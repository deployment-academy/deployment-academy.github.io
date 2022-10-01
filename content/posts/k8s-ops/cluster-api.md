---
title: "Self-managed Kubernetes with Cluster API in GCP (+ Cilium)"
description: "Cluster API is a Kubernetes sub-project focused on providing declarative APIs and tooling to simplify provisioning, upgrading, and operating multiple Kubernetes clusters."
date: 2022-10-01T11:32:38-04:00
lastmod: 2022-10-01T11:32:38-04:00
draft: false
tags:
  - "kubernetes"
  - "ops"
  - "k8s"
  - "cluster-api"
  - "gcp"
  - "google cloud"
  - "cilium"
---

We all know the benefits of using managed Kubernetes services like GKE, EKS, AKS, etc. Given the complexity of managing the cluster infrastructure and its core components (control plane, auto-scaling, monitoring, networking, storage, etc.), using a managed Kubernetes service is generally the first choice when running workloads in production.

However, in some situations, provisioning and managing the Kubernetes cluster from scratch might be necessary. Specific product features, security & compliance, costs, vendor independency, etc. are some factors that usually justify the decision to run Kubernetes by yourself. Of course, many challenges come with managing a Kubernetes cluster. I want to keep this discussion out of the scope of this tutorial since it requires special attention.

