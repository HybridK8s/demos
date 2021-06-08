# **Getting Started**

Welcome to HybridK8s Droid! 🎈

In this guide, we’ll walk you through how to install Droid's agent into your Kubernetes cluster. Then we’ll deploy a sample application to show off what it can do.

Installing Droid is easy. Just before we can do anything, we need to ensure you have access to modern Kubernetes cluster with a publicly exposed IP and a functioning `kubectl` command on your local machine.

You can validate your setup by running:

```bash
kubectl version --short
```

You should see output with both a `Client Version` and `Server Version` component.

## **Step 0: Account Creation**

To signup for HybridK8s Droid, create a HybridK8s account [here](https://hybridk8s.tech/signup). Once created, verify your account and you can use the username and password to sign in to HybridK8s platform. 

## **Step 1: Install Pre-requisites**

First, you will install the a service mesh, say, Linkerd onto your local machine. Using this CLI, you’ll then install the _control plane_ onto your Kubernetes cluster. Finally, we'll install flagger operator. 

Now that we have our cluster, we’ll install the Linkerd CLI and use it validate that your cluster is capable of hosting the Linkerd control plane.

(Note: if you’re using a GKE “private cluster”, there are some [extra steps required](https://linkerd.io/2.10/reference/cluster-configuration/#private-clusters) before you can proceed to the next step.)

If this is your first time running Linkerd, you will need to download the `linkerd` command-line interface (CLI) onto your local machine. The CLI will allow you to interact with your Linkerd deployment.

To install the CLI manually, run:

```bash
curl -sL https://run.linkerd.io/install | sh
```

**Be sure to follow the instructions to add it to your path, like export** `export PATH=$PATH:/Users/<`**`user`**`>/.linkerd2/bin`**.**

Alternatively, if you use [Homebrew](https://brew.sh/), you can install the CLI with `brew install linkerd`. You can also download the CLI directly via the [Linkerd releases page](https://github.com/linkerd/linkerd2/releases/).

Once installed, verify the CLI is running correctly with:

```bash
linkerd version
```

You should see the CLI version, and also `Server version: unavailable`. This is because you haven’t installed the control plane on your cluster. Don’t worry—we’ll fix that soon enough.

Flagger requires a Kubernetes cluster **v1.16** or newer and Linkerd **2.10** or newer.

Install Linkerd the Promethues (part of Linkerd Viz):

```shell
linkerd install | kubectl apply -f -
linkerd viz install | kubectl apply -f -
```

Install Flagger in the linkerd namespace:

```shell
kubectl apply -k github.com/fluxcd/flagger//kustomize/linkerd
```

## **Step 2: Bootstrap**

Flagger takes a Kubernetes deployment and optionally a horizontal pod autoscaler (HPA), then creates a series of objects (Kubernetes deployments, ClusterIP services and SMI traffic split). These objects expose the application inside the mesh and drive the canary analysis and promotion.

Create a test namespace and enable Linkerd proxy injection:

```shell
kubectl create ns test
kubectl annotate namespace test linkerd.io/inject=enabled
```

Install the load testing service to generate traffic during the canary analysis:

```shell
kubectl apply -k https://github.com/fluxcd/flagger//kustomize/tester?ref=main
```

If you want to install a **demo test app**. Create a deployment and a horizontal pod autoscaler:

```shell
git clone https://github.com/HybridK8s/demos && cd demos/droid

helm upgrade -i test-app test-app -n test
```

## **Step 3: Pair cluster with HybridK8s Droid**

Login to [HybridK8s Console](https://app.hybridk8s.tech). On the "Clusters" page, you should be able to see this quick form - 

![](https://user-images.githubusercontent.com/15074229/120476692-82c8b800-c3c8-11eb-8e9e-3b0210899917.png)

*   Click on "New Cluster"
*   Add your cluster name, environment.
*   Choose Mesh type as Linkerd (ISTIO support coming soon..)
*   If you have Prometheus Metric store already in your cluster, you can add the url otherwise leave it empty.
*   Choose service type
*   Click "Create"

You should be able to see cluster details like cluster key (separate for each cluster for security reasons) and other details you added. 

Now apply the commands shown on the cluster detail page. Following are commands we'll use to install agent in the cluster :

```shell
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
helm repo add hybridk8s https://hybridk8s.github.io/agent-chart && helm repo update && kubectl create ns agent
```

 Please ensure you use the right Cluster key. 

```shell
helm upgrade -i hybridk8s-agent -n agent hybridk8s/agent --set config.AGENT_AGENTINFO_APIKEY=<CLUSTER_KEY>
```

Congrats! Milestone achieved! 

## **Step 4: Applying Canary**

Create a [canary custom resource](https://github.com/HybridK8s/demos/blob/main/droid/canary.yaml) for the test-app deployment. Just make sure to add the **cluster key**(`<CLUSTER_KEY>)` in the **canary.yaml** before applying :  

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: test-app
  namespace: test
spec:
  # deployment reference
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: test-app
  # the maximum time in seconds for the canary deployment
  # to make progress before it is rollback (default 600s)
  progressDeadlineSeconds: 800
  service:
    # ClusterIP port number
    port: 80
    # container port number or name (optional)
    targetPort: 8080
  analysis:
    # schedule interval (default 60s)
    interval: 60s
    # max number of failed metric checks before rollback
    threshold: 1
    # max traffic percentage routed to canary
    # percentage (0-100)
    maxWeight: 50
    # canary increment step
    # percentage (0-100)
    stepWeight: 5
    # Linkerd Prometheus checks
    webhooks:
    - name: load-test
      type: rollout
      url: http://flagger-loadtester.test/
      metadata:
        cmd: "hey -z 60m -q 100 -c 2 http://test-app.demo/test"
    - name: verify
      type: rollout
      url: https://api.hybridk8s.tech/api/flagger/verify
      timeout: 120s
      metadata:
        api_key: "<CLUSTER_KEY>"
        app: "demo-app-1"
        primary: "test-app-primary"
        canary: "test-app"
        container: "test-app"
        duration: "60"
```

Now apply the canary to the cluster. 

```shell
kubectl apply -f ./canary.yaml
```

Go grab a cup of coffee ☕️ ... it'll take a few minutes to brew the magic! ✨

## **Step 5: Let's Try making a Faulty Deployment**

Let's try to change the docker image tag to "faulty" in the test-app. We can assume it to be similar to an error being introduced in any deployment.

```shell
helm upgrade -i test-app test-app -n test --set image.tag=faulty
```

☕️ ... Take some sips! It'll take a few minutes to realise the magic! ✨

You can see the magic happening via CLI or Linkerd dashboard. 

CLI fans, use : 

```
kubectl describe canary -n test test-app
```

Visualisation admirers, use : 

```shell
linkerd viz dashboard
```

We can see traffic splitting, response rates and other metrics. 

After a few minutes the canary will fail and automatically rollback because Droid automatically compared the primary metrics and logs with the canary metrics and logs. Things didn't seem better/fine. You can see why the deployment failed in detail on the platform.

**In case of metric failures :**

![](https://user-images.githubusercontent.com/15074229/120481033-506d8980-c3cd-11eb-9959-7496cbc8881f.png)

**In case of log failures :**

![](https://user-images.githubusercontent.com/15074229/120481444-bd811f00-c3cd-11eb-9e0c-4863244b6f4f.png)

Happy deploying! ✨☕️
