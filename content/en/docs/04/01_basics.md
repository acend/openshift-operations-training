---
title: "4.1 Troubleshooting basics"
weight: 41
sectionnumber: 4.1
---

In chapter 3 you got to know the monitoring components, its rules and more.
Now, imagine you get an alert.
"Where to begin?", you ask.
How do you find out what's amiss and led to the alert?

That's what we are going to look at in this first lab.


## Task {{% param sectionnumber %}}.1: Cluster operators

As you already know, OpenShift 4 usually consists of about 30 different operators.
Each operator is responsible that the actual state of the cluster matches the desired, configured state.
If this is not the case, the operator tells us.

Check the cluster operators' state:

```bash
oc get clusteroperators
```

{{% alert title="Note" color="primary" %}}
For most of the Kubernetes resource types, there's a short version.
In the case of the `clusteroperators` resources, it's `co`, so you could also check the state with `oc get co`.
{{% /alert %}}

This gives you something along the lines of:

```
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE
authentication                             4.7.6     True        False         False      14h
baremetal                                  4.7.6     True        False         False      10d
cloud-credential                           4.7.6     True        False         False      10d
cluster-autoscaler                         4.7.6     True        False         False      10d
config-operator                            4.7.6     True        False         False      10d
console                                    4.7.6     True        False         False      6d23h
csi-snapshot-controller                    4.7.6     True        False         False      6d23h
dns                                        4.7.6     True        False         False      6d23h
etcd                                       4.7.6     True        False         False      10d
image-registry                             4.7.6     True        False         False      6d23h
ingress                                    4.7.6     True        False         False      10d
insights                                   4.7.6     True        False         False      10d
kube-apiserver                             4.7.6     True        False         False      10d
kube-controller-manager                    4.7.6     True        False         False      10d
kube-scheduler                             4.7.6     True        False         False      10d
kube-storage-version-migrator              4.7.6     True        False         False      6d19h
machine-api                                4.7.6     True        False         False      10d
machine-approver                           4.7.6     True        False         False      10d
machine-config                             4.7.6     True        False         False      6d23h
marketplace                                4.7.6     True        False         False      6d23h
monitoring                                 4.7.6     True        False         False      6d20h
network                                    4.7.6     True        False         False      10d
node-tuning                                4.7.6     True        False         False      7d
openshift-apiserver                        4.7.6     True        False         False      6d20h
openshift-controller-manager               4.7.6     True        False         False      10d
openshift-samples                          4.7.6     True        False         False      7d
operator-lifecycle-manager                 4.7.6     True        False         False      10d
operator-lifecycle-manager-catalog         4.7.6     True        False         False      10d
operator-lifecycle-manager-packageserver   4.7.6     True        False         False      6d20h
service-ca                                 4.7.6     True        False         False      10d
storage                                    4.7.6     True        False         False      6d23h
```

The important part to see here is that all of the operators are available, that they are not progressing and that they are not degraded.

If you make a change to one of the operator's configuration, the operator usually changes its progressing state to true as long as the configuration change has not finished.

Degraded means that the operator cannot function properly because there is, e.g., a configuration error.


## Task {{% param sectionnumber %}}.2: Configuration error

In order to test this behaviour, we are going to introduce a configuration change.
Specifically, we want to add an authentication provider to our cluster:

{{< highlight yaml >}}{{< readfile file="content/en/docs/04/resources/oauth_cluster.yaml" >}}{{< /highlight >}}

Apply the authentication provider configuration to the cluster:

```bash
oc apply -f https://raw.githubusercontent.com/acend/openshift-4-ops-training/main/content/en/docs/04/resources/oauth_cluster.yaml
```

It might take a while, but after a moment, the authentication operator will change its "degraded" state to true.
Obviously something seems to be wrong with our introduced configuration; it'd be quite the coincidence something else happened at the same time.
But we need to know what exactly, so let's try to find out:

```bash
oc describe clusteroperator authentication
```

The output will be rather long.
Look for for the `Status` part to see the relevant message:

```
Status:
  Conditions:
    Last Transition Time:  2021-04-23T09:39:40Z
    Message:               OAuthServerConfigObservationDegraded: error validating secret openshift-config/github-secret: secret "github-secret" not found
    Reason:                OAuthServerConfigObservation_Error
    Status:                True
    Type:                  Degraded
    Last Transition Time:  2021-04-22T18:54:38Z
    Message:               All is well
    Reason:                AsExpected
    Status:                False
    Type:                  Progressing
    Last Transition Time:  2021-04-22T18:54:50Z
    Message:               OAuthServerDeploymentAvailable: availableReplicas==2
    Reason:                AsExpected
    Status:                True
    Type:                  Available
```

