---
title: "1.3 Further components"
weight: 13
sectionnumber: 1.3
---

In this lab, you will install additional components that are commonly used in production.


## Task {{% param sectionnumber %}}.1: Install `cert-manager`

//FIXME: What is cert-manager?

You will install `cert-manager` with Helm.

In order to install the Helm chart, you must follow these steps:

```bash
## Add the Jetstack Helm repository
$ helm repo add jetstack https://charts.jetstack.io

## Install the cert-manager Helm chart
$ helm install cert-manager jetstack/cert-manager \
  --namespace training-infra-cert-manager \
  --create-namespace \
  --version v1.3.0 \
  --values ~/ocp4-ops/resources/cert-manager/values.yaml
```

To be able to use the Amazon Route 53 for the DNS01 challenge of Let's Encrypt, you will need to create a secret containing the required credentials. We have already created the manifest in the folder `~/ocp4-ops/resources/cert-manager`.

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

After your `ClusterIssuer` is ready, you can request a wildcard certificate to be used on the Ingress Controller for the default subdomain `apps.user01-ops-training.openshift.ch`. #FIXME: Hugo var

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

Check, if the certificate is ready:

```bash
$ oc -n openshift-config get cert
NAME       READY   SECRET     AGE
cert-api   True    cert-api   3m7s
```

Patch the API server:

```bash
oc patch apiserver cluster \
     --type=merge -p \
     '{"spec":{"servingCerts": {"namedCertificates":
     [{"names": ["api.user01-ops-training.openshift.ch"],
     "servingCertificate": {"name": "cert-api"}}]}}}'
#FIXME: FQDN Hugo var
```

Since the `kubeconfig` file you have been working with so far (remember `export KUBECONFIG=...` in the installation lab), contains the CA of the self-signed certificate, the newly created certificate cannot be validated against this CA:

```bash
$ oc whoami
Unable to connect to the server: x509: certificate signed by unknown authority
```

To continue working with these credentials, you will need to delete the lines containing the CA certificate.

```yaml
$ vim $KUBECONFIG
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0...                          # delete this line
    server: https://api.user01-ops-training.openshift.ch:6443   #FIXME: Hugo var
    name: api-user01-ops-training-openshift-ch:6443             #FIXME: Hugo var
- cluster:
    certificate-authority-data: LS0...                          # delete this line
    server: https://api.user01-ops-training.openshift.ch:6443   #FIXME: Hugo var
  name: user01-ops-training                                     #FIXME: Hugo var
```

Check again:

```bash
$ oc whoami
system:admin
```


## Task {{% param sectionnumber %}}.4 Install `Velero`

//FIXME: What is Velero?

{{% alert title="Note" color="primary" %}}We already created S3 buckets for you to use as backup locations.{{% /alert %}}

You will install `Velero` with Helm.

//FIXME: Prereq.
In order to install the Helm chart, you must follow these steps:

```bash
## Add the VMware Tanzu Helm repository
$ helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts

## Install the Velero Helm chart
$ helm install velero vmware-tanzu/velero \
  --namespace training-infra-velero \
  --create-namespace \
  --version v2.16.0 \
  --set-file credentials.secretContents.cloud=/home/ec2-user/ocp4-ops/credentials \
  --values ~/ocp4-ops/resources/velero/values.yaml
```

After the installation has completed, you can verify the backup location:

```bash
$ velero -n training-infra-velero get backup-locations
NAME      PROVIDER   BUCKET/PREFIX                PHASE       LAST VALIDATED                  ACCESS MODE   DEFAULT
default   aws        user01-ops-training-backup   Available   2021-04-14 08:04:24 +0000 UTC   ReadWrite
#FIXME: Hugo var
#FIXME: Change to "oc"
```

In the next chapter you will learn how to use Velero for scheduled backups of cluster resources.

