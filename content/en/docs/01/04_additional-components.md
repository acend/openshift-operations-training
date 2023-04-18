---
title: "1.4 Additional components"
weight: 14
sectionnumber: 1.4
---

In this lab, you will install additional components that are commonly used in production.


## Task {{% param sectionnumber %}}.1: Install cert-manager

cert-manager massively simplifies certificate management for Kubernetes. It provides easy to use tools to issue, manage and automatically renew certificates and supports the ACME protocol. This allows us to easily and automatically get certificates signed by Let's Encrypt for our cluster.

You will install cert-manager with Helm.


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
  --values https://raw.githubusercontent.com/acend/openshift-operations-training/main/content/en/docs/01/resources/cert-manager/values.yaml
```

To be able to use Amazon Route 53 for Let's Encrypt's DNS01 challenges, you will need a secret containing the required credentials. You can find it on the bastion host at `~/ocp4-ops/resources/cert-manager`.

Create that secret.

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

```bash
oc -n training-infra-cert-manager apply -f ~/ocp4-ops/resources/cert-manager/secret_route53-credentials.yaml
```

{{% /details %}}

Now you can create the `ClusterIssuer` resource which is also supplied as a file inside the same directory.

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

```bash
oc apply -f ~/ocp4-ops/resources/cert-manager/clusterissuer_letsencrypt-producion.yaml
```

{{% /details %}}

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

After your `ClusterIssuer` is ready, you can request a wildcard certificate to be used on the Ingress Controller for the default subdomain `apps.+username+-ops-training.openshift.ch`.

Create a file named `certificate_ingress.yaml` and fill in the following content, replacing the placeholder `<username>` with your actual username +username+.

{{< readfile file="/content/en/docs/01/resources/cert-manager/certificate_ingress.yaml" code="true" lang="yaml" >}}

Now apply the file.

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

```bash
oc apply -f certificate_ingress.yaml
```

{{% /details %}}

It will take around 90 seconds for the certificate to be ready. Check with:

```bash
oc -n openshift-ingress get certificate
```

You're looking for the column `READY` to be `True`:

```
NAME           READY   SECRET         AGE
cert-ingress   True    cert-ingress   6m2s
```

Update the `IngressController` configuration [according to the documentation](https://docs.openshift.com/container-platform/latest/security/certificates/replacing-default-ingress-certificate.html) in order to use the certificate.

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

```bash
oc patch ingresscontroller.operator default \
   --type=merge -p \
   '{"spec":{"defaultCertificate": {"name": "cert-ingress"}}}' \
   -n openshift-ingress-operator
```

{{% /details %}}

After the router pods have all been replaced, the console URL should present a valid certificate.
Have a look inside the `openshift-console` namespace to see the state of the pods.

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

```bash
oc -n openshift-console get pods
```

{{% /details %}}


## Task {{% param sectionnumber %}}.3 Replace the API certificate

The default API server certificate is issued by an internal OpenShift Container Platform cluster CA. Clients outside of the cluster will not be able to verify the API server’s certificate by default. This certificate can be replaced by one that is issued by a CA that clients trust. Before you can apply the file you need to change the parameters `commonName` and `dnsNames` to match your cluster name.

You can find the file in the directory cert-manager directory mentioned above.

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

```bash
oc apply -f /home/ec2-user/ocp4-ops/resources/cert-manager/certificate_api.yaml
```

{{% /details %}}

Again, check if the certificate is ready, then [patch the API server to use the certificate](https://docs.openshift.com/container-platform/latest/security/certificates/api-server.html).

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

```bash
oc patch apiserver cluster \
     --type=merge -p \
     '{"spec":{"servingCerts": {"namedCertificates":
     [{"names": ["api.+username+-ops-training.openshift.ch"],
     "servingCertificate": {"name": "cert-api"}}]}}}'
```

oc patch apiserver cluster \
     --type=merge -p \
     '{"spec":{"servingCerts": {"namedCertificates":
     [{"names": ["api.+username+-ops-training.openshift.ch"],
     "servingCertificate": {"name": "cert-api"}}]}}}'

{{% /details %}}

Wait for the `kube-apiserver` operator to detect the configuration change and apply roll it out.

Since the `kubeconfig` file you have been working with so far contains the CA of the self-signed certificate, the newly created certificate cannot be validated against this CA:

```bash
$ oc whoami
Unable to connect to the server: x509: certificate signed by unknown authority
```

Open the kubeconfig file with your favourite editor (e.g. `vim $KUBECONFIG`) and remove the `certificate-authority-data` line under `clusters`.

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0...                          # delete this line
    server: https://api.+username+-ops-training.openshift.ch:6443
    name: api-+username+-ops-training-openshift-ch:6443
```

{{% /details %}}

Check again:

```bash
$ oc whoami
system:admin
```


## Task {{% param sectionnumber %}}.4 Install Velero

[Velero](https://velero.io/) is a tool to back up and restore Kubernetes resources.
We will use Velero in this training only for data protection of user workload.

{{% alert title="Note" color="primary" %}}
We already created S3 buckets for you to use as backup locations.
{{% /alert %}}

You will install Velero with Helm.

In order to install the Helm chart, follow these steps:

* Add the VMware Tanzu Helm repository:

```bash
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
```

* Create a `values.yaml` file and fill in the following content:

{{< readfile file="/content/en/docs/01/resources/velero/values.yaml" code="true" lang="yaml" >}}

* Open the `values.yaml` file and modify the paramter `configuration.backupStorageLocation.bucket` to reflect your username +username+

* Install the Velero Helm chart using your edited `values.yaml` file:

```bash
helm install velero vmware-tanzu/velero \
  --namespace training-infra-velero \
  --create-namespace \
  --set-file credentials.secretContents.cloud=/home/ec2-user/ocp4-ops/resources/velero/credentials \
  --values values.yaml
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
