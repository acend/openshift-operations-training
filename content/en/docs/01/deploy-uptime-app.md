---
title: "1.5 Uptime-App"
weight: 15
sectionnumber: 1.5
---


In this Lab you're going to install an application on your freshly installed OpenShift Cluster.


## Uptime-App

The application you're going to install is a simple python app. It's going to be scaled to 3 high available replicas.


## Task {{% param sectionnumber %}}.1: Deploy Uptime-App

First we need to create a new project (`uptime-app-prod`) for our Uptime-App.

```bash
oc new-project uptime-app-prod
```

Next you're going to deploy the following resources:

* Deployment
* Service
* Route

To deploy the resources, you can simply execute the following command:

```bash
oc apply -f https://raw.githubusercontent.com/acend/openshift-4-ops-training/main/content/en/docs/01/uptime-app.yaml -n uptime-app-prod
```

The deployed application is now available under <https://uptime-app-uptime-app-prod.trainings-cluster.puzzle.ch/>.

With 3 replicas and the [pod Anti-affinity](https://docs.openshift.com/container-platform/4.7/nodes/scheduling/nodes-scheduler-pod-affinity.html#nodes-scheduler-pod-affinity-example-antiaffinity_nodes-scheduler-pod-affinity), which will make sure all 3 Pods are never deployed on the same underlying node, we have a good enough high available setup for the training to continue.


## Uptime-App Main Goal

From now on the availability of your [Uptime-App deployment](https://uptime-app-uptime-app-prod.trainings-cluster.puzzle.ch/) will be monitored and recorded by an external monitoring system. You're main goal for the up coming labs is, to keep the Uptime-App running and responsive for the rest of the training.
