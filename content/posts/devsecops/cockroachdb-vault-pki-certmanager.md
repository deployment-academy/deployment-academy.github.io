---
title: "CockroachDB with Vault PKI and cert-manager"
description: "In this tutorial we are going to spin up a CockroachDB secure cluster running in Kubernetes with certificates managed by HashiCorp Vault and issued by cert-manager."
date: 2022-11-25T12:14:53-05:00
lastmod: 2022-11-25T12:14:53-05:00
draft: false
toc: true
tags:
  - "devsecops"
  - "vault"
  - "certificate management"
  - "pki"
  - "cockroachdb"
  - "kubernetes"
  - "cert-manager"
---

Recently, I have been experimenting with putting together a CockroachDB cluster using Vault PKI and cert-manager, and I
thought I'd take some time to organize my notes and share it here. This is a long tutorial because I'll drop here all
the experimentation process, but I hope the separation of the sections can help anyone who lands here looking for
specific insights in some parts of the setup. 

In this tutorial, we are going to spin up a CockroachDB secure cluster running in Kubernetes with certificates managed by
HashiCorp Vault and issued by cert-manager. Before we get to the final state, we are going to evolve the installation
step by step to understand how each component is contributing to the setup and what we are gaining.

We'll focus on the execution, assuming you are minimally familiar with these components. Links will be provided for
additional context when necessary.

This tutorial can be executed from your local machine, no cloud resources are necessary.

<!--more-->

## Prerequesites

Along this tutorial we'll use the following tools.

