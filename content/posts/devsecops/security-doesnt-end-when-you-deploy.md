---
title: "Part 4: Security Doesn't End When You Deploy"
description: "Why application logging, security observability, incident response, and recovery should be part of the way software engineers design and operate systems."
date: 2026-10-26
lastmod: 2026-10-26
draft: false
categories:
  - "Security Mindset for Software Engineers"
tags:
  - "application security"
  - "security mindset"
  - "incident response"
  - "cybersecurity"
---

When we deploy software to production, we expect it to work as designed. We have tests, code reviews, security controls, and other practices that help reduce the likelihood of something going wrong. But as we've discussed throughout this series, none of these controls can guarantee that a system will never be compromised. There are things we don't control, such as a security issue in an external service or API, or a newly discovered zero-day vulnerability in one of the dependencies we use.

If someone exploits a vulnerability, gains unauthorized access, or abuses legitimate functionality, would we know? And once we identify the problem, how prepared are we to investigate, contain, and recover from it?

<!--more-->

## Would we know if something went wrong?

As software engineers, we're already familiar with application monitoring. We collect logs, metrics, and traces to understand performance, troubleshoot errors, and identify availability issues.

Security monitoring builds on these capabilities, but the events we're interested in aren't always the same. An application can be functioning perfectly while someone is using it to access information they shouldn't, manipulate transactions, or abuse legitimate features.

Consider an account that suddenly downloads thousands of documents, attempts to access resources belonging to other users, or starts performing administrative operations it has never performed before. The application might successfully process every request, returning HTTP 200 responses without producing a single error.

From a functional perspective, everything could look normal. From a security perspective, however, some of those activities might deserve investigation.

## Logging what matters

One of the most useful things we can do as developers is make sure our applications and infrastructure produce enough information to understand security-relevant activities.

Authentication events, authorization failures, permission changes, administrative operations, access to sensitive resources, and unusual transaction patterns are examples of information that can help identify suspicious behavior. These are the kinds of events we often capture in application audit logs. The coverage and granularity we need will depend a lot on the application and the sensitivity of the data it handles.

One particular piece of information we often realize we don't have only when we need it is edge logs. When we notice suspicious activity in an application, one of the first questions is where it originated. Edge logs can help us identify source IP addresses observed at the edge, repeated requests, and the scope of that activity across different paths or services. We can collect these logs in different ways depending on the architecture. A common approach is at the WAF level, but API gateways, CDNs, and load balancers can also provide useful edge logs.

Recording an event isn't particularly useful if we can't establish what happened. Logs need enough context to answer questions such as who performed an action, when it happened, which resource was involved, and whether the operation succeeded. Depending on the architecture, request identifiers and correlation IDs can also help connect events across different services. For example, timestamps can help us correlate an edge log with an application-level event, but a shared correlation ID makes that relationship much easier to establish reliably.

Speaking of correlation and making things easier, centralizing logs in a log aggregator can make investigations much simpler. While the first priority is ensuring we capture the necessary data, centralization often comes next.

There are some important trade-offs we need to consider when thinking about logging. While we want to capture all relevant information, we must avoid unnecessarily logging sensitive data such as passwords, access tokens, session credentials, or private customer information. Logs themselves need appropriate access controls, protection against tampering, and retention policies that make sense for the system. And while we're here, remember that logging is expensive, so we need to tune it and choose the right approach based on volume, retention, storage tiers, and how often we need to query the data.

The [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html) is a useful resource.

Just as importantly, collecting logs doesn't mean we're monitoring security. Someone or something needs to evaluate those signals and identify when an activity deserves attention.

There's much more to security monitoring than what we've discussed here. Security teams often implement and operate additional capabilities, such as runtime threat detection across workloads and infrastructure, to identify suspicious activity that application monitoring alone might miss. My focus in this post is what we can influence as engineers building and maintaining a product: making security-relevant activity visible and ensuring our systems support investigation, containment, and recovery.

## Supporting incident response

Let's assume we identify suspicious activity and have reasonable evidence that an account or component has been compromised. The next question is what we can do about it.

Can we immediately revoke the affected credentials or terminate active sessions? Can we disable an account, restrict access to a service, or isolate a compromised workload without bringing down the entire application?

Depending on how the system was designed, these operations may be straightforward or surprisingly difficult. If credentials are embedded in configuration, sessions can't be invalidated, or services have broad dependencies on one another, containing an incident can become an engineering challenge of its own.

We should also consider how investigations are conducted. Logs that disappear when a container restarts, events without sufficient context, or systems that don't preserve useful audit trails can make it difficult to establish what happened.

Incident response is a collaborative effort often spanning multiple teams. As developers, we may not be responsible for coordinating that effort, but our design decisions influence what those teams can actually accomplish.

Keep in mind that security incidents shouldn't be treated exactly as we regularly treat operational incidents. During operational incidents, we often want to promote transparency for learning and collaboration. Security incidents can involve legal and privacy concerns. During an initial investigation, it's difficult to tell exactly what the severity and final impact are. For this reason, we should handle information about a suspected security incident discreetly, involving the people needed to investigate, remediate, and manage any required communications. This doesn't mean delaying escalation to the appropriate teams.

## Recovery is part of security

Containing an incident doesn't necessarily mean the system is ready to return to normal operation. We still need to understand what was affected, address the cause of the compromise, and determine whether unauthorized access or modifications remain.

For example, restoring a service from a backup might recover its data, but it won't necessarily resolve the vulnerability or compromised credential that allowed the incident to happen. Similarly, redeploying a container doesn't help much if an attacker still has access to the deployment pipeline.

Backups, tested restoration procedures, infrastructure automation, and the ability to replace compromised components all contribute to recovery. These capabilities are often discussed in the context of reliability and availability, but they're equally relevant to security.

And once an incident has been resolved, the findings should help us improve the system. Sometimes that means fixing a vulnerability; other times, it means improving logging, reducing unnecessary permissions, changing an architectural decision, or making a containment procedure more effective.

The [NIST Incident Response Recommendations (SP 800-61 Rev. 3)](https://csrc.nist.gov/pubs/sp/800/61/r3/final) provide a broader reference for how organizations can integrate detection, response, and recovery into their security practices.

## A security mindset throughout the lifecycle

Throughout this series, we've looked at security from several perspectives: what an attacker can discover, how far a vulnerability can lead, and how the components surrounding our applications can introduce additional risks.

Monitoring and incident response complete that picture. Security doesn't stop when we implement a control or deploy a new version. We also need to consider how the system behaves when something goes wrong, how we identify that problem, and what options we'll have to address it.

As software engineers, we make decisions throughout the development lifecycle that influence all of these outcomes. Developing a security mindset means making those decisions with a broader understanding of how our systems can be attacked, what the consequences might be, and how we can limit the damage.

> **AI assistance acknowledgment:** This post was written with AI support for paragraph construction, organization, and reviewing sentence clarity and arguments. The ideas, experiences described, perspective, and editorial decisions remain with the author.

---

[← Previous: Part 3](https://deployment.properties/posts/devsecops/security-beyond-your-applications-code/) | [Series introduction](https://deployment.properties/posts/devsecops/introducing-the-security-mindset-for-software-engineers/)
