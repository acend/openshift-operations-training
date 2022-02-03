---
title: "5.2 Argo CD"
weight: 52
sectionnumber: 5.2
---

## Task {{% param sectionnumber %}}.1: Argo CD

Go out into the new world of GitOps and explore Argo CD.
Find out how to access Argo CD (there's a very handy web interface for it).


### Solution {{% param sectionnumber %}}.1: Argo CD



## Task {{% param sectionnumber %}}.2: Gitea repository

* Create a new Git repository on your Gitea instance

{{% alert title="Warning" color="secondary" %}}
If you want to add Secrets with sensitive data, make sure the repository is not public!
{{% /alert %}}


### Solution {{% param sectionnumber %}}.2: Gitea repository



## Task {{% param sectionnumber %}}.3: Argo CD application

Figure out how to create an application based on the Git repository you just created.


### Solution {{% param sectionnumber %}}.3: Argo CD application

First, make sure you are in the **Applications** view, then click on **+ NEW APP**:

![Argo CD app creation](../argocd_appcreation.png)

Now fill in the appropriate information:

* **GENERAL**
  * **Application Name**: anything of your choosing
  * **Project**: `default`
  * Leave the rest as-is
* **SOURCE**
  * **Repository URL**: The URL to your Git repository
  * **Path**: Indicate in what folder the resource files reside; leave empty if they are in the root folder
  * Leave the rest as-is
* **DESTINATION**
  * **Cluster URL**: `https://kubernetes.default.svc`
  * **Namespace**: Leave empty


## Task {{% param sectionnumber %}}.4: Resource management


* Pick some of the resources you created in the course of this training, add them to your Git repository and apply them using Argo CD

{{% alert title="Note" color="primary" %}}
As [per the documentation](https://docs.openshift.com/container-platform/4.9/cicd/gitops/configuring_argo_cd_to_recursively_sync_a_git_repository_with_your_application/deploying-a-spring-boot-application-with-argo-cd.html) there's a namespace label `argocd.argoproj.io/managed-by=openshift-gitops` involved.
{{% /alert %}}


### Solution {{% param sectionnumber %}}.4: Resource management
