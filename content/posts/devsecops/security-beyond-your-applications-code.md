---
title: "Part 3: Security Beyond Your Application's Code"
description: "Repositories, CI/CD pipelines, container registries, cloud identities, and supporting services all influence the security of the application we put into production."
date: 2026-10-24
lastmod: 2026-10-24
draft: false
categories:
  - "Security Mindset for Software Engineers"
tags:
  - "application security"
  - "software supply chain"
  - "security mindset"
  - "cybersecurity"
---

When we think about securing an application, a lot of our attention goes to its code. We review changes, test functionality, manage dependencies, and look for vulnerabilities. That's important work, but our application doesn't go from a developer's laptop to production by itself. There are repositories, build pipelines, artifact stores, cloud services, and other systems involved along the way.

Each of these components has its own access controls, configuration, dependencies, and opportunities for mistakes. Even if the application code is secure, weaknesses in the surrounding environment can still put the system at risk.

<!--more-->

## Security starts in the development environment

Let's start with the development environment. Source repositories are usually treated as a place to store code and collaborate, but access to them can also mean access to sensitive configuration, deployment definitions, workflow files, or credentials. Developer machines and third-party development tools add another layer of dependencies and permissions.

For that reason, compromising a developer account or an integration can have consequences beyond reading the source code. Depending on the privileges involved, an attacker might be able to modify what the team builds or influence how it gets deployed. A compromised developer workstation may also provide a foothold into internal infrastructure through stored credentials, authenticated sessions, or network access. Branch protections, access reviews, and carefully scoped credentials help maintain the boundary between contributing code and changing what runs in production.

We also need to be careful about what we introduce into our development environments. Editor extensions, plugins, development tools, and unfamiliar repositories may contain code that executes with access to local resources and credentials. Campaigns such as [GlassWorm](https://socket.dev/blog/glassworm-sleeper-extensions-activated-on-open-vsx) have demonstrated how malicious editor extensions can be used to target developers. Installing an extension or running code from an unfamiliar repository isn't necessarily different from executing any other untrusted software. We should consider what permissions these tools receive, what they can access, and whether we trust their sources.

## CI/CD pipelines are part of the security boundary

Our continuous integration and delivery pipelines do much more than run tests. They may build artifacts, publish container images, access internal infrastructure, and deploy directly to production. To perform those tasks, they need credentials and permissions that can be more sensitive than the ones used by the application itself. This is why limiting workflow permissions and reducing the lifetime of credentials is so important. For example, using workload identity federation through OpenID Connect (OIDC) can allow pipelines to obtain short-lived credentials instead of storing long-lived cloud access keys.

Now consider what happens when a pipeline executes code from an untrusted pull request or depends on a compromised third-party action. A workflow that can run attacker-controlled commands and also has access to deployment credentials could become a path into other systems.

These attacks are not just hypothetical. The 2025 [tj-actions/changed-files compromise](https://github.com/advisories/ghsa-mrrh-fwg8-r2c3) demonstrated how a widely used GitHub Action could be modified to expose CI/CD secrets through workflow logs. In another example, the [Nx S1ngularity incident](https://nx.dev/blog/s1ngularity-postmortem) involved an exploitable GitHub Actions workflow that allowed attackers to steal publishing credentials and distribute malicious packages.

The question isn't only whether the workflow succeeds. It's also what the workflow is allowed to do, which inputs and dependencies it trusts, and whether an untrusted change can reach a privileged execution context. Isolating untrusted jobs, restricting deployment permissions, and controlling which third-party actions can execute are ways to reduce these risks.

## Can we trust the artifact?

Between the source repository and the running application, there is usually a build artifact. It might be a package, a binary, or a container image stored in a registry. We depend on the build process and the storage and distribution systems to deliver the artifact we actually intended to deploy.

A compromised build process or someone with unauthorized push access to a container registry might be able to replace or publish an artifact without changing the application's source code. If our deployment process trusts that artifact, the result can be a compromised application despite an otherwise clean code review.

This is one reason software supply-chain security includes more than scanning dependencies. We should think about who can publish artifacts, whether an artifact can be replaced after publication, and how we establish that what reaches production came from an authorized build process.

There are several ways to address these concerns. For example, referencing container images by immutable digests rather than mutable tags helps ensure that deployments use the expected artifact. Artifact signatures and build provenance can provide additional evidence about where an artifact came from and how it was produced. The [SLSA provenance specification](https://slsa.dev/spec/v1.2/build-provenance) describes a standardized way to record that information.

But generating provenance or signing an artifact isn't sufficient by itself. The deployment process must verify that evidence against trusted sources and expected build conditions. The goal isn't simply to produce evidence of how software was built, but to establish that what we're deploying is what we intended to build and trust.

## The cloud services around the application

Once the application is running, we also rely on cloud services and infrastructure that may not be represented in its codebase. Storage buckets, databases, messaging systems, secret stores, load balancers, and identity services all have security configurations of their own.

A storage bucket that unintentionally allows public reads can expose data even if the application has perfect authorization checks. Cloud platforms often provide safeguards against this—for example, [Amazon S3 Block Public Access](https://docs.aws.amazon.com/AmazonS3/latest/userguide/security-best-practices.html)—but those controls still need to be understood and managed. Applications and APIs may also rely on external services for authentication and authorization. For example, [Google Cloud Identity-Aware Proxy (IAP)](https://docs.cloud.google.com/iap/docs/concepts-overview) can control access to protected applications, [Amazon Cognito](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-how-to-authenticate.html) can handle user authentication, and [Istio authorization policies](https://istio.io/latest/docs/concepts/security/) can control access between services. If these controls are misconfigured, aren't applied where expected, or can be bypassed through another route, an application or API may be exposed without any change to its code. These are some examples, but this type of issue can happen with different types of cloud configuration and services.

These examples show how supporting services can introduce security issues independently of the application itself. One way to reduce these risks is to manage infrastructure and security configurations as code, keeping changes in version control and reviewing them through pull requests. Automated checks can help identify insecure configurations before deployment, while policy enforcement can prevent certain unsafe changes from being applied. We also need to verify the actual configuration of our environments, including changes that may occur outside the normal deployment process.

For services that enforce authentication and authorization, another important principle is to default to denying access and fail closed. We need to ensure that the protections are applied to the intended resources and that access is explicitly allowed rather than assumed. If a required authentication or authorization check fails or cannot be completed, the request should be denied rather than allowed through. A failure in a supporting security service shouldn't become a way to bypass the protection it was supposed to provide.

Security teams often monitor cloud environments for misconfigurations that could expose systems or data, but that monitoring shouldn't replace prevention by the teams building and operating those systems. As software engineers, we should understand the security implications of the configurations we depend on and, where possible, design applications to limit the consequences if a supporting control fails.

## Looking at the entire system

In larger environments, these components may be owned by different teams. Product engineering writes the application, a platform team runs the pipelines, and another team manages cloud infrastructure. Those ownership boundaries are useful for organizing work, but they don't necessarily correspond to security boundaries.

As engineers, we should understand how the components involved in delivering and running our software depend on one another. Which systems can modify the build? Who can publish an image? What credentials can the pipeline use? Which cloud resources can the application or its supporting services access?

Thinking about the system this way doesn't mean every developer has to administer every component. It means recognizing that application security depends on the entire path from source code to production and the services the application relies on once it gets there.

> **AI assistance acknowledgment:** This post was written with AI support for paragraph construction, organization, and reviewing sentence clarity and arguments. The ideas, experiences described, perspective, and editorial decisions remain with the author.
