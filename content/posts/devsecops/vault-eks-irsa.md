---
title: "HashiCorp Vault AWS Auth with Amazon EKS and IAM Roles for Service Accounts"
description: "Configure and explore the HashiCorp Vault AWS Auth method with Amazon EKS"
date: 2021-05-23
lastmod: 2021-08-22
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
  - "vault"
---

In this tutorial, we are going to configure and explore the [HashiCorp Vault AWS Auth method](https://www.vaultproject.io/docs/auth/aws) with [Amazon EKS](https://aws.amazon.com/eks). We will start performing the Vault authentication using the EC2 instances (Kubernetes nodes) identity and later we will use a Kubernetes service account to impersonate an AWS IAM Role and have more fine-grained control at the Pod level.

Before you begin, notice that this is a hands-on tutorial that focuses on configuring and playing around with the features. To learn more about HashiCorp Vault, Amazon EKS, or IAM Roles for Service Accounts, please, refer to the documentation linked in the text.

## Context

Using the Vault AWS Auth method with [IAM Roles for Service Accounts](https://aws.amazon.com/blogs/opensource/introducing-fine-grained-iam-roles-service-accounts/) allows you to read secrets from Vault using a token served based on a trust relationship with your workload identify (kubernetes namespace and service account) running on EKS. The clear benefit with this approach is that you don't need to deploy static credentials along with your applications such as a token or user password.

## Prerequisites

To execute this tutorial you need:

* An AWS Account
* Local environment configured with AWS credentials with the required IAM permissions to manage EKS and IAM resources
* Necessary CLI tools installed (awscli, eksctl and kubectl).

Check the [Getting started with Amazon EKS – eksctl Prerequisites](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html#eksctl-prereqs) section if you need help.

## Create the EKS cluster

We will start creating a simple EKS cluster with a managed Linux node group. See the [Getting started with Amazon EKS – eksctl](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html) for more details.

Declare some variables that we'll use throughout the tutorial.

```shell
CLUSTER_NAME=le-cluster
ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)
```

Create the cluster.

```shell
eksctl create cluster --name $CLUSTER_NAME \
    --with-oidc --managed
# output omitted
```

This is going to take several minutes. When the cluster is ready your `kubectl` will have access configured to the newly created cluster.

Check the access to the cluster.

```shell
kubectl get nodes
# output omitted

kubectl get po -A
# output omitted
```

Check the caller identity of a Pod running in the cluster.

```shell
kubectl run awscli -it --rm --restart=Never \
    --image amazon/aws-cli sts get-caller-identity

If you don't see a command prompt, try pressing enter.
{
    "UserId": "AROA000000425TR6IXYLJ:i-065600000008806af",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/eksctl-le-cluster-nodegroup-ng-b0-NodeInstanceRole-146NXG82R7UOU/i-065600000008806af"
}
pod "awscli" deleted
```

Note that the ARN in the output is the Node IAM Role ARN of the Node Group configuration created in the `eksctl create cluster` command. By default, all pods running on the Kubernetes node will assume the same role and share the same set of permissions.

## Configure the Vault AWS Auth method

The AWS auth method provides an automated mechanism to retrieve a Vault token for IAM principals and AWS EC2 instances. In other words, a client running on AWS will use the instance identity, usually through the [metadata service](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instancedata-data-retrieval.html), to communicate with Vault and retrieve a token. Check the [Vault AWS Auth docs](https://www.vaultproject.io/docs/auth/aws) for a more detailed explanation of the authentication workflow and different options.

The first step to configure the AWS authentication method in Vault is to provide AWS credentials to Vault. So, before we start the Vault server, let's create a policy and a user in AWS for Vault.

Create the AWS policy.

```shell

VAULT_AUTH_POLICY_NAME=vault-auth-policy

cat <<EOF > $VAULT_AUTH_POLICY_NAME.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "",
            "Effect": "Allow",
            "Action": [
                "iam:GetUser",
                "iam:GetRole",
                "iam:GetInstanceProfile",
                "ec2:DescribeInstances"
            ],
            "Resource": "*"
        }
    ]
}
EOF

VAULT_AUTH_POLICY_ARN=$(aws iam create-policy \
    --policy-name $VAULT_AUTH_POLICY_NAME \
    --description "Policy used by the Vault user to check instance identity" \
    --policy-document file://$VAULT_AUTH_POLICY_NAME.json | jq -r .Policy.Arn)

```

This policy is a simplified version of the [recommended Vault IAM policy](https://www.vaultproject.io/docs/auth/aws#recommended-vault-iam-policy) from the docs.

Create the AWS user and credentials.

```shell
VAULT_AUTH_USER_NAME=vault-auth

aws iam create-user --user-name $VAULT_AUTH_USER_NAME

aws iam attach-user-policy --user-name $VAULT_AUTH_USER_NAME --policy-arn $VAULT_AUTH_POLICY_ARN

aws iam create-access-key --user-name $VAULT_AUTH_USER_NAME > access_key.json

# output omitted

```

Now we are ready to start Vault and configure the auth method.

For the purpose of this tutorial, we'll run Vault locally in dev mode and make it reachable from AWS using [ngrok](https://ngrok.com/download).

Open a different terminal session and start the Vault server using Docker. We will keep this session running during the entire tutorial.

```shell
docker run --cap-add=IPC_LOCK --rm -it \
    -p 8200:8200 \
    -e VAULT_DEV_ROOT_TOKEN_ID=mytoken \
vault

# output omitted
```

Open another terminal session. Make sure you are in the same folder we started the tutorial since we are going to use the `access_key.json` file created before. We will use this container to execute the Vault client commands.

```shell
docker run --rm -it \
    -e VAULT_ADDR='http://localhost:8200' \
    -e VAULT_TOKEN=mytoken \
    -e VAULT_AUTH_ACCESS_KEY_ID=$(jq -r .AccessKey.AccessKeyId < access_key.json) \
    -e VAULT_AUTH_SECRET_ACCESS_KEY=$(jq -r .AccessKey.SecretAccessKey < access_key.json) \
    --net=host vault /bin/sh
    
# output omitted
```

The following commands will run from inside the container. Notice in the command above that, for convenience, we are exporting the secret and access key to the container environment. It's just a convenience so we don't need to copy and paste this data.

Enable and configure the AWS auth method.

```shell
vault auth enable aws

vault write auth/aws/config/client \
    secret_key=$VAULT_AUTH_SECRET_ACCESS_KEY \
    access_key=$VAULT_AUTH_ACCESS_KEY_ID

# output omitted
```

## Optional: Authenticate using the Node IAM Role

Still from inside the container, we are going to create a Vault role and bind the EC2 data.

```shell
vault write auth/aws/role/dev-role-iam auth_type=iam \
    inferred_entity_type=ec2_instance \
    inferred_aws_region=us-east-1 \
    bound_account_id="123456789012" \
    bound_iam_instance_profile_arn="arn:aws:iam::123456789012:instance-profile/eks1-*" \
    policies=dev \
    max_ttl=768h
```

The instance profile ARN provided in the `bound_iam_instance_profile_arn` field is from the Node IAM Role and identifies the EC2 instances of the Node group. We can verify this instance profile ARN with the following command.

```shell
aws ec2 describe-instances \
    --filters Name=tag:"eks:cluster-name",Values=$CLUSTER_NAME \
    --query 'Reservations[0].Instances[0].IamInstanceProfile.Arn' \
    --output text
```

Note that you need to fill the other values for `inferred_aws_region` and `bound_account_id` in the Vault role with your own. Also, note that the Vault policy `dev` used in the role does not exist in this Vault instance, it's just to illustrate.

Start ngrok in a different terminal session.

```shell
./ngrok http 8200

Session Status                online
Session Expires               1 hour, 59 minutes
Version                       2.3.40
Region                        Mars (mars)
Web Interface                 http://127.0.0.1:4040
Forwarding                    http://2a7f0319443a.ngrok.io -> http://localhost:8200
Forwarding                    https://2a7f0319443a.ngrok.io -> http://localhost:8200
```

Perform the Vault login from a pod running in EKS (make sure you use the ngrok address as the Vault address).

```shell
kubectl run vault -it --rm --restart=Never --image vault -- \
    vault login -address=http://2a7f0319443a.ngrok.io \
    -method=aws \
    role="dev-role-iam"

[...]
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                      Value
---                      -----
token                    s.A4HmMdld2oOdRdewylaj6qLy
token_accessor           QaNhxJxywmUasNFMmihdgoBB
token_duration           768h
token_renewable          true
token_policies           ["default" "dev"]
identity_policies        []
policies                 ["default" "dev"]
token_meta_account_id    123456789012
token_meta_auth_type     iam
token_meta_role_id       231bdbec-7246-6c24-e255-b1065e2eaa74
pod "vault" deleted

```

You should see an output with the Vault token.

## Configure the AWS IAM Roles for Service Accounts

Now let's get to the AWS IAM Roles for Service Accounts configuration. Please, note that I'll configure the AWS IAM resources and k8s service account individually so we can have a better understanding of what is happening behind the scenes. But the eksctl has a command `eksctl create iamserviceaccount` that simplifies the following steps to a single command.

Get the OIDC provider identifier for your cluster. This is available because we used the `--with-oidc` flag when creating the cluster, otherwise we'd need to create the OIDC provider and associate to the cluster ([more about this here](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html)).

```shell
OIDC_PROVIDER=$(aws eks describe-cluster --name $CLUSTER_NAME \
    --query "cluster.identity.oidc.issuer" \
    --output text | sed -e "s/^https:\/\///")
```

Create the trust policy and an IAM Role to be used by the Vault authentication.

```shell
VAULT_AUTH_ROLE_NAME=vault-auth
VAULT_AUTH_K8S_SERVICE_ACCOUNT=$VAULT_AUTH_ROLE_NAME

cat <<EOF > ${VAULT_AUTH_ROLE_NAME}.trust.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/${OIDC_PROVIDER}"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "${OIDC_PROVIDER}:sub": "system:serviceaccount:default:${VAULT_AUTH_K8S_SERVICE_ACCOUNT}"
        }
      }
    }
  ]
}
EOF

VAULT_AUTH_ROLE_ARN=$(aws iam create-role \
    --role-name $VAULT_AUTH_ROLE_NAME \
    --assume-role-policy-document file://${VAULT_AUTH_ROLE_NAME}.trust.json \
    --description "Defines Pod identity for Vault AWS authentication" \
    --query Role.Arn \
    --output text)
```

Now let's create and annotate a Kubernetes service account with this IAM Role and validate the caller identity within a pod.

```shell
kubectl create sa $VAULT_AUTH_K8S_SERVICE_ACCOUNT

kubectl annotate sa $VAULT_AUTH_K8S_SERVICE_ACCOUNT eks.amazonaws.com/role-arn=$VAULT_AUTH_ROLE_ARN

kubectl run awscli -it --rm --restart=Never --serviceaccount $VAULT_AUTH_K8S_SERVICE_ACCOUNT --image amazon/aws-cli sts get-caller-identity

{
    "UserId": "AROA6HH0000000CRMM46T:botocore-session-1621778135",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/vault-auth/botocore-session-1621778135"
}
pod "awscli" deleted
```

You should see the ARN output of the last command referring to the previously created IAM role for Vault.

Now, let's get back to our Vault instance running locally and create a Vault role to satisfy this new identity.

Start a container to perform the Vault client commands. Note that this container is a convenience to use the Vault client, if you have a Vault client installed in your machine you can just point it to the Vault server running in Docker.

```shell
docker run --rm -it \
    -e VAULT_ADDR='http://localhost:8200' \
    -e VAULT_TOKEN=mytoken \
    --net=host vault /bin/sh
```

Let's create the Vault AWS role. Change the `bound_iam_principal_arn` below with the Vault IAM Role created before - you can check this out with `echo $VAULT_AUTH_ROLE_ARN`. Run from inside the container.

```shell
vault write auth/aws/role/dev-role-iam2 auth_type=iam \
    bound_iam_principal_arn=arn:aws:iam::123456789012:role/vault-auth \
    policies=dev \
    max_ttl=768h
```

Exit the container.

Make sure ngrok is still running and the URL is still the same.

Run a pod to perform the Vault authentication using the new role and the pod identity.

```shell
kubectl run vault -it --rm --restart=Never \
    --serviceaccount $VAULT_AUTH_K8S_SERVICE_ACCOUNT \
    --image vault -- \
    vault login -address=http://2a7f0319443a.ngrok.io \
    -method=aws \
    role="dev-role-iam2"
    
If you don't see a command prompt, try pressing enter.
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                      Value
---                      -----
token                    s.RSqVT87jxMwPjZOS9gv6N7eg
token_accessor           gyU6SDHP3YJ6yIDeGESc0loO
token_duration           768h
token_renewable          true
token_policies           ["default" "dev"]
identity_policies        []
policies                 ["default" "dev"]
token_meta_account_id    123456789012
token_meta_auth_type     iam
token_meta_role_id       f4588750-45cc-19f2-b7d1-1c570979dcc3
pod "vault" deleted
```

If everything is ok, you should see the Vault token as in the output above. Make changes in the Vault role and note how the login will fail if you provide an invalid principal ARN. You can also experiment with a different non-authorized Kubernetes service account to see what happens.

## Optional: IAM Roles for Service Accounts Overview

Instead of using the Node IAM Role, the IAM Roles for Service Accounts feature makes pods first-class citizens in IAM, enabling the AWS identity APIs to recognize Kubernetes pods. This approach uses an OpenID Connect (OIDC) identity provider and Kubernetes service account annotations to allow a Pod to assume an AWS IAM role.

The flow works as follows, from the docs:
> OIDC federation allows the user to assume IAM roles with the Secure Token Service (STS), effectively receiving a JSON Web Token (JWT) via an OAuth2 flow that can be used to assume an IAM role with an OIDC provider. In Kubernetes we then use projected service account tokens, which are valid OIDC JWTs, giving each pod a cryptographically-signed token which can be verified by STS against the OIDC provider for establishing identity.

To simplify things on our side, a mutating admission controller running in EKS (via a webhook) automatically injects in the pod the necessary environment variables and a volume with a [projected service account token](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#service-account-token-volume-projection). The admission controller is just a utility, this step can also be made manually.

Check the blog post [Introducing fine-grained IAM roles for service accounts](https://aws.amazon.com/blogs/opensource/introducing-fine-grained-iam-roles-service-accounts/) and the [AWS IAM Roles for Service Accounts documentation](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts-technical-overview.html) for more details.

## Optional: Terraform configuration

In this tutorial we used the AWS and Vault CLI's for the configuration. If you are interested in the Terraform approach for the same configuration we did here, please, check [this gist](https://gist.github.com/soeirosantos/1ac8478c917ea47bd092974c9a96c003).

## Clean up

To avoid costs with EKS and to remove the IAM resources created in this tutorial, execute the following commands.

```shell
eksctl delete cluster --name $CLUSTER_NAME
aws iam detach-user-policy --user-name vault-auth --policy-arn $VAULT_AUTH_POLICY_ARN
aws iam delete-policy --policy-arn $VAULT_AUTH_POLICY_ARN
aws iam delete-access-key --user-name $VAULT_AUTH_USER_NAME --access-key-id="$(jq -r .AccessKey.AccessKeyId < access_key.json)"
aws iam delete-user --user-name $VAULT_AUTH_USER_NAME
aws iam delete-role --role-name $VAULT_AUTH_ROLE_NAME
```
