---
title: "3.4 Logging"
weight: 34
sectionnumber: 3.4
---

In this lab we will install and configure the logging stack.


## Task {{% param sectionnumber %}}.1 Install cluster logging

You can install [OpenShift Logging](https://docs.openshift.com/container-platform/latest/logging/cluster-logging.html) to aggregate all the logs from your OpenShift cluster, such as node logs, application logs and infrastructure logs.

To deploy the cluster logging stack from the CLI, we need to create the following objects:

* OpenShift Elasticsearch Operator Namespace

{{< highlight yaml >}}{{< readfile file="content/en/docs/03/resources/logging/ns_openshift-operators-redhat.yaml" >}}{{< /highlight >}}

* Red Hat OpenShift Logging Namespace

{{< highlight yaml >}}{{< readfile file="content/en/docs/03/resources/logging/ns_openshift-logging.yaml" >}}{{< /highlight >}}

* OpenShift Elasticsearch Operator `OperatorGroup`

{{< highlight yaml >}}{{< readfile file="content/en/docs/03/resources/logging/og_openshift-operators-redhat.yaml" >}}{{< /highlight >}}

* Red Hat OpenShift Logging `OperatorGroup`

{{< highlight yaml >}}{{< readfile file="content/en/docs/03/resources/logging/og_cluster-logging.yaml" >}}{{< /highlight >}}

* OpenShift Elasticsearch Operator `Subscription`

{{< highlight yaml >}}{{< readfile file="content/en/docs/03/resources/logging/sub_elasticsearch-operator.yaml" >}}{{< /highlight >}}

* Red Hat OpenShift Logging `Subscription`

{{< highlight yaml >}}{{< readfile file="content/en/docs/03/resources/logging/sub_cluster-logging.yaml" >}}{{< /highlight >}}

You can also use our provided files:

```bash
oc apply -f https://raw.githubusercontent.com/acend/openshift-4-ops-training/main/content/en/docs/03/resources/logging/ns_openshift-operators-redhat.yaml
oc apply -f https://raw.githubusercontent.com/acend/openshift-4-ops-training/main/content/en/docs/03/resources/logging/ns_openshift-logging.yaml
oc apply -f https://raw.githubusercontent.com/acend/openshift-4-ops-training/main/content/en/docs/03/resources/logging/og_openshift-operators-redhat.yaml
oc apply -f https://raw.githubusercontent.com/acend/openshift-4-ops-training/main/content/en/docs/03/resources/logging/og_cluster-logging.yaml
oc apply -f https://raw.githubusercontent.com/acend/openshift-4-ops-training/main/content/en/docs/03/resources/logging/sub_elasticsearch-operator.yaml
oc apply -f https://raw.githubusercontent.com/acend/openshift-4-ops-training/main/content/en/docs/03/resources/logging/sub_cluster-logging.yaml
```

Verify the Elasticsearch Operator installation:

```bash
oc get csv --all-namespaces
```

Example output:

```
NAMESPACE                                 NAME                              DISPLAY                            VERSION    REPLACES   PHASE
default                                   elasticsearch-operator.5.1.2-7    OpenShift Elasticsearch Operator   5.1.2-7               Succeeded
kube-node-lease                           elasticsearch-operator.5.1.2-7    OpenShift Elasticsearch Operator   5.1.2-7               Succeeded
kube-public                               elasticsearch-operator.5.1.2-7    OpenShift Elasticsearch Operator   5.1.2-7               Succeeded
kube-system                               elasticsearch-operator.5.1.2-7    OpenShift Elasticsearch Operator   5.1.2-7               Succeeded
openshift-apiserver-operator              elasticsearch-operator.5.1.2-7    OpenShift Elasticsearch Operator   5.1.2-7               Succeeded
openshift-apiserver                       elasticsearch-operator.5.1.2-7    OpenShift Elasticsearch Operator   5.1.2-7               Succeeded
openshift-authentication-operator         elasticsearch-operator.5.1.2-7    OpenShift Elasticsearch Operator   5.1.2-7               Succeeded
openshift-authentication                  elasticsearch-operator.5.1.2-7    OpenShift Elasticsearch Operator   5.1.2-7               Succeeded
...
```

There should be an OpenShift Elasticsearch Operator in each namespace. The version number might be different than shown.

Verify the Cluster Logging Operator isntallation:

```bash
oc get csv -n openshift-logging
```

Example output:

```
NAME                      DISPLAY                     VERSION   REPLACES                   PHASE
...
cluster-logging.5.1.2-7   Red Hat OpenShift Logging   5.1.2-7   cluster-logging.5.1.1-36   Succeeded
...
```

There should be a Cluster Logging Operator in the openshift-logging Namespace. The Version number might be different than shown.

Now you can create an OpenShift Logging instance:

{{< highlight yaml >}}{{< readfile file="content/en/docs/03/resources/logging/clusterlogging_instance.yaml" >}}{{< /highlight >}}

Again, you can use the provided file:

```bash
oc apply -f https://raw.githubusercontent.com/acend/openshift-4-ops-training/main/content/en/docs/03/resources/logging/clusterlogging_instance.yaml
```

{{% alert title="Note" color="primary" %}}
The provided configuration is a basic setup that should support most use cases. For details on how to configure the logging stack, consult the chapter [About the Cluster Logging custom resource](https://docs.openshift.com/container-platform/latest/logging/config/cluster-logging-configuring-cr.html) in the official documentation.
{{% /alert %}}

Verify the install by listing the pods in the openshift-logging project.

You should see several pods for OpenShift Logging, Elasticsearch, Fluentd, and Kibana.

```bash
oc get pods -n openshift-logging
```

Example output:

```
NAME                                            READY   STATUS    RESTARTS   AGE
cluster-logging-operator-664b7f76d8-v65hn       1/1     Running   0          12d
collector-74jkk                                 2/2     Running   0          12d
collector-g2djj                                 2/2     Running   0          12d
collector-gzrt5                                 2/2     Running   0          12d
collector-j2q86                                 2/2     Running   0          12d
collector-nkngv                                 2/2     Running   0          12d
collector-pbvhn                                 2/2     Running   0          12d
elasticsearch-cdm-0w2i8ldc-1-58b948bd84-cc7t2   2/2     Running   0          12d
elasticsearch-cdm-0w2i8ldc-2-7b6989ddb7-c78tg   2/2     Running   0          12d
elasticsearch-cdm-0w2i8ldc-3-64b5f55f58-9455z   2/2     Running   0          12d
kibana-687fdb6fb5-64jpw                         2/2     Running   0          12d
```


## Task {{% param sectionnumber %}}.2 Enable audit log forwarding

By default, the logging stack does not store the audit logs in Elasticsearch, since Elasticsearch does not provide encryption.  
For the purpose of this lab you will configure the logging stack to store the audit logs in the central Elasticsearch cluster.

We can make use of the cluster log forwarding feature of the OpenShift Logging stack, which allows us to route logs to different log stores by defining forwarding pipelines.

{{% alert title="Note" color="primary" %}}
Because we do not have an external log store to forward our logs to, we will make use of the cluster-internal Elasticsearch instance. Check the documentation on [Forwarding logs to third party systems](https://docs.openshift.com/container-platform/latest/logging/cluster-logging-external.html) to learn more about the supported third-party log stores.
{{% /alert %}}

To forward the audit logs to the internal Elasticsearch instance, we need to define a `ClusterLogForwarder` object:

{{< highlight yaml >}}{{< readfile file="content/en/docs/03/resources/logging/clusterlogforwarder_instance.yaml" >}}{{< /highlight >}}

You can also use our provided file:

```bash
oc apply -f https://raw.githubusercontent.com/acend/openshift-4-ops-training/main/content/en/docs/03/resources/logging/clusterlogforwarder_instance.yaml
```


## Task {{% param sectionnumber %}}.3 Configure and use Kibana

Now that you have installed and confugred the logging stack, it is time to check the log visualizer - Kibana.

To get to Kibana you can either click the Application Launcher ![Launcher](../app-launcher-icon.png) and select **Logging**, or by clicking on **Show in Kibana** in the log browser of the console:

![Show in Kibana](../kibana-link.png)

The log store of the logging stack (Elasticsearch) stores the logs in three index categories: Application, Infrastructure and - if enabled - Audit.  
The first time you log in to Kibana, you need to create index patterns for your user:

* Create the infra index pattern:
  * Define the infra index pattern ![Kibana Infra Index Pattern](../kibana-define-pattern01.png)
  * Select `@timestamp` as the filed name ![Kibana Infra Index Pattern](../kibana-define-pattern02.png)
* Repeat these steps for the remaining indices (audit, apps)
* Go to **Discover** to see the logs

The following screenshot shows the logs of the infra index for the pods in the namespace `openshift-kube-apiserver` for the last 12 hours:

![Kibana example](../kibana-example.png)

Try to see the same logs on your cluster by exploring the features of Kibana.
