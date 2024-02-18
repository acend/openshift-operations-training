---
title: "2.3 Restore"
weight: 23
sectionnumber: 2.3
---

## Task {{% param sectionnumber %}}.1: Restoring user workload backups

In the backup lab you created a scheduled backup of all resources in namespace `uptime-app-prod`. Now we are going to test the restore procedure.

{{% alert title="Warning" color="secondary" %}}
To keep your restore time as short as possible, make sure you read through the whole procedure, understand it and maybe even prepare all resources for the restore before deleting the project!
{{% /alert %}}

Make sure you read above warning, then go ahead and delete the project `uptime-app-prod`.

{{% details title="Hints" mode-switcher="normalexpertmode" %}}

```bash
oc delete project uptime-app-prod
```

{{% /details %}}

Update your backup storage location to read-only mode (this prevents backup objects from being created or deleted in the backup storage location during the restore process):

```bash
oc patch backupstoragelocation default \
   --namespace openshift-adp \
   --type merge \
   --patch '{"spec":{"accessMode":"ReadOnly"}}'
```

In order to restore a backup, we need to create a `Restore` object. A `Restore` object looks like this:

{{< readfile file="/content/en/docs/02/resources/restore_uptime-app-prod.yaml" code="true" lang="yaml" >}}

Copy above `Restore` resource into a file on your bastion host. Replace the placeholder `<backup-name>` with the name of the backup you want to restore. To list your available backups:

```bash
oc -n openshift-adp get backups
```

Start the restore by creating the `Restore` object:

```bash
oc apply -f <restore-file>
```

After the restore has completed, your uptime app should be up and running again:

```bash
oc -n uptime-app-prod get pods
```

Example output:

```
NAME                          READY   STATUS    RESTARTS   AGE
uptime-app-56df4df7d8-hnsps   1/1     Running   0          20m
uptime-app-56df4df7d8-jfkzx   1/1     Running   0          20m
```

When ready, revert your backup storage location to read-write mode:

```bash
oc patch backupstoragelocation default \
   --namespace openshift-adp \
   --type merge \
   --patch '{"spec":{"accessMode":"ReadWrite"}}'
```