- [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
- [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
- [cockroach CLI](https://www.cockroachlabs.com/docs/v22.1/install-cockroachdb-linux)
- [helm](https://helm.sh/docs/intro/install/)
- [vault CLI](https://developer.hashicorp.com/vault/docs/install)

## Create a local Kubernetes cluster

We'll start creating a Kubernetes local cluster with Kind with the following profile:

- one control plane node, 
- three worker nodes dedicated to CockroachDB and
- one general-purpose worker node for other pods. 

This is completely optional for the main goal of this tutorial, but it's a scenario particularly interesting for a
CockroachDB deployment. **Feel free to skip it and use your preferred way to set up Kubernetes locally.**

Create a file with the following Kind configuration.

```shell
cat <<EOF >./kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: roaches
nodes:
  - role: control-plane
  - role: worker
    kubeadmConfigPatches:
      - |
        kind: JoinConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "role=roach"
          taints:
            - key: "dedicated"
              value: "roach"
              effect: "NoSchedule"
  - role: worker
    kubeadmConfigPatches:
      - |
        kind: JoinConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "role=roach"
          taints:
            - key: "dedicated"
              value: "roach"
              effect: "NoSchedule"
  - role: worker
    kubeadmConfigPatches:
      - |
        kind: JoinConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "role=roach"
          taints:
            - key: "dedicated"
              value: "roach"
              effect: "NoSchedule"
  - role: worker
    kubeadmConfigPatches:
      - |
        kind: JoinConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "role=general"
EOF
```

We are using the `role=roach` node label and the `dedicated=roach` taint to prepare the
nodes for the CockroachDB pods scheduling. See [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
for more details.

Create the Kubernetes cluster.

```shell
kind create cluster --config kind-config.yaml \
  --image kindest/node:v1.23.13@sha256:ef453bb7c79f0e3caba88d2067d4196f427794086a7d0df8df4f019d5e336b61
```

Reference for the [Kind configuration](https://kind.sigs.k8s.io/docs/user/configuration/),
[kind images](https://github.com/kubernetes-sigs/kind/releases)
and for the
[kubeadm JoinConfiguration config](https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta3/#kubeadm-k8s-io-v1beta3-JoinConfiguration).

## CockroachDB Cluster

In this section we are going to create a CockroachDB cluster with certificates issued by the `cockroach cert` command
([see reference](https://www.cockroachlabs.com/docs/stable/cockroach-cert.html)).
The approach followed here is described in the [CockroachDB docs](https://www.cockroachlabs.com/docs/stable/orchestrate-a-local-cluster-with-kubernetes.html?filters=manual)
with a few changes.

In a CockroachDB cluster, each node must be able to initiate HTTP requests to any of the others. In a normal operating
mode, these requests are TLS encrypted with mutual authentication, requiring that each node present its own certificate,
signed by the same CA. Check the
[PKI in CockroachDB](https://www.cockroachlabs.com/docs/v22.1/security-reference/transport-layer-security#pki-in-cockroachdb)
documentation for more details.

### Create the Certificates

Before we launch the cluster we need to create the certificates.

Create the CA certificate and key pair.

```shell
mkdir -p certs ca-key

cockroach cert create-ca \
  --certs-dir=certs \
  --ca-key=ca-key/ca.key
```

Create the certificate and key pair for the nodes. We are adding the domain names for the internal Kubernetes
communication. In a real scenario, you should add any other name that your cluster is expected to be reached.

```shell
cockroach cert create-node \
    localhost 127.0.0.1 \
    cockroachdb-public \
    cockroachdb-public.default.svc.cluster.local \
    \*.cockroachdb \
    \*.cockroachdb.default.svc.cluster.local \
    --certs-dir=certs \
    --ca-key=ca-key/ca.key
```

Create the client certificate and key for the
[root user](https://www.cockroachlabs.com/docs/stable/security-reference/authorization.html#root-user).

```shell
cockroach cert create-client \
  root \
  --certs-dir=certs \
  --ca-key=ca-key/ca.key
```

We can check the certificate created with OpenSSL.

```shell
openssl x509 -in certs/node.crt" -text | less
```

Note some defaults applied by cert command like the CN and validity. Check the list of subject alternative names provided. You can
also take a look at the client and CA certificates.

To mount these certificates into the Kubernetes pods we will create Kubernetes secrets out for the `certs` folder
containing the node certificate and private key pair, root user certificate and private key pair, and the CA certificate.
The name of this secret matches what is expected in the StatefulSet that we will download next.

Create the kubernetes secret.

```shell
kubectl create secret \
  generic cockroachdb.node \
  --from-file=certs
```

### Deploy CockroachDB

This is a manual simplified deployment on a local Kubernetes cluster. Check
the [documentation](https://www.cockroachlabs.com/docs/stable/kubernetes-overview.html) for more info about how to
deploy CockroachDB to Kubernetes.

Download the CockroachDB StatefulSet manifest.

```shell
mkdir cluster
curl -o cluster/cockroachdb-statefulset.yaml \
  https://raw.githubusercontent.com/cockroachdb/cockroach/master/cloud/kubernetes/bring-your-own-certs/cockroachdb-statefulset.yaml
```

For the main goal of this tutorial we could just apply this manifest, but we will make some adjustments to reduce the
resources requests and limits and to ensure that the CockroachDB pods get scheduled to the dedicated nodes. We'll use
[kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/) for this. **This customization is
optional, so feel free to skip these editions - you can make edits directly in the YAML file if you want.**

Create the following customization file.

```shell
cat <<EOF >cluster/kustomization.yaml
resources:
- cockroachdb-statefulset.yaml

patchesStrategicMerge:
- |-
  apiVersion: apps/v1
  kind: StatefulSet
  metadata:
    name: cockroachdb
  spec:
    template:
      spec:
        containers:
          - name: cockroachdb
            resources:
              requests:
                cpu: "300m"
                memory: "1Gi"
              limits:
                cpu: "300m"
                memory: "1Gi" 
        nodeSelector:
          role: roach
        tolerations:
        - key: "dedicated"
          operator: "Equal"
          value: "roach"
          effect: "NoSchedule"
EOF
```

Check the customization.

```shell
kubectl kustomize cluster
```

Apply it.

```shell
kubectl apply -k cluster
```

Check that the pods are running but no containers are ready until we initialize the CockroachDB cluster.

```shell
kubectl get po -o wide
NAME            READY   STATUS    RESTARTS   AGE     IP           NODE              NOMINATED NODE   READINESS GATES
cockroachdb-0   0/1     Running   0          2m39s   10.244.3.4   roaches-worker2   <none>           <none>
cockroachdb-1   0/1     Running   0          2m39s   10.244.4.4   roaches-worker    <none>           <none>
cockroachdb-2   0/1     Running   0          2m39s   10.244.1.3   roaches-worker3   <none>           <none>
```

Also note that the pods have been scheduled for the selected nodes (if you followed the customization above).

Initialize the cluster.

```shell
kubectl exec -it cockroachdb-0 \
  -- /cockroach/cockroach init \
  --certs-dir=/cockroach/cockroach-certs
```

Now if you check the pods again you should expect to see they are ready.
The cluster is operational so let's connect a client and check.

Before we deploy the client, create a Kubernetes secret with only the root client certificate and key to be mounted
into the client container.

```shell
kubectl create secret \
  generic cockroachdb.client.root \
  --from-file=certs/client.root.key \
  --from-file=certs/client.root.crt \
  --from-file=certs/ca.crt
```

Deploy the client pod.

```shell
mkdir client

curl -o client/cockroachdb-client.yaml \
  https://raw.githubusercontent.com/cockroachdb/cockroach/master/cloud/kubernetes/bring-your-own-certs/client.yaml

kubectl create \
  -f client/cockroachdb-client.yaml
```

Connect to it.

```shell
kubectl exec -it cockroachdb-client-secure \
  -- ./cockroach sql \
  --certs-dir=/cockroach-certs \
  --host=cockroachdb-public
```

Alternatively, you can use the `url` argument providing the connection string (it's easier to visualize what's going on
and validate some assumptions).

```shell
kubectl exec -it cockroachdb-client-secure \
  -- ./cockroach sql \
  --url="postgres://root@cockroachdb-public:26257/?sslmode=verify-full&sslcert=/cockroach-certs/client.root.crt&sslkey=/cockroach-certs/client.root.key&sslrootcert=/cockroach-certs/ca.crt"
```

In the CockroachDB terminal use the following commands to try it out:

```shell
CREATE DATABASE bank;
CREATE TABLE bank.accounts (id INT PRIMARY KEY, balance DECIMAL);
INSERT INTO bank.accounts VALUES (1, 1000.50);
SELECT * FROM bank.accounts;
```

{{< notice type="warning" id="too-many-files" title="Too many open files error" >}}
If you are on Linux and bump into the error `Failed to create inotify object: Too many open files`. Increase the
following values:

```shell
sudo sysctl fs.inotify.max_user_watches=1048576
sudo sysctl fs.inotify.max_user_instances=8192
```
{{< /notice >}}

### Checkpoint

What we have done so far:

- Created a local Kubernetes cluster
- Issued CA, node, and client certificates and key pairs for the CockroachDB cluster
- Created the Kubernetes secrets with the certificate and key pairs
- Deployed the CockroachDB cluster

The security of our CockroachDB cluster relies on these certificates. If any certificate key pair (CA, nodes or root
client) gets compromised a threat actor will have access to data or the ability to disrupt operations.
From time to time, we'll have to rotate these certificates before they expire or when they get compromised.

## HashiCorp Vault PKI

The next step is to replace the certificate generation with the Vault PKI. There are a couple of reasons you might want to
do that. One of the main reasons is that you can use the Vault access control to grant access to the CA via Vault PKI
roles allowing node and client operators to issue certificates in a self-service fashion while ensuring the principle of
least privilege (no one needs direct access to the CA key or is allowed to generate certificates for clusters they are
not authorized). The Vault PKI will also offer a smooth path to automating the entire process.

### Deploy Vault

Let's start installing and configuring Vault.

Add the HashiCorp Helm repository.

```shell
helm repo add hashicorp https://helm.releases.hashicorp.com
```

Install Vault.

```shell
helm install vault hashicorp/vault --set "injector.enabled=false"
```

Check that the pod is running but no the container is not ready.

```shell
kubectl get po -l app.kubernetes.io/name=vault -o wide
```

Initialize.

```shell
kubectl exec vault-0 -- vault operator init -key-shares=1 -key-threshold=1 \
  -format=json > init-keys.json
```

Unseal.

```shell
VAULT_UNSEAL_KEY=$(cat init-keys.json | jq -r ".unseal_keys_b64[]")

kubectl exec vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY
```

We are going to run the Vault commands from our machine pointing to the Vault instance running on Kubernetes.
For that, we'll use the `kubectl port-forward` command. 

Note that the approach described here is for simplicity and demonstrations only. For a complete view of how to deploy and
access Vault on Kubernetes check the
[HashiCorp Vault Kubernetes Reference Architecture](https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-reference-architecture).

Open a different terminal and execute.

```shell
kubectl port-forward vault-0 8200:8200
```

Keep it running.

Now, go back to the previous terminal session and export the following variables.

```shell
export VAULT_ADDR=http://127.0.0.1:8200
export VAULT_TOKEN=$(cat init-keys.json | jq -r ".root_token") 
```

Check your access to Vault.

```shell
vault secrets list
```

You should see the list of default secret mounts.

### Configure Vault PKI

Now let's jump into what really matters: the PKI configuration and issuing of the CockroachDB certificates.

Let's enable and configure the PKI backend.

```shell
vault secrets enable -path=roach/pki pki
```

Note that we enable a backend that identifies the CockroachDB purpose by its path. That’s a questionable decision.
I don’t get into too much detail about how to organize the PKI backends. 
But this setup will be enough to ensure access segregation at the level we want using Vault policies in this tutorial.

Let's tune the backend to set the max TTL for certificates to 1 year instead of the default 30 days.

```shell
vault secrets tune -max-lease-ttl=8760h roach/pki

# you can check the tune config with
vault read sys/mounts/roach/pki/tune
```

Let's generate a self-signed root CA.

```shell
vault write roach/pki/root/generate/internal ttl=8760h
```

This is probably a very naive approach to mount the PKI backend and generate the root CA. Refer to the
[Vault PKI documentation](https://developer.hashicorp.com/vault/tutorials/secrets-management/pki-engine) for more
details about how to set up a CA and manage the PKI backend.

Configure the PKI secrets engine certificate issuing and certificate revocation list (CRL) endpoints.

```shell
vault write roach/pki/config/urls \
    issuing_certificates="http://vault-internal.default.svc.cluster.local:8200/v1/roach/pki/ca" \
    crl_distribution_points="http://vault-internal.default.svc.cluster.local:8200/v1/roach/pki/crl"
```

The next step is to configure the Vault PKI node and client roles.

Let's start with the node role.

```shell
vault write "roach/pki/roles/cockroachdb_nodes" \
    allow_wildcard_certificates=true \
    client_flag=true \
    server_flag=true \
    require_cn=true \
    cn_validations="disabled" \
    enforce_hostnames=false \
    allow_localhost=true \
    ttl=48h \
    max_ttl=8760h \
    allowed_domains="cockroachdb-public,cockroachdb-public.default.svc.cluster.local,*.cockroachdb,*.cockroachdb.default.svc.cluster.local"
```

The client role.

```shell
vault write roach/pki/roles/cockroachdb_client \
    allow_bare_domains=true \
    allow_wildcard_certificates=false \
    client_flag=true \
    server_flag=false \
    require_cn=true \
    cn_validations="disabled" \
    enforce_hostnames=false \
    allow_ip_sans=false \
    allow_localhost=false \
    ttl=48h \
    max_ttl=8760h \
    allowed_domains="cockroachdb-public,cockroachdb-public.default.svc.cluster.local"
```

### Create the Certificates

Let's proceed and generate the certificates.

Create a different directory to store the raw Vault output and the certificates and generate the nodes certificate.

```shell
mkdir -p certs/vault/json
```

```shell
vault write roach/pki/issue/cockroachdb_nodes \
    common_name="node" \
    alt_names="localhost,cockroachdb-public,cockroachdb-public.default.svc.cluster.local,*.cockroachdb,*.cockroachdb.default.svc.cluster.local" \
    exclude_cn_from_sans=true \
    ttl=24h \
    -format=json > certs/vault/json/node.json
```

Extract the CA, certificate, and key for the nodes from the Vault json.

```shell
jq -r .data.issuing_ca certs/vault/json/node.json > certs/vault/ca.crt
jq -r .data.certificate certs/vault/json/node.json > certs/vault/node.crt
jq -r .data.private_key certs/vault/json/node.json > certs/vault/node.key
```

Generate the root client certificate.

```shell
vault write roach/pki/issue/cockroachdb_client \
    common_name=root \
    exclude_cn_from_sans=true \
    alt_names="cockroachdb-public,cockroachdb-public.default.svc.cluster.local" \
    -format=json > certs/vault/json/client.root.json
```

Extract the certificate and key for the root client from the Vault json.

```shell
jq -r .data.certificate certs/vault/json/client.root.json > certs/vault/client.root.crt
jq -r .data.private_key certs/vault/json/client.root.json > certs/vault/client.root.key
```

Let's clean up our CockroachDB installation so we can start from scratch and ensure a clean test.

```shell
kubectl delete sts cockroachdb
kubectl delete pvc -l app=cockroachdb
kubectl delete secrets cockroachdb.node

kubectl delete po cockroachdb-client-secure
kubectl delete secrets cockroachdb.client.root
```

Now follow the same steps we followed previously to deploy CockroachDB and test but create the Kubernetes secrets out
of the Vault certificates this time.

```shell
kubectl create secret \
  generic cockroachdb.node \
  --from-file=certs/vault

kubectl apply -k cluster

kubectl exec -it cockroachdb-0 \
  -- /cockroach/cockroach init \
  --certs-dir=/cockroach/cockroach-certs

kubectl create secret \
  generic cockroachdb.client.root \
  --from-file=certs/vault/client.root.key \
  --from-file=certs/vault/client.root.crt \
  --from-file=certs/vault/ca.crt

kubectl create \
  -f client/cockroachdb-client.yaml

kubectl exec -it cockroachdb-client-secure \
  -- ./cockroach sql \          
  --certs-dir=/cockroach-certs \
  --host=cockroachdb-public

CREATE DATABASE bank;
CREATE TABLE bank.accounts (id INT PRIMARY KEY, balance DECIMAL);
INSERT INTO bank.accounts VALUES (1, 1000.50);
SELECT * FROM bank.accounts;
```

### Checkpoint

So far, we basically replaced the `cockroachdb cert` command with Vault. As mentioned before, it has some benefits, like
delegating certificate management to your already existing secrets management solution and making it easier to control
access to the CA via PKI roles restricting who can issue certificates (we didn't create any special policies yet).

Now we are going to add cert-manager to the setup and automate the flow end to end.

## Automatically Issuing / Renewing Certificates

### Install cert-manager

Following the same approach we used for CockroachDB and Vault we will use a simplified installation method. Check the
[installation documentation](https://cert-manager.io/docs/installation/) for more details.

Install cert-manager.

```shell
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.10.0/cert-manager.yaml
```

### Configure the Issuer access to Vault

cert-manager needs access to the same Vault PKI roles endpoints we used to issue the certificates. For that, we will
use the Vault Kubernetes Auth method with policies granting access to the issuer.

Enable the auth backend.

```shell
vault auth enable kubernetes
```

Retrieve the service address of the kube-apiserver.

```shell
kubernetes_port_443_tcp_addr=$(kubectl exec vault-0 -- sh -c 'echo -n $KUBERNETES_PORT_443_TCP_ADDR')
```

Configure the backend.

```shell
vault write auth/kubernetes/config \
    kubernetes_host="https://$kubernetes_port_443_tcp_addr:443"
```

For more details on configuring the [Kubernetes Auth Method](https://developer.hashicorp.com/vault/docs/auth/kubernetes),
check the Vault documentation.

So far, these steps are just preparation. Now we are going to configure the Vault policy and Kubernetes roles that allow
the cert-manager issuer to access the Vault PKI roles.

Note that we are going to use the same cert-manager issuer to issue the nodes and the root client certificates. For
other clients, you will likely want to create more specific Vault policies and a different cert-manager issuer with more
restricted privileges.

```shell
vault policy write cockroachdb-issuer - <<EOF
path "roach/pki/sign/cockroachdb_nodes" { 
    capabilities = ["create", "update"] 
}
path "roach/pki/issue/cockroachdb_nodes" { 
    capabilities = ["create"]
}
path "roach/pki/sign/cockroachdb_client" { 
    capabilities = ["create", "update"]
}
path "roach/pki/issue/cockroachdb_client" {
    capabilities = ["create"]
}
EOF
```

Create a Kubernetes service account to identify our cert-manager issuer.

```shell
kubectl create serviceaccount cockroachdb-issuer
```

Create the Vault Kubernetes role.

```shell
vault write auth/kubernetes/role/cockroachdb-issuer \
  bound_service_account_names=cockroachdb-issuer \
  bound_service_account_namespaces=default \
  policies=cockroachdb-issuer \
  ttl=20m
```

Ok, now we are going to configure the cert-manager Issuer object. The issuer needs the name of a service account token
associated with the service account that we configured for the Kubernetes role. We can use the default service account
token automatically generated (for k8s <v1.23) but instead we will create our own token, which despite being generally a
good practice is necessary for k8s >v1.24.

```shell
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: cockroachdb-issuer-token
  annotations:
    kubernetes.io/service-account.name: cockroachdb-issuer
EOF
```

### Deploy the Certificate Manager Configuration

Now let's proceed with the actual cert-manager configuration. Create the issuer for the node certificate.

```shell
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: cockroachdb-nodes-issuer
spec:
  vault:
    server: http://vault-internal.default.svc.cluster.local:8200
    path: roach/pki/sign/cockroachdb_nodes
    auth:
      kubernetes:
        mountPath: /v1/auth/kubernetes
        role: cockroachdb-issuer
        secretRef:
          name: cockroachdb-issuer-token
          key: token
EOF
```

Check the issuer - it should report ready.

```shell
kubectl get issuer cockroachdb-nodes-issuer
```

Create the cert-manager Certificate using the secret name we have been using for nodes certificates.
Delete the previous secret first `kubectl delete secrets cockroachdb.node`. If we don't delete the existing secret, the
Certificate object will append the new secret data to the existing secret.

```shell
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: cockroachdb.node
  namespace: default
spec:
  secretName: cockroachdb.node
  issuerRef:
    name: cockroachdb-nodes-issuer
  commonName: node
  dnsNames:
  - localhost
  - cockroachdb-public
  - cockroachdb-public.default.svc.cluster.local
  - "*.cockroachdb"
  - "*.cockroachdb.default.svc.cluster.local"
EOF
```

Create the client issuer.

```shell
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: cockroachdb-client-root-issuer
spec:
  vault:
    server: http://vault-internal.default.svc.cluster.local:8200
    path: roach/pki/sign/cockroachdb_client
    auth:
      kubernetes:
        mountPath: /v1/auth/kubernetes
        role: cockroachdb-issuer
        secretRef:
          name: cockroachdb-issuer-token
          key: token
EOF
```

Check the issuer - it should report ready.

```shell
kubectl get issuer cockroachdb-client-root-issuer
```

In the same way, create the cert-manager Certificate object using the secret name we have been using for the root client
certificate. Delete the previous secret first `kubectl delete secrets cockroachdb.client.root`.

```shell
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: cockroachdb.client.root
  namespace: default
spec:
  secretName: cockroachdb.client.root
  issuerRef:
    name: cockroachdb-client-root-issuer
  commonName: root
  dnsNames:
  - cockroachdb-public
  - cockroachdb-public.default.svc.cluster.local
EOF
```

Let's clean up, so we can perform a test creating everything from scratch.

```shell
kubectl delete sts cockroachdb
kubectl delete pvc -l app=cockroachdb
kubectl delete po cockroachdb-client-secure
```

Now we have to make some adjustments in our CockroachDB StatefulSet. Remember that we mounted both nodes and root client
certificates into the CockroachDB container from the same secret. Now the certificates are entirely separated into two
secrets. We need to change the StatefulSet to use a projected volume. And we also need to adjust the name of the files in
the file system - besides being duplicated in the different secrets, CockroachDB expects the files to be prefixed with
specific terms, or we are going to get this error
`bad filename ‹/cockroach/cockroach-certs/tls.crt›: unknown prefix ‹"tls"›`.

Adjust our StatefulSet customization.

```shell
cat <<EOF >cluster/kustomization.yaml
resources:
- cockroachdb-statefulset.yaml

patchesStrategicMerge:
- |-
  apiVersion: apps/v1
  kind: StatefulSet
  metadata:
    name: cockroachdb
  spec:
    template:
      spec: 
        containers:
        - name: cockroachdb
          resources:
            requests:
              cpu: "300m"
              memory: "1Gi"
            limits:
              cpu: "300m"
              memory: "1Gi" 
        nodeSelector:
          role: roach
        tolerations:
        - key: "dedicated"
          operator: "Equal"
          value: "roach"
          effect: "NoSchedule"
        volumes:
        - name: certs
          projected:
            defaultMode: 256
            sources:
            - secret:
                name: cockroachdb.node
                items:
                - key: ca.crt
                  path: ca.crt
                - key: tls.crt
                  path: node.crt
                - key: tls.key
                  path: node.key
            - secret:
                name: cockroachdb.client.root
                items:
                - key: tls.crt
                  path: client.root.crt
                - key: tls.key
                  path: client.root.key
          secret:
            $patch: delete
            defaultMode: 256
            secretName: cockroachdb.node
EOF
```

Apply, check and initialize.

```shell
kubectl apply -k cluster

kubectl exec -it cockroachdb-0 \
  -- /cockroach/cockroach init \
  --certs-dir=/cockroach/cockroach-certs
```

Now we also need to adjust the client pod we have been using. Add the following customization
to adjust the crt and key names.

```shell
cat <<EOF >client/kustomization.yaml
resources:
- cockroachdb-client.yaml

patchesStrategicMerge:
- |-
  apiVersion: v1
  kind: Pod
  metadata:
    name: cockroachdb-client-secure
  spec:
    volumes:
    - name: client-certs
      secret:
        secretName: cockroachdb.client.root
        items:
        - key: ca.crt
          path: ca.crt
        - key: tls.crt
          path: client.root.crt
        - key: tls.key
          path: client.root.key
        defaultMode: 256
EOF
```

Apply, connect and check.

```shell
kubectl apply -k client

kubectl exec -it cockroachdb-client-secure \
  -- ./cockroach sql \          
  --certs-dir=/cockroach-certs \
  --host=cockroachdb-public

CREATE DATABASE bank;
CREATE TABLE bank.accounts (id INT PRIMARY KEY, balance DECIMAL);
INSERT INTO bank.accounts VALUES (1, 1000.50);
SELECT * FROM bank.accounts;
```

### Final Checkpoint

Ok, now we don't need to ever bother again about issuing and distributing certificates again
for CockroachDB. However, note that CockroachDB doesn't automatically reload certificates. To reload the certificates
you need to perform a rolling restart of the cluster's pods. Another alternative that I haven't tried but sounds
feasible is to add a sidecar container that will send a SIGHUP signal to the cockroach process running in the
container. From the [docs](https://www.cockroachlabs.com/docs/stable/create-security-certificates-custom-ca.html):

> In a cluster deployed using the Kubernetes Operator, there is no way to send a SIGHUP signal to the individual
> cockroach process on each cluster node. Instead, perform a rolling restart of the cluster's pods.

Check the 
[Rotate Security Certificates documentation](https://www.cockroachlabs.com/docs/stable/rotate-certificates.html)
for more info.

## Bonus Challenge

Suppose you want to add a client application to access the CockroachDB cluster; add an issuer that issues/renews
certificates for this application.

<<<<<<< HEAD
- Add a client user to CockroachDB and grant read access.
- Create a Vault policy that allows access to the client PKI role but locks access to this specific user (identified by the CN).
- Create a Vault Kubernetes role associated with this policy and the application identity (k8s SA).
- Create a cert-manager Issuer and Certificate for the application.
- Launch a test container consuming this certificate and verify the setup.

## Other relevant links

Besides the links shared along the text, these are some other relevant links for reference:

- https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-cert-manager
- https://www.cockroachlabs.com/docs/stable/orchestrate-a-local-cluster-with-kubernetes.html?filters=manual
- https://www.cockroachlabs.com/docs/stable/manage-certs-vault.html
=======
- Create a Vault policy that allows access to the client PKI role but locks access to a specific CN.
- Create a Vault Kubernetes role associated with this policy and the application identity (k8s SA).
- Create a cert-manager Issuer for the application.
- Create the cert-manager certificate for the application.
- Launch a test container consuming the k8s secret created by this certificate and verify the setup.
>>>>>>> 97af209 (cockroachdb with vault pki + cert-manager)
