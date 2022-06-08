---
title: "2.2 Infrastructure components"
weight: 22
sectionnumber: 2.2
---

Now that we have fresh infra nodes up and running, let's see to it that they have something to do.
The usual suspects to run on infrastructure nodes are:

* routers
* registry
* most of the monitoring components
* most of the logging components

Red Hat provides a detailed [list of what components don't incur OpenShift Container Platform worker subscriptions](https://docs.openshift.com/container-platform/latest/machine_management/creating-infrastructure-machinesets.html#infrastructure-components_creating-infrastructure-machinesets).
So if you have any other components listed there that are not going to be looked at in this training, it could make perfect sense to move these components as well.

"Moving" components from one node to another is usually done with either using node selectors and labels or using taints and tolerations.
Using node selectors and labels exists since OpenShift 3 and works the same way as the use of selector in services to define which pods they should cover.
A possibility would therefore be to simply configure all infrastructure components with a node selector that points to the infra nodes' role label (`node-role.kubernetes.io/infra=''`).

There might however be use cases where node selectors reach their limit of flexibility.
Imagine you had different types of worker nodes and labelled them accordingly and additionally to the worker role label (`node-role.kubernetes.io/infra=''`).
What if you wanted to prevent all but a specific subset of pods to be deployed to a certain node?
We can limit said pods to this one node, but we can't prevent other pods to be deployed on it as well.

This is where taints and tolerations come into play.
When node selectors and labels represent an allowlist of nodes, taints and tolerations can be used to disallow pods from being deployed to specific nodes.
This is the exact situation we are in right now.
Our new infrastructure nodes have the infra and the worker label, but we want to make sure that only infra components can be deployed on them.
Which means we are going to use a combination of the two variants:

* Use node selectors to move the infra components onto the infra nodes
* Use taints to prevent other pods from being deployed onto the infra nodes


## Task {{% param sectionnumber %}}.1: Taints

The first step we need to take is taint the new infra nodes.
Nodes that have taints on them effectively tell the scheduler to only deploy pods onto them if they comply with the corresponding toleration.

A taint consists of a key, value, and effect and is expressed as `key=value:effect`; the value is optional.
As an effect we can choose from `NoSchedule`, `PreferNoSchedule` or `NoExecute`.

First of all, taint your infra nodes with key `node-role.kubernetes.io/infra` and effect `NoSchedule`.

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

```bash
oc adm taint nodes --selector node-role.kubernetes.io/infra node-role.kubernetes.io/infra:NoSchedule
```

{{% /details %}}

This action's effect is that new pods are not scheduled onto infra nodes if they do not match the taint.
Just what we wanted.

{{% alert title="Note" color="primary" %}}
Instead of manually tainting the infra nodes, the taints could be added to the infra `MachineSet` object.
The [OpenShift documentation](https://docs.openshift.com/container-platform/latest/nodes/scheduling/nodes-scheduler-taints-tolerations.html#nodes-scheduler-taints-tolerations-adding-machineset_nodes-scheduler-taints-tolerations) explains how to do that.
This would be the more elegant solution because it automatically taints infra nodes when new ones are added or existing ones replaced.
{{% /alert %}}

Before we continue with moving all the different infra components onto the infra nodes, if you want to know more about taints and tolerations, have a look at [OpenShift's documentation](https://docs.openshift.com/container-platform/latest/nodes/scheduling/nodes-scheduler-taints-tolerations.html).


## Task {{% param sectionnumber %}}.2: Router

The [OpenShift documentation](https://docs.openshift.com/container-platform/latest/machine_management/creating-infrastructure-machinesets.html#infrastructure-moving-router_creating-infrastructure-machinesets) explains how to move the router and other infra pods.
However, it uses node selectors and labels to do so.

In order to find out how to define a toleration for the Ingress Controller, a possible way is to have a look at its CustomResourceDefinition:

```bash
oc get crd ingresscontrollers.operator.openshift.io -o yaml
```

In there we find:

```
tolerations:
  description: "tolerations is a list of tolerations applied to ingress controller deployments. \n The default is an empty list. \n See https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/"
  items:
    description: The pod this Toleration is attached to tolerates any taint that matches the triple <key,value,effect> using the matching operator <operator>.
    properties:
      effect:
        description: Effect indicates the taint effect to match. Empty means match all taint effects. When specified, allowed values are NoSchedule, PreferNoSchedule and NoExecute.
        type: string
      key:
        description: Key is the taint key that the toleration applies to. Empty means match all taint keys. If the key is empty, operator must be Exists; this combination means to match all values and all keys.
        type: string
      operator:
        description: Operator represents a key's relationship to the value. Valid operators are Exists and Equal. Defaults to Equal. Exists is equivalent to wildcard for value, so that a pod can tolerate all taints of a particular category.
        type: string
      tolerationSeconds:
        description: TolerationSeconds represents the period of time the toleration (which must be of effect NoExecute, otherwise this field is ignored) tolerates the taint. By default, it is not set, which means tolerate the taint forever (do not evict). Zero and negative values will be treated as 0 (evict immediately) by the system.
        format: int64
        type: integer
```

This means the toleration definition will have to look like this:

```yaml
spec:
  nodePlacement:
    tolerations:
    - effect: NoSchedule
      key: node-role.kubernetes.io/infra
```

Additionally, we need to define a node selector.
Again looking at the CRD's definition, the node selector by itself looks like this:

```yaml
spec:
  nodePlacement:
    nodeSelector:
      matchLabels:
        node-role.kubernetes.io/infra: ""
```

So what we effectively want is the combination of the two.

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

```yaml
spec:
  nodePlacement:
    nodeSelector:
      matchLabels:
        node-role.kubernetes.io/infra: ""
    tolerations:
    - effect: NoSchedule
      key: node-role.kubernetes.io/infra
```

{{% /details %}}

Add the node selector and toleration to the `default` `ingresscontroller` resource.

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

You could either directly edit the resource:

```bash
oc edit ingresscontroller/default -n openshift-ingress-operator
```

Or use the following patch command:

```bash
oc patch ingresscontroller/default -n openshift-ingress-operator --type=merge -p '{"spec":{"nodePlacement": {"nodeSelector": {"matchLabels": {"node-role.kubernetes.io/infra": ""}},"tolerations": [{"effect":"NoSchedule","key": "node-role.kubernetes.io/infra"}]}}}'
```

{{% /details %}}

You can now observe the Ingress Controller Operator do its work and immediately start redeploying the router pods onto the infra nodes:

{{% alert title="Note" color="primary" %}}
You'll have to manually terminate the following command with ctrl+c.
{{% /alert %}}

```bash
oc get pods -n openshift-ingress -o wide -w
```


## Task {{% param sectionnumber %}}.3: Registry

Repeating the same procedure as we did with the router but with the `configs.imageregistry.operator.openshift.io` CRD, we end up with a similar-looking definition:

```yaml
spec:
  nodeSelector:
    node-role.kubernetes.io/infra: ""
  tolerations:
  - effect: NoSchedule
    key: node-role.kubernetes.io/infra
```

Add it to the `configs.imageregistry.operator.openshift.io` custom resource `cluster`.

{{% details title="Hints" mode-switcher="normalexpertmode" %}}
Again, either edit the resource directly or use the following command:

```bash
oc patch configs.imageregistry.operator.openshift.io/cluster --type=merge -p '{"spec": {"nodeSelector": {"node-role.kubernetes.io/infra": ""}, "tolerations": [{"effect": "NoSchedule", "key": "node-role.kubernetes.io/infra"}]}}'
```

{{% /details %}}

Watch the pods how they're being redeployed one by one:

```bash
oc get pods -n openshift-image-registry -o wide -w
```


## Task {{% param sectionnumber %}}.4: Monitoring

Moving the monitoring stack involves moving a number of different pods to the infra nodes.
Placement of these pods is defined in a ConfigMap and is documented [here](https://docs.openshift.com/container-platform/latest/monitoring/configuring-the-monitoring-stack.html#assigning-tolerations-to-monitoring-components_configuring-the-monitoring-stack).

Gather all the relevant information and create an appropriate ConfigMap.

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

{{< readfile file="/content/en/docs/02/resources/configmap_cluster-monitoring-config.yaml" code="true" lang="yaml" >}}

{{% /details %}}

Apply it to the cluster.

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

```bash
oc create -f https://raw.githubusercontent.com/acend/openshift-4-ops-training/main/content/en/docs/02/resources/configmap_cluster-monitoring-config.yaml
```

{{% /details %}}

As soon as the ConfigMap is created, the monitoring operator begins redeploying the pods:

```bash
oc get pods -n openshift-monitoring -o wide -w
```

This concludes moving all the current infrastructure components to the infra nodes.
