---
title: "4.5 Chaos monkey"
weight: 45
sectionnumber: 4.5
---

You might have already heard of a concept called chaos engineering, or probably the solution that made it prominent, [Netflix's Chaos Monkey](https://github.com/netflix/chaosmonkey).

Even though you don't always need a chaos monkey because you might already have a human one in your organization (e.g. acend has Tobi), this is what we are going to do in this lab.
Most notably, the solution we are going to use is not limited to only applications running _on top of_ Kubernetes, but can also be used to wreck chaos on the infrastructure level.

Let's see how your cluster fares and how quickly you're able to solve problems if they persist.

The tool we are going to use is [Litmus](https://litmuschaos.io/), from the project's description, Litmus is:

> an open source Chaos Engineering platform that enables teams to identify weaknesses & potential outages in infrastructures by inducing chaos tests in a controlled way.

Sounds fun!
Let's install it.


## Task {{% param sectionnumber %}}.1: Preparations

Create a new project:

```bash
oc new-project litmus
```

Add Litmus' helm repository:

```bash
helm repo add litmuschaos https://litmuschaos.github.io/litmus-helm/
```

Get the chart's values file:

```bash
wget https://raw.githubusercontent.com/litmuschaos/litmus-helm/master/charts/litmus-2-0-0-beta/values.yaml
```


## Task {{% param sectionnumber %}}.2: Installation

Before installing the helm chart, we need to configure it.
Create a file named values.yaml with the following content or [download it from here](https://raw.githubusercontent.com/acend/openshift-4-ops-training/main/content/en/docs/04/resources/values_litmus.yaml):

{{< highlight yaml >}}{{< readfile file="content/en/docs/04/resources/values_litmus.yaml" >}}{{< /highlight >}}

{{% alert title="Warning" color="secondary" %}}
Note the two environment variables under `portal.server.authServer.env`.
These are the initial credentials for logging in.
As soon as the pods are deployed and running, you will need to log in and change these values.
Or change them right now in the `values.yaml` file.
{{% /alert %}}

Install Litmus:

```bash
helm install chaos litmuschaos/litmus-2-0-0-beta --namespace=litmus --devel --values values.yaml
```

The output should look similar to this:

```
NAME: chaos
LAST DEPLOYED: Fri Aug 6 15:20:09 2021
NAMESPACE: litmus
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Thank you for installing litmus-2-0-0-beta 😀

Your release is named chaos and its installed to namespace: litmus.

Visit https://docs.litmuschaos.io/docs/getstarted/ to find more info.
```

Find out the URL at which Litmus was exposed:

```bash
oc --namespace litmus get route chaos-litmus-2-0-0-beta-route -o go-template='https://{{ .spec.host }}{{ "\n" }}'
```

Open the URL in your browser and log in using the credentials you can find in the `values.yaml` file.
Immediately after logging in you'll be prompted to change the password.
If you used the password provided in the `values.yaml` file, absolutely change it to something else.


## Evictions the Litmus way

Remember the very manual test we did back in [lab 4.2](../../04/02_evictions/) artificially using up all memory on a node in order to provoke eviction of all pods running on that one node?
Using Litmus, this looks quite different:

{{< youtube id="ECxlWgQ8F5w" title="Node-memory-hog chaos experiment" >}}

So what did you see just now?

1. Up until FIXME, the video shows the Litmus' installation which you already did
1. Then, the [node-memory-hog experiment](https://hub.litmuschaos.io/api/chaos/2.0.0-RC1?file=charts/generic/node-memory-hog/experiment.yaml) from Litmus' Chaos Hub gets instantiated
1. From FIXME to FIXME, you can see in the lower left terminal corner that the node's memory goes up to about FIXME % usage
1. This lasts for 2 minutes and then stops automatically

