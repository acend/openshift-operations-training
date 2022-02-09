---
title: "2.1 Infrastructure nodes"
weight: 21
sectionnumber: 2.1
---

Not exactly a day 2-operations task, but one that immensily simplifies operations.
The concept of infrastructure nodes was more prominently advertised by Red Hat when OpenShift 3 was the latest and greatest.
This evidently changed somehow for OpenShift 4, but with a bit of legwork we're able to create those, too.

Infra nodes are dedicated worker nodes intended for infrastructure components such as the image registry, routers or the whole monitoring stack.
Providing dedicated nodes helps ensuring that there's always enough resources for infrastructure components.


## Task {{% param sectionnumber %}}.1: Create a machine set

The best way to add infra nodes to the cluster is via machine sets.
Machine sets are usually only supported on clusters installed with the IPI installation method.
They could be compared to deployments: If deployments are responsible for creating replica sets and ultimately pods, machine sets create machines to then add them as nodes to the cluster.

The probably easiest way to create a machine set for infra nodes is to copy and adapt the existing `worker` MachineSet in the `openshift-machine-api` namespace.

First of all, get the yaml definition of the existing worker MachineSet resource inside the `openshift-machine-api` namespace.

{{% details title="Hints" mode-switcher="normalexpertmode" %}}
Check the name of the existing worker MachineSet resource inside the `openshift-machine-api` namespace:

```bash
oc get machinesets -n openshift-machine-api
```

Using this information, write the existing worker MachineSet resource into a file:

```bash
oc get machinesets -n openshift-machine-api -o yaml <machineset name> > machineset_infra.yaml
```

{{% /details %}}

Now, adapt the file so its content describes an `infra` MachineSet.

{{% alert title="Warning" color="secondary" %}}
By far not all occurrences of `worker` must be changed into `infra`.

Refer to the [documentation](https://docs.openshift.com/container-platform/latest/machine_management/creating-infrastructure-machinesets.html#machineset-yaml-aws_creating-infrastructure-machinesets) if you are not sure what to replace.
{{% /alert %}}

{{% details title="Hints" mode-switcher="normalexpertmode" %}}
Edit the file using your favorite editor (still `vim` of course):

* Remove all the unnecessary clutter such as the `.metadata.managedFields` part or `creationTimestamp` fields
* Replace all occurrences of `worker` with `infra` **except for everything under `providerSpec`**
* Ensure that `replicas` is set to 3
* Add the infra role label to the `.spec.template.spec.metadata.labels` field like so:

```yaml
    spec:
      metadata:
        creationTimestamp: null
        labels:
          node-role.kubernetes.io/infra: ""
      providerSpec:
```

When you're finished, your MachineSet resource file should look similar to this:

{{< highlight yaml >}}{{< readfile file="content/en/docs/02/resources/machineset_infra.yaml" >}}{{< /highlight >}}
{{% /details %}}

{{% alert title="Warning" color="secondary" %}}
You cannot use the sample MachineSet resource definition above without further modification.
It's recommended you use your own worker MachineSet definition and make the described changes.
{{% /alert %}}

Now, create the MachineSet.

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

```bash
oc create -f machineset_infra.yaml
```

{{% /details %}}

Check if the machines were created and that they're in the `Provisioning` state:

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

```bash
oc get machines -n openshift-machine-api
```

{{% /details %}}

It shouldn't take too long and you will see new infra nodes popping up.
