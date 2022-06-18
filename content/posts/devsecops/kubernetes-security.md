---
title: "A Kubernetes-centered security approach from the trenches"
description: "In this post, I want to share the approach I have been using to scope and decide what to do when defending a Kubernetes environment. The approach presented defines ten high-level security concerns and is used with other frameworks and benchmarks to contextualize and guide the security efforts."
date: 2022-06-11T13:16:56-04:00
lastmod: 2022-06-11T13:16:56-04:00
draft: false
toc: true 
tags:
  - "devsecops"
  - "kubernetes security"
  - "kubernetes"
  - "supply chain"
---

In this post, I want to share the approach I have been using to scope and decide what to do when defending a Kubernetes environment. The approach presented defines ten high-level security concerns and is used with other frameworks and benchmarks to contextualize and guide the security efforts.

<!--more-->

## Context

Ok, we all know that Kubernetes is a dynamic platform with a fast-growing ecosystem with new features and tools coming out almost every week. There are many moving parts and an extensive attack surface that involves not just the Kubernetes components and workloads but also the underlying Cloud and software distribution infrastructure. While securing Kubernetes, it turns out that even when we have tools and are technically equipped, the threat model may still be unclear. It’s often challenging to determine how the different ongoing initiatives and tools installed across the board come together to tell if we are going in the right direction.

The approach presented here aims to help with this scenario by providing a complete and concise way to evaluate the effort to secure a Kubernetes environment. It organizes the Kubernetes security concerns in ten groups that should cover three security aspects - **prevention**, **monitoring**, and **response** - across four areas: **Cloud infrastructure**, **Kubernetes components**, **supply chain infrastructure**, and **workloads**.

In practice, we use these groups to understand better the things we are working on, identify gaps, assess risks, and get more clarity of where/what we should be focusing the efforts. It helps the team with the organization and prioritization of the roadmap, discovery, prospection, comparison of tools to implement or buy, skills to develop, etc. Most important, it moves the focus from the tooling and gives a comprehensive view of the security concerns.