You can see that the operator's status first changed from "Available" (`Type` field) to "Progressing" and then to "Degraded".
This means that it tried to apply our configuration change but then failed to do so.

Looking at the topmost "Message" reveals the problem:
The configuration defines a secret in `clientSecret` which contains the GitHub client secret.
This secret doesn't exist (on purpose).

Let's revert our change so the authentication operator goes back to a functioning state:

```bash
oc patch oauth cluster --type merge -p '{"spec": {"identityProviders": []}}'
```


## Task {{% param sectionnumber %}}.3: Node debugging

OpenShift 4 introduces a new capability to debug cluster nodes.
Using `oc debug node/<node name>`, a debug pod is started on the specified node and you get a terminal session in that debug pod.

Let's try this out.
Pick a node of your choosing:

```bash
oc get nodes
```

Then, using the node's name, start the debug session:

```bash
oc debug node/<node name>
```

{{% alert title="Warning" color="secondary" %}}
Note the `/` between the resource type (`node`) and the name.
The command will error out if you don't write it.
{{% /alert %}}

After a quick moment, you are presented with a root shell on the desired node.
However, you're not yet really on the node, you're limited to what the container sees of it.
You need to chroot onto the node first before you'll be able to, e.g., use host binaries:

```bash
chroot /host
```

If you're not sure if you're on the node, there are different possibilities to find out, one of which is looking at the release file:

```bash
cat /etc/redhat-release
```

If the output says `Red Hat Enterprise Linux release [...]`, you're still contained in the container.
As we know, OpenShift nodes use CoreOS as operating system, so the output should read `Red Hat Enterprise Linux CoreOS release [...]`.

We can now do all the things as if we had connected via ssh.
Which, of course, is still possible but might be less convenient.

One of these things you might want to do is have a look at the running containers on that node.
We do this using `crictl`:

```bash
crictl ps
```

As you probably know from the good old days when you still used Docker on your machines, `ps` lists all running containers.
You can also list all pods:

```bash
crictl pods
```

To exit the debug pod, either simply type `exit` or press ctrl+d until you're back on your own shell.

For more information on `crictl`, check out [this debugging how-to](https://kubernetes.io/docs/tasks/debug-application-cluster/crictl/) or [its documentation](https://github.com/kubernetes-sigs/cri-tools/blob/master/docs/crictl.md).


## Task {{% param sectionnumber %}}.4: Node logs

You might be tempted now to use the `oc debug node` feature to use it for all kinds of debugging tasks.
However, there is one very useful `oc adm` subcommand that comes in especially handy when you want to look at a node's logs.

`oc adm node-logs` allows you to retrieve node logs.
By default, the command queries the systemd journal.
Using `--path`, you can override this behaviour and use it to show logs within the node's `/var/logs` directory.

Let's use this to query all masters' (`--role master`) kubelet (`--unit`) logs:

```bash
oc adm node-logs --role master --unit kubelet --tail 10
```

{{% alert title="Note" color="primary" %}}
As with the `journalctl` command you can use `--tail`, `--since` and `--until` to limit the amount of logs or to focus on a certain period of time.
{{% /alert %}}

Of course above command can be used for all the other systemd units that are running on a node.

As already mentioned, the `--path` parameter can be used to show logs in the `/var/logs` directory.
The following command lists all available log directories:

```bash
oc adm node-logs --role master --path /
```

This way, we can find a specific log file and finally show its content, e.g. the audit log's:

```bash
oc adm node-logs --role master --path /audit/audit.log
```

{{% alert title="Note" color="primary" %}}
You might have to interrupt the `node-logs` command using `--path` with ctrl+c depending on the number of logs.
The `--tail`, `--since` and `--until` parameters cannot be used to with `--path`.
{{% /alert %}}


## SSH

You might be wondering why all these new subcommands were introduced with OpenShift 4.
Why not simply ssh into a node you want to analyze?

The reason is the introduction of CoreOS as the default underlying operating system of OpenShift 4.
CoreOS changes the way we interact with an operating system because it follows the core principles of containers:

* immutability
* statelessness

This means that it is discouraged to ssh into a node because this might introduce manual, undocumented changes to the operating system that could lead to unexpected behaviour.
Always apply configuration changes via `MachineConfig` objects.

However, there are cases where ssh represents the only viable means of analyzing a malfunctioning node.
Imagine the OpenShift API is down or the node's kubelet is not responding.
`oc` will not work in these cases.
So it still is a good idea to configure ssh keys in those installation configuration files.


## Troubleshooting references

The OpenShift documentation has multiple troubleshooting documentation pages such as [this one](https://docs.openshift.com/container-platform/latest/support/troubleshooting/troubleshooting-installations.html) which are worth checking out.
