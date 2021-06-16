---
title: "1.4 Backup"
weight: 14
sectionnumber: 1.4
---

In this lab, you will create scheduled backups for the most important cluster components.


## Task {{% param sectionnumber %}}.1: Create user workload backups

As a cluster admin, you can create scheduled backups of your users' resources. This allows you to restore them in cases like an accidental deletion.

{{% alert title="Note" color="primary" %}}
You will be using Velero to create user workload backups.
{{% /alert %}}

Setup a daily backup of the resources in namespace `uptime-app-prod` with a lifetime of 10 days by creating the following schedule:

{{< highlight yaml >}}{{< readfile file="content/en/docs/01/resources/schedule_daily-backup-uptime-app-prod.yaml" >}}{{< /highlight >}}

{{% alert title="Note" color="primary" %}}
This resource file is also available at https://raw.githubusercontent.com/acend/openshift-4-ops-training/main/content/en/docs/01/resources/schedule_daily-backup-uptime-app-prod.yaml.
{{% /alert %}}

```bash
oc -n training-infra-velero apply -f https://raw.githubusercontent.com/acend/openshift-4-ops-training/main/content/en/docs/01/resources/schedule_daily-backup-uptime-app-prod.yaml
```

The schedule will create the first backup immediately. Have a look at the status part of the backup:

```bash
oc -n training-infra-velero describe backup | grep -A 9 Status
```

The output should look similar to this:

```
Status:
  Completion Timestamp:  2021-04-14T09:31:11Z
  Expiration:            2021-04-24T09:30:50Z
  Format Version:        1.1.0
  Phase:                 Completed
  Progress:
    Items Backed Up:  31
    Total Items:      31
  Start Timestamp:    2021-04-14T09:30:50Z
  Version:            1
```


## Task {{% param sectionnumber %}}.2 Create etcd backup

To be able to restore the cluster in case of a disaster, you will create a scheduled `etcd` backup.

Create a new project to be used for the backup:

```bash
oc new-project training-infra-etcd-backup
```

Since `etcd` is running on the masters, we need to make sure the cronjob pods creating the `etcd` snapshots are running on the master as well. We can do this by adding the following annotation:

```bash
oc patch namespace training-infra-etcd-backup -p \
    '{"metadata":{"annotations":{"openshift.io/node-selector": "node-role.kubernetes.io/master="}}}'
```

For the cronjob pod to be able to access the `etcd` files, we need to create a service account with additional permissions by attaching a custom `SecurityContextConstraints` (SCC) policy:

* Service account:

{{< highlight yaml >}}{{< readfile file="content/en/docs/01/resources/etcd-backup/sa_etcd-backup.yaml" >}}{{< /highlight >}}

* SCC:

{{< highlight yaml >}}{{< readfile file="content/en/docs/01/resources/etcd-backup/scc_privileged-etcd-backup.yaml" >}}{{< /highlight >}}

```bash
oc -n training-infra-etcd-backup apply -f https://raw.githubusercontent.com/acend/openshift-4-ops-training/main/content/en/docs/01/resources/etcd-backup/sa_etcd-backup.yaml
oc -n training-infra-etcd-backup apply -f https://raw.githubusercontent.com/acend/openshift-4-ops-training/main/content/en/docs/01/resources/etcd-backup/scc_privileged-etcd-backup.yaml
```

Now we can create the secret containing the AWS credentials and S3 bucket configuration:

```bash
oc -n training-infra-etcd-backup apply -f /home/ec2-user/ocp4-ops/resources/etcd-backup/secret_etdc-backup-s3-bucket.yaml
```

Finally we can create the `ConfigMap` containing the backup script and the `CronJob` resources. Here's what they look like:

{{< highlight yaml >}}{{< readfile file="content/en/docs/01/resources/etcd-backup/cm_backup-script.yaml" >}}{{< /highlight >}}

{{< highlight yaml >}}{{< readfile file="content/en/docs/01/resources/etcd-backup/cronjob_etcd-backup.yaml" >}}{{< /highlight >}}