Nowadays, when considering provision and managing a Kubernetes cluster, the tool of choice is [Cluster API](https://cluster-api.sigs.k8s.io/). From the docs:

> _Cluster API is a Kubernetes sub-project focused on providing declarative APIs and tooling to simplify provisioning, upgrading, and operating multiple Kubernetes clusters._
>
> _[...] The supporting infrastructure, like virtual machines, networks, load balancers, and VPCs, as well as the Kubernetes cluster configuration are all defined in the same way that application developers operate deploying and managing their workloads. This enables consistent and repeatable cluster deployments across a wide variety of infrastructure environments._

<!--more-->

Cluster API supports a comprehensive [list of providers](https://cluster-api.sigs.k8s.io/reference/providers.html) covering the main public Cloud providers and other popular platforms.

This tutorial will use the Cluster API to provision a cluster in GCP and install Cilium as the network solution. It follows the same idea as the [Cluster API quick start guide](https://cluster-api.sigs.k8s.io/user/quick-start.html) for GCP with more complete and detailed steps.

Before we start, note that Cluster API introduces important [concepts](https://cluster-api.sigs.k8s.io/user/concepts.html) that I'll use here with minimal explanation. So, I recommend reading the concepts and other relevant parts of the documentation. It's not a requirement to follow along, tho.

## Prerequisites

For this tutorial, you need to create or select a Google Cloud Project and ensure the following tools are installed.

- [gcloud](https://cloud.google.com/sdk/gcloud#downloading_the_gcloud_command-line_tool)
- [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl) (v1.23)
- [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) (v0.14.0+)
- [clusterctl](https://cluster-api.sigs.k8s.io/user/quick-start.html#install-clusterctl) (v1.2.2+)
- [packer](https://www.packer.io/intro/getting-started/install.html) (v1.8.3)
- [ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) (v2.13.4)
- [helm](https://helm.sh/docs/intro/install/) (v3.10.0)


## Preparing the GCP project

In the terminal export the following variables:

```shell
export GCP_PROJECT_ID=cluster-api-363616
export GCP_REGION=us-east1
export GCP_NETWORK_NAME=default
export CLUSTER_NAME=starks
```

### Create a Cloud NAT

To ensure the cluster can communicate with the outside world and the load balancer, let's create a Cloud NAT. Note that we are using the `default` network.

If it's a new project enable the compute API.

```shell
gcloud services enable compute.googleapis.com --project "$GCP_PROJECT_ID"
```

Create a Cloud Router and Cloud NAT.

```shell
gcloud compute routers create "${CLUSTER_NAME}-router" \
  --region "${GCP_REGION}" --network "default" \
  --project "${GCP_PROJECT_ID}"

gcloud compute routers nats create "${CLUSTER_NAME}-nat" \
  --router-region "${GCP_REGION}" --router "${CLUSTER_NAME}-router" \
  --nat-all-subnet-ip-ranges --auto-allocate-nat-external-ips \
  --project "${GCP_PROJECT_ID}"
```

These instructions can be found in the [GCP Cluster API Provider documentation](https://github.com/kubernetes-sigs/cluster-api-provider-gcp/blob/main/docs/book/src/topics/prerequisites.md#setup-a-network-and-cloud-nat). 

### Create a Service Account

We need a service account key to grant the Cluster API access to the GCP project. As per [these instructions](https://github.com/kubernetes-sigs/cluster-api-provider-gcp/blob/main/docs/book/src/topics/prerequisites.md#create-a-service-account) this Service Account needs `Editor` permissions.

Generate the service account and attach the `Editor` role.

```shell
gcloud iam service-accounts create "${CLUSTER_NAME}-sa" \
  --display-name "${CLUSTER_NAME}-sa" \
  --project "${GCP_PROJECT_ID}"

gcloud projects add-iam-policy-binding "${GCP_PROJECT_ID}" \
  --member "serviceAccount:${CLUSTER_NAME}-sa@${GCP_PROJECT_ID}.iam.gserviceaccount.com" \
  --role "roles/editor"
```

Create a service account key and store it in a file called `key.json`.

```shell
gcloud iam service-accounts keys create key.json \
  --iam-account "${CLUSTER_NAME}-sa@${GCP_PROJECT_ID}.iam.gserviceaccount.com" \
  --project "${GCP_PROJECT_ID}"
```

### Build a Kubernetes GCE Image

We need a GCE image with the Kubernetes dependencies and configuration that will be used in the nodes instances. For this, we will use the [Kubernetes Image Builder](https://github.com/kubernetes-sigs/image-builder) project.

Export the `GOOGLE_APPLICATION_CREDENTIALS` variable pointing to the service account key file.

```shell
export GOOGLE_APPLICATION_CREDENTIALS="$PWD"/key.json
```

Clone the Image Builder repository.

```shell
git clone https://github.com/kubernetes-sigs/image-builder.git image-builder; cd image-builder/images/capi
```

Build the image.

```shell
make build-gce-ubuntu-2004
```

{{< notice type="note" id="tip-id" title="Note on the Kubernetes version and customization" >}}
In this tutorial, we use the default Kubernetes version defined in the configuration cloned from the repository at the time of this writing (v1.23.10). To change the Kubernetes version or customize other build parameters check the [customization section](https://github.com/kubernetes-sigs/image-builder/blob/master/docs/book/src/capi/capi.md#customization) of the Image Builer CAPI documentation.
{{< /notice >}}

 

**This step takes several minutes. You can probably move forward with other steps until before the `kubectl apply` that starts the creation of the cluster.**

Retrieve the image name and set the image ID variable.

```shell
IMAGE_NAME=$(gcloud compute images list --no-standard-images \
  --sort-by=~creationTimestamp --filter="family:capi-ubuntu-2004-k8s" \
  --project "${GCP_PROJECT_ID}" --format json | jq -r '.[0].name')

export IMAGE_ID=projects/"${GCP_PROJECT_ID}"/global/images/"${IMAGE_NAME}"
```

For more details about this process check the doc for the [Image Builder GCP](https://image-builder.sigs.k8s.io/capi/providers/gcp.html).

Return to the original directory (where the `key.json` file is stored).

```shell
cd -
```

{{< notice type="tip" id="ssh-ansible" title="SSH errors during the Ansible execution" >}}
I'm on Ubuntu 22.04 and had to add these lines to my SSH config for the Ansible script connect with the remote server.
```
cat ~/.ssh/config
HostKeyAlgorithms +ssh-rsa
PubkeyAcceptedKeyTypes +ssh-rsa
```
{{< /notice >}}


## Spin up the Management Cluster

A [Management Cluster](https://cluster-api.sigs.k8s.io/user/concepts.html#management-cluster) is a Kubernetes cluster that manages the lifecycle of the cluster that we are going to create with the Cluster API (aka [Workload Cluster](https://cluster-api.sigs.k8s.io/user/concepts.html#workload-cluster)).

Set a variable with the Kubernetes version.

```shell
KUBERNETES_VERSION=$(jq -r .kubernetes_semver image-builder/images/capi/packer/config/kubernetes.json)
```

We will use `kind` to create a Management Cluster locally. This is only for development and experimentation. In production, you should use an actual production-ready Kubernetes cluster.


Create the local cluster.

```shell
kind create cluster --name management-cluster --image kindest/node:"$KUBERNETES_VERSION"
```

Initialize the Management Cluster using the GCP provider.

```shell
export GCP_B64ENCODED_CREDENTIALS=$(base64 key.json | tr -d '\n')

clusterctl init --infrastructure gcp
```

Inspect the Management Cluster and check the CAPI controllers that have been created.

{{< notice type="warning" id="capi-bootstrap-creds" title="CAPI Manager Bootstrap Credentials" >}}
Note that the service account key has been stored in the `capg-manager-bootstrap-credentials` kube secret in the namespace `capg-system`. This is a really high-privileged static credential to have stored in the cluster. In a production environment, this is something that you definetely want to keep track of and secure properly.
```shell
kubectl get secrets capg-manager-bootstrap-credentials \
  -n capg-system -o json | jq -r '.data."credentials.json"' | base64 -d
```
{{< /notice >}}


## Create the Workload Cluster

Export the following variables.

```shell
export GCP_PROJECT="$GCP_PROJECT_ID"
export GCP_CONTROL_PLANE_MACHINE_TYPE=n1-standard-2
export GCP_NODE_MACHINE_TYPE=n1-standard-2
```

The name of these variables and others we have exported before (e.g. `IMAGE_ID` and `KUBERNETES_VERSION`) must match what is defined in the [cluster-template.yaml](https://github.com/kubernetes-sigs/cluster-api-provider-gcp/blob/main/templates/cluster-template.yaml).

Generate the cluster configuration.

```shell
clusterctl generate cluster "$CLUSTER_NAME" \
  --kubernetes-version "$KUBERNETES_VERSION" \
  --control-plane-machine-count=3 \
  --worker-machine-count=3 \
  > "$CLUSTER_NAME".yaml
```

Inspect the YAML file created. This YAML contains the specification for the GCP infrastructure that will be created. You apply this to the Management Cluster and the CAPI controllers deployed to it will take care of the infrastructure provisioning.

In a production environment, you'll adjust and version control this file with all of your specific requirements.

Apply it.


```shell
kubectl apply -f "$CLUSTER_NAME".yaml 

```

It will create the resources:

```
cluster.cluster.x-k8s.io
gcpcluster.infrastructure.cluster.x-k8s.io
kubeadmcontrolplane.controlplane.cluster.x-k8s.io
gcpmachinetemplate.infrastructure.cluster.x-k8s.io
machinedeployment.cluster.x-k8s.io
gcpmachinetemplate.infrastructure.cluster.x-k8s.io
kubeadmconfigtemplate.bootstrap.cluster.x-k8s.io
```

This will initialize the creation of the GCP resources for the Workload Cluster.

Inspect and explore the resources created. These are some useful commands to check the progress of cluster provisioning.

```shell
kubectl get cluster
kubectl describe cluster "$CLUSTER_NAME" 
kubectl get machines
kubectl get gcpmachines
kubectl get machinedeployments 
kubectl describe kubeadmcontrolplane "$CLUSTER_NAME"-control-plane
```

Check the GCP project and note the resources created as part of the provisioning. Some useful commands to explore the GCP resoruces.

```shell
gcloud compute instance-groups list --project "$GCP_PROJECT_ID"
gcloud compute instances list  --filter="tags.items=$CLUSTER_NAME" --project "$GCP_PROJECT_ID"
gcloud compute firewall-rules list --project "$GCP_PROJECT"
```

Checking the CAPI controllers logs is probably your best bet if you need to debug problems.

It will take a couple minutes. Wait until the all the 6 nodes are up and running.

```shell
kubectl get machines
NAME                           CLUSTER   NODENAME                     PROVIDERID                                                       PHASE     AGE     VERSION
starks-control-plane-7d7f4     starks    starks-control-plane-8qbrq   gce://project-id/us-east1-c/starks-control-plane-8qbrq   Running   4m53s   v1.23.10
starks-control-plane-jdqnz     starks    starks-control-plane-rxhl4   gce://project-id/us-east1-d/starks-control-plane-rxhl4   Running   3m2s    v1.23.10
starks-control-plane-kgdhb     starks    starks-control-plane-c7t74   gce://project-id/us-east1-b/starks-control-plane-c7t74   Running   9m24s   v1.23.10
starks-md-0-6f6f4956f8-2lj5w   starks    starks-md-0-lbxw9            gce://project-id/us-east1-b/starks-md-0-lbxw9            Running   10m     v1.23.10
starks-md-0-6f6f4956f8-ctstr   starks    starks-md-0-fqd89            gce://project-id/us-east1-b/starks-md-0-fqd89            Running   10m     v1.23.10
starks-md-0-6f6f4956f8-tqw5n   starks    starks-md-0-nlknx            gce://project-id/us-east1-b/starks-md-0-nlknx            Running   10m     v1.23.10

```

## Install Cilium

Now, we need to install a CNI to get the Workload Cluster ready to work. 

Export the kubeconfig.

```shell
clusterctl get kubeconfig "$CLUSTER_NAME" > "$CLUSTER_NAME".kubeconfig
```

Open a new terminal session and in this same directory export the `KUBECONFIG` environment variable.

```shell
export KUBECONFIG="$PWD/$CLUSTER_NAME".kubeconfig
```

Now your kubectl is pointing to the Workload Cluster.

Inspect the pods created in this cluster `kubectl get po n- kube-system`. Also, check the nodes and note that they are reporting `NotReady` status.

```shell
kubectl get nodes
```

Install Cilium.

```shell
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium --version 1.12.2 --namespace kube-system
```

Check that the Cilium pods are running.

```shell
kubectl get pod -l k8s-app=cilium -n kube-system
```

Check the nodes again and verify that now they are ready.

Refer to the [Cilium docs](https://docs.cilium.io/en/stable/gettingstarted/) for more details.

## Check the cluster

To verify that our cluster is completely operational, let's deploy and expose an application.

```shell
kubectl apply -f https://k8s.io/examples/controllers/nginx-deployment.yaml

kubectl expose deployment nginx-deployment --type=LoadBalancer --name=nginx-service
```

Wait until the service provides an external IP and run.

```shell
curl -s $(kubectl get services nginx-service -o json | jq -r '.status.loadBalancer.ingress[0] | .ip') | grep '<h1>Welcome to nginx!</h1>'
<h1>Welcome to nginx!</h1>
```

This LoadBalancer service creates a Network Load Balancer in GCP and is good for this type of test or a few use cases. To expose your application you probably want to use an Ingress Controller such as [Ingress NGINX Controller](https://github.com/kubernetes/ingress-nginx), which provides a richer set of features.

## Clean up

Pointing you kubeconfig to the workload cluster.
```shell
kubectl delete svc nginx-service
```
(it will remove the load balancer and the firewall rules)

Pointing you kubeconfig to the management cluster.
```shell
kubectl delete cluster "$CLUSTER_NAME"
```

Remove the management cluster.
```shell
kind delete cluster --name management-cluster
```

Remove the GCP objects that we created.
```shell
gcloud compute routers nats delete "$CLUSTER_NAME"-nat --router="$CLUSTER_NAME"-router --router-region="$GCP_REGION" --project "$GCP_PROJECT_ID"
gcloud compute routers delete "$CLUSTER_NAME"-router --region="$GCP_REGION" --project "$GCP_PROJECT_ID"
gcloud iam service-accounts delete "${CLUSTER_NAME}-sa@${GCP_PROJECT_ID}.iam.gserviceaccount.com" --project "$GCP_PROJECT_ID"
```

If you have any questions or comments feel free to reach out in the Twitter thread below or directly.

{{< tweet user="soeiro_santos" id="1576269542590148611" >}}

