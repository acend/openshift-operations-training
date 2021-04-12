---
title: "1.5 Uptime app"
weight: 15
sectionnumber: 1.5
---

In this lab you're going to deploy an application on your freshly-installed OpenShift cluster.

The application's availability will be monitored and recorded by an external monitoring system.
Your main goal for the rest of this training is to keep this application up and running, no matter what you do.


## Uptime app

The application you're going to install is a simple Python app. It's going to be scaled to 3 replicas in order to achieve high availability.


## Task {{% param sectionnumber %}}.1: Deploy the uptime app

First you need to create a new project named `uptime-app-prod` for the application:

```bash
oc new-project uptime-app-prod
```

Next, you're going to deploy the following resources:

* Deployment
* Service
* Route

To deploy these resources, you can simply execute the following command:

```bash
oc apply -f https://raw.githubusercontent.com/acend/openshift-4-ops-training/main/content/en/docs/01/uptime-app.yaml -n uptime-app-prod
```

The deployed application should now be available at <https://uptime-app-uptime-app-prod.trainings-cluster.puzzle.ch/>.

With 3 replicas and the [Pod anti-affinity](https://docs.openshift.com/container-platform/latest/nodes/scheduling/nodes-scheduler-pod-affinity.html#nodes-scheduler-pod-affinity-example-antiaffinity_nodes-scheduler-pod-affinity), which will make sure all 3 Pods are never deployed on the same underlying node, we have a good setup for the training to continue.
