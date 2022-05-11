---
title: "4.2 Evictions"
weight: 42
sectionnumber: 4.2
---

This second troubleshooting lab will go into detail on what evictions are, how to analyze and how to avoid them.


## Task {{% param sectionnumber %}}.1: Setup

First, let's get set up.
Create a new namespace using the following command:

{{% alert title="Warning" color="secondary" %}}
It is essential that you use this command or else this lab won't work!
{{% /alert %}}

```bash
oc create namespace <namespace>
```

Now, deploy a test pod into the freshly created namespace using the provided definition:

```bash
oc apply -f https://raw.githubusercontent.com/acend/openshift-4-ops-training/main/content/en/docs/04/resources/deployment_stress2much.yaml --namespace <namespace>
```

Watch the deployment and wait for a new event after the pod successfully ran for some time (give it 1 to 2 minutes).

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

```bash
oc get pods --watch --namespace <namespace>
```

{{% alert title="Note" color="primary" %}}
As always with `--watch`/`-w`, terminate the command using ctrl+c.
{{% /alert %}}

{{% /details %}}

What you should see is that the pod gets evicted and recreated periodically.


## Task {{% param sectionnumber %}}.2: Analysis

So what happened?
Why does the pod periodically stop running and show the status `Evicted`?

Try to find an answer to these questions.

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

There are at least two ways to find that out:
Describing an evicted pod shows its events where you'll find the reason:

```bash
oc describe pod <evicted pod> --namespace <namespace>
```

```
...

Events:
  Type     Reason          Age   From               Message
  ----     ------          ----  ----               -------
  Normal   Scheduled       27s   default-scheduler  Successfully assigned eviction-lab/stress2much-945db9-74z7m to host-10-110-0-110
  Normal   AddedInterface  24s   multus             Add eth0 [10.125.2.107/23]
  Normal   Pulling         24s   kubelet            Pulling image "quay.io/acend/stress"
  Normal   Pulled          22s   kubelet            Successfully pulled image "quay.io/acend/stress" in 1.670548529s
  Normal   Created         22s   kubelet            Created container stress
  Normal   Started         22s   kubelet            Started container stress
  Warning  Evicted         7s    kubelet            The node was low on resource: memory. Container stress was using 5093448Ki, which exceeds its request of 0.
  Normal   Killing         7s    kubelet            Stopping container stress
```

Depending on the pods running on the cluster, it may be the case that you already had to scale down or delete the culprit(s).
This would mean you don't have any pods to describe.
However, looking at all the collected events from the namespace's resources, we can still find out what happened:

```bash
oc get events --namespace <namespace>
```

```
...
34s         Normal    Scheduled             pod/stress2much-945db9-74z7m        Successfully assigned eviction-lab/stress2much-945db9-74z7m to host-10-110-0-110
32s         Normal    AddedInterface        pod/stress2much-945db9-74z7m        Add eth0 [10.125.2.107/23]
32s         Normal    Pulling               pod/stress2much-945db9-74z7m        Pulling image "quay.io/acend/stress"
30s         Normal    Pulled                pod/stress2much-945db9-74z7m        Successfully pulled image "quay.io/acend/stress" in 1.670548529s
30s         Normal    Created               pod/stress2much-945db9-74z7m        Created container stress
30s         Normal    Started               pod/stress2much-945db9-74z7m        Started container stress
15s         Warning   Evicted               pod/stress2much-945db9-74z7m        The node was low on resource: memory. Container stress was using 5093448Ki, which exceeds its request of 0.
15s         Normal    Killing               pod/stress2much-945db9-74z7m        Stopping container stress
...
```

{{% /details %}}


## Task {{% param sectionnumber %}}.3: Cleanup

The pod's eviction and recreation would repeat endlessly, so let's scale down the deployment.

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

```bash
oc scale deployment/stress2much --replicas 0 --namespace <namespace>
```

{{% /details %}}

Notice how many pods were created and evicted in the meantime:

