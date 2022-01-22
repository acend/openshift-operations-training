---
title: "5.1 Setup"
weight: 51
sectionnumber: 5.1
---

In order to explore GitOps processes, we are going to need appropriate tools: Gitea and ArgoCD.


## Task {{% param sectionnumber %}}.1: Gitea

Gitea is a lightweight Git server written in Go.
It allows us to easily host a Git repository which we can use as our single source of truth.

Unfortunately, the default Operator catalog sources don't contain a Gitea operator, but let's change that.

Add the following catalog source to your cluster:

```bash
oc apply -f https://raw.githubusercontent.com/redhat-gpte-devopsautomation/gitea-operator/master/catalog_source.yaml
```

You can now find and install the Gitea Operator from the OperatorHub:

* In the **Administrator** view of your web console, navigate to **Operators**, then **OperatorHub**
* Filter for "gitea"

{{% alert title="Note" color="primary" %}}
If the Gitea Operator does not appear yet, wait a few more seconds.
{{% /alert %}}

* Choose the Operator and click **Install**
* Don't change any of the pre-chosen values and again click **Install**
* Wait for the Operator to finish its installation successfully

This was just the Operator part.
We now have to create a Gitea instance.

First, create a new project:

```bash
oc new-project gitea
```

The instance you are going to create has the following content:

{{< highlight yaml >}}{{< readfile file="content/en/docs/05/resources/gitea_gitea.yaml" >}}{{< /highlight >}}

Create the instance with:

```bash
oc apply -f https://raw.githubusercontent.com/acend/openshift-4-ops-training/main/content/en/docs/05/resources/gitea_gitea.yaml
```

That's it!
Watch the pods start:

```bash
oc -n gitea get pods -w
```

As soon as the postgresql as well as the gitea pods are running, get the hostname from the route:

```bash
oc -n gitea get route gitea -o go-template='https://{{ .spec.host }}{{ "\n" }}'
```

Open the URL in your browser and register yourself a user.
Note the username and password you chose, you will need them later.


## Task {{% param sectionnumber %}}.2: Argo CD

We are going to install Argo CD in the form of the OpenShift GitOps Operator:

* Head over to the **OperatorHub** on your cluster, filter for "gitops" and choose **Red Hat OpenShift GitOps**
* Click **Install**
* Leave the pre-filled values as-is and again click **Install**

This installs a nearly ready-to-use Argo CD instance.
You can see that when looking into the `openshift-gitops` namespace:

* An `argocd` custom resource named `openshift-gitops` was created
* This in turn led the Operator to create all the pods and routes you can see

What we need to do to make it fully operational is slightly adjust certain parameters in its custom resources configuration:

* Add tolerations and node selectors to make all pods run on infra nodes
* Change the route's termination to reencrypt


### Solution {{% param sectionnumber %}}.2: Argo CD

Apply the following resources.

* The Argo CD custom resource to change the route termination and add the node placement definition:

{{< highlight yaml >}}{{< readfile file="content/en/docs/05/resources/argocd_openshift-gitops.yaml" >}}{{< /highlight >}}

* The GitOpsService custom resource to move the Operator itself onto infra nodes as well:

{{< highlight yaml >}}{{< readfile file="content/en/docs/05/resources/gitopsservice_cluster.yaml" >}}{{< /highlight >}}