---
title: "1.3 Further components"
weight: 13
sectionnumber: 1.3
---

In this lab, you will install additional components that are commonly used in production.


## Task {{% param sectionnumber %}}.1: Install `cert-manager`

//FIXME: What is cert-manager?
With [`cert-manager`](https://cert-manager.io/) you can automate certificate management.

You will install `cert-manager` with Helm.

In order to install the Helm chart, you must follow these steps:

* Add the Jetstack Helm repository:

```bash
helm repo add jetstack https://charts.jetstack.io
```

* Install the cert-manager Helm chart:

```bash
helm install cert-manager jetstack/cert-manager \
  --namespace training-infra-cert-manager \
  --create-namespace \
  --version v1.3.0 \
  --values ~/ocp4-ops/resources/cert-manager/values.yaml
```

To be able to use Amazon Route 53 for Let's Encrypt's DNS01 challenge, you will need to create a secret containing the required credentials. You can find these on the bastion host at `~/ocp4-ops/resources/cert-manager`.

```bash
oc apply -f ~/ocp4-ops/resources/cert-manager/secret_route53-credentials.yaml
```

Now you can create the first `ClusterIssuer`:

```bash
oc apply -f ~/ocp4-ops/resources/cert-manager/clusterissuer_letsencrypt-producion.yaml
```

Verify that the `ClusterIssuer` is ready to issue certificates:

```bash
oc describe clusterissuer letsencrypt-production
```

Which should give you an output similar to this:

```
Status:
  Acme:
    Last Registered Email:  hello@openshift.ch
    Uri:                    https://acme-v02.api.letsencrypt.org/acme/acct/119084055
  Conditions:
    Last Transition Time:  2021-04-13T09:44:14Z
    Message:               The ACME account was registered with the ACME server
    Observed Generation:   1
    Reason:                ACMEAccountRegistered
    Status:                True
    Type:                  Ready
```


## Task {{% param sectionnumber %}}.2 Replace the ingress controller certificate

After your `ClusterIssuer` is ready, you can request a wildcard certificate to be used on the Ingress Controller for the default subdomain `apps.+username+-ops-training.openshift.ch`:

```bash
oc apply -f ~/ocp4-ops/resources/cert-manager/certificate_wildcard-ingress.yaml
```

It will take around 90 seconds for the certificate to be ready. Check with:

```bash
oc -n openshift-ingress get certificate
```

You're looking for the column `READY` to be `True`:

```
NAME                    READY   SECRET                  AGE
cert-wildcard-ingress   True    cert-wildcard-ingress   6m2s
```

Update the Ingress Controller configuration to use the certificate:

```bash
oc patch ingresscontroller.operator default \
   --type=merge -p \
   '{"spec":{"defaultCertificate": {"name": "cert-wildcard-ingress"}}}' \
   -n openshift-ingress-operator
```

After the Ingress Controller has finished rolling out, the console URL should now present a valid certificate.


## Task {{% param sectionnumber %}}.3 Replace the API certificate

The default API server certificate is issued by an internal OpenShift Container Platform cluster CA. Clients outside of the cluster will not be able to verify the API server’s certificate by default. This certificate can be replaced by one that is issued by a CA that clients trust.

```bash
oc apply -f /home/ec2-user/ocp4-ops/resources/cert-manager/certificate_api.yaml
```

Again, check if the certificate is ready, then patch the API server:

```bash
oc patch apiserver cluster \
     --type=merge -p \
     '{"spec":{"servingCerts": {"namedCertificates":
     [{"names": ["api.+username+-ops-training.openshift.ch"],
     "servingCertificate": {"name": "cert-api"}}]}}}'
```

Since the `kubeconfig` file you have been working with so far contains the CA of the self-signed certificate, the newly created certificate cannot be validated against this CA:

```bash
$ oc whoami
Unable to connect to the server: x509: certificate signed by unknown authority
```

Open the kubeconfig file with your favourite editor (e.g. `vim $KUBECONFIG`) and remove these marked lines:

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0...                          # delete this line
    server: https://api.+username+-ops-training.openshift.ch:6443
    name: api-+username+-ops-training-openshift-ch:6443
- cluster:
    certificate-authority-data: LS0...                          # delete this line
    server: https://api.+username+-ops-training.openshift.ch:6443
  name: +username+-ops-training
```

Check again:

```bash
$ oc whoami
system:admin
```


## Task {{% param sectionnumber %}}.4 Install Velero

//FIXME: What is Velero?
[Velero](https://velero.io/) is a tool to back up and restore Kubernetes resources.
We will use Velero in this training only for data protection of user workload.

{{% alert title="Note" color="primary" %}}
We already created S3 buckets for you to use as backup locations.
{{% /alert %}}

You will install Velero with Helm.

//FIXME: Prereq.
In order to install the Helm chart, you must follow these steps:

* Add the VMware Tanzu Helm repository:

```bash
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
```

* Install the Velero Helm chart

```bash
helm install velero vmware-tanzu/velero \
  --namespace training-infra-velero \
  --create-namespace \
  --version v2.16.0 \
  --set-file credentials.secretContents.cloud=/home/ec2-user/ocp4-ops/credentials \
  --values ~/ocp4-ops/resources/velero/values.yaml
```

After the installation has completed, you can verify the backup location:

```bash
oc -n training-infra-velero get backupstoragelocations
```

The status of `PHASE` should be `Available`:

```
NAME      PHASE       LAST VALIDATED   AGE
default   Available   10s              156m
```

In the next chapter you will learn how to use Velero for scheduled backups of cluster resources.


## Task {{% param sectionnumber %}}.5 Install Cluster Logging

You can install [OpenShift Logging](https://docs.openshift.com/container-platform/latest/logging/cluster-logging.html) to aggregate all the logs from your OpenShift cluster, such as node logs, application logs, and infrastructure logs.

To deploy the Cluster Logging stack from the CLI, we need to create the following objects:

* OpenShift Elasticsearch Operator Namespace

{{< highlight yaml >}}{{< readfile file="content/en/docs/02/resources/logging/ns_openshift-operators-redhat.yaml" >}}{{< /highlight >}}

* Cluster Logging Operator Namespace

{{< highlight yaml >}}{{< readfile file="content/en/docs/02/resources/logging/ns_openshift-logging.yaml" >}}{{< /highlight >}}

* Elasticsarch `OperatorGroup`

{{< highlight yaml >}}{{< readfile file="content/en/docs/02/resources/logging/og_openshift-operators-redhat.yaml" >}}{{< /highlight >}}

* Cluster Logging `OperatorGroup`

{{< highlight yaml >}}{{< readfile file="content/en/docs/02/resources/logging/og_cluster-logging.yaml" >}}{{< /highlight >}}

* Elasticsearch Operator `Subscription`

{{< highlight yaml >}}{{< readfile file="content/en/docs/02/resources/logging/sub_elasticsearch-operator.yaml" >}}{{< /highlight >}}

* Cluster Logging Operator `Subscription`

{{< highlight yaml >}}{{< readfile file="content/en/docs/02/resources/logging/sub_cluster-logging.yaml" >}}{{< /highlight >}}

You can also use our provided files:

```bash
oc apply -f https://raw.githubusercontent.com/acend/openshift-4-ops-training/main/content/en/docs/02/resources/logging/ns_openshift-operators-redhat.yaml
oc apply -f https://raw.githubusercontent.com/acend/openshift-4-ops-training/main/content/en/docs/02/resources/logging/ns_openshift-logging.yaml
oc apply -f https://raw.githubusercontent.com/acend/openshift-4-ops-training/main/content/en/docs/02/resources/logging/og_openshift-operators-redhat.yaml
oc apply -f https://raw.githubusercontent.com/acend/openshift-4-ops-training/main/content/en/docs/02/resources/logging/og_cluster-logging.yaml
oc apply -f https://raw.githubusercontent.com/acend/openshift-4-ops-training/main/content/en/docs/02/resources/logging/sub_elasticsearch-operator.yaml
oc apply -f https://raw.githubusercontent.com/acend/openshift-4-ops-training/main/content/en/docs/02/resources/logging/sub_cluster-logging.yaml
```

Verify the Elasticsearch Operator installation:

```bash
oc get csv --all-namespaces
```

Example output:

```
NAMESPACE                                               NAME                                            DISPLAY                  VERSION               REPLACES   PHASE
default                                                 elasticsearch-operator.5.0.0-202007012112.p0    OpenShift Elasticsearch Operator   5.0.0-202007012112.p0               Succeeded
kube-node-lease                                         elasticsearch-operator.5.0.0-202007012112.p0    OpenShift Elasticsearch Operator   5.0.0-202007012112.p0               Succeeded
kube-public                                             elasticsearch-operator.5.0.0-202007012112.p0    OpenShift Elasticsearch Operator   5.0.0-202007012112.p0               Succeeded
kube-system                                             elasticsearch-operator.5.0.0-202007012112.p0    OpenShift Elasticsearch Operator   5.0.0-202007012112.p0               Succeeded
openshift-apiserver-operator                            elasticsearch-operator.5.0.0-202007012112.p0    OpenShift Elasticsearch Operator   5.0.0-202007012112.p0               Succeeded
openshift-apiserver                                     elasticsearch-operator.5.0.0-202007012112.p0    OpenShift Elasticsearch Operator   5.0.0-202007012112.p0               Succeeded
openshift-authentication-operator                       elasticsearch-operator.5.0.0-202007012112.p0    OpenShift Elasticsearch Operator   5.0.0-202007012112.p0               Succeeded
openshift-authentication                                elasticsearch-operator.5.0.0-202007012112.p0    OpenShift Elasticsearch Operator   5.0.0-202007012112.p0               Succeeded
...
```

There should be an OpenShift Elasticsearch Operator in each Namespace. The version number might be different than shown.

Verify the Cluster Logging Operator isntallation:

```bash
oc get csv -n openshift-logging
```

Example output:

```
NAMESPACE                                               NAME                                         DISPLAY                  VERSION               REPLACES   PHASE
...
openshift-logging                                       clusterlogging.5.0.0-202007012112.p0         OpenShift Logging          5.0.0-202007012112.p0              Succeeded
...
```

There should be a Cluster Logging Operator in the openshift-logging Namespace. The Version number might be different than shown.

Now you can create a OpenShift Logging instance:

{{< highlight yaml >}}{{< readfile file="content/en/docs/02/resources/logging/og_cluster-logging.yaml" >}}{{< /highlight >}}

Again, you can use the provided file:

```bash
oc apply -f https://raw.githubusercontent.com/acend/openshift-4-ops-training/main/content/en/docs/02/resources/logging/clusterlogging_instance.yaml
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
cluster-logging-operator-66f77ffccb-ppzbg       1/1     Running   0          7m
elasticsearch-cdm-ftuhduuw-1-ffc4b9566-q6bhp    2/2     Running   0          2m40s
elasticsearch-cdm-ftuhduuw-2-7b4994dbfc-rd2gc   2/2     Running   0          2m36s
elasticsearch-cdm-ftuhduuw-3-84b5ff7ff8-gqnm2   2/2     Running   0          2m4s
fluentd-587vb                                   1/1     Running   0          2m26s
fluentd-7mpb9                                   1/1     Running   0          2m30s
fluentd-flm6j                                   1/1     Running   0          2m33s
fluentd-gn4rn                                   1/1     Running   0          2m26s
fluentd-nlgb6                                   1/1     Running   0          2m30s
fluentd-snpkt                                   1/1     Running   0          2m28s
kibana-d6d5668c5-rppqm                          2/2     Running   0          2m39s
```

