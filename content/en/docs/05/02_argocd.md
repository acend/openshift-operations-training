---
title: "5.2 Argo CD"
weight: 52
sectionnumber: 5.2
---

## Task {{% param sectionnumber %}}.1: Argo CD

Go out into the new world of GitOps and explore Argo CD.

Find out how to access Argo CD's web interface (which is very handy).

{{% details title="Hints" mode-switcher="normalexpertmode" %}}
As usual, there are multiple ways to find out what Argo CD's dashboard URL is:

* OpenShift exposes the link via its web console:

![Argo CD link in OpenShift's web console](../argocd_ocpwebconsole.png)

* Or get it via route from the `openshift-gitops` namespace:

```bash
oc -n openshift-gitops get routes
```

{{% /details %}}


## Task {{% param sectionnumber %}}.2: Gitea repository

Create a new Git repository on your Gitea instance.

{{% alert title="Warning" color="warning" %}}
If you want to add Secrets with sensitive data, make sure the repository is not public!
{{% /alert %}}

{{% details title="Hints" mode-switcher="normalexpertmode" %}}
* On the upper right corner, click on the **+** icon and then choose **New Repository**:

![Repository creation in Gitea](../gitea_createrepo.png)

* On the **New Repository** page, choose a name such as "argocd-gitops-test"
* Choose the repo's visibility
* On the bottom of the page, click on the **Create Repository** button
{{% /details %}}


## Task {{% param sectionnumber %}}.3: Argo CD application

Figure out how to create an application based on the Git repository you just created.

{{% details title="Hints" mode-switcher="normalexpertmode" %}}
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
{{% /details %}}


## Task {{% param sectionnumber %}}.4: Resource management

* Choose some of the resources you created in the course of this training, e.g.:
  * The [`KubeletConfig` resources from task 1.2.1](../../01/02_configuration/#task-121-configure-kubelet-arguments)
  * The [`MachineSet` resource from task 2.1.1](../../02/01_infra-nodes/#task-211-create-a-machine-set)
  * The `IngressController` definition
  * The [custom Prometheus rule from task 3.3.2](../../03/03_user-defined-rules/#task-332-add-a-custom-prometheus-rule)
* Add these resource definitions to your Git repository

{{% alert title="Note" color="info" %}}
If you are unfamiliar with Git, check out [Git's tutorial introduction to Git](https://git-scm.com/docs/gittutorial) or ask your trainer.
{{% /alert %}}

{{% details title="Hints" mode-switcher="normalexpertmode" %}}
* Create a new directory for your repository files:

```bash
mkdir cluster-configs
cd cluster-configs
```

* Initialize it for usage with Git:

```bash
git init
git remote add origin <your git repository's URL>
git push -u origin main
```

* Taking the [KubeletConfig from task 1.2.1](https://raw.githubusercontent.com/acend/openshift-4-ops-training/main/content/en/docs/01/resources/kubeletconfig_master.yaml) as an example, save its definition as a new file inside the Git directory and push it to your Gitea repository:

```bash
wget https://raw.githubusercontent.com/acend/openshift-4-ops-training/main/content/en/docs/01/resources/kubeletconfig_master.yaml
git add kubeletconfig_master.yaml
git commit -m "Add master's kubelet configuration"
git push
```

* When visiting your Gitea web interface, you should now see the added file(s)
{{% /details %}}

* Apply your chosen resources using Argo CD
