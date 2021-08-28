---
title: "Workload Identity in Practice"
description: "Workload Identity is the recommended way to access Google Cloud APIs from within GKE due to its improved security properties and manageability."
date: 2020-06-15
lastmod: 2020-06-15
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
  - "workload identity"
---
In this tutorial, we're going to go through the [Workload Identity](https://cloud.google.com/blog/products/containers-kubernetes/introducing-workload-identity-better-authentication-for-your-gke-applications) feature and see how it helps to improve the way we manage access to Google Services and APIs from applications running in [Google Kubernetes Engine (GKE)](https://cloud.google.com/kubernetes-engine).

Workload Identity is the recommended way to access Google Cloud APIs from within GKE due to its improved security properties and manageability. With Workload Identity you can control access to APIs using [Google service accounts](https://cloud.google.com/iam/docs/service-accounts) and [IAM roles](https://cloud.google.com/iam/docs/understanding-roles) without deploying static service account JSON keys to Pods and without relying on the node's service account.

<!--more-->

## Create a GKE Cluster

Let's get started.

Assuming you already have a GCP project, create a variable with the name of your project and two other variables for cluster name and region.

```shell
gcp_project=my-project
cluster_name=le-cluster
cluster_region=us-east1

gcloud config set project $gcp_project
```

Start creating a simple cluster with Workload Identity enabled, which is defined with the property `--workload-pool`.

```shell
gcloud beta container clusters create ${cluster_name} \
  --release-channel regular \
  --enable-ip-alias \
  --region $cluster_region \
  --workload-pool=${gcp_project}.svc.id.goog
```

Get cluster credentials:

```shell
gcloud container clusters get-credentials $cluster_name --region $cluster_region
```

## Configure Workload Identity

With Workload Identity, you configure a Kubernetes service account (KSA) to act as a Google service account (GSA). Any workload running as the KSA, automatically authenticates as the GSA when accessing Google Cloud APIs.

Let's create a Google Service Account:

```shell
gcloud iam service-accounts create le-application-stg
```

Now we create a k8s namespace and k8s service account:

```shell
kubectl create namespace staging

kubectl create serviceaccount le-application -n staging
```

{{< notice type="note" id="note-k8s-declarative-config" title="A note on Kubernetes Declarative Configuration" >}}
In this tutorial, we are using the `kubectl` CLI to create and annotate the namespace and service account, and to run Pods. The purpose is to play around and quickly evaluate how Workload Identity works. In the real world you'll likely want to use YAML configuration to declaratively manage your Kubernetes resources.
{{< /notice >}}

Associate both SAs using an [IAM Policy binding](https://cloud.google.com/sdk/gcloud/reference/iam/service-accounts/add-iam-policy-binding):

```shell
gcloud iam service-accounts add-iam-policy-binding \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:${gcp_project}.svc.id.goog[staging/le-application]" \
  le-application-stg@${gcp_project}.iam.gserviceaccount.com
```

And, finally, annotate your service account:

```shell
kubectl annotate serviceaccount \
  le-application \
  iam.gke.io/gcp-service-account=le-application-stg@${gcp_project}.iam.gserviceaccount.com \
  -n staging
```

## Validate the config

The workload identity configuration is in place and we are ready to use it. First, we'll create a Google Cloud Storage (GCS) bucket and upload a file.

```shell
gsutil mb -p $gcp_project gs://le-application-stg/
echo foo > my_application_file
gsutil cp my_application_file gs://le-application-stg/
```

Now let's run a Pod and try to download the file from the GCS bucket just created:

```shell
kubectl run -it --rm --restart=Never \
  --serviceaccount le-application \
  --image google/cloud-sdk:slim bb8 -n staging

root@bb8:/ gsutil cp gs://le-application-stg/my_application_file .
AccessDeniedException: 403 le-application-stg@my-project.iam.gserviceaccount.com does not have storage.objects.list access to the Google Cloud Storage bucket.
```

It fails. To fix it we grant the GSA access to the bucket:

```shell
gsutil iam ch serviceAccount:le-application-stg@${gcp_project}.iam.gserviceaccount.com:objectViewer gs://le-application-stg/
```

From now on, we manage access to services and APIs using the GSA and Google IAM and in Kubernetes we grant access to applications using the KSA.

Run again and verify:

```shell
kubectl run -it --rm --image google/cloud-sdk:slim bb8 --serviceaccount le-application -n staging --restart=Never
root@bb8:/ gsutil cp gs://le-application-stg/my_application_file .
root@bb8:/ cat my_application_file
```

## Clean up

```shell
gsutil rm -r gs://le-application-stg
gcloud iam service-accounts delete le-application-stg@${gcp_project}.iam.gserviceaccount.com
gcloud container clusters delete $cluster_name --region $cluster_region
```

## Next Steps

If you liked this content you might be interested to check my tutorial [Restrict Google Kubernetes Engine Workload Identity with Kyverno](https://cloud.google.com/community/tutorials/restrict-workload-identity-with-kyverno) on the Google Cloud Community tutorials.
