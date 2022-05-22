---
title: "IaC Security and Compliance with Regula"
description: "In this tutorial, we will review the Regula policy engine and see how to check infrastructure security and compliance rules even before it’s deployed."
date: 2021-10-16T23:37:41-04:00
lastmod: 2021-10-17T23:37:41-04:00
draft: false
tags:
  - "devsecops"
  - "kubernetes"
  - "aws"
  - "iac"
  - "opa"
  - "rego"
---

Infrastructure as Code (IaC) is an essential piece of the modern software development landscape. IaC tools allow teams to define their infrastructure in a reproducible-automated fashion, increasing speed and consistency while preventing errors, misconfiguration, and configuration drifts.

IaC also plays an important role in security. It gives visibility over the whole infrastructure, including its security aspects and components, even before they have been deployed. Development and/or security and operations teams can define sensible security defaults and a security baseline enforcing best practices that are automated and applied via the CICD pipeline, where changes can be reviewed and approved following the separation of duties principle.

<!--more-->

Following this idea of anticipating security checks in the CICD pipeline, we can then define policies that automatically run against our infrastructure code and check for compliance rules as a step of the pipeline. In this tutorial, we will review the [Regula](https://regula.dev/index.html) policy engine, a tool that evaluates infrastructure as code files for potential Cloud security and compliance violations.

Regula includes a [list of pre-defined rules](https://regula.dev/rules.html) for AWS, Azure, Google Cloud, and Kubernetes based on their respective CIS benchmarks. Regula also allows you to [define custom rules](https://regula.dev/development/writing-rules.html) written in [Rego](https://www.openpolicyagent.org/docs/latest/policy-language/), OPA’s native query language.

## Download Regula

Here we are going to download and get Regula ready for use following the most straightforward approach. Follow the [installation guide](https://regula.dev/getting-started.html#installation) for a complete list of methods for different platforms.

```shell
version=1.6.0
platform=macOS_x86_64
curl -L https://github.com/fugue/regula/releases/download/v1.6.0/regula_${version}_${platform}.tar.gz -o regula.tar.gz ; tar xzvf $_
```

Change the `version` and `platform` accordingly to your needs.

Check it.

```shell
./regula version
v1.6.0, build e2ae463, built with OPA v0.28.0
```

## Verify Terraform Security and Compliance Violations

For this tutorial, we are going to use code examples from the [microservices-demo](https://github.com/microservices-demo/microservices-demo) repository. Let's start downloading some Terraform configuration.

```shell
mkdir terraform

demo_base_folder=https://raw.githubusercontent.com/microservices-demo/microservices-demo/master/deploy/kubernetes

for f in main.tf variables.tf outputs.tf; do
  curl -L $demo_base_folder/terraform/$f -o ./terraform/$f
done
```

{{< notice type="tip" id="use-your-config" title="Use your own infrastructure as code" >}}
Feel free to use your infrastructure as code to perform these tests.
{{< /notice >}}

Now let's run Regula. For the simplest version of the `regula run` command, we have to provide the root location of the configuration files.

```shell
./regula run ./terraform

# ommiting results

FG_R00377: VPC security group rules should not permit ingress from '0.0.0.0/0' except to ports 80 and 443 [Medium]

  [1]: aws_security_group.k8s-security-group
       in terraform/main.tf:5:1

Found 10 problems.
```

The output of the run command is human-friendly, listing all the non-compliant rules along with their descriptions, location, and severity.

Also, note that the command returns
 a non-zero exit code `echo $?` given the minimum severity set as default (`unknown`, the lowest severity used for rules without a severity specified).

You can change the minimum severity to return a non-zero exit code using the `-s, --severity` flag.

```shell
./regula run --severity critical terraform
# ommiting results
echo $?
0
```

The possible values are `informational`, `low`, `medium`, `high`, `critical` and `off` (never exit with a non-zero exit code).

Being able to check the output is useful. Still, we often want to send this result to a place where we can better visualize this data and/or monitor compliance rules across different code repositories. For this, you can use the `-f, --format` to output the result as json and send it to your preferred place for consumption.

```shell
./regula run --format json terraform
```

### Optional: Send the results to SQS

You need to have AWS credentials with the proper permissions configured in your local environment to follow this step.

Create an SQS Queue.

```shell
iac_compliance_queue=iac_compliance 
iac_compliance_queue_url=$(aws sqs create-queue --queue-name $iac_compliance_queue --query QueueUrl --output text)
```

Send the results to SQS.

```shell
./regula run --format json terraform | aws sqs send-message --queue-url $iac_compliance_queue_url --message-body file:///dev/stdin
```

From here, you can consume this message and send this data for aggregation, monitoring or alerting in other tools.

Check and clean up.

```shell
# if you want to check the message created
aws sqs receive-message --queue-url $iac_compliance_queue_url | jq -r '.Messages[] | .Body'

# remove the queue to clean up
aws sqs delete-queue --queue-url $iac_compliance_queue_url
```

In time, note that the maximum message size in SQS is 256 KB. If you have a large codebase to be scanned, you need a different approach, such as sending the results to S3 and a notification to SQS.

## Kubernetes Security and Compliance Violations

As per the date of this writing, a [recent feature added to Regula](https://www.fugue.co/press/releases/fugue-adds-kubernetes-security-checks-to-its-saas-platform-and-open-source-regula-project) is the support for Kubernetes.

Let's see how it works.

Download an example from the microservices-demo repo as we did for Terraform.

```shell
mkdir k8s

curl -L $demo_base_folder/complete-demo.yaml -o ./k8s/complete-demo.yaml
```

Run Regula.

```shell
./regula run -t k8s ./k8s

# ommiting results

FG_R00496: Pods and containers should apply a security context [Medium]

  [1]: Deployment.sock-shop.catalogue-db
       in k8s/complete-demo.yaml:205:1

  [2]: Deployment.sock-shop.queue-master
       in k8s/complete-demo.yaml:516:1

  [3]: Deployment.sock-shop.rabbitmq
       in k8s/complete-demo.yaml:568:1

Found 55 problems.
```

This time we are providing the `-t, --input-type` flag for Kubernetes.

## Kubernetes Custom Policy

One of the main Regula features, in my opinion, is the ability to write your policies using Rego.

Let's create a custom Rego policy to check whether a Kubernetes Pod has set the [containers' resources](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) requests and limits for cpu and memory.

```shell
mkdir k8s/custom

cat << EOF > k8s/custom/containers_resources.rego
package rules.k8s_containers_resources

import data.fugue
import data.k8s

__rego__metadoc__ := {
	"id": "CUSTOM_R00001",
	"title": "Pods should specify containers compute resources (cpu and memory) requests and limits",
	"description": "Pods should specify containers compute resources (cpu and memory) requests and limits. Specifying containers resources requests and limits is considered a best practice.",
	"custom": {
		"severity": "Low"
	}
}

input_type = "k8s"

resource_type = "MULTIPLE"

container_resources_set(template) {
	container = template.spec.containers[_]

	container.resources.limits.cpu != null
	container.resources.limits.memory != null

	container.resources.requests.cpu != null
	container.resources.requests.memory != null
}

policy[p] {
	obj := k8s.resources_with_pod_templates[_]
	count(obj.pod_template.spec.containers) > 0
	container_resources_set(obj.pod_template)
	p := fugue.allow_resource(obj.resource)
}

policy[p] {
	obj := k8s.resources_with_pod_templates[_]
	count(obj.pod_template.spec.containers) > 0
	not container_resources_set(obj.pod_template)
	p := fugue.deny_resource(obj.resource)
}
EOF
```

Now we can check the custom policy.

```shell
./regula run -t k8s --include ./k8s/custom --user-only ./k8s/complete-demo.yaml

CUSTOM_R00001: Pods should specify containers compute resources (cpu and memory) requests and limits [Low]

# ommiting results

Found 6 problems.
```

In this command, we include the custom policy with `-i, --include` and disable the default rules with `-u, --user-only` to make it easier to visualize the output of our custom rule.

Check the [Open Policy Agent documentation](https://www.openpolicyagent.org/docs/latest/policy-language/) to learn more about Rego and the Regula documentation to learn more about how to [write custom rules](https://regula.dev/development/writing-rules.html).
