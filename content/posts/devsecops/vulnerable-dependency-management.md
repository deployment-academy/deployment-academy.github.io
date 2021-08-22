---
title: "Vulnerable Dependency Management"
description: "changeme"
date: 2021-08-22T12:52:20-04:00
lastmod: 2021-08-22T12:52:20-04:00
draft: false
sidebar: "right"
widgets:
  - "ddg-search"
  - "recent"
  - "social"
tags:
---

Let's talk about something that most developers and engineering teams don't pay the necessary attention. Do you have a process to update your application dependencies? How likely is it that your application could be compromised because of a vulnerable dependency?

> **TL;DR** - In this post, you will see that you shouldn't be omissive in terms of upgrading your dependencies and you should strongly consider automating dependency vulnerability check in your pipeline.

## Context

Regardless of the platform you use (Java, Go, Python, PHP, .Net, etc) your application likely relies on external dependencies - probably dozens and often hundreds of them. In fact, we can fairly say that most of the application code for any modern application comes from third-party open source libraries and frameworks. The [State of Software Security Report v11](https://www.veracode.com/state-of-software-security-report) points that 97% of a typical Java application is composed of open source libraries. While this number is unusual, on average we should expect that [about 70% of modern applications should be composed of third-party open source libraries](https://www.synopsys.com/software-integrity/resources/analyst-reports/open-source-security-risk-analysis.html).

Modern applications use dependency management tools like Maven, NuGet, NPM, Pip, etc to automate the process of installing and configuring external libraries. Using third-party components can lead to increased developer productivity allowing them to focus on the target domain while reusing cross-cutting specialized features. These features can vary from simple utilities, like string formatting or arrays manipulation, to more complex features that take care of some core behavior of an application, such as an HTTP handling, an Object Relationship Mapping, or a Dependency Injection framework.

## Security Concerns

Having visibility and control over the security risks presented by external dependencies is as critical as having good security practices for your own code. You can design and write secure code and still be compromised by a bad design choice or implementation made by a library. The same [State of Software Security Report v11]((https://www.veracode.com/state-of-software-security-report)) mentioned before has also identified that **about seven in every ten applications were found to have flaws in their open source libraries** on initial scan. And, **about three in every ten applications have more flaws in their open source libraries than in the primary codebase**.

Vulnerable dependencies issues have been on the industry's radar for a while. The OWASP Top 10 [A9-Using Components with Known Vulnerabilities](https://owasp.org/www-project-top-ten/2017/A9_2017-Using_Components_with_Known_Vulnerabilities) item was introduced in 2013. And other controls, like the [PCI DSS 3.0](https://www.pcisecuritystandards.org/document_library?category=pcidss&document=pci_dss), require that third-party system components should be properly patched.

A classic example of an incident caused by a vulnerable dependency is the [Equinox data breach](https://www.reuters.com/article/us-equifax-cyber/criticism-of-equifax-data-breach-response-mounts-shares-tumble-idUSKCN1BJ1NF) exploited through an Apache Struts vulnerability that exposed personal data of millions of Americans.

## Dependencies Upgrade Approaches

To properly understand how vulnerable dependencies get and last into the codebase, we need to understand some different approaches engineering teams take to keep dependencies up to date. In this way, you can evaluate what you are doing and what options work best for you.

These are some of the approaches that we can see in terms of dependencies upgrades:

**Omissive** - The most common and what I'd call the default approach is to upgrade a dependency when you need a new feature. Besides the obvious issue of using old and often deprecated and not supported features, one of the main problems with this approach is that the longer you take to upgrade a dependency, the more extensive the effort could be. Evaluate the changelog and the impact of an upgrade is more complicated when there is a significant drift between versions. Of course, this isn't good from a security perspective. Security patches may not be backported to older versions and the time and effort to patch a critical vulnerability may not meet the business expectations.

**Latest version** - Staying up to date with the latest version of your dependencies is usually a recommended approach. Being up to date with the latest version of libraries keeps you up to date with new features, bugs and security fixes, but it also brings a risk that cannot be ignored. You'll be impacted by new bugs or vulnerabilities flaws introduced in the third-party library. Depending on the requirements and criticality of your application, it might be a good idea to keep a more conservative cadence of updates mainly for core frameworks. To follow this approach it's ideal to have an automated CICD pipeline and a mature continuous delivery process. Reasonable code coverage and automated tests will help you to validate that the fast pace for upgrading dependencies will not break your application.

**Follow the release cycle of your dependency** - Many teams will consider a more conservative approach and follow the release cycle of dependencies when available. This approach is particularly interesting for core frameworks where being on top of the latest version can be riskier. Some frameworks have a release process that supports and backport fixes up to a certain number of supported versions. Being within a supported version would be considered a reasonable enough approach for most cases.

**Security patches** - Another common approach in the industry is to update dependencies only when critical security patches are available. While this approach might help to secure your application, you are still subjected to merging with a large backlog of changes when an issue is disclosed. It might require a significant refactory and a change that could take minutes may take hours or sometimes days.

## Tools

Regardless of your upgrade approach, it's essential to have an automated step baked into your pipeline to check and alert for vulnerabilities. A good vulnerability scanner or software composition analysis tool will help you find issues earlier in the SDLC and take the necessary actions.

There are some good tools in the industry. Some are commercial, others are open source, and others have a hybrid model. They will vary in features and focus (for example, some focus on scanning container images, others on code repository scanning). I don't have any particular recommendation, and you should evaluate the tool that fits your requirements and profile best. Below is a list of some well-known tools, in no specific order:

* [Snyk](https://snyk.io/)
* [Nexus Lifecycle](https://www.sonatype.com/products/open-source-security-dependency-management?topnav=true)
* [Black Duck SCA](https://www.synopsys.com/blogs/software-security/manage-open-source-black-duck/)
* [GitHub Dependabot](https://docs.github.com/en/code-security/supply-chain-security/keeping-your-dependencies-updated-automatically/about-dependabot-version-updates)
* [WhiteSource](https://www.whitesourcesoftware.com/)
* [Veracode SCA](https://www.veracode.com/products/software-composition-analysis)
* [Prisma Cloud Compute Edition](https://docs.paloaltonetworks.com/prisma/prisma-cloud/prisma-cloud-admin-compute/vulnerability_management.html)
* [Trivy](https://aquasecurity.github.io/trivy)
* [OWASP Dependency Check](https://owasp.org/www-project-dependency-check/)

With a quick research you'll find several comparative posts and reports.

## Conclusion

Most of your application's code is composed of third-party open source libraries. Taking care of this code is as important as taking care of your own code. An omissive approach to handling dependencies upgrades can lead you to a terrible security position and expose your application. Act proactively to upgrade your dependencies and bake vulnerability discovery into your development life cycle, preferably in the CICD pipeline, to anticipate remediation.
