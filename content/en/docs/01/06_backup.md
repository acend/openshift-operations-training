---
title: "1.6 Backup"
weight: 16
sectionnumber: 1.6
---

In this lab, you will create scheduled backups for the most important cluster components.


## Task {{% param sectionnumber %}}.1: Create user workload backups

First, check the `backupStorageLocation`'s health and name:

```bash
oc -n openshift-adp get backupstoragelocations.velero.io 
```

The output should show you that you've got `backupStorageLocation` resource named `default` with `PHASE` `Available`. If the `PHASE` shows something else, there's most probably a permissions or typo problem in your `dataProtectionApplication` resource.

```yaml
NAME      PHASE       LAST VALIDATED   AGE   DEFAULT
default   Available   4s               3m    true
```

Now, create a daily backup of the resources in namespace `uptime-app-prod` with a lifetime of 5 days by creating the following `Schedule` manifest:

{{< readfile file="/content/en/docs/01/resources/schedule_daily-backup-uptime-app-prod.yaml" code="true" lang="yaml" >}}

{{% alert title="Note" color="primary" %}}
This resource file is also available at https://raw.githubusercontent.com/acend/openshift-operations-training/main/content/en/docs/01/resources/schedule_daily-backup-uptime-app-prod.yaml.
{{% /alert %}}

```bash
oc apply -f https://raw.githubusercontent.com/acend/openshift-operations-training/main/content/en/docs/01/resources/schedule_daily-backup-uptime-app-prod.yaml
```

In order to test it, let's trigger a manual backup. We are using the `velero` cli for this which is pre-installed on your bastion host:

```bash
velero -n openshift-adp backup create --from-schedule daily-backup-uptime-app-prod
```

When the backup is completed, have a look at its status part:

```bash
oc -n openshift-adp describe backup | grep -A 9 Status
```

The output should look similar to this:

```
Status:
  Completion Timestamp:  2023-04-18T10:04:32Z
  Expiration:            2023-04-28T10:03:59Z
  Format Version:        1.1.0
  Phase:                 Completed
  Progress:
    Items Backed Up:  55
    Total Items:      55
  Start Timestamp:    2023-04-18T10:03:59Z
  Version:            1
```


## Task {{% param sectionnumber %}}.2 Create etcd backup

To be able to restore the cluster in case of a disaster, you will create a scheduled `etcd` backup.

Create a new project named `training-infra-etcd-backup`.

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

```bash
oc new-project training-infra-etcd-backup
```

{{% /details %}}

Since `etcd` is running on the control plane nodes, we need to make sure the cronjob's pods that are creating the `etcd` snapshots are running on one as well. We can do this by annotating the namespace with the corresponding node selector:

```bash
oc patch namespace training-infra-etcd-backup -p \
    '{"metadata":{"annotations":{"openshift.io/node-selector": "node-role.kubernetes.io/master="}}}'
```

For the cronjob pod to be able to access the `etcd` files, we need to create a service account with additional permissions by attaching a custom `SecurityContextConstraints` (SCC) policy:

* Service account:

{{< readfile file="/content/en/docs/01/resources/etcd-backup/sa_etcd-backup.yaml" code="true" lang="yaml" >}}

* SCC:

{{< readfile file="/content/en/docs/01/resources/etcd-backup/scc_privileged-etcd-backup.yaml" code="true" lang="yaml" >}}

```bash
oc -n training-infra-etcd-backup apply -f https://raw.githubusercontent.com/acend/openshift-operations-training/main/content/en/docs/01/resources/etcd-backup/sa_etcd-backup.yaml
oc -n training-infra-etcd-backup apply -f https://raw.githubusercontent.com/acend/openshift-operations-training/main/content/en/docs/01/resources/etcd-backup/scc_privileged-etcd-backup.yaml
```

Edit the file `~/ocp4-ops/resources/etcd-backup/secret_etcd-backup-s3-bucket.yaml` and add your S3 bucket name for etcd backups:

```yaml
[...]
AWS_S3_BUCKET: +username+-ops-training-backup-etcd
type: Opaque
```

Now we can create the secret:

```bash
oc -n training-infra-etcd-backup apply -f ~/ocp4-ops/resources/etcd-backup/secret_etcd-backup-s3-bucket.yaml
```

Finally we can create the `ConfigMap` containing the backup script and the `CronJob` resources. Here's what they look like:

{{< readfile file="/content/en/docs/01/resources/etcd-backup/cm_backup-script.yaml" code="true" lang="yaml" >}}

{{< readfile file="/content/en/docs/01/resources/etcd-backup/cronjob_etcd-backup.yaml" code="true" lang="yaml" >}}

Create them with:

```bash
oc -n training-infra-etcd-backup apply -f https://raw.githubusercontent.com/acend/openshift-operations-training/main/content/en/docs/01/resources/etcd-backup/cm_backup-script.yaml
oc -n training-infra-etcd-backup apply -f https://raw.githubusercontent.com/acend/openshift-operations-training/main/content/en/docs/01/resources/etcd-backup/cronjob_etcd-backup.yaml
```

