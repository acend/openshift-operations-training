---
title: "3. Monitoring and logging"
weight: 30
sectionnumber: 3
---

OpenShift provides a cluster monitoring stack, which can be used for [monitor default platform components](https://docs.openshift.com/container-platform/4.7/monitoring/understanding-the-monitoring-stack.html) or [user-defined projects components](https://docs.openshift.com/container-platform/4.7/monitoring/enabling-monitoring-for-user-defined-projects.html).

The OpenShit provided [cluster logging](https://docs.openshift.com/container-platform/4.7/logging/cluster-logging.html) uses ElasticSearch to store the cluster logs, which then made be available to users using Kibana.

In this lab you are going to configure alerting for the cluster monitoring stack, enabling monitoring for user-defined projects and deploy the logging stack.
