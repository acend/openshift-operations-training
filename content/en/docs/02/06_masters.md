---
title: "2.6 Control plane nodes"
weight: 26
sectionnumber: 2.6
---

{{% alert title="Note" color="secondary" %}}
While many occurrences of "master" have been replaced throughout OpenShift and Kubernetes, this is unfortunately not yet the case for `Machine` resources. This is why you are going to encounter a mixture of "control plane" and "master" terms in this lab.
{{% /alert %}}

During this lab, we will learn how to replace a failed control plane node. As long as we do not lose the majority of our control plane nodes, we can simply replace those that failed. If we lose the majority (e.g., 2 out of 3), we need to restore the state of `etcd` from a previously created snapshot, which will not be covered in this lab.

Since the control plane nodes are hosting `etcd`, the key-value store that holds all the cluster's information and state, it is not as straightforward as replacing a worker node backed by a machine set.

{{% alert title="Note" color="info" %}}
Above statement is not completely correct anymore. [OpenShift 2.12 introduced the control plane machine sets for specific infrastructure providers](https://docs.openshift.com/container-platform/latest/machine_management/control_plane_machine_management/cpmso-getting-started.html#cpmso-platform-matrix_cpmso-getting-started). Because the control plane machine set on AWS is supported and active by default, we are going to delete it in the first step of this lab.\
This enables you to do this lab and experience the replacement of a control plane node anyway, in case you need to do it on infrastructure that is not supported by or has no active control plane machine set.
{{% /alert %}}


## Task {{% param sectionnumber %}}.1: Replacing the machine

As mentioned in above note, the first task is to delete the control plane machine set. The control plane machine set will immediately be recreated, however, it will not be active anymore.

```bash
oc -n openshift-machine-api delete controlplanemachinesets.machine.openshift.io cluster
```

Now, save the resource definition of the master machine you want to replace.

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

```bash
oc -n openshift-machine-api get machine <failed-master-machine> -o yaml > machine_<failed-master-machine>.yaml
```

{{% /details %}}

Edit the file to reflect the following changes:

* Remove:
  * `metadata.annotations`
  * `metadata.creationTimestamp`
  * `metadata.generations`
  * `metadata.uid`
  * `metadata.resourceVersion`
  * `spec.providerID`
  * all of `status`

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

The `machine` resource should look similar to this:

{{< readfile file="/content/en/docs/02/resources/machine_master-02.yaml" code="true" lang="yaml" >}}

{{% /details %}}

Delete the master machine you want to replace.

{{% alert title="Warning" color="secondary" %}}
Make sure you delete the master machine you just saved the resource definition from in the previous step.
{{% /alert %}}

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

```bash
oc -n openshift-machine-api delete machine <failed-master-machine>
```

{{% /details %}}

This will queue the machine for deletion. You can verify this by checking the machines' state.

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

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
user01-ops-training-75gjz-master-2                   Deleting   m5.xlarge    eu-north-1   eu-north-1c   3d3h   ip-10-0-212-27.eu-north-1.compute.internal    aws:///eu-north-1c/i-02210ddbae879539e   running
user01-ops-training-75gjz-worker-eu-north-1a-vv59v   Running    m5.2xlarge   eu-north-1   eu-north-1a   3d3h   ip-10-0-139-81.eu-north-1.compute.internal    aws:///eu-north-1a/i-0d243001e9ba4cb9b   running
user01-ops-training-75gjz-worker-eu-north-1b-zddgj   Running    m5.2xlarge   eu-north-1   eu-north-1b   3d3h   ip-10-0-173-170.eu-north-1.compute.internal   aws:///eu-north-1b/i-028cec3fbf26c0f13   running
user01-ops-training-75gjz-worker-eu-north-1c-6mqls   Running    m5.2xlarge   eu-north-1   eu-north-1c   3d3h   ip-10-0-195-93.eu-north-1.compute.internal    aws:///eu-north-1c/i-0d7f8ed2c8b9c0932   running
```

{{% /details %}}

This machine will not be deleted right away. You can see in the resource definition that its lifecycle contains a pre-drain-hook, executed before draining the machine. This step must run before the draining of the machine, before the deletion itself:

```
spec:
  lifecycleHooks:
    preDrain:
    - name: EtcdQuorumOperator
      owner: clusteroperator/etcd
```

The hook ensures that the collection of etcd nodes (the "quorum") remains in a healthy state. This is no longer granted if we remove one node out of the three expected ones. If we manually decrease the number of required nodes in the healthy state, then the third node can be deleted.


## Task {{% param sectionnumber %}}.2: Deleting the `etcd` member

For the master machine to be fully removed, we need to also remove the `etcd` member, since it still has the configuration of the old control plane node.

Connect to an `etcd` pod that is not running on the control plane node you just removed:

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

```bash
oc -n openshift-etcd rsh <etcd-pod>
```

{{% /details %}}

View the member list and take note of the ID and name of the `etcd` member you want to remove.

{{% alert title="Note" color="primary" %}}
To identify the member you need to remove, compare the list with the existing pod names.
In the example, the list shows a member with the name `ip-10-0-212-27.eu-north-1.compute.internal` which does not correspond with any of the control plane node names.
{{% /alert %}}

All relevant information can be found in [OpenShift's documentation](https://docs.openshift.com/container-platform/latest/backup_and_restore/control_plane_backup_and_restore/replacing-unhealthy-etcd-member.html#restore-replace-stopped-etcd-member_replacing-unhealthy-etcd-member).

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

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

{{% /details %}}

Remove the old peer entry.

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

```bash
etcdctl member remove <id>
```

{{% /details %}}

Verify the member was removed.

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

```bash
etcdctl member list -w table
```

{{% /details %}}

This is it! You can now disconnect from the etcd pod.

The machine we marked for deletion can now finally proceed. We can check the machines' state again.

{{% alert title="Note" color="secondary" %}}
The machine deletion might take several minutes to complete.
{{% /alert %}}

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

```bash
oc -n openshift-machine-api get machines -o wide
```

{{% /details %}}


## Task {{% param sectionnumber %}}.3: Recreate the control plane node

The cluster is again in a properly running state, of course however with only two instead of three control plane nodes and etcd members.

Recreate the master machine using the file definition you created before.

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

```bash
oc -n openshift-machine-api apply -f machine_<failed-master-machine>.yaml
```

{{% /details %}}

Verify the machine was created successfully:

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

```bash
oc -n openshift-machine-api get machines -o wide
```

{{% /details %}}

The `etcd` operator should automatically scale up a new member and add it to the cluster.

Verify that three `etcd` pods are up and running.

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

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
{{% /details %}}

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

Delete those secrets.

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

```bash
oc -n openshift-etcd delete secrets \
  etcd-peer-<removed-peer-name> \
  etcd-serving-<removed-peer-name> \
  etcd-serving-metrics-<removed-peer-name>
```

{{% /details %}}
