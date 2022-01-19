---
title: "2.6 Masters"
weight: 26
sectionnumber: 2.6
---

In this lab we will learn how to replace a failed master node. As long as we do not lose the majority of our masters, we can simply replace those that failed. If we lose the majority (e.g. 2 out of 3), we need to restore the state of `etcd` from a previously created snapshot, which will not be covered in this lab.


## Task {{% param sectionnumber %}}.1: Replacing a master node

Since the masters are hosting `etcd`, the key-value store that holds all the cluster's information and state, it is not as straightforward as replacing a worker node backed by a machine set.


### Task {{% param sectionnumber %}}.1.1: Replacing the machine

First, create a copy of the master machine you want to replace:

```bash
oc -n openshift-machine-api get machine <failed-master-machine> -o yaml > machine_<failed-master-machine>.yaml
```

Edit the file to reflect the following changes:

* Remove:
  * `metadata.annotations`
  * `metadata.creationTimestamp`
  * `metadata.generations`
  * `metadata.uid`
  * `metadata.managedFields`
  * `metadata.resourceVersion`
  * `spec.providerID`
  * all of `status`
* Change `metadata.name` to a unique name, e.g. if the name was `<clustername>-<id>-master-0` use `<clsutername>-<id>-master00`.

The Machine resource should look similar to this:

{{< highlight yaml >}}{{< readfile file="content/en/docs/02/resources/machine_master-02.yaml" >}}{{< /highlight >}}

Delete the master machine you want to replace:

{{% alert title="Warning" color="secondary" %}}
Make sure you delete the master machine you created the copy from in the previous step.
{{% /alert %}}

```bash
oc -n openshift-machine-api delete machine <failed-master-machine>
```

Wait for the machine to be deleted:

```bash
oc -n openshift-machine-api get machines -o wide
```

Example output:

```
NAME                                                 PHASE      TYPE         REGION       ZONE          AGE    NODE                                          PROVIDERID                               STATE
user01-ops-training-75gjz-infra-eu-north-1a-9mzd9    Running    m5.2xlarge   eu-north-1   eu-north-1a   46h    ip-10-0-130-69.eu-north-1.compute.internal    aws:///eu-north-1a/i-0a7b134c9b0d07d4d   running
user01-ops-training-75gjz-infra-eu-north-1a-h88ln    Running    m5.2xlarge   eu-north-1   eu-north-1a   46h    ip-10-0-154-3.eu-north-1.compute.internal     aws:///eu-north-1a/i-0b98e80b4cd46d367   running
user01-ops-training-75gjz-infra-eu-north-1a-smgzs    Running    m5.2xlarge   eu-north-1   eu-north-1a   46h    ip-10-0-154-185.eu-north-1.compute.internal   aws:///eu-north-1a/i-0750413e1557c439f   running
user01-ops-training-75gjz-master-0                   Running    m5.xlarge    eu-north-1   eu-north-1a   3d3h   ip-10-0-155-28.eu-north-1.compute.internal    aws:///eu-north-1a/i-0736635506a9eb2cc   running
user01-ops-training-75gjz-master-1                   Running    m5.xlarge    eu-north-1   eu-north-1b   3d3h   ip-10-0-184-171.eu-north-1.compute.internal   aws:///eu-north-1b/i-067f7bac5768ab117   running
user01-ops-training-75gjz-master-2                   Deleting   m5.xlarge    eu-north-1   eu-north-1c   3d3h   ip-10-0-212-27.eu-north-1.compute.internal    aws:///eu-north-1c/i-02210ddbae879539e   shutting-down
user01-ops-training-75gjz-worker-eu-north-1a-vv59v   Running    m5.2xlarge   eu-north-1   eu-north-1a   3d3h   ip-10-0-139-81.eu-north-1.compute.internal    aws:///eu-north-1a/i-0d243001e9ba4cb9b   running
user01-ops-training-75gjz-worker-eu-north-1b-zddgj   Running    m5.2xlarge   eu-north-1   eu-north-1b   3d3h   ip-10-0-173-170.eu-north-1.compute.internal   aws:///eu-north-1b/i-028cec3fbf26c0f13   running
user01-ops-training-75gjz-worker-eu-north-1c-6mqls   Running    m5.2xlarge   eu-north-1   eu-north-1c   3d3h   ip-10-0-195-93.eu-north-1.compute.internal    aws:///eu-north-1c/i-0d7f8ed2c8b9c0932   running
```


