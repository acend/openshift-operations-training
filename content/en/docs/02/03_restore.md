---
title: "2.3 Restore"
weight: 23
sectionnumber: 2.3
---

## Task {{% param sectionnumber %}}.1: Restoring user workload backups

In the backup lab you created a scheduled backup of all resources in namespace `uptime-app-prod`. Now we are going to test the restore procedure.

Go ahead and delete the namespace `uptime-app-prod`:

```bash
oc delete ns uptime-app-prod
```

Update your backup storage location to read-only mode (this prevents backup objects from being created or deleted in the backup storage location during the restore process):

```bash
oc patch backupstoragelocation default \
   --namespace training-infra-velero \
   --type merge \
   --patch '{"spec":{"accessMode":"ReadOnly"}}'
```

In order to restore a backup, we need to create a `Restore` object. A `Restore` object looks like this:

{{< highlight yaml >}}{{< readfile file="content/en/docs/02/resources/restore_uptime-app-prod.yaml" >}}{{< /highlight >}}

Copy above `Restore` resource into a file on your bastion host. Replace the placeholder `<backup-name>` with the name of the backup you want to restore. To list your available backups:

```bash
oc -n training-infra-velero get backups

After the restore has completed, your uptime app should be up and running again:

```bash
$ oc -n uptime-app-prod get pods
NAME                          READY   STATUS    RESTARTS   AGE
uptime-app-56df4df7d8-hnsps   1/1     Running   0          20m
uptime-app-56df4df7d8-jfkzx   1/1     Running   0          20m
uptime-app-56df4df7d8-mx6bt   1/1     Running   0          20m
```

When ready, revert your backup storage location to read-write mode:

```bash
oc patch backupstoragelocation default \
   --namespace training-infra-velero \
   --type merge \
   --patch '{"spec":{"accessMode":"ReadWrite"}}'
```