```bash
...
stress2much-945db9-pt2kr   0/1     Evicted   0          7m33s
stress2much-945db9-qf5gc   0/1     Evicted   0          7m31s
stress2much-945db9-qq52c   0/1     Evicted   0          7m28s
stress2much-945db9-rgtsp   0/1     Evicted   0          7m35s
stress2much-945db9-rq4mh   0/1     Evicted   0          7m30s
stress2much-945db9-rwtwl   0/1     Evicted   0          7m29s
stress2much-945db9-rz7kg   0/1     Evicted   0          7m32s
stress2much-945db9-s8gxl   0/1     Evicted   0          7m34s
stress2much-945db9-sgvqm   0/1     Evicted   0          7m33s
stress2much-945db9-smq8f   0/1     Evicted   0          7m31s
stress2much-945db9-tg4q2   0/1     Evicted   0          7m30s
stress2much-945db9-tm2q5   0/1     Evicted   0          7m29s
stress2much-945db9-tvjfs   0/1     Evicted   0          7m29s
stress2much-945db9-v9grm   0/1     Evicted   0          7m30s
stress2much-945db9-vjb4v   0/1     Evicted   0          7m59s
stress2much-945db9-vk2qk   0/1     Evicted   0          7m31s
stress2much-945db9-vn86d   0/1     Evicted   0          7m32s
stress2much-945db9-wkpkx   0/1     Evicted   0          7m31s
stress2much-945db9-wpf6t   0/1     Evicted   0          7m33s
stress2much-945db9-xq5t2   0/1     Evicted   0          7m34s
stress2much-945db9-z6ddv   0/1     Evicted   0          7m30s
stress2much-945db9-z8j6j   0/1     Evicted   0          7m33s
stress2much-945db9-z8v7c   0/1     Evicted   0          7m32s
stress2much-945db9-zk5pn   0/1     Evicted   0          7m28s
stress2much-945db9-zvz7p   0/1     Evicted   0          7m34s
stress2much-945db9-zwmzk   0/1     Evicted   0          7m34s
...
```

In order to delete all the evicted pods at once, execute the following command:

```bash
oc delete pods --field-selector=status.phase=Failed --namespace <namespace>
```


## Reasons

So we now know that the pod got evicted because it simply used too much memory.
In contrast to an out-of-memory event caused by overstepping the defined memory limit on the container or pod though, this event happened because the worker node itself had no memory left.

It's easy to see why the pod used too much memory:

{{< highlight yaml >}}{{< readfile file="content/en/docs/04/resources/deployment_stress2much.yaml" >}}{{< /highlight >}}

Have a good look at the last 3 lines:
The deployment created at the beginning of this lab deployed a pod using an image containing the stress-testing tool `stress`.
It also instructed the pod to execute `stress` with parameter `--vm-bytes 8G` which means the process tries to allocate 8GB of memory when started.


### Limitrange

But why could the pod even allocate this much memory and was not killed before?
Let's look at the LimitRange resource we defined earlier in the default project template:

```bash
oc get limitranges --namespace <namespace>
```

Now you know why it was crucial to create the namespace using `oc create namespace` at the beginning of this lab.
This command only creates the namespace but does not respect the default project template. That is why the LimitRange object was never created.

{{% alert title="Note" color="primary" %}}
Notice how OpenShift automatically creates a corresponding Project resource when only a namespace is created.
{{% /alert %}}


### Kubelet arguments

Now it's clear why the pod used too much memory and why it wasn't stopped from doing so earlier.
The last remaining question is, why was it evicted?

Remember the kubelet arguments you configured after installing the cluster in [lab 1.2](../../01/02_configuration/)?
We never looked at them in greater detail, so let's do that now:

{{< highlight yaml >}}{{< readfile file="content/en/docs/01/resources/kubeletconfig_worker.yaml" >}}{{< /highlight >}}

There's only three arguments defined, but those three were what saved our worker nodes from completely crashing.

* `kubeReserved` and `systemReserved` do exactly what their names suggest: They reserve a certain amount of resources for the kubelet and the operating system, respectively. This means that pod workload cannot, e.g., use more than the node's total memory less the reserved values.
* `evictionHard` defines a threshold which, when exceeded, triggers the eviction of pods. Pods are evicted based on their quality of service class which we already looked at during the Kubernetes/OpenShift Basics training.

There's also the `evictionSoft` argument which only starts to evict pods if node resources were over a certain threshold for a defined period of time.
However, the `evictionHard` argument should always be present as a last resort in case too many resources are consumed too fast.


## Conclusion

A properly configured cluster is well-defended against any kinds of resource overuse.
If however you run into any sort of evictions, you now know where to optimize parameters.

Additionally, proper capacity management is always part of operating an OpenShift cluster.
So make sure to set one up and add new nodes when needed, or even configure node autoscaling if possible.
