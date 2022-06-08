---
title: "3.3 User-defined monitoring rules"
weight: 33
sectionnumber: 3.3
---

## Task {{% param sectionnumber %}}.1: Enable the user-workload monitoring stack

As mentioned in a lab before, it is not supported to alter the OpenShift-provided monitoring stack. If you want to add additional monitoring targets or add custom Prometheus rules, you need to enable the user-workload monitoring stack.

And this is exactly what we are going to do. First, edit the `cluster-monitoring-config` configmap in the `openshift-monitoring` namespace.

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

```bash
oc -n openshift-monitoring edit cm cluster-monitoring-config
```

{{% /details %}}

Add the line `enableUserWorkload: true` under `data/config.yaml` (leave the rest as it is).

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

```yaml
...
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    enableUserWorkload: true
...
```

{{% /details %}}

Saving the configmap will automatically deploy your own Prometheus instance in a newly created namespace `openshift-user-workload-monitoring`.
Verify that all pods are in state `Running`.

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

```bash
oc -n openshift-user-workload-monitoring get pods
```

```bash
NAME                                  READY   STATUS    RESTARTS   AGE
prometheus-operator-764495967-z5tln   2/2     Running   0          74s
prometheus-user-workload-0            5/5     Running   1          65s
prometheus-user-workload-1            5/5     Running   1          65s
thanos-ruler-user-workload-0          3/3     Running   0          65s
thanos-ruler-user-workload-1          3/3     Running   0          65s
```

{{% /details %}}

**Optional:** You may also want to enable storage and persistence for your user workload monitoring stack, which is done in the same way as you learned in the previous lab.


## Task {{% param sectionnumber %}}.2: Add a custom Prometheus rule

With the user-workload monitoring stack now running, you can add your own custom Prometheus rules. To do so, create a PrometheusRule custom resource with the following content:

{{< highlight yaml >}}{{< readfile file="/content/en/docs/03/resources/custom-prometheus-rule.yaml" >}}{{< /highlight >}}

Either create a file with above content or apply it directly:

```bash
oc apply -f https://raw.githubusercontent.com/acend/openshift-4-ops-training/main/content/en/docs/03/resources/custom-prometheus-rule.yaml
```

Let's take a look at the main components that can be configured applying a custom PrometheusRule:

* **alert:** The alert's name should be as short and precise as possible because it is used by most receivers as the title of the alert message.
* **annotations:** Human-readable metadata like a brief summary or a link to a playbook explaining the alert and actions on the alert.
* **expr:** PromQL expression that defines the condition in which Prometheus should trigger the alert
* **for:** Duration of the fulfilled condition until the alarm is triggered. If not set, alerts are triggered at their first occurrence.
* **labels:** Metadata that can be used to dynamically route the alert to a corresponding receiver.

Verify the rule by accessing the Thanos ruler web interface.
You can find out its URL by looking at the `thanos-ruler` route in the `openshift-user-workload-monitoring`.

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

```bash
oc -n openshift-user-workload-monitoring get route thanos-ruler -o go-template='https://{{ .spec.host }}/alerts{{ "\n" }}'
```

{{% /details %}}


## Task {{% param sectionnumber %}}.3: Allowing users to add monitoring resources

By default, only cluster administrators have access to the user-workload monitoring stack and are able to create Prometheus custom resources like PrometheusRules. You can grant platform users access to the stack by giving them one of the cluster roles below:

* `monitoring-rules-view`: View `PrometheusRule` resources in their projects
* `monitoring-rules-edit`: Edit `PrometheusRule` resources in their projects
* `monitoring-edit`: Edit `PrometheusRule`, `ServiceMonitor` and `PodMonitor` resources in their projects
* `user-workload-monitoring-config-edit`: Let's users alter the `user-workload-monitoring-config` configmap in the `openshift-user-workload-monitoring` namespace

This is particularly useful as it allows users to change their monitoring configuration in a self-service manner.
