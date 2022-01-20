---
title: "5. GitOps"
weight: 50
sectionnumber: 5
---

This chapter explores how to use GitOps processes and tools.
GitOps is a way to do Kubernetes cluster management and application delivery.
It works by using Git as a single source of truth for declarative infrastructure and applications.
With GitOps, the use of software agents can alert on any divergence between Git and what's running in a cluster, and if there's a difference, automatically update or rollback the cluster depending on the case.

Argo CD is such a software agent, or GitOps tool.
It is a declarative, GitOps continuous delivery tool for Kubernetes.

Argo CD follows the GitOps pattern of using Git repositories as the source of truth for defining the desired application state.
Kubernetes manifests can be specified in several ways:

* kustomize applications
* helm charts
* ksonnet applications
* jsonnet files
* Plain directory of YAML/json manifests
* Any custom config management tool configured as a config management plugin

Argo CD automates the deployment of the desired application states in the specified target environments.
Application deployments can track updates to branches, tags, or pinned to a specific version of manifests at a Git commit.
See tracking strategies for additional details about the different tracking strategies available.

For a quick 10 minute overview of Argo CD, check out the demo presented to the Sig Apps community meeting:

{{< youtube aWDIQMbp1cc >}}
