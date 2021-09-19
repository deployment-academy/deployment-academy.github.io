---
title: "GKE Dataplane V2 and Network Policies in Practice"
description: "changeme"
date: 2021-09-18T13:48:06-04:00
lastmod: 2021-09-18T13:48:06-04:00
draft: false
sidebar: "right"
widgets:
  - "ddg-search"
  - "recent"
  - "social"
tags:
  - "devsecops"
  - "kubernetes"
  - "gcp"
  - "google cloud"
  - "gke"
  - "dataplane v2"
  - "network policies"
---

In this tutorial, we are going to play with the [Google Kubernetes Engine Dataplane V2](https://cloud.google.com/kubernetes-engine/docs/concepts/dataplane-v2) and check how we can use it along with [Kubernetes Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) to limit traffic to Pods and to obtain real-time visibility on network activity.

Dataplane V2 is a recent feature in GKE whith GA starting on version 1.20.6-gke.700, as of May 10, 2021 - [see this feature announcement for more details](https://cloud.google.com/blog/products/containers-kubernetes/bringing-ebpf-and-cilium-to-google-kubernetes-engine). Dataplane V2 uses [eBPF](https://ebpf.io/) and [Cilium](https://cilium.io/) to performantly process network packets in-kernel using Kubernetes-specific metadata without relying on the kube-proxy and iptables for service routing. Dataplane V2 brings some interesting features for cluster operations and security, such as:

- Built-in Network Policies enforcement without the need of Calico
- Real-time visibility, enabling cluster networking troubleshooting, auditing and alerting

See the [Dataplane V2 documentation](https://cloud.google.com/kubernetes-engine/docs/concepts/dataplane-v2) for more details.

## Pre-requisites

To follow this tutorial you need a [Google Cloud project created](https://console.cloud.google.com/cloud-resource-manager?pli=1) with [billing enabled](https://cloud.google.com/billing/docs/how-to/modify-project).

Also, make sure you have the most recent version of the `gcloud` CLI installed. See [this doc for installation instructions](https://cloud.google.com/sdk/gcloud/#downloading_the_gcloud_command-line_tool). If you already have `gcloud` installed, you can ensure it's up to date by running `gcloud components update`.

Note that this tutorial uses billable components of Google Cloud. Make sure you execute the [clean up](#clean-up) steps after you finish.

## Create a GKE Cluster

Let's get started by creating a GKE cluster with Dataplane V2 enabled.

Let's set some defaults:

```shell
gcp_project=your-project
cluster_name=your-cluster
cluster_region=us-east1

gcloud config set project $gcp_project
```

And create the cluster:

```shell
gcloud container clusters create $cluster_name \
    --enable-dataplane-v2 \
    --enable-ip-alias \
    --release-channel regular \
    --region $cluster_region
```

The current default version in the `regular` channel already enables the use of the Dataplane V2. If you want to know which versions are available in the release channels, you can execute:

```shell
# optional command to check versions in the channels, no need to run
gcloud container get-server-config --format "yaml(channels)" --region $cluster_region
```

After the cluster is created your `kube-config` will be already be pointing to the newly created cluster. If you need to reconnect to it, you can use the following command:

```shell
# optional command to re-configure the kube-config, no need to run
gcloud container clusters get-credentials $cluster_name --region $cluster_region
```

## Deploy a Demo Application

To check the Dataplane V2 feature, we want to use a non-trivial scenario. For this, we will deploy the [Online Boutique](https://github.com/GoogleCloudPlatform/microservices-demo) - a cloud-native microservices demo application used to demonstrate Kubernetes/GKE features.

Clone the GitHub repository:

```shell
git clone https://github.com/GoogleCloudPlatform/microservices-demo
```

Apply the Kubernetes manifests:

```shell
kubectl apply -f microservices-demo/release/kubernetes-manifests.yaml
```

Check that the pods are running:

```shell
kubectl get po
NAME                                     READY   STATUS    RESTARTS   AGE
adservice-5844cffbd4-k5nnq               1/1     Running   0          65s
cartservice-fdc659ddc-lf8ph              1/1     Running   0          66s
checkoutservice-64db75877d-stqwv         1/1     Running   0          68s
currencyservice-9b7cdb45b-qsgzb          1/1     Running   0          66s
emailservice-64d98b6f9d-s528z            1/1     Running   0          68s
frontend-76ff9556-nsb6w                  1/1     Running   0          67s
loadgenerator-589648f87f-nnfzt           1/1     Running   0          66s
paymentservice-65bdf6757d-rqq8x          1/1     Running   0          67s
productcatalogservice-5cd47f8cc8-d2x8d   1/1     Running   0          67s
recommendationservice-b75687c5b-nz6rt    1/1     Running   0          68s
redis-cart-74594bd569-wcvlz              1/1     Running   0          65s
shippingservice-778554994-8l2pr          1/1     Running   0          66s
```

Check the external IP of the `LoadBalancer` service:

```shell
kubectl get svc frontend-external
NAME                TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)        AGE
frontend-external   LoadBalancer   10.60.3.181   34.139.243.117   80:30595/TCP   2m1s
```

> **Optional:** Access the application from the browser and add some products to the cart.

## Undesired Network Access

This microservice application has quite a few components and, among them, we can see that the `CartService` communicates with the `Cache` (a Redis instance) to store the product items in the user's shopping cart. See the [architecutre diagram](https://github.com/GoogleCloudPlatform/microservices-demo#architecture) for details on how the components are related to each other.

List the Kubernetes services in the cluster to identify the Redis service:

```shell
kubectl get svc
NAME                    TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)        AGE
adservice               ClusterIP      10.60.5.28     <none>           9555/TCP       6m
cartservice             ClusterIP      10.60.6.160    <none>           7070/TCP       6m
checkoutservice         ClusterIP      10.60.9.50     <none>           5050/TCP       6m
currencyservice         ClusterIP      10.60.9.235    <none>           7000/TCP       6m
emailservice            ClusterIP      10.60.8.126    <none>           5000/TCP       6m
frontend                ClusterIP      10.60.1.70     <none>           80/TCP         6m
frontend-external       LoadBalancer   10.60.3.181    34.139.243.117   80:30595/TCP   6m
kubernetes              ClusterIP      10.60.0.1      <none>           443/TCP        17m
paymentservice          ClusterIP      10.60.12.233   <none>           50051/TCP      6m
productcatalogservice   ClusterIP      10.60.1.237    <none>           3550/TCP       6m
recommendationservice   ClusterIP      10.60.6.61     <none>           8080/TCP       6m
redis-cart              ClusterIP      10.60.1.152    <none>           6379/TCP       6m
shippingservice         ClusterIP      10.60.4.213    <none>           50051/TCP      6m
```

You can see the `redis-cart` `ClusterIP` service configured on port `6379`. You can verify that this service is fronting the `redis-cart` Pod by checking the label selector:

```shell
kubectl describe svc redis-cart
Name:              redis-cart
Namespace:         default
Labels:            <none>
Annotations:       cloud.google.com/neg: {"ingress":true}
Selector:          app=redis-cart
Type:              ClusterIP
IP Families:       <none>
IP:                10.60.1.152
IPs:               10.60.1.152
Port:              redis  6379/TCP
TargetPort:        6379/TCP
Endpoints:         10.56.8.4:6379
Session Affinity:  None
Events:            <none>
```

And confirm by checking the `redis-cart` deployment:

```shell
kubectl get deploy redis-cart -o json | jq .spec.template.metadata.labels
{
  "app": "redis-cart"
}
```

Note that since describing a deployment is much more verbose than describing a service, we opted to use the `json` output piping to `jq`.

Let's play the role of a malicious actor that got access to a container in our cluster. For this, we will launch a container using the `redis` image and check connectivity to the `redis-cart` Pod.

```shell
kubectl run rogue-1 --image=redis --restart=Never --rm -it -- /bin/bash
```

From the container's prompt run:

```shell
root@rogue-1:/data# redis-cli -h redis-cart ping
PONG
```

With this we can confirm conectivity to the Redis database. A malicious actor can use different commands to access data or disrupt the service. Also keep in mind that this behavior can be explored across all components of the application. We are using the Redis database just as an example.

## Kubernetes Network Policies

To improve the security posture, the first thing that we'll do is create a `NetworkPolicy` that blocks this kind of undesired access, restricting ingress traffic to the `redis-cart` Pod only from the `CartService` Pods.

```shell
kubectl apply -f- << EOF
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: redis-cart-allow-from-cartservice
spec:
  policyTypes:
  - Ingress
  podSelector:
    matchLabels:
      app: redis-cart
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: cartservice
EOF
```

The network policy is self-explanatory and you can see that we are using label selectors to match the Pods where we want the policy to be applied to and the pods where we want to enable ingress.

Launch the rogue container again and try to execute the `ping` command and notice that it hangs:

```shell
kubectl run rogue-1 --image=redis --restart=Never --rm -it -- /bin/bash
root@rogue-1:/data# redis-cli -h redis-cart ping
```

Use `CTRL+C` to stop.

This feature is already helping to improve our security posture. Still, depending on the level of access that a malicious actor has gotten, they could notice that a network policy is in play and try a few tricks to bypass it.

> **Optional:** Access the application from the browser and add some products to the cart and check that the `CartService -> Cache` connectivity still works as expected.

So, besides preventing undesired access, we also want to log this activity. This log allows us to audit and alert on suspicious events.

## Network Policy Logging

When you create a cluster with Dataplane V2 enabled, a default `NetworkLogging` object is created. We need to change this object to configure the network policy logging behavior.

Let's see how the default `NetworkLogging` object looks like initially:

```shell
kubectl get nl default -o yaml
apiVersion: networking.gke.io/v1alpha1
kind: NetworkLogging
metadata:
  name: default
spec:
  cluster:
    allow:
      delegate: false
      log: false
    deny:
      delegate: false
      log: false
```

Use this yaml configuration to change the default network logging configuration and enable logging for denied connections:

```shell
kubectl apply -f- << EOF
kind: NetworkLogging
apiVersion: networking.gke.io/v1alpha1
metadata:
  name: default
spec:
  cluster:
    allow:
      log: false
      delegate: false
    deny:
      log: true
      delegate: false
EOF
```

You should expect to see a warning after applying this change, it's ok.

Check the [Network Policy Logging documentation](https://cloud.google.com/kubernetes-engine/docs/how-to/network-policy-logging) for a complete description of the `NetworkLogging` specification and configuration options.

Now we can check the [Google Cloud Logging](https://cloud.google.com/logging/docs/quickstart-sdk) to verify the log events related to the network policy activity.

Execute the Redis `ping` command again:

```shell
kubectl run rogue-1 --image=redis --restart=Never --rm -it -- /bin/bash
root@rogue-1:/data# redis-cli -h redis-cart ping
```

And check the logs:

```shell
# replace <your-cluster-name> and <your-project-name>
gcloud logging read --project $gcp_project 'resource.type="k8s_node" resource.labels.location=us-east1 resource.labels.cluster_name=<your-cluster-name> logName="projects/<your-project-name>/logs/policy-action" jsonPayload.dest.pod_name=~"redis-cart" jsonPayload.disposition="deny"'
```

You will see the logs entries coming through with details about the events. Check the documentation for the [Log Format definition](https://cloud.google.com/kubernetes-engine/docs/how-to/network-policy-logging#log_format).

{{< notice type="note" id="cloud-logging" title="A note on Google Cloud Logging" >}}
You are able to check the logs with the previous command because [Cloud Logging](https://cloud.google.com/logging/docs) is enabled by default in new clusters, and network policy logs are automatically shipped to Cloud Logging. See more about how to [configure Cloud Operations for GKE](https://cloud.google.com/stackdriver/docs/solutions/gke/installing).

Using Cloud Logging you can set up a [Log-based metric](https://cloud.google.com/logging/docs/logs-based-metrics) and create an [alerting policy](https://cloud.google.com/logging/docs/logs-based-metrics/charts-and-alerts#alert-on-lbm) upon suspicious activity.

Keep in mind that network policy logs generated on each cluster are available at `/var/log/network/policy_action.log*` and can be shipped to your preferred log aggregator.
{{< /notice >}}

## Optional: Delegating Network Policy Logging Configuration

Instead of enabling logging for all denied connections as we did with our previous `NetworkLogging` configuration, you can delegate it using the annotation `policy.network.gke.io/enable-deny-logging: "true"`.

Update the `NetworkLogging` object:

```shell
kubectl apply -f- << EOF
kind: NetworkLogging
apiVersion: networking.gke.io/v1alpha1
metadata:
  name: default
spec:
  cluster:
    allow:
      log: false
      delegate: false
    deny:
      log: true
      delegate: true
EOF
```

Execute the Redis `ping` command again:

```shell
kubectl run rogue-1 --image=redis --restart=Never --rm -it -- /bin/bash
root@rogue-1:/data# redis-cli -h redis-cart ping
```

Check the logs and observe that there are no new entries:

```shell
# replace <your-cluster-name> and <your-project-name>
gcloud logging read --project $gcp_project 'resource.type="k8s_node" resource.labels.location=us-east1 resource.labels.cluster_name=<your-cluster-name> logName="projects/<your-project-name>/logs/policy-action" jsonPayload.dest.pod_name=~"redis-cart" jsonPayload.disposition="deny"'
```

Annotate the namespace where the `redis-cart` (Pod where the connection was denied) is running with the following annotation:

```shell
kubectl annotate namespaces default policy.network.gke.io/enable-deny-logging="true"
```

Execute the test again and observe that logs will show up.

{{< notice type="tip" id="allowed-connections-delegation" title="Allowed connections delegation" >}}
Note that to delegate logging for **allowed connections**, the configuration is different. We have to add the annotation `policy.network.gke.io/enable-logging: "true"` to the `NetworkPolicy` instead of the namespace.
{{< /notice >}}

## Clean up

```shell
kubectl delete -f microservices-demo/release/kubernetes-manifests.yaml
gcloud container clusters delete $cluster_name --region $cluster_region
```
