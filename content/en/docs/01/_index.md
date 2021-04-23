---
title: "1. Installation and configuration"
weight: 10
sectionnumber: 1
---

Installing the predecessor, OpenShift 3, meant that you had to configure everything in advance using an Ansible inventory.
Nearly every aspect of the cluster had to be configured with variables, most could be changed afterwards, but not all.
And some did not work well with others, leading to a hard-to-do installation when you had no experience or a complex setup.

This changed in an essential way with OpenShift 4.
Instead of configuring everything in advance, you start with the installation of a basic cluster.
The cluster can then be configured using Kubernetes resources, mostly custom resources, sometimes configmaps or secrets.

In this first lab section of the OpenShift 4 Operations training, you are going to install your own OpenShift 4 cluster, configure it and add some additional, important components.
