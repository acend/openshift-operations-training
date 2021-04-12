---
title: "1.1 Installation"
weight: 11
sectionnumber: 1.1
---

In this lab, we are going to install an OpenShift 4 cluster on AWS.


## Task {{% param sectionnumber %}}.1: Installing OpenShift 4 on AWS

In this task we will prepare and carry out the installation.


### Customizing the installation

See the [OpenShift installation documentation](https://docs.openshift.com/container-platform/4.7/installing/installing_aws/installing-aws-customizations.html#installation-configuration-parameters_installing-aws-customizations) for a list of available parameters.

Edit the file `install-config.yaml`

```yaml
apiVersion: v1
baseDomain: ops.openshift.ch
compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  platform: {}
  replicas: 3 #FIXME: change to 2
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  platform: {}
  replicas: 3
metadata:
  creationTimestamp: null
  name: user01
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 10.0.0.0/16
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16
platform:
  aws:
    region: eu-north-1
    userTags:
      acend-training: ocp4-ops
publish: External
pullSecret: '{"auths":{...}}'
sshKey: 'ssh-ed25519 AAAA...'
```

Backup the file `install-config.yaml`, since it will be consumed during the installation process.

```bash
$ cp $(date +"%Y-%m-%d")/install-config.yaml ~/backup/install-config.yaml
```


### Deploying the cluster

Now we are ready to create our cluster:

```bash
$ ./openshift-install create cluster --dir=$(date +"%Y-%m-%d") --log-level=info
...
INFO Install complete!
INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/home/myuser/install_dir/auth/kubeconfig'
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.mycluster.example.com
INFO Login to the console with user: "kubeadmin", and password: "4vYBz-Ee6gm-ymBZj-Wt5AL"
INFO Time elapsed: 36m22s
```


### Logging in to the cluster by using the CLI

We will no check, whether we can log in to the cluster by exporting the `kubeadmin` credentials:

```bash
$ export KUBECONFIG=/home/ec2-user/ocp4-ops/$(date +"%Y-%m-%d")/auth/kubeconfig
$ oc whoami
system:admin
$ oc whoami --show-server
https://api.ops.openshift.ch
```