Create them with:

```bash
oc -n training-infra-etcd-backup apply -f https://raw.githubusercontent.com/acend/openshift-4-ops-training/main/content/en/docs/01/resources/etcd-backup/cm_backup-script.yaml
oc -n training-infra-etcd-backup apply -f https://raw.githubusercontent.com/acend/openshift-4-ops-training/main/content/en/docs/01/resources/etcd-backup/cronjob_etcd-backup.yaml
```

{{% alert title="Note" color="primary" %}}
In our example of the `CronJob` the backup job is started every six hours. If you do not want to wait that long, change the schedule of the `CronJob` accordingly.
{{% /alert %}}

You can verify the backup job by looking at the result or by checking the logs of the pod:

* Job status:

```bash
oc -n training-infra-etcd-backup get jobs
```

* Check the logs:

```bash
oc -n training-infra-etcd-backup logs jobs/<job-name>
```

The logs should not contain any errors and should show the successful upload of the backup to the S3 bucket:

```
ac3805eda3a0200224fbe10d8eabbd429bcd6fb12ddd7a575b83bc6f68fb49b7
etcdctl version: 3.4.9
API version: 3.4
found latest kube-apiserver: /etc/kubernetes/static-pod-resources/kube-apiserver-pod-12
found latest kube-controller-manager: /etc/kubernetes/static-pod-resources/kube-controller-manager-pod-12
found latest kube-scheduler: /etc/kubernetes/static-pod-resources/kube-scheduler-pod-12
found latest etcd: /etc/kubernetes/static-pod-resources/etcd-pod-3
{"level":"info","ts":1618472588.6842906,"caller":"snapshot/v3_snapshot.go:119","msg":"created temporary db file","path":"/home/core/assets/snapshot_2021-04-15_074306.db.part"}
{"level":"info","ts":"2021-04-15T07:43:08.691Z","caller":"clientv3/maintenance.go:200","msg":"opened snapshot stream; downloading"}
{"level":"info","ts":1618472588.6915584,"caller":"snapshot/v3_snapshot.go:127","msg":"fetching snapshot","endpoint":"https://10.0.184.171:2379"}
{"level":"info","ts":"2021-04-15T07:43:10.032Z","caller":"clientv3/maintenance.go:208","msg":"completed snapshot read; closing"}
{"level":"info","ts":1618472590.2327292,"caller":"snapshot/v3_snapshot.go:142","msg":"fetched snapshot","endpoint":"https://10.0.184.171:2379","size":"112 MB","took":1.548386092}
{"level":"info","ts":1618472590.232812,"caller":"snapshot/v3_snapshot.go:152","msg":"saved","path":"/home/core/assets/snapshot_2021-04-15_074306.db"}
Snapshot saved at /home/core/assets/snapshot_2021-04-15_074306.db
snapshot db and kube resources are successfully saved to /home/core/assets
upload: ../host/home/core/assets/snapshot_2021-04-15_071919.db to s3://user01-ops-training-backup/etcd-backup/snapshot_2021-04-15_071919.db
upload: ../host/home/core/assets/snapshot_2021-04-15_071932.db to s3://user01-ops-training-backup/etcd-backup/snapshot_2021-04-15_071932.db
upload: ../host/home/core/assets/snapshot_2021-04-15_072307.db to s3://user01-ops-training-backup/etcd-backup/snapshot_2021-04-15_072307.db
upload: ../host/home/core/assets/snapshot_2021-04-15_072026.db to s3://user01-ops-training-backup/etcd-backup/snapshot_2021-04-15_072026.db
upload: ../host/home/core/assets/snapshot_2021-04-15_072128.db to s3://user01-ops-training-backup/etcd-backup/snapshot_2021-04-15_072128.db
upload: ../host/home/core/assets/snapshot_2021-04-15_071953.db to s3://user01-ops-training-backup/etcd-backup/snapshot_2021-04-15_071953.db
upload: ../host/home/core/assets/snapshot_2021-04-15_072559.db to s3://user01-ops-training-backup/etcd-backup/snapshot_2021-04-15_072559.db
upload: ../host/home/core/assets/snapshot_2021-04-15_073934.db to s3://user01-ops-training-backup/etcd-backup/snapshot_2021-04-15_073934.db
upload: ../host/home/core/assets/static_kuberesources_2021-04-15_071919.tar.gz to s3://user01-ops-training-backup/etcd-backup/static_kuberesources_2021-04-15_071919.tar.gz
upload: ../host/home/core/assets/static_kuberesources_2021-04-15_072026.tar.gz to s3://user01-ops-training-backup/etcd-backup/static_kuberesources_2021-04-15_072026.tar.gz
upload: ../host/home/core/assets/static_kuberesources_2021-04-15_071932.tar.gz to s3://user01-ops-training-backup/etcd-backup/static_kuberesources_2021-04-15_071932.tar.gz
upload: ../host/home/core/assets/snapshot_2021-04-15_074009.db to s3://user01-ops-training-backup/etcd-backup/snapshot_2021-04-15_074009.db
upload: ../host/home/core/assets/static_kuberesources_2021-04-15_072128.tar.gz to s3://user01-ops-training-backup/etcd-backup/static_kuberesources_2021-04-15_072128.tar.gz
upload: ../host/home/core/assets/static_kuberesources_2021-04-15_072307.tar.gz to s3://user01-ops-training-backup/etcd-backup/static_kuberesources_2021-04-15_072307.tar.gz
upload: ../host/home/core/assets/static_kuberesources_2021-04-15_072559.tar.gz to s3://user01-ops-training-backup/etcd-backup/static_kuberesources_2021-04-15_072559.tar.gz
upload: ../host/home/core/assets/static_kuberesources_2021-04-15_073913.tar.gz to s3://user01-ops-training-backup/etcd-backup/static_kuberesources_2021-04-15_073913.tar.gz
upload: ../host/home/core/assets/static_kuberesources_2021-04-15_071953.tar.gz to s3://user01-ops-training-backup/etcd-backup/static_kuberesources_2021-04-15_071953.tar.gz
upload: ../host/home/core/assets/static_kuberesources_2021-04-15_073905.tar.gz to s3://user01-ops-training-backup/etcd-backup/static_kuberesources_2021-04-15_073905.tar.gz
upload: ../host/home/core/assets/static_kuberesources_2021-04-15_074009.tar.gz to s3://user01-ops-training-backup/etcd-backup/static_kuberesources_2021-04-15_074009.tar.gz
upload: ../host/home/core/assets/snapshot_2021-04-15_073905.db to s3://user01-ops-training-backup/etcd-backup/snapshot_2021-04-15_073905.db
upload: ../host/home/core/assets/static_kuberesources_2021-04-15_074105.tar.gz to s3://user01-ops-training-backup/etcd-backup/static_kuberesources_2021-04-15_074105.tar.gz
upload: ../host/home/core/assets/static_kuberesources_2021-04-15_074306.tar.gz to s3://user01-ops-training-backup/etcd-backup/static_kuberesources_2021-04-15_074306.tar.gz
upload: ../host/home/core/assets/static_kuberesources_2021-04-15_073934.tar.gz to s3://user01-ops-training-backup/etcd-backup/static_kuberesources_2021-04-15_073934.tar.gz
upload: ../host/home/core/assets/snapshot_2021-04-15_073913.db to s3://user01-ops-training-backup/etcd-backup/snapshot_2021-04-15_073913.db
upload: ../host/home/core/assets/snapshot_2021-04-15_074105.db to s3://user01-ops-training-backup/etcd-backup/snapshot_2021-04-15_074105.db
upload: ../host/home/core/assets/snapshot_2021-04-15_074306.db to s3://user01-ops-training-backup/etcd-backup/snapshot_2021-04-15_074306.db
```

