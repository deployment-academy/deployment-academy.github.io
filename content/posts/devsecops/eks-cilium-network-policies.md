---
title: "Using Network Policies in EKS with Cilium"
description: "In this tutorial, we will deploy Cilium to an Amazon EKS Kubernetes cluster and limit traffic to Pods using Network Policies."
date: 2021-09-26T14:15:28-04:00
lastmod: 2021-09-26T14:15:28-04:00
draft: false
sidebar: "right"
widgets:
  - "ddg-search"
  - "recent"
  - "social"
tags:
  - "devsecops"
  - "kubernetes"
  - "aws"
  - "eks"
  - "network policies"
  - "cilium"
---

In this tutorial, we will deploy [Cilium](https://cilium.io/) to an [Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html) Kubernetes cluster and limit traffic to Pods using [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/).

Network Policies are not available in Kubernetes out-of-the-box. To leverage Network Policies, you must use a [network plugin](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/) that implements such a feature - here is where Cilium comes into play.

It's worth noting that while this is our scope for this tutorial, Cilium is not limited to Network Policies and not even limited to Kubernetes. Check the [Introduction to Cilium](https://docs.cilium.io/en/stable/intro/) to learn more about it and the [Kubernetes Integration doc](https://docs.cilium.io/en/stable/concepts/kubernetes/intro/) about how Cilium works with Kubernetes.

<!--more-->

## Prerequisites

To follow this tutorial, you need an [AWS Account](https://aws.amazon.com/account/) and credentials to create and manage an EKS cluster properly provisioned and installed in your local environment. You also need the `eksctl` and `kubectl` CLIs installed.

Check the [Getting started with Amazon EKS – eksctl Prerequisites](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html#eksctl-prereqs) section if you need help.

## Create the EKS cluster

{{< notice type="warning" id="minimal-configuration" title="Quick note about minimal configuration" >}}
Note that in this tutorial (and all other tutorials in this blog, unless said the contrary), we will use minimal configuration to get things running quickly and focus on the feature we are evaluating. **Never use this configuration to set up a production environment.** Refer to the official installation guides linked in the tutorial for more thorough instructions.
{{< /notice >}}

Declare some variables that we'll use throughout the tutorial.

```shell
cluster_name=le-cluster
cluster_region=us-east-1  
```

Create the cluster.

```shell
eksctl create cluster --name $cluster_name --region $cluster_region
```

Check the `eksctl` docs for more details about [creating and managing EKS clusters](https://eksctl.io/usage/creating-and-managing-clusters/).

Verify that your kube-config is currently pointing to the created cluster.

```shell
kubectl config current-context
```

You can also check the nodes for more details about your installation.

```shell
kubectl get nodes -o wide
```

## Install Cilium

Download the Cilium CLI.

```shell
curl -L https://github.com/cilium/cilium-cli/releases/latest/download/cilium-darwin-amd64.tar.gz \
  -o cilium-darwin-amd64.tar.gz ; tar xzvf $_
```

Run the install command.

```shell
./cilium install
[...]
Auto-detected IPAM mode: eni
Auto-detected datapath mode: aws-eni
Custom datapath mode: aws-eni
[...]
```

The `cilium install` command uses your current kube context to detect the best configuration for your Kubernetes distribution and perform the installation. As you can see in the partial output listed above, Cilium is being installed on [AWS ENI](https://docs.cilium.io/en/stable/concepts/networking/ipam/eni/#ipam-eni) mode. It's also possible to use Cilium with the [AWS VPC CNI plugin](https://docs.cilium.io/en/stable/gettingstarted/cni-chaining-aws-cni/#chaining-aws-cni) - refer to the docs for more details.

When the installation finishes, you can check that Cilium is running with the CLI using:

```shell
./cilium status --wait
```

and validate the installation with:

```shell
./cilium connectivity test
```

To manually check that Cilium is running you can also check that the Cilium Pods are in running status.

```shell
kubectl get po -n kube-system

NAMESPACE     NAME                               READY   STATUS    RESTARTS   AGE
kube-system   cilium-9vnf8                       1/1     Running   0          3m43s
kube-system   cilium-bjdb4                       1/1     Running   0          3m43s
kube-system   cilium-operator-7c9b465f5c-blk4w   1/1     Running   0          3m43s
kube-system   coredns-7d74b564bd-2d2t9           1/1     Running   0          3m34s
kube-system   coredns-7d74b564bd-tkxgp           1/1     Running   0          3m19s
kube-system   kube-proxy-cnvph                   1/1     Running   0          23m
kube-system   kube-proxy-hzkt9                   1/1     Running   0          23m
```

Note that when running Cilium in an autoscaling cluster, you'll want to add the `node.cilium.io/agent-not-ready=true:NoSchedule` taint to the nodes to prevent that Pods are scheduled in the node before Cilium is ready. Check the [Cilium installation guide](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/) for more details.

## Deploy a Demo Application

Let's deploy a demo application to perform some tests.

```shell
git clone https://github.com/microservices-demo/microservices-demo

kubectl create namespace sock-shop

kubectl apply -f microservices-demo/deploy/kubernetes/complete-demo.yaml
```

[Sock Shop](https://microservices-demo.github.io/) is an application that implements an online socks shop and is usually used to demonstrate and test microservices and cloud-native technologies.

Wait until all the Pods are in running status.

```shell
watch 'kubectl get po -n sock-shop'
NAME                            READY   STATUS    RESTARTS   AGE
carts-b4d4ffb5c-t8w2r           1/1     Running   0          3m45s
carts-db-6c6c68b747-5wdm7       1/1     Running   0          3m45s
catalogue-5bd97b4988-dsg6b      1/1     Running   0          3m45s
catalogue-db-96f6f6b4c-dbwts    1/1     Running   0          3m45s
front-end-6649c54d45-p7c66      1/1     Running   0          3m44s
orders-7664c64d75-ffw4d         1/1     Running   0          3m44s
orders-db-659949975f-x7cqq      1/1     Running   0          3m44s
payment-dd67fc96f-rvrx4         1/1     Running   0          3m44s
queue-master-5f6d6d4796-vx4f5   1/1     Running   0          3m44s
rabbitmq-5bcbb547d7-z7bxf       2/2     Running   0          3m43s
session-db-7cf97f8d4f-w8mhg     1/1     Running   0          3m43s
shipping-7f7999ffb7-4bnhj       1/1     Running   0          3m43s
user-888469b9f-lcv4n            1/1     Running   0          3m43s
user-db-6df7444fc-77ht7         1/1     Running   0          3m43s
```

When the Pods are running, you can optionally test it locally by port-fowarding the `frond-end` service.

```shell
kubectl port-forward service/front-end -n sock-shop 8080:80
```

You can now access it on the browser at [localhost:8080](http://localhost:8080). Use the username and password `user / password` to login.

## Lateral Movement

As you can see in the [Architecture Diagram](https://github.com/microservices-demo/microservices-demo/blob/master/internal-docs/design.md#architecture), this application has a few components. Among them is the `user` component that interacts with the `user-db`, which is a [MongoDB](https://www.mongodb.com/) instance used to store users' information.

We will play the role of a malicious actor that has gained access to create a Pod in our cluster and use this access to connect to the `user-db`.

List the services and observe the `users-db` service responding on port `27017`.

```shell
kubectl get svc -n sock-shop
```

Let's launch a temporary Pod in interactive mode using the `mongo` image to execute commands against the `user-db`.

```shell
kubectl run rogue-1 --image=mongo:3.2 --restart=Never --rm -it -- /bin/bash
```

From inside the container run the following command to connect to the MongoDB instance:

```shell
root@rogue-1:/# mongo --host user-db.sock-shop.svc.cluster.local:27017
```

Now, from the mongo session, run:

```shell
> show dbs
local  0.000GB
users  0.000GB

> use users
switched to db users

> show collections
addresses
cards
customers

> db.cards.find()
{ "_id" : ObjectId("57a98d98e4b00679b4a830ae"), "longNum" : "5953580604169678", "expires" : "08/19", "ccv" : "678" }
{ "_id" : ObjectId("57a98d98e4b00679b4a830b1"), "longNum" : "5544154011345918", "expires" : "08/19", "ccv" : "958" }
{ "_id" : ObjectId("57a98d98e4b00679b4a830b4"), "longNum" : "0908415193175205", "expires" : "08/19", "ccv" : "280" }
{ "_id" : ObjectId("57a98ddce4b00679b4a830d2"), "longNum" : "5429804235432", "expires" : "04/16", "ccv" : "432" }
```

## Kubernetes Network Policies

Let's use a [Kubernetes Network Policy](https://kubernetes.io/docs/concepts/services-networking/network-policies/) to restrict ingress traffic to the `users-db` Pod to only traffic from the `users` Pod.

To create this Network Policy, we need to know the labels used for each of these Pods. We can check this by running:

```shell
kubectl describe deploy user-db -n sock-shop
Name:                   user-db
Namespace:              sock-shop
[...]
Pod Template:
  Labels:  name=user-db
  Containers:
[...]

kubectl describe deploy user -n sock-shop
Name:                   user
Namespace:              sock-shop
[...]
Pod Template:
  Labels:  name=user
  Containers:
[...]

```

Note the Pod labels `name=user-db` and `name=user`.

Create the Network Policy.

```shell
kubectl apply -f- << EOF
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: user-db-allow-from-user
  namespace: sock-shop
spec:
  policyTypes:
  - Ingress
  PodSelector:
    matchLabels:
      name: user-db
  ingress:
  - from:
    - PodSelector:
        matchLabels:
          name: user
EOF
```

Create a temporary Pod again and try to connect to the MongoDB instance.

```shell
kubectl run rogue-1 --image=mongo:3.2 --restart=Never --rm -it -- /bin/bash

root@rogue-1:/# mongo --host user-db.sock-shop.svc.cluster.local:27017
[...]
```

It should fail now.

{{< notice type="tip" id="k8s-security" title="A note on Kubernetes Security" >}}
The goal of this tutorial is to check how Cilium works with EKS allowing us to experiment with Network Policies and other features. The focus is not to provide best practices on how to secure your Kubernetes cluster. For instance, notice that a malicious actor, depending on the level of access that has been granted, can use different approaches to bypass this policy and get access to PII data or lead to service disruption. To properly secure your cluster, you should consider a combination of techniques such as using an [Admission Controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/) (e.g. [Gatekeeper](https://www.openpolicyagent.org/docs/latest/kubernetes-introduction/#what-is-opa-gatekeeper) or [Kyverno](https://kyverno.io/docs/introduction/)); tuned RBAC permissions; proper monitoring and alerting on suspicious behavior (e.g. [Falco](https://falco.org/docs/)); among others.
{{< /notice >}}

Access the application on the browser at [localhost:8080](http://localhost:8080) and verify that everything is still working fine.

Besides being able to use standard Kubernetes Network Policies, Cilium also allows you to use [CiliumNetworkPolicy](https://docs.cilium.io/en/stable/concepts/kubernetes/policy/#ciliumnetworkpolicy) and [CiliumClusterwideNetworkPolicy](https://docs.cilium.io/en/stable/concepts/kubernetes/policy/#ciliumclusterwidenetworkpolicy) that extend the standard policies, providing [more capabilities](https://docs.cilium.io/en/stable/concepts/kubernetes/intro/#what-does-cilium-provide-in-your-kubernetes-cluster). See the [Network Policies tutorials](https://docs.cilium.io/en/stable/gettingstarted/#network-policy-security-tutorials) for examples. It's also worth taking a look at the [functionality overview doc](https://docs.cilium.io/en/stable/intro/#functionality-overview) for a list of Cilium features that go beyond Network Policies.

## Clean up

```shell
eksctl delete cluster --name $cluster_name --region $cluster_region
```