{{% alert title="Note" color="primary" %}}
In our example of the `CronJob` the backup job is started every six hours. If you do not want to wait that long, trigger the job manually with\
`oc create job --from=cronjob/etcd-backup -n training-infra-etcd-backup etcd-test-job`
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
Certificate /etc/kubernetes/static-pod-certs/configmaps/etcd-serving-ca/ca-bundle.crt is missing. Checking in different directory
Certificate /etc/kubernetes/static-pod-resources/etcd-certs/configmaps/etcd-serving-ca/ca-bundle.crt found!
found latest kube-apiserver: /etc/kubernetes/static-pod-resources/kube-apiserver-pod-13
found latest kube-controller-manager: /etc/kubernetes/static-pod-resources/kube-controller-manager-pod-8
found latest kube-scheduler: /etc/kubernetes/static-pod-resources/kube-scheduler-pod-8
found latest etcd: /etc/kubernetes/static-pod-resources/etcd-pod-7
5c933a1f3ca2a665cbbf37f5155359592eede1e9cdb4f25cebd10cdc9dad7d0b
etcdctl version: 3.5.11
API version: 3.5
{"level":"info","ts":"2024-02-18T12:02:53.735252Z","caller":"snapshot/v3_snapshot.go:65","msg":"created temporary db file","path":"/home/core/assets/snapshot_2024-02-18_120252.db.part"}
{"level":"info","ts":"2024-02-18T12:02:53.741068Z","logger":"client","caller":"v3@v3.5.11/maintenance.go:212","msg":"opened snapshot stream; downloading"}
{"level":"info","ts":"2024-02-18T12:02:53.741097Z","caller":"snapshot/v3_snapshot.go:73","msg":"fetching snapshot","endpoint":"https://10.0.89.148:2379"}
{"level":"info","ts":"2024-02-18T12:02:54.940107Z","logger":"client","caller":"v3@v3.5.11/maintenance.go:220","msg":"completed snapshot read; closing"}
{"level":"info","ts":"2024-02-18T12:02:55.569259Z","caller":"snapshot/v3_snapshot.go:88","msg":"fetched snapshot","endpoint":"https://10.0.89.148:2379","size":"113 MB","took":"1 second ago"}
{"level":"info","ts":"2024-02-18T12:02:55.569335Z","caller":"snapshot/v3_snapshot.go:97","msg":"saved","path":"/home/core/assets/snapshot_2024-02-18_120252.db"}
Snapshot saved at /home/core/assets/snapshot_2024-02-18_120252.db
{"hash":2933310916,"revision":480301,"totalKey":10630,"totalSize":112947200}
snapshot db and kube resources are successfully saved to /home/core/assets
upload: ../host/home/core/assets/snapshot_2024-02-18_120022.db to s3://<redacted-bucket-name>/etcd-backup/snapshot_2024-02-18_120022.db
upload: ../host/home/core/assets/static_kuberesources_2024-02-18_120015.tar.gz to s3://<redacted-bucket-name>/etcd-backup/static_kuberesources_2024-02-18_120015.tar.gz
upload: ../host/home/core/assets/static_kuberesources_2024-02-18_120022.tar.gz to s3://<redacted-bucket-name>/etcd-backup/static_kuberesources_2024-02-18_120022.tar.gz
upload: ../host/home/core/assets/static_kuberesources_2024-02-18_120038.tar.gz to s3://<redacted-bucket-name>/etcd-backup/static_kuberesources_2024-02-18_120038.tar.gz
upload: ../host/home/core/assets/snapshot_2024-02-18_120252.db to s3://<redacted-bucket-name>/etcd-backup/snapshot_2024-02-18_120252.db
upload: ../host/home/core/assets/static_kuberesources_2024-02-18_120109.tar.gz to s3://<redacted-bucket-name>/etcd-backup/static_kuberesources_2024-02-18_120109.tar.gz
upload: ../host/home/core/assets/snapshot_2024-02-18_120109.db to s3://<redacted-bucket-name>/etcd-backup/snapshot_2024-02-18_120109.db
upload: ../host/home/core/assets/static_kuberesources_2024-02-18_120252.tar.gz to s3://<redacted-bucket-name>/etcd-backup/static_kuberesources_2024-02-18_120252.tar.gz
upload: ../host/home/core/assets/static_kuberesources_2024-02-18_120155.tar.gz to s3://<redacted-bucket-name>/etcd-backup/static_kuberesources_2024-02-18_120155.tar.gz
upload: ../host/home/core/assets/snapshot_2024-02-18_120015.db to s3://<redacted-bucket-name>/etcd-backup/snapshot_2024-02-18_120015.db
upload: ../host/home/core/assets/snapshot_2024-02-18_120038.db to s3://<redacted-bucket-name>/etcd-backup/snapshot_2024-02-18_120038.db
upload: ../host/home/core/assets/snapshot_2024-02-18_120155.db to s3://<redacted-bucket-name>/etcd-backup/snapshot_2024-02-18_120155.db
```
