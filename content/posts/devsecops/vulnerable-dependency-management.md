---
title: "Vulnerable Dependency Management: The Invisible Enemy"
description: "Security considerations about dependency management."
date: 2021-08-22T12:52:20-04:00
lastmod: 2021-08-22T12:52:20-04:00
draft: false
sidebar: "right"
widgets:
  - "ddg-search"
  - "recent"
  - "social"
tags:
  - "devsecops"
  - "application security"
  - "appsec"
  - "dependency management"
  - "vulnerability management"
  - "software composition analysis"
  - "sca"
---

Let's talk about something that most developers and engineering teams don't pay the necessary attention to. Do you have a process to update your application dependencies? How likely is it that your application could be compromised because of a vulnerable dependency?

<!--more-->

> **TL;DR** - In this post, we are going to review a few points and see why you shouldn't be omissive in terms of upgrading your dependencies and that you should strongly consider automating dependency vulnerability checks in your CICD pipeline.

## Dependency Management Overview

Regardless of the platform you use (Java, Go, Python, PHP, .Net, etc.), your application likely relies on external dependencies - probably tens or hundreds of them. In fact, we can fairly say that most of the application code for any modern application comes from third-party open-source libraries and frameworks. The [State of Software Security Report v11](https://www.veracode.com/state-of-software-security-report) points that **97% of a typical Java application is composed of open-source libraries**. While this number is unusual, on average we should expect that **about 70% of modern applications should be composed of third-party open-source libraries**, according to the [Synopsys Opn Source Security Risk Analysis](https://www.synopsys.com/software-integrity/resources/analyst-reports/open-source-security-risk-analysis.html).

Modern applications use dependency management tools like Maven, NuGet, NPM, Pip, etc. to automate installing and configuring external libraries during the build time. Using third-party components can lead to increased developer productivity allowing developers to focus on the target domain while reusing cross-cutting specialized features. These features can vary from simple utilities, like string formatting or arrays manipulation, to more complex features that take care of some core behavior of an application, such as HTTP handling, Object Relationship Mapping, or a Dependency Injection framework.

## Security Considerations

Having visibility and control over the security risks presented by external dependencies is as critical as having good security practices for your own code. You can design and write secure code and still be compromised by a bad design choice or implementation made by a library. The same [State of Software Security Report v11]((https://www.veracode.com/state-of-software-security-report)) mentioned before has also identified that **about seven in every ten applications were found to have flaws in their open-source libraries** on initial scan. And, **about three in every ten applications have more flaws in their open-source libraries than in the primary codebase**.

Vulnerable dependencies issues have been causing losses and are on the industry's radar for a while. The OWASP Top 10 [A9-Using Components with Known Vulnerabilities](https://owasp.org/www-project-top-ten/2017/A9_2017-Using_Components_with_Known_Vulnerabilities) item was introduced in 2013 and is still present in the 2017 version. Other important controls and standards, like the [PCI DSS 3.0](https://www.pcisecuritystandards.org/document_library?category=pcidss&document=pci_dss) and the [FS-ISAC](https://www.fsisac.com/hubfs/Resources/FSISAC-ThirdPartySecurityControlTypes-Whitepaper_2015.pdf), require that third-party system components should be properly handled.

The classic example of an incident caused by a vulnerable dependency is the [Equifax data breach](https://www.trendmicro.com/en_us/research/17/c/cve-2017-5638-apache-struts-vulnerability-remote-code-execution.html) where a [Remote Code Execution (RCE) vulnerability](https://nvd.nist.gov/vuln/detail/CVE-2017-5638) in the Apache Struts 2 framework led to the expose of personal data (PII) of millions of Americans.

## Dependency Upgrade Approaches

To properly understand how vulnerable dependencies get and last into the codebase, we need to understand some different approaches engineering teams take to keep dependencies up to date. In this way, you can evaluate what you are doing and what options work best for you.

These are some common approaches we can see in terms of dependency upgrades:

**Omissive** - The most common (and what I would call the default approach) is to upgrade a dependency when you need a new feature or not upgrade at all otherwise. Besides the obvious issue of using old and often deprecated and not supported features, one of the main problems with this approach is that the longer you take to upgrade a dependency, the more extensive the effort will be. Evaluate the changelog and the impact of an upgrade is more complicated when there is a significant drift between versions. Of course, this isn't good from a security perspective as well. Besides keeping vulnerable code in your codebase, security patches may not be backported to older versions and the time and effort to patch a critical vulnerability may not meet the business expectations.

**Latest version** - Staying up to date with the latest version of your dependencies is usually the recommended approach. Being up to date with the latest version of libraries will keep you up to date with new features, bug fixes and security patches. But it also brings a risk that cannot be ignored. You might be impacted by new bugs or vulnerabilities flaws introduced in the third-party library. Depending on the requirements and criticality of your application, it might be a good idea to keep a more conservative cadence of updates - mainly for core frameworks. To follow this approach, it's ideal to have an automated CICD pipeline and a mature continuous delivery process to support frequent rollouts and quick rollbacks. Proper code coverage and automated tests will help you to validate upgrades and changes.

**Follow the dependency release cycle** - Many teams will consider a more conservative approach and follow the release cycle of dependencies (when it's available). This approach is particularly interesting for core frameworks where being on top of the latest version can be risky (from reliability and security perspectives) and/or operationally expensive. Some frameworks have a release process that supports and backports fixes up to a certain number of supported versions, with some having what we call long-term support (LTS) version. Being within a supported version would be considered a reasonable enough approach for most cases.

**Security patches** - Another common approach in the industry is to update dependencies only when critical security patches are available. While this approach might help secure your application, you are still prone to merging with a large backlog of changes when an issue is disclosed. It might require a significant code change and something that could take minutes engineering time may take hours or sometimes days.

It's also not necessary to pick one of these approaches. You might come up with a mixed approach where you would be always on the latest version for utility libraries and follow the dependency release cycle for more critical components.

While being omissive is not an advised posture, **choosing the best approach to manage your dependencies should consider your application characteristics and threat model**.

## Tools Overview

Regardless of your dependency management approach, it's essential to have an automated step baked into your pipeline to check and alert for vulnerabilities. A good vulnerability scanner or software composition analysis tool will help you find issues earlier in the SDLC and take the necessary actions.

There are some interesting tools in the industry to help with this. Some are commercial, others are open-source software, and others have a hybrid model. They will vary in features and focus (for example, some focus on scanning container images, others on code repository scanning, while others are complete security suites with various features). You should evaluate the tool that fits your requirements and profile best. Below is a list of some well-known tools, in no specific order:

* [Snyk](https://snyk.io/)
* [Nexus Lifecycle](https://www.sonatype.com/products/open-source-security-dependency-management?topnav=true)
* [Black Duck SCA](https://www.synopsys.com/blogs/software-security/manage-open-source-black-duck/)
* [GitHub Dependabot](https://docs.github.com/en/code-security/supply-chain-security/keeping-your-dependencies-updated-automatically/about-dependabot-version-updates)
* [WhiteSource](https://www.whitesourcesoftware.com/)
* [Veracode SCA](https://www.veracode.com/products/software-composition-analysis)
* [Prisma Cloud Compute Edition](https://docs.paloaltonetworks.com/prisma/prisma-cloud/prisma-cloud-admin-compute/vulnerability_management.html)
* [Trivy](https://aquasecurity.github.io/trivy)
* [OWASP Dependency Check](https://owasp.org/www-project-dependency-check/)

This list is not final, and I recommend that you do your own research if you are interested in finding a provider. You might also be able to solve this problem with a complete open-source solution (which sounds like an interesting topic for another blog post :slightly_smiling_face:). The [Open Guide To Evaluating Software Composition Analysis Tools](
https://www.linuxfoundation.org/wp-content/uploads/An-Open-Guide-To-Evaluating-Software-Composition-Analysis-Tools_V2.pdf) provides a comprehensive baseline for researching tools to prevent vulnerable dependencies (which is not limited to SCA tools). Another valuable resource is the [OWASP Vulnerable Dependency Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Vulnerable_Dependency_Management_Cheat_Sheet.html).

## Conclusion

Most of your application is composed of third-party open-source libraries. Taking care of this code is as important as taking care of your own code. An omissive dependency management posture can lead you to a high-risk and vulnerable position. You should act proactively to upgrade your dependencies and bake vulnerability discovery into your development life cycle, preferably in the CICD pipeline, to anticipate remediation.