Important to mention that this model doesn’t intend to replace or reinvent existing resources. On the other hand, it’s used with other benchmarks like [MITRE](https://attack.mitre.org/matrices/enterprise/containers/), [CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes), the [NIST Cybersecurity framework](https://csf.tools/reference/nist-cybersecurity-framework/v1-1/), and others to contextualize and guide the security efforts.

Below I’ll enumerate the concerns groups with a brief description for each. Many of these areas are enough to fill entire books. It’s not my intention to cover all the points or exhaust a subject. The goal is to provide enough information to contextualize what we are talking about for each topic. It’s also not my intention to make this post a list of tools or a recipe to be followed. Examples or tools will be mentioned when helpful. Finally, I have no commercial association with any product or company mentioned here.

## Security Concerns

### Application code and dependency vulnerabilities

The security of the workloads running in the Kubernetes cluster - application code and 3rd-party dependencies - is one of the first areas to consider tackling. Generally, the scope of this area goes beyond the Kubernetes efforts and is supported by an entire application security and vulnerability program that will cover different practices and tools.

Some examples of activities that will help to keep the workload secure are: 
* maintain security baseline documentation with general guidance, secure coding best practices, and standards for developers (usually accompanied by policy documents);
* continuously run static analysis tools (SAST) against the codebase in the CI/CD pipeline;
* continuously scan code and container images for common vulnerabilities;
* continuous developer education and training; 
* maintain a vulnerability disclosure or bug bounty program; 
* etc.

Besides implementing and rolling out tools, an application security and vulnerability program requires more complex tasks, such as: 
* consolidating and analyzing findings;
* filtering false positives;
* prioritizing remediation;
* maintaining a fast-paced and change-friendly culture and development environment that allows continuously rebuilding and deploying;
* assessing complex software design and architectural decisions and trade-offs;
* and others.

On the monitoring side, most of the activity is restricted to the development process and issues should be discovered and treated early in the SDLC. Besides notifying and enumerating findings, the available tooling should facilitate aggregation and visualization of issues across multiple components in a centralized way where they can be manually or, ideally, automatically triaged and directed to the target teams.

### Pod and container misconfiguration

According to security reports, like [Red Hat's State of Kubernetes security report](https://www.redhat.com/rhdc/managed-files/cl-state-of-kubernetes-security-report-2022-ebook-f31209-202205-en.pdf), misconfiguration is a top concern in the Kubernetes security world and the number one issue related to security incidents in Kubernetes environments. A badly configured container can compromise the enterprise network or lead to a cloud account takeover if properly chained to other techniques/issues.

Misconfigured containers, or containers using the default configuration, can lead to privilege escalation or breach the container isolation allowing a threat actor to abuse host resources, which can potentially lead to accessing other containers, network services, databases, storage buckets, cloud metadata service, and others. The way to protect against this type of problem is to harden the Pod configuration, ensuring that security standards and best practices are applied. The [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/) documentation provides a good baseline that can be implemented. Other benchmarks and guides such as [CIS Kubernetes](https://www.cisecurity.org/benchmark/kubernetes), [CIS Docker](https://www.cisecurity.org/benchmark/docker), and [NIST SP 800-190](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-190.pdf) can also be used to harden Pod and container configuration.

There are different tools in the industry that can be used to check container compliance rules. A common approach is using a [dynamic admission controller](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/) such as [OPA/Gatekeeper](https://open-policy-agent.github.io/gatekeeper/website/docs/), [Kyverno](https://kyverno.io/docs/introduction/), or the new [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/) (introduced in v1.23) to prevent badly configured resources from being deployed to the cluster or alerting upon violations. Another method is to use static analysis tools such as [Regula](https://regula.dev/rules.html#kubernetes) and [KubeLinter](https://github.com/stackrox/kube-linter) to check the configuration in the source code repository.

Some special use cases (multi-tenant clusters and clusters whose containers run untrusted workloads) can benefit from an extra layer of security using sandboxed container features, such as [gVisor](https://gvisor.dev/docs/).

Container compliance violations should be reported to a centralized tool where they can be aggregated, filtered, and triaged at the cluster, namespace, or application level. This type of issue should be treated early in the pipeline. 

### Supply chain threats

Supply chain threats refer to how the build and distribution infrastructure can be used to compromise the software system. Modern software relies on complex continuous delivery pipelines that will likely involve CI/CD servers and agents, build machines, package managers, public and private artifact repositories, public and private container registries, etc.

A threat actor can use any of these components to compromise the integrity of your code or 3rd-party dependencies to gain access, steal data or disrupt your system’s operations. For example, someone with access to the build environment can tamper a container image with a payload that will allow remote code execution in running containers. Or someone with access to your container registry can replace an actual image with another one with malicious intent. The tl;dr of supply chain issues is that you should not deploy any artifact that you can’t trust or can't verify the integrity against attestations.

To protect against supply chain threats, you want to ensure that all the components in the build and distribution pipeline are properly hardened (including software configuration, security patches, etc), access to registries and artifact repositories follows the principle of least privilege, artifacts are properly identified with metadata and attestations, and only trusted code is deployed. The CNCF blog post [A MAP for Kubernetes supply chain security](https://www.cncf.io/blog/2022/04/12/a-map-for-kubernetes-supply-chain-security/) gives a comprehensive view of the current state of the Kubernetes supply chain with valuable pointers for other resources, for example the [SLSA security framework](https://slsa.dev/).

The software supply chain tooling ecosystem is evolving fast and becoming more mature every day. With some research, you can find a variety of tools that will help to secure your supply chain environment, such as CI/CD service providers, artifact and container registries, both SaaS and self-hosted offers, etc. The main cloud providers offer code and container image build managed services such as [Google Cloud Container Build](https://cloud.google.com/blog/products/gcp/google-cloud-container-builder-a-fast-and-flexible-way-to-package-your-software). For attestations, you can find cloud-specific solutions such as [GKE Binary Authorization](https://cloud.google.com/binary-authorization/docs) or more general approaches based on [dynamic admission controllers and signing keys](https://neonmirrors.net/post/2022-05/harbor-cosign-and-kyverno/). 

On the monitoring side, you should ensure that all of your build and distribution infrastructure is continuously monitored for security and compliance violations, ideally using endpoint security, anomaly detection, and event management (SIEM) tools. Security violations in the supply chain environment should trigger the security operations or incident response team and be treated immediately.


### Cluster misconfiguration

As mentioned before, misconfiguration is one of the most significant concerns and one of the leading causes of incidents in Kubernetes environments. While the previous section discusses container issues, here we are interested in the cluster’s control plane and worker nodes misconfigurations. A Kubernetes cluster is a combination of [different components](https://kubernetes.io/docs/concepts/overview/components/) that are deployed independently. Each of these components has many bootstrap [options](https://kubernetes.io/docs/reference/command-line-tools-reference/) that configure their expected behavior. When provisioning a new cluster, you need to ensure that these components are configured in a way that reduces the blast radius. It’s equally important to ensure that the underlying Cloud infrastructure is configured correctly.

The [CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes) is the primary resource for best practices to harden the configuration of a Kubernetes cluster. The Kubernetes documentation also has a [dedicated page](https://kubernetes.io/docs/tasks/administer-cluster/securing-a-cluster/) that covers topics related to protecting a cluster from accidental or malicious access and provides recommendations on overall security.

Using a managed Kubernetes service like GKE or EKS will save you some effort with hardening and defending the cluster. Via a shared responsibility model, the cloud provider takes over a lot of the effort to secure the control plane components and underlying infrastructure. Even in this situation, it’s essential to remember that relying on defaults or minimal setups will not buy you the best security posture. You should make sure that you have a good security baseline, preferably managed as code, to provision clusters. There are CIS Benchmarks and specific recommendations for each of the most popular providers (for instance, the [EKS security best practice guide](https://aws.github.io/aws-eks-best-practices/security/docs/) and [GKE security hardening guide](https://cloud.google.com/kubernetes-engine/docs/how-to/hardening-your-cluster)).

Cloud config and infrastructure security tools are a good way to continuously monitor your Kubernetes environment to detect Kubernetes components and underlying infrastructure issues. These tools usually implement rules based on the CIS Kubernetes Benchmark and other benchmarks, including cloud-specific checks. Problems with Kubernetes configuration are usually discovered and treated by the development operations team during the provisioning or remediation phases. Aqua Security [kube-bench](https://github.com/aquasecurity/kube-bench) is a popular tool to check CIS Kubernetes Benchmark compliance.

### API Server access issues

The kube-apiserver is the entry point for the Kubernetes cluster and how developers, operators, and other systems interact with it. Access to the Kubernetes API goes through 3 steps: authentication, authorization, and admission control. 

For authentication, it’s important to choose a method that allows credentials rotation and revocation. Using an OIDC provider is generally recommended. If using a managed service, relying on the cloud provider tooling to configure user access is the best approach.

The access to Kubernetes resources should follow the principle of least privilege using the RBAC API. It’s a good idea to define and configure roles for different profiles: operator, developer, pipeline, etc. Restricting these roles to specific namespaces is a good idea, even when not in a multi-tenant environment, because it reduces the blast radius. The most important take in terms of authorization is not to leave the cluster open wide. A threat actor with standard access to a Kubernetes cluster can read Kubernetes secrets, execute into pods to check environment variables and file system, escape to the host, launch pods with privileges to Cloud APIs and services, etc.

Dynamic admission controllers (validating or mutating webhooks) can also be leveraged when more sophisticated rules or behavior are required.

The best way to monitor API Server access issues is to ship audit logs to a SIEM tool or log aggregator.

### Cloud API and services access issues

Applications often need access to Cloud API and Services such as storage buckets, messaging services, databases, etc. It’s generally not a good idea to issue and deploy static cloud credentials to containers. These credentials are long-lived and not easy to be rotated or revoked. 

Managed Kubernetes services like GKE and EKS offer specific features to dynamically provision credentials into the container ([EKS IAM roles for service accounts](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) and [GKE workload identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity)). These features allow containers to impersonate Identity and Access Management (IAM) roles and service accounts to access the Cloud API and Services.

When these features are not available it’s possible to leverage [HashiCorp Vault secrets engines](https://www.vaultproject.io/docs/secrets) combined with the [Vault Kubernetes auth method](https://www.vaultproject.io/docs/auth/kubernetes) to generate short-lived dynamic Cloud credentials.

Cloud config and security tools can help to monitor existing static credentials.

### Secrets leaks

In any typical SDLC, secrets are needed across multiple phases. In the CI/CD pipeline to deploy infrastructure and applications, in the runtime environment so applications can access external services, in the developers’ machines to run manual operations and administrative tasks, and so on and so forth. The rule of thumb for handling secrets is to avoid long-lived static credentials as much as possible, which means aiming eliminating the existence of static secrets. When they are necessary, we should ensure they are stored in a safe place and encrypted at rest. Access to secrets should be limited and follow the principle of least privilege, and there should be a clear auditing track to identify who has access to which secrets, and all access should be logged.

In the CI/CD pipeline consider using short-lived dynamic secrets that can be issued via OIDC (see this [GitHub Actions documentation](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect) and [this complete Google Cloud tutorial](https://cloud.google.com/blog/products/identity-security/enabling-keyless-authentication-from-github-actions) as an example). Alternatively, HashiCorp Vault can be used to issue short-lived dynamic credentials via its several secret engines. A Vault authentication method can be used to create a trusting relationship between the CI/CD self-hosted agents and Vault, eliminating the need for any static credentials in this process.

Applications should benefit from IAM impersonation methods to access Cloud APIs and Services as described in the previous section when available. If static secrets are needed, they should be stored and fetched from a secrets manager tool. Most, if not all, cloud providers offer a managed secrets manager service. There are still other products that can help with this. Vault, again, is a very popular tool to support your static secrets and identity and access management needs. As mentioned for CI/CD pipelines, when using Vault you can leverage auth methods to connect applications with it without the need to deploy any long-lived credential (e.g. static tokens or AppRole credentials). Applications can use the Vault API or Vault SDK to fetch secrets in runtime or use the [Agent Sidecar Injector](https://www.vaultproject.io/docs/platform/k8s/injector).

Kubernetes secrets are also generally fine to load secrets into applications. The most common and recommended method to create Kubernetes secrets is to integrate with a secrets manager via the CI/CD pipeline or the [secrets store CSI driver](https://secrets-store-csi-driver.sigs.k8s.io/). It's very important to remember that the access control to the kube-apiserver should be tightly controlled since secrets are accessible from the k8s secrets objects or from containers. We should also ensure that etcd is encrypted at rest.

It doesn't matter if you implement the most sophisticated secrets management mechanism across the board if good practices for credentials hygiene on dev machines are not encouraged. Impersonation features should always be used to avoid developers' need for static credentials on their machines. Vault can be leveraged as a credentials broker for virtually any modern tool or component and used by developers to issue temporary credentials. When static secrets are necessary for local operations, they should be read via CLI or API and piped into other commands without going to the developers' file system (including the bash history ;)) or favorite online note-keeping app :P. There are still infrastructure access solutions like strongDM or Teleport that can be used to prevent static credentials on developers' machines. When the tooling available can't help, we should ensure that developers are informed and aware (and accountable) of the risks involved with handling static credentials.

For monitoring, use continuous automated scans to monitor secret leaks in the VCS, artifacts, or container images (yeah secrets baked in images are not nice). Keep a clear and documented plan that describes severity level, priority, and path for remediation to handle secrets leaks. Cloud config and infrastructure monitoring tools can help to identify long-lived cloud credentials and track usage. These features can be used to identify existing keys that could be cleaned up, or during an incident to audit usage. High-privileged credentials should be tracked via logs and piped into an events manager tool (SIEM) or log aggregator with rules to identify abnormal activity.

Well, it is a really long topic. What does it have to do with Kubernetes at all? I must say everything. Trust me, you are more likely to be compromised by a credential living on a hidden commit in the GitHub history of a public repository than a remote code execution chained into a container escape in your environment.

### Network vulnerabilities

When talking about network vulnerabilities in a Kubernetes environment, we focus on two main aspects: the cloud provider's VPC network (where your control plane and workers node live) and the actual Kubernetes network (where the workloads are deployed to).

Every cloud provider has recommendations and best practices around VPC security. You should consult your providers' documentation for guidance. The general guideline is to ensure that the VPC is properly configured to only allow authorized inbound and outbound traffic from and to the control plane and workers nodes and between them. From a security perspective, you want to be as restrictive as possible and keep things locked. However, sometimes you have to balance this with operational needs (as an example, often it’s not possible or desired to have the kube-apiserver endpoint not accessible from the internet). Other resources on the VPC network (messaging systems, databases, etc.) should also limit access to only the downstream resources. If you use a managed Kubernetes service, you can use a cloud-specific feature that will help lock down your cluster's network out of the box - for example, if using GKE, you can leverage the [GKE private cluster](https://cloud.google.com/kubernetes-engine/docs/concepts/private-cluster-concept) feature to create a cluster that only depends on internal IP addresses.

Within a Kubernetes cluster, all Pod-to-Pod communication is allowed by default, and there isn’t much control over ingress and egress traffic (unless you are messing with iptables directly). Using [Kubernetes Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) will buy you an extra layer of security. With Network Policies, you can create boundaries between Pods or namespaces and have more fine-grained control over the inbound and outbound traffic for applications running on the cluster. Network Policies ensure network isolation at the Pod level with many configuration options. The main Network Policies providers, such as [Calico](https://projectcalico.docs.tigera.io/security/calico-network-policy) and [Cilium](https://kubernetes.io/docs/tasks/administer-cluster/network-policy-provider/cilium-network-policy/) will still offer advanced features on top of the standard configuration.

To ensure that you have visibility over what’s happening in your network you should make sure that you are collecting access logs for all public endpoints, have enabled flow logs at the VPC level, and have features in place that allow you to collect flow logs at the cluster level as well as monitor Network Policies violations - for example, [Calico Enterprise](https://projectcalico.docs.tigera.io/security/calico-enterprise/network-visibility), [Hubble](https://github.com/cilium/hubble), and [GKE Dataplane V2](https://cloud.google.com/kubernetes-engine/docs/concepts/dataplane-v2). Having the right visualization and event management capabilities on top of this data is essential to optimize detection and response.

Other necessary network security measures are: to ensure that you have WAF (and API Security) in place for public endpoints and, of course, end-to-end traffic encryption. Many people consider that a service mesh can help achieve a better security posture since it embeds (or in theory could) many of these features. You should assess your needs and remember that configuring a tool won’t buy you security for free.

### Runtime exploitation

Most of the topics covered so far are related to prevention and mitigation. However, if someone is able to get access to your Kubernetes cluster or hosting infrastructure, you must be able to detect and respond accordingly. In the real world, all these security controls we talked about so far take time to be implemented and, honestly, some of them may not even be prioritized and implemented when compared to other priorities. For this reason, it’s very important to have tooling in place to detect suspicious activity inside your cluster from the very early stages.

Commonly, runtime defense tools monitor your cluster via agents installed in the nodes and support sensors for file system, network, process activity, syscall, etc. This means if someone is able to exec into a container and launch or modify a process or access a sensitive directory, an event will be triggered. The tables [on this page](https://docs.paloaltonetworks.com/prisma/prisma-cloud/prisma-cloud-admin-compute/runtime_defense/runtime_audits) provide a list of runtime detections to exemplify what types of threats a Kubernetes environment is exposed to. 

The biggest challenge with this type of tool is to identify what types of events are actionable (this is probably true for all security tooling, but core in this context where timely detection and response are important). The trick is that it’s not easy to differentiate between a developer accessing a container to check connectivity with a database from a smart threat actor performing reconnaissance in your environment. There is also a huge amount of application behavior that usually just falls into the anomalous behavior category. The experience has shown that a typical Kubernetes environment can generate hundreds (or thousands) of events per day while only a handful of events require attention from the security team per week.

In this way, runtime defense tools must be able to collect, aggregate, and filter events in a meaningful way so that the security team has a clear view of what is relevant and what is not. Popular products in the industry use some sort of automatic learning methodology to differentiate what is regular application behavior from anomaly activity. There are a couple of products in the industry that can help you with Kubernetes runtime defense and anomaly detection and I recommend that you do some research to figure out what fits better for your environment. To give an example, the [Falco project](https://falco.org/) is a very popular open-source option.

It’s equally important to ensure that control plane and worker nodes are also continuously monitored for runtime exploitation. Usually, endpoint security, host-based and network intrusion detection systems can be used to monitor the cloud boxes. In both cases, control plane and workloads, all relevant events should be sent to a SIEM tool where rules will be in place to notify the security operations or incident response team.

### Forensics, auditing and incident response

Last but not least, you should ensure that you collect and have timely access to everything that happens inside your cluster. It goes from audit logs and Kubernetes events to all the container activity. When exploring an incident in a cluster, commonly, you’ll need a timeline of all the activity that happened inside a container (container started, process spawned, connection established, etc.). This information should be contextualized and correlated with other data from your environment. For example, after detecting that a suspicious binary was executed inside a container you may want to check the VPC network flow logs to identify if an non-identified IP is connected to the node.

Different products in the industry can support you with tooling and processes to investigate and audit behavior in your Kubernetes environment. As in all other cases, this information should be centralized and available from a single pane of glass. Data visualization, query and filtering capabilities, as well as contextualized data from different parts of your environment, will help investigate an ongoing incident or collect evidence and analyze the scope and impact of an attack.

## Wrap up

The security concerns presented and described in this post are used to guide the security efforts when defending a Kubernetes environment. They help the team to revise the security roadmap ensuring that the initiatives are aligned and that there are no gaps or overlapping efforts.

* Application code and dependency vulnerabilities
* Pod and container misconfiguration
* Supply chain threats
* Cluster misconfiguration
* API Server access issues
* Cloud API and services access issues
* Secrets leaks
* Network vulnerabilities
* Runtime exploitation
* Forensics, auditing and incident response

This model has also been used as a baseline to evaluate tools, methods, assess risks, and other Kubernetes-centered security capacities.

This overall approach has been used and continuously improved in the real world for a couple of years now. The current form is heavily influenced by the container threat model presented by [Liz Rice](https://twitter.com/lizrice) in [Container Security: Fundamental Technology Concepts that Protect Containerized Applications 1st Edition](https://www.amazon.com/Container-Security-Fundamental-Containerized-Applications/dp/1492056707).

If you have any questions or comments feel free to reach out in the Twitter thread below or directly.

{{< tweet user="soeiro_santos" id="1536109625665363969" >}}
