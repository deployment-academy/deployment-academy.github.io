---
title: "How AI Is Changing Software Engineering Work"
description: "As AI makes more software work practical to delegate, technical foundations, verification, and operational ownership become more important."
date: 2026-08-06T02:04:12Z
lastmod: 2026-08-06T02:04:12Z
draft: false
sidebar: "right"
widgets:
  - "ddg-search"
  - "recent"
  - "social"
tags:
  - "artificial intelligence"
  - "software engineering"
  - "opinion"
---

Over the past few years, we have watched AI move from completing lines of code to taking on work we might once have handed to another engineer. Coding agents can inspect a repository, propose an implementation, modify several files, write tests, and return something that looks remarkably close to finished. What has not become equally easy is deciding whether that work belongs in a production system.

Someone still has to understand what the change does, whether it solves the right problem, how it can fail, what it exposes, how it behaves under load, and whether the surrounding system can absorb it. As of today, the code may arrive faster, but the confidence required to move it into production still has to be established separately.

This changes where engineering value is concentrated: less in producing each line of code, and more in understanding systems, verifying outcomes, and operating software safely. It also means AI will amplify the strengths and weaknesses of the engineering environment around it.

<!--more-->

## Sixteen engineers who felt fast


In July 2025, [METR published a randomized controlled trial](https://arxiv.org/abs/2507.09089) involving sixteen experienced open-source developers working through 246 real tasks in repositories they had maintained for an average of five years. The developers predicted that AI would make them 24% faster. Afterward, they reported that it had made them roughly 20% faster. They were 19% slower, according to METR’s results.

Sixteen people is a small sample, the confidence interval was wide, and expert engineers working in familiar repositories are close to the worst case for AI assistance. When [METR revisited the experiment](https://metr.org/blog/2026-02-24-uplift-update/), a returning subset leaned toward a speedup, but with too much uncertainty for a clean conclusion. The original result is therefore best read as a snapshot of early-2025 tooling in a narrow setting, not as a general estimate of AI’s effect on software productivity.

What remains useful is the distance between measured and perceived productivity: roughly forty percentage points, with perceived productivity running well ahead of measured productivity. METR’s results suggests that time saved on writing was consumed by prompting, reading, checking, and repairing. The work moved from *making* the code to *establishing that it was fit to run*. That second activity has no keystroke count and may produce no visible artifact. It is often invisible to dashboards that celebrate code volume or time-to-merge.

What practitioners and organizations may be mispricing is assurance.

## Where the compiler comparison stops helping

An increasingly common framing says that [AI is the new compiler](https://abolinsky.io/blog/ai-compiler/): another step in the progression from low-level implementation toward expressing intent at a higher level.

The comparison is not entirely misplaced. Compilers, garbage collection, libraries, frameworks, cloud platforms, and CI/CD each removed work engineers once performed directly and changed, to varying degrees, how we produce and deliver software.

AI may prove broader and more consequential than any of those changes. It works with ambiguous instructions, makes choices, uses tools, and can participate across much of the delivery lifecycle. Still, the historical comparison is useful because it forces us to ask which work disappears and which work moves.

The comparison stops helping when we consider how trust is established. Previous tools were not flawless. Compilers contain bugs, runtimes fail, and frameworks and cloud services introduce risk. But much of their verification cost can be spread across repeated use. Once tested, stabilized, and versioned, they can be trusted within known constraints. Engineers do not inspect machine code after every compilation or re-audit a managed database for every query.

Generative AI produces a different kind of artifact. The fitness of its output depends on the request, repository, architecture, business rule, security boundary, and moment in time. Confidence in the model does not transfer cleanly to confidence in the change. The output still has to be evaluated in context.

Martin Fowler offered a useful posture: treat each AI contribution as [a pull request from a rather dodgy collaborator](https://thenewstack.io/martin-fowler-on-preparing-for-ais-nondeterministic-computing/) who produces a great deal of code but cannot be trusted without close review. The collaborator may improve substantially. The pull request still enters a specific system with specific consequences.

Earlier tools often retired categories of inspection. Generative AI increases the volume of context-specific artifacts requiring inspection.

## Where the work is moving

The clearest evidence that engineering work is moving toward assurance appears inside the workflow itself. AI increases the volume of code entering repositories, but review, integration, security validation, and operational confidence do not scale at the same rate. The pressure moves downstream.

This isn't a new argument; others are describing the same pattern. [Google Cloud's Office of the CTO](https://cloud.google.com/transform/when-ai-writes-the-code-who-reviews-it-cto-google-cloud) places the constraint at review and integration, while [TypeDB](https://typedb.com/blog/the-new-economics-of-code-verification-is-the-new-bottleneck) calls verification the limiting step.

[Faros AI](https://www.faros.ai/research/ai-acceleration-whiplash), analysing two years of data across roughly 22,000 developers, reports more completed work alongside longer reviews, more incidents and bugs, and more changes merged without review. [LinearB](https://linearb.io/blog/8-million-prs-engineering-productivity), across 8.1 million pull requests, found larger AI-assisted changes and much longer waits for agent-generated pull requests to be reviewed.

The quality data points in the same direction. [GitClear](https://www.gitclear.com/ai_assistant_code_quality_2025_research) found duplication increasing while refactoring declined. [Veracode](https://www.veracode.com/blog/genai-code-security-report/) found known security flaws in 45% of generated samples, and its [2026 update](https://www.veracode.com/blog/spring-2026-genai-code-security/) showed security results remaining flat while syntax correctness improved. [Apiiro](https://apiiro.com/blog/4x-velocity-10x-vulnerabilities-ai-coding-assistants-are-shipping-more-risks/) reported trivial defects falling while privilege-escalation paths and design flaws rose sharply.

Of course, these are vendor-published sources, and their precise numbers deserve scrutiny. But they converge on a pattern I am witnessing firsthand: output arrives faster, while review, validation, integration, and system-level judgment remain stubbornly human.

An important observation is that defects may not simply increase; their distribution also appears to be moving from shallow to deep. While AI can eliminate errors that a linter, type system, or competent reviewer would catch quickly, it can also introduce errors that require someone to understand the system’s trust model, state transitions, operational assumptions, and blast radius—defects that may look reasonable during review and announce themselves only after deployment.

A common response in the industry is to use AI to scale verification alongside AI adoption. AI can review AI-generated work, and that can substantially expand verification capacity. Models can generate tests, inspect changes, compare alternatives, identify suspicious patterns, and direct attention toward likely problems. But the generator and reviewer may share the same incomplete context or mistaken assumptions. AI can scale the search for defects; it does not, by itself, establish that a change belongs in a particular production system. It should remain an additional layer of evidence rather than a substitute for deterministic controls, system understanding, and accountable judgment.

This is also why [DORA’s 2025 research](https://dora.dev/research/2025/dora-report/) complicates any simple anti-AI reading. It found AI adoption associated with better throughput and product performance, but also with increased delivery instability. Its broader conclusion—that AI amplifies an organization’s existing strengths and weaknesses—is more useful than any single effect size.

AI amplifies the engineering system around it. Teams with strong platforms, fast feedback, reliable tests, and disciplined review can convert generation speed into useful delivery. Teams without them become faster at producing work their systems are not prepared to absorb.

Some organizations will move faster not simply because they adopt better models, but because their systems are easier to understand, test, observe, and change safely.

Generated code makes complexity easy to create and expensive to carry.

## What this changes for software engineers

The evidence does not show that AI is about to replace software teams, but these tools are no longer merely better autocomplete. The direction worth preparing for is one in which supervised delegation becomes ordinary. Engineers will not stop coding, but manual implementation may become a smaller part of what distinguishes them. One engineer may supervise more work and become responsible for a wider surface.

The role I expect to remain most valuable is that of the engineer who holds enough technical and business context to take responsibility for a production system: what it should do, how it fits the architecture, and what happens when it is live—its reliability, security, performance, data integrity, operability, and the pager.

“Write the endpoint” is a task. “Be answerable for whether the endpoint should exist, whether it leaks data, whether its retries amplify an outage, and whether it survives Election Day” is accountability. The tasks can change substantially while the need remains for someone who understands the context and is answerable for the outcome.

The future engineer may write less of the code while needing to understand more of the system.

That points to several areas that deserve engineers’ attention.

- **Technical foundations and first-principles reasoning.** This is the area many assume engineers already have covered, and in my experience that assumption is often wrong. Networking, operating systems, databases, concurrency, distributed systems, identity, and failure modes provide the mental models needed to evaluate generated work. The lower layers do not disappear because our tools make them easier to ignore.

- **Specification and system design.** Clear requirements, constraints, invariants, failure behavior, and explicit non-goals become more valuable when implementation can be delegated. As “can we build it?” becomes less constraining, “should this exist?” and “is this the simplest safe form?” matter more.

- **Verification, security, and failure-mode judgment.** Engineers need to reconstruct intent, reason about adversarial and degraded conditions, and treat “it looks right” as the beginning of review rather than the end. Deterministic controls—tests, types, schemas, static analysis, tightly scoped permissions, and reliable rollback—remain essential. AI review can provide another signal, but it does not turn probabilistic output into verified fact.

- **Calibrated delegation.** The question is not whether to use AI, but how much control to delegate, what evidence to require back, and when verification costs exceed generation benefits. Engineers need to recognize where models are useful and where local context, business rules, security concerns, or blast radius require tighter supervision.

- **Operational reasoning and ownership.** As more production code is generated or reviewed quickly, engineers will rely less on authorship memory and more on architecture, telemetry, code, logs, traces, and first principles. They will increasingly own systems they did not assemble line by line.

These are ordinary software-engineering disciplines, not AI-engineering specialties. They become more important because of the environment in which software is now being produced.

## Conclusion

AI is not making software engineering disappear. It is changing the scarce part of the work. As more implementation can be delegated, the engineer’s advantage moves toward understanding systems, defining behavior, verifying outcomes, managing failure, and owning consequences. That makes technical foundations more valuable, not less.

AI will not produce the same gains everywhere. Teams that can test, observe, review, and change software safely will move faster. Teams that cannot may simply accumulate complexity and technical debt faster.

The timing of this shift may unfold differently. Model progress may slow, accelerate beyond what I expect, or be accompanied by forms of automated verification that reduce assurance costs. But the practical response changes little: strong foundations, precise specification, disciplined verification, security judgment, and operational maturity remain valuable whether AI completes thirty minutes of work or three days.

I do not know exactly where this settles, and I do not think anyone does. Based on what we can observe today, this is the direction I would prepare for.

## References

- Becker, J., Rush, N., Barnes, E., and Rein, D. - [*Measuring the Impact of Early-2025 AI on Experienced Open-Source Developer Productivity*](https://arxiv.org/abs/2507.09089). METR, July 2025.
- METR - [*We Are Changing Our Developer Productivity Experiment Design*](https://metr.org/blog/2026-02-24-uplift-update/). February 2026.
- DORA / Google Cloud - [*2025 State of AI-Assisted Software Development*](https://dora.dev/research/2025/dora-report/).
- Faros AI - [*The Acceleration Whiplash*](https://www.faros.ai/research/ai-acceleration-whiplash), based on two years of telemetry from roughly 22,000 developers across more than 4,000 teams.
- LinearB - [*8 Million Pull Requests Reveal Where Engineering Productivity Breaks Down*](https://linearb.io/blog/8-million-prs-engineering-productivity), based on its 2026 engineering benchmarks.
- GitClear - [*AI Copilot Code Quality: 2025 Look Back at 12 Months of Data*](https://www.gitclear.com/ai_assistant_code_quality_2025_research), analysing 211 million changed lines from 2020 through 2024.
- Apiiro - [*4x Velocity, 10x Vulnerabilities: AI Coding Assistants Are Shipping More Risks*](https://apiiro.com/blog/4x-velocity-10x-vulnerabilities-ai-coding-assistants-are-shipping-more-risks/), based on large-enterprise repository data.
- Veracode - [*2025 GenAI Code Security Report*](https://www.veracode.com/blog/genai-code-security-report/) and [*Spring 2026 GenAI Code Security Update*](https://www.veracode.com/blog/spring-2026-genai-code-security/).
- Boonstra, L. - [*When AI writes the code, who reviews it?*](https://cloud.google.com/transform/when-ai-writes-the-code-who-reviews-it-cto-google-cloud). Google Cloud, 2026.
- Hananda, G. - [*The new economics of code: verification is the new bottleneck*](https://typedb.com/blog/the-new-economics-of-code-verification-is-the-new-bottleneck). TypeDB, 2026.
- Bolinsky, A. - [*AI is the New Compiler*](https://abolinsky.io/blog/ai-compiler/).
- Jackson, J. - [*Martin Fowler on Preparing for AI's Nondeterministic Computing*](https://thenewstack.io/martin-fowler-on-preparing-for-ais-nondeterministic-computing/), including Fowler's "dodgy collaborator" analogy.

> **AI assistance acknowledgment**: This article was produced with AI assistance. Research synthesis, source gathering, and multiple rounds of drafting involved AI. The thesis, argument, editorial decisions, and final responsibility are mine.
