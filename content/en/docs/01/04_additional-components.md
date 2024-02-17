---
title: "1.4 Additional components"
weight: 14
sectionnumber: 1.4
---

In this lab, you will install additional components that are commonly used in production.


## Task {{% param sectionnumber %}}.1: Install cert-manager

cert-manager massively simplifies certificate management for Kubernetes. It provides easy to use tools to issue, manage and automatically renew certificates and supports the ACME protocol. This allows us to easily and automatically get certificates signed by Let's Encrypt for our cluster.

Install cert-manager as an Operator using the OperatorHub:

* In the **Administrator** view, navigate to **Operators** -> **OperatorHub**
* Enter **cert-manager** into the filter box
* Select the **cert-manager Operator for Red Hat OpenShift** and click **Install**
* Leave everything as it is, except that you set the **Update approval** to **Manual**
* Click **Install**
* As soon as it appears, approve the install plan presented to you by clicking **Approve**
* Check for the existence of the `cert-manager-operator` pod

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

```bash
oc --namespace cert-manager-operator get pods
```

{{% /details %}}

* Also, the Operator should already have created at least the webhook and cainjector pods in the `cert-manager` namespace

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

```bash
oc --namespace cert-manager get pods
```

{{% /details %}}

The Operator will also automatically create a `CertManager` custom resource named `cluster`.
In order to use DNS01 challenges on OpenShift clusters installed on AWS, we need to [add some configuration to cert-manager's configuration](https://docs.openshift.com/container-platform/latest/security/cert_manager_operator/cert-manager-operator-issuer-acme.html#cert-manager-acme-dns01-explicit-aws_cert-manager-operator-issuer-acme):

```yaml
apiVersion: operator.openshift.io/v1alpha1
kind: CertManager
metadata:
  name: cluster
  ...
spec:
  ...
  controllerConfig:
    overrideArgs:
      - '--dns01-recursive-nameservers-only'
      - '--dns01-recursive-nameservers=1.1.1.1:53'
```

This effectivley adds these two parameters to the cert-manager deployment. They are needed because of AWS' different DNS zones for private and public resolution. This way cert-manager knows that, in order to check for the DNS01 challenge, it needs to call a public DNS server.

The last piece of configuration missing are the credentials. On supported infrastructure providers, you can simply get these credentials via the Cloud Credential Operator. [As AWS is supported, we're going to use this mechanic](https://docs.openshift.com/container-platform/latest/security/cert_manager_operator/cert-manager-authentication-non-sts.html):

* Create a CredentialsRequest resource YAML file named `credentialsrequest_cert-manager.yaml` with:

```yaml
apiVersion: cloudcredential.openshift.io/v1
kind: CredentialsRequest
metadata:
  name: cert-manager
  namespace: openshift-cloud-credential-operator
spec:
  providerSpec:
    apiVersion: cloudcredential.openshift.io/v1
    kind: AWSProviderSpec
    statementEntries:
    - action:
      - "route53:GetChange"
      effect: Allow
      resource: "arn:aws:route53:::change/*"
    - action:
      - "route53:ChangeResourceRecordSets"
      - "route53:ListResourceRecordSets"
      effect: Allow
      resource: "arn:aws:route53:::hostedzone/*"
    - action:
      - "route53:ListHostedZonesByName"
      effect: Allow
      resource: "*"
  secretRef:
    name: aws-creds
    namespace: cert-manager
  serviceAccountNames:
  - cert-manager
```

* Create it:

```bash
oc create -f credentialsrequest_cert-manager.yaml
```

* Update the subscription object for cert-manager Operator for Red Hat OpenShift by running the following command:

```bash
oc --namespace cert-manager-operator patch subscription openshift-cert-manager-operator --type=merge -p '{"spec":{"config":{"env":[{"name":"CLOUD_CREDENTIALS_SECRET_NAME","value":"aws-creds"}]}}}'
```

* Verify that the cert-manager controller pod shows a volume and a mountPath entry each for the secret called `aws-creds`

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

```bash
oc --namespace cert-manager get pod --selector app.kubernetes.io/component=controller -o yaml
```

* You are looking for the following entries:

```yaml
...
spec:
  containers:
  - args:
    ...
    - mountPath: /.aws
      name: cloud-credentials
  ...
  volumes:
  ...
  - name: cloud-credentials
    secret:
      ...
      secretName: aws-creds
```

{{% /details %}}

Now you can create the `ClusterIssuer` resource which is supplied as a file inside the directory `~/ocp4-ops/resources/cert-manager/`.

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

```bash
oc create -f ~/ocp4-ops/resources/cert-manager/clusterissuer_letsencrypt-producion.yaml
```

{{% /details %}}

Verify that the `ClusterIssuer` is ready to issue certificates:

```bash
oc get clusterissuer letsencrypt-production
```

Look for column `READY` to be true:

```
NAME                     READY   AGE
letsencrypt-production   True    37s
```


## Task {{% param sectionnumber %}}.2 Replace the ingress controller certificate

After your `ClusterIssuer` is ready, you can request a wildcard certificate to be used on the Ingress Controller for the default subdomain `apps.+username+-ops-training.openshift.ch`.

Create a file named `certificate_ingress.yaml` and fill in the following content, replacing the placeholder `<username>` with your actual username `+username+`.

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

The default API server certificate is issued by an internal OpenShift Container Platform cluster CA. Clients outside of the cluster will not be able to verify the API server’s certificate by default. This certificate can be replaced by one that is issued by a CA that clients trust.

Create a file named `certificate_api.yaml` and fill in the following content.  
Adjust the paramaters `commonName` and `dnsNames` to match your cluster name.

{{< readfile file="/content/en/docs/01/resources/cert-manager/certificate_api.yaml" code="true" lang="yaml" >}}

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

```bash
oc apply -f certificate_api.yaml
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


## Task {{% param sectionnumber %}}.4 Install OADP

[OpenShift API for Data Protection](https://docs.openshift.com/container-platform/latest/backup_and_restore/application_backup_and_restore/oadp-intro.html), or short OADP, is a tool to back up and restore Kubernetes resources.

{{% alert title="Note" color="primary" %}}
S3 buckets were already created for you to use as backup locations.
{{% /alert %}}

OADP is provided as an Operator, you will therefore install it using the OperatorHub:

* Navigate to **Operators** -> **OperatorHub**
* Search for **OADP**
* Select the **OADP Operator** provided by the Red Hat catalog (indicated by the **Red Hat** tag in the upper right of the box)
* Click **Install**
* Leave everything as it is, except that you set the **Update approval** to **Manual**
* Click **Install**
* As soon as it appears, approve the install plan presented to you by clicking **Approve**

What's left is to configure OADP:

* Create a secret containing the access credentials for OADP to access the S3 bucket:

```bash
oc --namespace openshift-adp create secret generic cloud-credentials --from-file cloud=~/ocp4-ops/resources/velero/credentials
```

* Create a `DataProtectionApplication` manifest as follows, named `dataprotectionapplication_ops-training-aws.yaml`, replacing the placeholder `<username>` with your actual username `+username+`:

{{< readfile file="/content/en/docs/01/resources/dataprotectionapplication_ops-training-aws.yaml" code="true" lang="yaml" >}}

* Lastly, apply the manifest

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

```bash
oc create -f dataprotectionapplication_ops-training-aws.yaml
```

{{% /details %}}

In one of the next chapters, you will learn how to use OADP for scheduled backups of cluster resources.
