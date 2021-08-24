---
title: "Google Cloud Endpoints in GKE with Container-native Load Balancing"
description: "Google Cloud Endpoints in GKE walkthrough"
date: 2021-03-08
lastmod: 2021-08-23
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
---
In the source code for this tutorial, we extend the [Getting started with Cloud Endpoints for GKE with ESP](https://cloud.google.com/endpoints/docs/openapi/get-started-kubernetes-engine) documentation guide to provide an example of how to configure HTTPS between the LB and the ESP.

<!--more-->

In the step-by-step below, we will configure the ESP to communicate with the LB over HTTP. I'll let you verify the HTTPS configuration between the ESP and the LB. The purpose of this tutorial is for you to use the existing code in [the GitHub repository](https://github.com/soeirosantos/cloud-endpoints-gke-container-native-lb) to make changes and experiment.

Start cloning the source code.

```shell
git clone https://github.com/soeirosantos/cloud-endpoints-gke-container-native-lb.git
```

{{< notice type="note" id="fork" title="Note" >}}
The source code used in the [repo for this tutorial](https://github.com/soeirosantos/cloud-endpoints-gke-container-native-lb) is a fork of [Google Cloud golang-samples/endpoints/getting-started](github.com/GoogleCloudPlatform/golang-samples/endpoints/getting-started).
{{< /notice >}}

## Prepare the environment

Create variables with your GCP Project ID, GCP Region and name of the GKE cluster to be created.

```shell
gcp_project=your_project_id
gcp_region=us-central1
gke_cluster=le-cluster
```

```shell
gcloud config set project $gcp_project
```

Create the GKE cluster.

```shell
gcloud beta container clusters create $gke_cluster \
  --release-channel stable \
  --enable-ip-alias \
  --region $gcp_region
```

After the cluster is created, your `kubeconfig` is already pointing to the new cluster. If you change contexts and need to re-connect, use:

```shell
gcloud container clusters get-credentials $gke_cluster --region $gcp_region
```

Create an External IP that will be used in the Ingress later.

```shell
gcloud compute addresses create esp-echo --global
```

Store the IP address in a variable.

```shell
ingress_ip=$(gcloud compute addresses describe \
    esp-echo --global --format json | jq -r .address)
```

## Deploy the Cloud Endpoints configuration

Inspect the [openapi.yaml](https://github.com/soeirosantos/cloud-endpoints-gke-container-native-lb/blob/main/openapi.yaml) file and update the attribute `host` with the name of your `gcp_project`.

```yaml
host: "echo-api.endpoints.YOUR_PROJECT_ID.cloud.goog"
```

```shell
sed -i.original "s/YOUR_PROJECT_ID/${gcp_project}/g" openapi.yaml
```

The value of the attribute host will be the name of the Cloud Endpoints service.

Now, upload the configuration and create a managed service.

```shell
gcloud endpoints services deploy openapi.yaml
```

Check the Google services enabled in your project and enable the necessary services if they aren't enabled.

```shell
gcloud services list

gcloud services enable servicemanagement.googleapis.com
gcloud services enable servicecontrol.googleapis.com
gcloud services enable endpoints.googleapis.com
gcloud services enable echo-api.endpoints."${gcp_project}".cloud.goog
```

## Deploy the Kubernetes configuration

Deploy the Kubernetes config using ESP with HTTP.

Adjust the name of your Endpoints service in the ESP configuration.

```yaml
- name: esp
  image: gcr.io/endpoints-release/endpoints-runtime:1
  args: [
      "--http_port", "8081",
    "--backend", "127.0.0.1:8080",
    "--service", "echo-api.endpoints.YOUR_PROJECT_ID.cloud.goog",
    "--rollout_strategy", "managed",
  ]
```

```shell
sed -i.original "s/YOUR_PROJECT_ID/${gcp_project}/g" .kube-http.yaml
```

In the ESP configuration above, the `--rollout_strategy=managed` option configures ESP to use the latest deployed service configuration. When you specify this option, up to 5 minutes after you deploy a new service configuration, ESP detects the change and automatically begins using it. Alternatively, you can use the `--version` / `-v` flag to have more control of which configuration id / version of your Cloud Endpoints the ESP is using. [See more details about the ESP options](https://cloud.google.com/endpoints/docs/openapi/specify-proxy-startup-options). Using the `-v` flag is recommended for production so you can have control over configuration deployed to prod via Git.

Deploy the Kubernetes config.

```shell
kubectl apply -f .kube-http.yaml
```

Check if the pod was properly initialized.

```shell
kubectl get po
```

Check if the ingress has an external IP assigned. This IP is the same IP we defined earlier.

It will take several minutes until the Ingress becomes available. Wait until the backend service reports `HEALTHY`.

```shell
watch "kubectl get ingress esp-echo \
    -o jsonpath='{.metadata.annotations.ingress\.kubernetes\.io/backends}'"
```

Use the following commands to observe how the GCP Backend Service and Health Check get configured based on your Ingress, Service and Pod configuration.

```shell
backend_service=$(kubectl get ingress esp-echo \
    -o jsonpath='{.metadata.annotations.ingress\.kubernetes\.io/backends}' | jq -r keys[0])

gcloud compute backend-services describe $backend_service --global

gcloud compute health-checks describe $backend_service --global
```

Finally, check your service.

```shell
curl -v http://"${ingress_ip}"/healthz

curl --request POST \
   --header "content-type:application/json" \
   --data '{"message":"hello world"}' \
   "http://${ingress_ip}/echo"
```

Execute the same steps with the [.kube-https.yaml](https://github.com/soeirosantos/cloud-endpoints-gke-container-native-lb/blob/main/.kube-https.yaml) configuration. Notice that you will test from the `ingress_ip` still using HTTP. This is because when you configure the ESP container with HTTPS you are using TLS only for traffic from `LB -> ESP`. Configuring TLS for your Ingress is important and something you should definitely do, but it's out of the scope in this tutorial.

## Optional - Test different types of Security Definitions

The Open API configuration provides examples of how to configure different types of `securityDefinitions` using Cloud Endpoints. These `securityDefinitions` after proper configured can be tested with [this client](https://github.com/soeirosantos/cloud-endpoints-gke-container-native-lb/blob/main/client/main.go).

## Clean up

```shell
kubectl delete -f .kube-http.yaml
gcloud services disable echo-api.endpoints."${gcp_project}".cloud.goog
gcloud endpoints services delete echo-api.endpoints."${gcp_project}".cloud.goog
gcloud compute addresses delete esp-echo --global
gcloud container clusters delete "$gke_cluster" --region "$gcp_region"
```

## Troubleshooting

The main problems you may have in this setup are related to the Ingress configuration. You can check the Google Cloud [troubleshooting page](https://cloud.google.com/kubernetes-engine/docs/how-to/container-native-load-balancing#troubleshooting) for Container-native load balancing.

If the Ingress health check for the https example is consistently showing Unhealthy you may need to create a firewall rule to allow the Google LB to reach the backend in the 8443 port used in this example.

```shell
gcloud compute firewall-rules create fw-allow-health-check-and-proxy \
  --network=NETWORK_NAME \
  --action=allow \
  --direction=ingress \
  --target-tags=GKE_NODE_NETWORK_TAGS \
  --source-ranges=130.211.0.0/22,35.191.0.0/16 \
  --rules=tcp:8443
 ```

## References

Resources that helped me during this experiment:

https://cloud.google.com/endpoints/docs/openapi/get-started-kubernetes-engine
https://cloud.google.com/endpoints/docs/openapi/specify-proxy-startup-options
https://cloud.google.com/endpoints/docs/openapi/configure-endpoints
https://cloud.google.com/kubernetes-engine/docs/concepts/ingress-xlb#https_tls_between_load_balancer_and_your_application
https://cloud.google.com/kubernetes-engine/docs/concepts/container-native-load-balancing
https://cloud.google.com/kubernetes-engine/docs/how-to/container-native-load-balancing#troubleshooting
