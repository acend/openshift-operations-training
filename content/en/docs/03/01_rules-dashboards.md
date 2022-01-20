---
title: "3.1 Rules and dashboards"
weight: 31
sectionnumber: 3.1
---

## Task {{% param sectionnumber %}}.1: Provided dashboards

OpenShift provides a set of default dashboards for cluster adminstrators and OpenShift users.

* **Cluster dashboards**

To get a sense of where these dashboards can be found, let's take a look at the performance of the etcd cluster.

Navigate to **Monitoring** -> **Dashboard** on the web console and select **etcd** in the dashboard drop-down list.

![cluster dashboard](../cluster-dashboard.png)

If you are more comfortable using Grafana you can display those dashboards in the OpenShift-provided Grafana. 

Run the following command to get the URL to your Prometheus instance:

```bash
oc -n openshift-monitoring get route grafana -o go-template='https://{{ .spec.host }}{{ "\n" }}'
```

* **User dashboards**

Switch to the **Developer** console and select the **uptime-app-prod** project. Then navigate to the **Monitoring** tab. Select the **uptime-app** in the **Workload** drop-down list. This will display a wide selection of different metrics relevant to your app like CPU usage, memory usage and networking performance.

![cluster dashboard](../user-dashboard.png)


## Task {{% param sectionnumber %}}.2: Provided alerting rules

OpenShift provides an extensive set of alerts, based on the [kubernetes-mixin](https://github.com/kubernetes-monitoring/kubernetes-mixin) project. Check out the configured alerts by navigating to your Prometheus web user interface and take a look at the predefined alerting rules.

Run the following command to get the URL to your Prometheus instance:

```bash
oc -n openshift-monitoring get route prometheus-k8s -o go-template='https://{{ .spec.host }}/alerts{{ "\n" }}'
```

Active alerts (state `Pending` or `Firing`) will also be displayed in the administrator web console at **Monitoring** -> **Alerting**.

As you can see, these alerts are as generic as possible to fit most platforms. They cannot be altered and adding more rules in `openshift-monitoring` is not supported. In a later lab, you will learn how to add rules by enabling monitoring for user-defined projects.


## Task {{% param sectionnumber %}}.3: Create silences

Sometimes you have to silence an alert because someone is already working on it, or the issue is scheduled to be fixed in a future maintenance window, or it might be false positive.

To do so, navigate to **Monitoring** -> **Alerting**. Select the **Silences** tab and hit the **Create Silence** button. A common task is to silence the `UpdateAvailable` alert, as we may have already scheduled the corresponding update.

Silence the alert for 1 week.
![silece](../create-silence.png)

This will suppress all `UpdateAvailable` alerts for one week.
