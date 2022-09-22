---
title: "1.3 Configuration"
weight: 13
sectionnumber: 1.3
---

After the initial installation, we can start configuring the cluster.
Because of the extended use of operators, nearly everything in OpenShift 4 is configured via custom resources.
In some cases configmaps or templates are used, but this is clearly the minority.

Most of the configuration is done post-installation and can be used to better adapt the cluster to its environment or change its default behaviour.
We are going to configure our own kubelet arguments and change the default project template in order to automatically create some NetworkPolicy resources.


## Task {{% param sectionnumber %}}.1: Configure kubelet arguments

OpenShift lets us change the kubelet configuration via the custom resource `KubeletConfig`.
But why change it at all?

The default kubelet configuration doesn't reserve any memory or cpu for the kubelet or the operating system itself.
It also defines an eviction threshold which, in some cases, is too low for the kubelet to react fast enough.

Configure a `KubeletConfig` resource each for master and worker nodes defining the following parameters:

* Set a hard eviction threshold to 500Mi available memory
* Set the reserved kubelet resources to 250m cpu and 1Gi memory
* Set the reserved system resources to 250m cpu and 1Gi memory

You can find relevant information on the OpenShift documentation pages [here](https://docs.openshift.com/container-platform/latest/scalability_and_performance/recommended-host-practices.html#create-a-kubeletconfig-crd-to-edit-kubelet-parameters_recommended-host-practices) and [here](https://docs.openshift.com/container-platform/4.10/nodes/nodes/nodes-nodes-managing.html).

{{% details title="Hints" mode-switcher="normalexpertmode" %}}
Change the masters' configuration to this:

{{< readfile file="/content/en/docs/01/resources/kubeletconfig_master.yaml" code="true" lang="yaml" >}}

And the worker nodes' configuration to this:

{{< readfile file="/content/en/docs/01/resources/kubeletconfig_worker.yaml" code="true" lang="yaml" >}}

{{% alert title="Note" color="primary" %}}
Instead of defining specific `systemReserved` values, you could also simply supply a line defining `.spec.autoSizingReserved: true`.
That way OpenShift calculates recommended values for you.
{{% /alert %}}

{{% alert title="Note" color="primary" %}}
These resource files are also available at https://raw.githubusercontent.com/acend/openshift-4-ops-training/main/content/en/docs/01/resources/kubeletconfig_master.yaml and https://raw.githubusercontent.com/acend/openshift-4-ops-training/main/content/en/docs/01/resources/kubeletconfig_worker.yaml, respectively.
{{% /alert %}}

There are multiple possible ways to apply these configuration resources to the cluster.
We are going to use one of the quickest methods and apply via URL:

```bash
oc apply -f https://raw.githubusercontent.com/acend/openshift-4-ops-training/main/content/en/docs/01/resources/kubeletconfig_master.yaml
oc apply -f https://raw.githubusercontent.com/acend/openshift-4-ops-training/main/content/en/docs/01/resources/kubeletconfig_worker.yaml
```

{{% /details %}}

To learn more about kubelet configuration, check the [Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/) and [OpenShift documentation](https://docs.openshift.com/container-platform/latest/nodes/nodes/nodes-nodes-managing.html).


## Task {{% param sectionnumber %}}.2: Generate the default project template

When creating a new project using `oc new-project`, OpenShift instantiates a template using the information provided.
We can modify this template if we want to alter it or add more resources, such as NetworkPolicies or LimitRanges.

And this is exactly what you are going to do first of all:
Generate the default template according to the [documentation](https://docs.openshift.com/container-platform/latest/applications/projects/configuring-project-creation.html#modifying-template-for-new-projects_configuring-project-creation) and change its name to `project-request-extended`.

{{% alert title="Note" color="primary" %}}
You might want to change the filename the template is written to to something more meaningful than what is used in the documentation (`template.yaml`).
A good practice is to use a naming convention such as `<resource type>_<resource name>.yaml`.
This allows you to quickly find the resource definition you're looking for based on the filename in case it is part of a larger collection.
{{% /alert %}}

{{% details title="Hints" mode-switcher="normalexpertmode" %}}
Simply execute the command from the documentation with a slight change to the filename (according to the tip above):

```bash
oc adm create-bootstrap-project-template -o yaml > template_project-request-extended.yaml
```

Open the file using your favorite editor and change the name:

```yaml
kind: Template
metadata:
  creationTimestamp: null
  name: project-request-extended
objects:
```

{{% /details %}}


## Task {{% param sectionnumber %}}.3: Network policies

Now, what you usually want to ensure first on a new cluster is that the different namespaces are isolated from each other in terms of network traffic.
User A's pods in project A should not be able to directly talk to user B's pods in project B.
Except of course for certain use cases, but thanks to the flexibility of network policies, we can easily define a sane default and change it on a namespace-basis if necessary.

Add the necessary network policies to the default project template. A sane default is already provided by Red Hat in its [documentation](https://docs.openshift.com/container-platform/latest/networking/network_policy/multitenant-network-policy.html).

{{% details title="Hints" mode-switcher="normalexpertmode" %}}
Your file should now look like this:

{{% alert title="Note" color="primary" %}}
For your convenience, we also provide the file at <https://raw.githubusercontent.com/acend/openshift-4-ops-training/main/content/en/docs/01/resources/networkpolicies.yaml>.
{{% /alert %}}

```yaml
apiVersion: template.openshift.io/v1
kind: Template
metadata:
  creationTimestamp: null
  name: project-request-extended
  namespace: openshift-config
objects:
- apiVersion: project.openshift.io/v1
  kind: Project
  metadata:
    annotations:
      openshift.io/description: ${PROJECT_DESCRIPTION}
      openshift.io/display-name: ${PROJECT_DISPLAYNAME}
      openshift.io/requester: ${PROJECT_REQUESTING_USER}
    creationTimestamp: null
    name: ${PROJECT_NAME}
  spec: {}
  status: {}
- apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    creationTimestamp: null
    name: admin
    namespace: ${PROJECT_NAME}
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: admin
  subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: ${PROJECT_ADMIN_USER}
- apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: allow-from-openshift-ingress
    namespace: ${PROJECT_NAME}
  spec:
    ingress:
    - from:
      - namespaceSelector:
          matchLabels:
            network.openshift.io/policy-group: ingress
    podSelector: {}
    policyTypes:
    - Ingress
- apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: allow-from-openshift-monitoring
    namespace: ${PROJECT_NAME}
  spec:
    ingress:
    - from:
      - namespaceSelector:
          matchLabels:
            network.openshift.io/policy-group: monitoring
    podSelector: {}
    policyTypes:
    - Ingress
- apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: allow-same-namespace
    namespace: ${PROJECT_NAME}
  spec:
    podSelector:
    ingress:
    - from:
      - podSelector: {}
parameters:
- name: PROJECT_NAME
- name: PROJECT_DISPLAYNAME
- name: PROJECT_DESCRIPTION
- name: PROJECT_ADMIN_USER
- name: PROJECT_REQUESTING_USER
```

{{% alert title="Note" color="primary" %}}
Pay extra attention to the additional `metadata.namespace` fields added to the NetworkPolicy resources.
If you don't add them, the network policies might end up being created in another namespace.
{{% /alert %}}

{{% alert title="Note" color="primary" %}}
Also note the additional namespace definition (`openshift-config`) at the top of the resource definition.
This is to make sure the template is going to be created in the correct namespace where OpenShift will be looking for it.
{{% /alert %}}
{{% /details %}}


## Task {{% param sectionnumber %}}.4: Limit range

Limit ranges are very important in keeping the resource usage on the cluster in check.
In this training, we assume you already know about limit ranges.
If not, the [documentation](https://docs.openshift.com/container-platform/latest/nodes/clusters/nodes-cluster-limit-ranges.html) provides a good foundation.

A good starting point could look as follows:

{{< readfile file="/content/en/docs/01/resources/limitrange.yaml" code="true" lang="yaml" >}}

Add the LimitRange resource to your default project template.

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

Your file should now look like this:

{{< readfile file="/content/en/docs/01/resources/template_project-request-extended.yaml" code="true" lang="yaml" >}}

{{% /details %}}


## Task {{% param sectionnumber %}}.5: Configure the template

The only thing missing now is to configure the cluster to use the new template.

Create the template and configure all necessary resources so new projects use our custom template.

{{% alert title="Note" color="primary" %}}
You might want to have another look at the different documentation pages so you don't forget anything.
{{% /alert %}}

When you think everything is set up, create a new project and check that the NetworkPolicy and LimitRange resources were created automatically.

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

Create the template on the cluster by either using your own template:

```bash
oc apply -f template_project-request-extended.yaml
```

Or use our provided solution:

```bash
oc apply -f https://raw.githubusercontent.com/acend/openshift-4-ops-training/main/content/en/docs/01/resources/template_project-request-extended.yaml
```

You then need to label the `default` Namespace in order for our network policies to work (they refer to a namespace labelled `network.openshift.io/policy-group=ingress`):

```bash
oc label namespace default 'network.openshift.io/policy-group=ingress'
```

And finally, we need to configure the Project custom resource:

```bash
oc patch project.config.openshift.io cluster --type=merge --patch '{"spec": {"projectRequestTemplate": {"name": "project-request-extended"}}}'
```

{{% /details %}}


## Task {{% param sectionnumber %}}.6: Console URL

You probably already noticed the quite unwieldy console URL <https://console-openshift-console.apps.+username+-{{% param baseDomain %}}>.
It's among a number of other route hostnames that are created by OpenShift this way by default.

Because the console URL is the one we probably will use the most, we are going to simplify it.
Find the appropriate instructions in OpenShift's documentation and adapt the route name.

{{% details title="Hints" mode-switcher="normalexpertmode" %}}
The [proper way](https://docs.openshift.com/container-platform/latest/web_console/customizing-the-web-console.html#customizing-the-console-route_customizing-web-console) is to edit the ingress config resource:

```bash
oc edit ingress.config.openshift.io cluster
```

Append the `componentRoutes` part so your resource looks similar to the example below:

```yaml
apiVersion: config.openshift.io/v1
kind: Ingress
metadata:
  name: cluster
spec:
  componentRoutes:
    - name: console
      namespace: openshift-console
      hostname: console.apps.+username+-{{% param baseDomain %}}
  domain: apps.+username+-{{% param baseDomain %}}
```

{{% /details %}}
