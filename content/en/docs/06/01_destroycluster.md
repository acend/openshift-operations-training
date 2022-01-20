---
title: "6.1 Destroy the cluster"
weight: 61
sectionnumber: 6.1
---

Usually our work centers around keeping everything up and running, building and engineering stuff and so on.
This is not always easy, so there's some extra joy when it comes to finally be allowed to do the opposite and destroy something :)


## Task {{% param sectionnumber %}}.1: Destroy your cluster

Change into the installation directory you created on day 1, it should be under `/home/ec2-user/ocp4-ops` and have a timestamp as its name.

Destroy the cluster:

```bash
openshift-install destroy cluster
```

Above command removes all created resources on AWS including VM instances, load balancers etc.

The really only thing left to do now is: Please inform your trainer that you destroyed your cluster.

Thank you!
