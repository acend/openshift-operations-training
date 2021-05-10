---
title: "3. Monitoring and logging"
weight: 30
sectionnumber: 3
---

OpenShift provides a cluster monitoring stack which can be used to [monitor default platform components](https://docs.openshift.com/container-platform/latest/monitoring/understanding-the-monitoring-stack.html) or even [user-defined projects components](https://docs.openshift.com/container-platform/latest/monitoring/enabling-monitoring-for-user-defined-projects.html).

The OpenShit-provided [cluster logging](https://docs.openshift.com/container-platform/latest/logging/cluster-logging.html) uses ElasticSearch to store cluster and application logs, which can then be visualised using Kibana.

In this lab, you are going to configure alerting for the cluster monitoring stack, enable monitoring for user-defined projects and deploy the logging stack.