### Task {{% param sectionnumber %}}.1.2: Deleting the `etcd` member

Now that the master node has been fully removed, we need to also remove the `etcd` member, since it still has the configuration of the old master node.

Connect to an `etcd` pod that is not running on the master you just removed:

```bash
oc -n openshift-etcd rsh -c etcdctl <etcd-pod>
```

View the member list and take note of the ID and name of the `etcd` member you want to remove:

{{% alert title="Note" color="primary" %}}
To identify the member you need to remove, compare the list with the existing pod names.
In the example, the list shows a member with the name `ip-10-0-212-27.eu-north-1.compute.internal` which does not correspond with any of the master node names.
{{% /alert %}}

```bash
etcdctl member list -w table
```

Example output:

```
+------------------+---------+---------------------------------------------+---------------------------+---------------------------+------------+
|        ID        | STATUS  |                    NAME                     |        PEER ADDRS         |       CLIENT ADDRS        | IS LEARNER |
+------------------+---------+---------------------------------------------+---------------------------+---------------------------+------------+
|  ce90578971689b5 | started |  ip-10-0-155-28.eu-north-1.compute.internal |  https://10.0.155.28:2380 |  https://10.0.155.28:2379 |      false |
| 2c53f4526118a2eb | started | ip-10-0-184-171.eu-north-1.compute.internal | https://10.0.184.171:2380 | https://10.0.184.171:2379 |      false |
| 43997d8a75197361 | started |  ip-10-0-212-27.eu-north-1.compute.internal |  https://10.0.212.27:2380 |  https://10.0.212.27:2379 |      false |
+------------------+---------+---------------------------------------------+---------------------------+---------------------------+------------+
```

Remove the old peer entry.

```bash
etcdctl member remove <id>
```

Verify the member was removed:

```bash
etcdctl member list -w table
```

This is it! You can now disconnect from the etcd pod at this point.


### Task {{% param sectionnumber %}}.1.3: Recreate the master node

The cluster is again in a properly running state, of course however with only two instead of three masters and etcd members.

Recreate the master machine using the file definition you created before:

```bash
oc -n openshift-machine-api apply -f machine_<failed-master-machine>.yaml
```

Verify the machine was created successfully:

```bash
oc -n openshift-machine-api get machines -o wide
```

The `etcd` operator should automatically scale up a new member and add it to the cluster.

Verify that three `etcd` pods are up and running:

```etcd
oc -n openshift-etcd get pods -l etcd
```

{{% alert title="Note" color="primary" %}}
If only two pods are running, you can force a redeployment:

```bash
oc patch etcd cluster \
  -p='{"spec": {"forceRedeploymentReason": "recovery-'"$( date --rfc-3339=ns )"'"}}' \
  --type=merge
```

{{% /alert %}}

All that is left to do is clean up the old secrets that are not used anymore.

List the secrets of the old peer that can be safely deleted:

```bash
oc -n openshift-etcd get secrets | grep <removed-peer-name>
```

Example output:

```
etcd-peer-ip-10-0-212-27.eu-north-1.compute.internal               kubernetes.io/tls                     2      3d4h
etcd-serving-ip-10-0-212-27.eu-north-1.compute.internal            kubernetes.io/tls                     2      3d4h
etcd-serving-metrics-ip-10-0-212-27.eu-north-1.compute.internal    kubernetes.io/tls                     2      3d4h
```

Delete those secrets:

```bash
oc -n openshift-etcd delete secrets \
  etcd-peer-<removed-peer-name> \
  etcd-serving-<removed-peer-name> \
  etcd-serving-metrics-<removed-peer-name>
```
