---
title: "1.1 Installation"
weight: 11
sectionnumber: 1.1
---

In this lab, you are going to install an OpenShift 4 cluster on AWS.


## Task {{% param sectionnumber %}}.1: Preparing the environment

* Download public key
* SSH into your bastion host
* Create working directory


## Task {{% param sectionnumber %}}.2: Customizing the installation

See the [OpenShift installation documentation](https://docs.openshift.com/container-platform/latest/installing/installing_aws/installing-aws-customizations.html#installation-configuration-parameters_installing-aws-customizations) for a list of available parameters.

It is best practice to create a timestamped directory for each cluster installation, for example:

```bash
cd /home/ec2-user/ocp4-ops
mkdir $(date +"%Y-%m-%d")
```

Create a file called `install-config.yaml` in the previously created directory on the bastion host, add the following content and change the values of `metadata.name` and `platform.aws.userTags.user` to reflect your username +username+.
Additonally, add an SSH key (use an existing keypair or create a new one if needed) and the `pull-secret` (available at `~/ocp4ops/pull-secret` on the bastion host) for the installer to be able to pull all necessary images from the Red Hat container registry.

{{< highlight yaml >}}{{< readfile file="content/en/docs/01/resources/install-config.yaml" >}}{{< /highlight >}}

{{% alert title="Note" color="primary" %}}
This file is also available at https://raw.githubusercontent.com/acend/openshift-4-ops-training/main/content/en/docs/01/resources/install-config.yaml.
{{% /alert %}}

Backup the file `install-config.yaml`, since it will be consumed during the installation process:

```bash
mkdir backup
cp $(date +"%Y-%m-%d")/install-config.yaml backup/install-config.yaml
```


## Task {{% param sectionnumber %}}.3: Deploying the cluster

Now you are ready to create your own cluster:

```bash
openshift-install create cluster --dir=$(date +"%Y-%m-%d") --log-level=info
```

{{% alert title="Note" color="primary" %}}
The installation usually takes about 30 minutes to completed.
{{% /alert %}}

Example output:

```
...
INFO Install complete!
INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/home/myuser/install_dir/auth/kubeconfig'
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.mycluster.example.com
INFO Login to the console with user: "kubeadmin", and password: "4vYBz-Ee6gm-ymBZj-Wt5AL"
INFO Time elapsed: 36m22s
```


## Task {{% param sectionnumber %}}.4: Verifying the installation

By setting the environment variable `KUBECONFIG` you can provide credentials to the OpenShift CLI (`oc`):

```bash
export KUBECONFIG=$HOME/ocp4-ops/$(date +"%Y-%m-%d")/auth/kubeconfig
```

You will now check whether you can log in to the cluster with the `kubeadmin` credentials:

```bash
oc whoami --show-server
```

The output should look like this:

```
https://api.+username+-ops-training.openshift.ch:6443
```

The cluster operators are a good indicator whether the cluster is healhty or not:

```bash
oc get clusteroperators
```

Example output:

```
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE
authentication                             4.7.6     True        False         False      105m
baremetal                                  4.7.6     True        False         False      3d2h
cloud-credential                           4.7.6     True        False         False      3d3h
cluster-autoscaler                         4.7.6     True        False         False      3d2h
config-operator                            4.7.6     True        False         False      3d2h
console                                    4.7.6     True        False         False      113m
csi-snapshot-controller                    4.7.6     True        False         False      107m
dns                                        4.7.6     True        False         False      111m
etcd                                       4.7.6     True        False         False      3d2h
image-registry                             4.7.6     True        False         False      98m
ingress                                    4.7.6     True        False         False      3d2h
insights                                   4.7.6     True        False         False      3d2h
kube-apiserver                             4.7.6     True        False         False      3d2h
kube-controller-manager                    4.7.6     True        False         False      3d2h
kube-scheduler                             4.7.6     True        False         False      3d2h
kube-storage-version-migrator              4.7.6     True        False         False      106m
machine-api                                4.7.6     True        False         False      3d2h
machine-approver                           4.7.6     True        False         False      3d2h
machine-config                             4.7.6     True        False         False      103m
marketplace                                4.7.6     True        False         False      107m
monitoring                                 4.7.6     True        False         False      3h47m
network                                    4.7.6     True        False         False      3d2h
node-tuning                                4.7.6     True        False         False      146m
openshift-apiserver                        4.7.6     True        False         False      105m
openshift-controller-manager               4.7.6     True        False         False      3d2h
openshift-samples                          4.7.6     True        False         False      146m
operator-lifecycle-manager                 4.7.6     True        False         False      3d2h
operator-lifecycle-manager-catalog         4.7.6     True        False         False      3d2h
operator-lifecycle-manager-packageserver   4.7.6     True        False         False      108m
service-ca                                 4.7.6     True        False         False      3d2h
storage                                    4.7.6     True        False         False      107m
```
