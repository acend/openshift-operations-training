---
title: "3.2 Monitoring stack configuration"
weight: 32
sectionnumber: 3.2
---

## Task {{% param sectionnumber %}}.1: Enable alerting

Let's take a look at an example of how you can send alerts to a receiver of your choice.
Configuring a new receiver can either be done using the web console or by directly configuring the appropriate secret.
Before going into detail on where this configuration lies, let's have a look at an example:

{{< highlight yaml >}}{{< readfile file="content/en/docs/03/resources/alertmanager-main.yaml" >}}{{< /highlight >}}

The main components that can be configured when applying a custom Alertmanager configuration comprise:

* `receivers`: With a receiver, one or more notification types such as mails, webhooks or messaging platforms like Slack or PagerDuty can be defined.
* `routes`: With routing blocks, a tree of routes and child routes can be defined. Each routing block has a matcher which can match one or several labels of an alert. Per block, one receiver can be specified, or if empty, the default receiver is taken.
* `Watchdog`: There is a default Watchdog alert that is periodically firing to make sure that the complete alerting pipeline is functional. It is best practice to implement a Deadman's switch on the receiving side to get notified when sending alerts should fail.
* `inhibit_rules`: You can group alerts so if one alert of this group was triggered, others will not. This is to prevent notification flooding. As an example, imagine an etcd member is unavailable (**severity: critical**). As a consequence, this would also trigger a high number of failed leader proposals (**severity: warning**). When fixing the unavailable etcd member, both alerts will be resolved, so you only care about one of them.

You are now going to configure Alertmanager so that alerts created by Prometheus are sent to your alerts channel on Slack.
In order to do this, navigate to **Administration**, **Cluster Settings**, **Global Configuration**, **Alertmanager** on the web console.
Here you can see the pre-configured **Receivers**.
Click on **Configure** next to the **Default** receiver.
Now fill in the **Slack API URL** and **Channel** information provided to you by your trainer and click **Save**.

{{% alert title="Note" color="primary" %}}
By clicking on **Show advanced configuration**, you have the option of customizing the messages' appearance in Slack.
Feel free to do so if you'd like.
{{% /alert %}}

Switch to Slack and check if notifications are showing up in your alerts channel.
This might take some minutes so don't hesitate to continue the lab and check back later.

Configuring Alertmanager receivers via web console has its advantages because it takes out most of its difficulty.
However, the time will come when you want to configure receivers automatically or with more advanced settings not exposed on the web console.
In order to do so, you adapt the `alertmanager-main` secret in the `openshift-monitoring` namespace.
Have a look at how Alertmanager is now configured:

```bash
oc -n openshift-monitoring extract secrets/alertmanager-main
```


## Task {{% param sectionnumber %}}.2: Persistence and metrics retention

By default, the monitoring stack loses all scraped metrics on restarts because it does not persist its data. Let's make our Prometheus time series database persistent and increase the metrics retention to 2 days so the volume won't fill up.

To do so, you can edit the `cluster-monitoring-config` configmap in the `openshift-monitoring` namespace:

```bash
oc -n openshift-monitoring edit cm cluster-monitoring-config
```

Add the following snippet:

```yaml
    ...
    prometheusK8s:
      ...
      retention: 2d
      volumeClaimTemplate:
       spec:
         storageClassName: gp2
         resources:
           requests:
             storage: 5Gi
    ...
```

It is advisable to persist the Alertmanager, too. This allows Alertmanager to retain active alerts on restarts and therefore prevents triggering existing alerts again:

```bash
oc -n openshift-monitoring edit cm cluster-monitoring-config
```

Add the following config:

```yaml
    ...
    alertmanagerMain:
      ...
      volumeClaimTemplate:
       spec:
         storageClassName: gp2
         resources:
           requests:
             storage: 1Gi
      ...
```

After adding the above configuration to the ConfigMap, the Cluster Monitoring Operator will create the requested persistent volumes:

```bash
oc -n openshift-monitoring get pvc
```

The output should look similar to this:

```bash
NAME                                       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
alertmanager-main-db-alertmanager-main-0   Bound    pvc-49453e5f-1e04-4d00-88cb-cd04528c9700   1Gi        RWO            gp2            5m43s
alertmanager-main-db-alertmanager-main-1   Bound    pvc-42b8083b-5dc2-4218-a8e2-15111e0a87e7   1Gi        RWO            gp2            5m43s
alertmanager-main-db-alertmanager-main-2   Bound    pvc-824824e3-ee0b-49c6-a786-3efd86055acb   1Gi        RWO            gp2            5m42s
prometheus-k8s-db-prometheus-k8s-0         Bound    pvc-19894227-c8ad-4ac3-a793-71a4a409fd23   5Gi        RWO            gp2            6m32s
prometheus-k8s-db-prometheus-k8s-1         Bound    pvc-11fcf37e-5803-41e7-95ae-f2779010b201   5Gi        RWO            gp2            6m32s
```
