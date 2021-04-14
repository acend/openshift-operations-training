---
title: "1.4 Backup"
weight: 14
sectionnumber: 1.4
---

In this lab, you will create scheduled backups for the most important cluster components.


## Task {{% param sectionnumber %}}.1: Create user workload backups

As a cluster admin, you can create scheduled backups of your users resources, to be able to restore them in case of emergency, like an accidental deletion.

Setup a daily backup of the resources in namespace `uptime-app-prod` with a lifetime of 10 days: #FIXME: Hugo var

```bash
velero create schedule uptime-app-prod-daily-backup \
  --namespace training-infra-velero \
  --schedule "@every 24h" \
  --ttl 240h0m0s \
  --include-namespaces uptime-app-prod
#FIXME: Hugo var
```

The schedule will create the first backup immeditately:

```bash
# check the schedule
$ velero -n training-infra-velero get schedule
NAME                           STATUS    CREATED                         SCHEDULE     BACKUP TTL   LAST BACKUP   SELECTOR
daily-backup-uptime-app-prod   Enabled   2021-04-14 09:30:50 +0000 UTC   @every 24h   240h0m0s     4m ago        <none>
#
# check the backup 
$ velero -n training-infra-velero get backup
NAME                                          STATUS      ERRORS   WARNINGS   CREATED                         EXPIRES   STORAGE LOCATION   SELECTOR
daily-backup-uptime-app-prod-20210414093050   Completed   0        0          2021-04-14 09:30:50 +0000 UTC   9d        default            <none>
```

{{% alert title="Note" color="primary" %}
For get the details  of the schedule and the backups, run the following commands:

```bash
# details of the schedule
velero -n training-infra-velero describe schedule <schedule-name>
#
# details of the backup
velero -n training-infra-velero describe backup <backup-name>
```
{{% /alert %}}


## Task {{% param sectionnumber %}}.2 Create etcd backup


