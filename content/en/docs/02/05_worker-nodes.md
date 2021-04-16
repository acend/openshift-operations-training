---
title: "2.5 Worker nodes"
weight: 25
sectionnumber: 2.5
---

We initially installed this cluster using only worker nodes.
We then successfully created infra nodes and moved all infrastructure components onto them.
Now, essentially the only real workload that's running on the 3 worker nodes is our uptime application.

Without going into detail yet on how and where you can find dashboards and metrics for capacity monitoring, it's safe to say that one small Python app doesn't need 3 worker nodes with 8 CPU cores and 32 GB of memory.
So what we are going to do next is reduce the number of worker nodes to 2 and reduce their size.

We admit that in real life, this would probably be the other way round.
Instead of reducing the number of nodes and their capacity, we would add more nodes and maybe even make them bigger.
However, be it reducing or enlarging the cluster, the process is the same and the forecast for this training tells us that we're not going to see much more workload.


## Task {{% param sectionnumber %}}.1: Change the number of nodes

Changing the number of nodes can be either done via CLI or in the web console.


### Change the number of nodes in the web console

* Make sure you're in the Administrator view (see the dropdown menu top left)
* In the menu on the left, choose Compute and then MachineSets
* Click the 3-dot button on the right next to the worker machineset:

![machineset](machineset.png)

* Choose to edit Machine count
* Reduce the number of machines to 2 and click Save

If you now switch to the Nodes or Machines overview in the menu on the left, you can see that OpenShift already began the deprovisioning process.


### Change the number of nodes on the command line

Scaling OpenShift nodes via CLI works the exact same way as scaling pods.

We first want to check which MachineSet resource we want to scale:

```bash
oc get machinesets -n openshift-machine-api
```

Now pick the worker machineset and use it in the following command to scale it:

```bash
oc scale machinesets --replicas=2 -n openshift-machine-api <machineset name>
```

Yes, it's that easy.


## Task {{% param sectionnumber %}}.2: Change the node size

The way nodes are resized depends on the installation method.
If the cluster was installed using UPI, the underlying infrastructure solution is responsible for sizing the machines.
Thus, if you want to resize a node in a UPI installation, you need to do it yourself, OpenShift doesn't directly manage these machines.

In an IPI installation, the infrastructure is managed by OpenShift, too.
This is done using machine sets which we just used to resize the number of worker nodes.

In order to find out what exactly we have to change, let's have a closer look at the MachineSet resource.
Here's an excerpt of the more relevant fields:

```
apiVersion: machine.openshift.io/v1beta1
kind: MachineSet
metadata:
  annotations:
    machine.openshift.io/GPU: "0"
    machine.openshift.io/memoryMb: "32768"
    machine.openshift.io/vCPU: "8"
  name: user01-ops-training-75gjz-worker-eu-north-1a
  namespace: openshift-machine-api
spec:
  replicas: 3
  template:
    metadata:
      labels:
        machine.openshift.io/cluster-api-cluster: user01-ops-training-75gjz
        machine.openshift.io/cluster-api-machine-role: worker
        machine.openshift.io/cluster-api-machine-type: worker
        machine.openshift.io/cluster-api-machineset: user01-ops-training-75gjz-worker-eu-north-1a
    spec:
      providerSpec:
        value:
          ami:
            id: ami-0080eb90a48d9655e
          iamInstanceProfile:
            id: user01-ops-training-75gjz-worker-profile
          instanceType: m5.2xlarge
          kind: AWSMachineProviderConfig
          placement:
            availabilityZone: eu-north-1a
            region: eu-north-1
```

We can see that the annotations contain the information of how many resources machines from this set have.
But changing these values won't change anything.
The field we are interested in is the `instanceType` field.
It contains the [AWS instance flavor](https://aws.amazon.com/ec2/instance-types/).

Change this field to `m5.large` (2 vCPUs, 8 GiB Memory).

After the change, nothing happens.
OpenShift only applies this change to new nodes.
We'll have to delete every machine with the old instance type one by one so they get replaced by new ones.

Delete one of the worker machines and wait until it is up again.
Then, repeat the same step for the other machine.

{{% alert title="Note" color="primary" %}}
It might make perfect sense to first scale up the nodes before you begin the renewal process.
This helps to ensure high availability on the cluster.
{{% /alert %}}
