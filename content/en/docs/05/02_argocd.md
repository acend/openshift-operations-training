---
title: "5.2 Argo CD"
weight: 52
sectionnumber: 5.2
---

## Task {{% param sectionnumber %}}.1: Argo CD

Go out into the new world of GitOps and explore Argo CD:

* Find out how to access Argo CD (there's a very handy web interface for it)
* Create a new Git repository on your Gitea instance

{{% alert title="Warning" color="secondary" %}}
If you want to add Secrets with sensitive data, make sure the repository is not public!
{{% /alert %}}

* Figure out how to create an application based on this Git repository
* Pick some of the resources you created in the course of this training, add them to your Git repository and apply them using Argo CD

{{% alert title="Note" color="primary" %}}
As [per the documentation](https://docs.openshift.com/container-platform/4.9/cicd/gitops/configuring_argo_cd_to_recursively_sync_a_git_repository_with_your_application/deploying-a-spring-boot-application-with-argo-cd.html) there's a namespace label `argocd.argoproj.io/managed-by=openshift-gitops` involved.
{{% /alert %}}
