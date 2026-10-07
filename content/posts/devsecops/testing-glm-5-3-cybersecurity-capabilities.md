---
title: "Testing GLM-5.3's Cybersecurity Capabilities"
description: "A hands-on experiment with Z.ai's GLM-5.3, Claude Code, Kali Linux, OWASP Juice Shop, and a custom vulnerable target to evaluate autonomous cybersecurity reasoning."
date: 2026-10-06T23:16:00-04:00
lastmod: 2026-10-06T23:16:00-04:00
draft: false
tags:
  - "artificial intelligence"
  - "cybersecurity"
  - "devsecops"
---

Last week, [Anthropic published an interesting post](https://www.anthropic.com/research/glm-5-3-and-the-spread-of-advanced-cyber-capabilities) analyzing the cybersecurity capabilities of Z.ai’s open-weight GLM-5.3 model and comparing some of its exploit-development performance with Claude Mythos Preview. Anthropic’s testing found GLM-5.3 surprisingly close to Mythos Preview on some of the harder exploit-development benchmarks.

A couple of weeks earlier, [NIST’s Center for AI Standards and Innovation (CAISI) had published its own assessment](https://www.nist.gov/news-events/news/2026/09/caisis-assessment-zais-glm-53-cyber-capabilities). CAISI reached a somewhat more conservative conclusion: it described GLM-5.3 as the most cyber-capable open-weight model released so far, while estimating that its overall cyber capability still trails the current U.S. frontier by roughly four months.

There is an important bit of context here. Anthropic has deliberately restricted access to some of the most capable cybersecurity functionality in its models. [Fable 5.1 and Mythos 5.1](https://www.anthropic.com/claude-fable-and-mythos-5-1), for example, are the same underlying model but are deployed with different safeguard and access regimes: Fable is generally available, while Mythos is available through trusted-access programs with safeguards intended to support professional cybersecurity and life-sciences work. Anthropic has been explicit that these restrictions are driven in part by the cybersecurity capabilities of the models.

Obviously, there are vendor incentives and some amount of marketing framing behind announcements like these. Still, there is a meaningful signal here. If an open-weight model that anyone can download is approaching the same general class of exploit-development capability that caused Anthropic to restrict access to Mythos-class models, that is worth paying attention to, and I wanted to take a closer look and see it for myself.

The first challenge was figuring out how to get practical access to GLM-5.3. Downloading the weights is easy enough; running the full model well is a very different story. GLM-5.3 is a 753-billion-parameter model, and the published checkpoint is hundreds of gigabytes. My old Linux server with a Xeon processor and 128 GB of RAM was not going to run it at anything resembling useful agentic performance.

I looked at renting enough cloud GPU capacity to run it myself, but that quickly became more expensive than I was willing to spend on an experiment. After comparing a few API-based options, I decided to use GLM-5.3 through [Together AI](https://www.together.ai/).

I created an account, added $15 in credit, which was the minimum the interface allowed me to add at the time, and set that as my budget. [Together Link](https://www.together.ai/link) turned out to be particularly convenient because it can connect Claude Code directly to models hosted by Together, including GLM-5.3. Within a few minutes, I had Claude Code running with GLM-5.3 as the underlying model.

Before testing anything, I needed to give the agent access to a reasonable set of security tools without installing a pile of offensive-security packages directly on my laptop.

I have been using [Lima](https://lima-vm.io/) for experiments for some time and have [written about it before](https://deployment.properties/tags/networking/). I wondered whether I could simply spin up a Kali Linux VM with Lima and let Claude Code use that environment for security testing.

It turned out to be surprisingly easy. I created a Kali template for Lima and used Lima’s MCP integration to expose the Kali environment to Claude Code. The result was a fairly clean architecture: Claude Code and GLM-5.3 could reason and orchestrate from the host, while security tooling and command execution happened inside the Kali VM, keeping my laptop clean.

I documented the Kali/Lima setup and the supporting artifacts from these experiments in [this GitHub Gist](https://gist.github.com/soeirosantos/23e69d4f187b41b1557f7ae281eee5c4#file-z_kali_on_lima_via_mcp_claude_code-md).

For the first target, I used [OWASP Juice Shop](https://owasp.org/www-project-juice-shop/) purely as a smoke test. GLM-5.3 made very short work of it. It compromised the application in roughly a minute and produced a detailed report. I asked the model to be explicit about whether previous knowledge of Juice Shop had influenced its approach, and it acknowledged that it had. That was already obvious from watching the methodology. It knew several Juice Shop-specific paths and techniques in advance. The Juice Shop report generated by Claude Code is also available in [the experiment Gist](https://gist.github.com/soeirosantos/23e69d4f187b41b1557f7ae281eee5c4#file-y_juice_shop_report-md).

Even with that caveat, one thing stood out: it moved extremely quickly through the environment, validated assumptions against the live target, adapted when remembered techniques no longer worked, and chained findings autonomously.

For the second test, I partnered with ChatGPT—GPT-5.6 Sol, which also helped me throughout this experiment—to build a completely new vulnerable target. The result is an application that you can see on [Bookshop VWA](https://github.com/soeirosantos/bookshop-vwa). The application was intentionally designed to be small enough to run locally in Docker, but to require a multi-stage compromise.

The intended path looked roughly like this:

**web application → initial vulnerability → internal service → low-privilege code execution → credential discovery → normal user access → local privilege escalation → root**

The runtime username, credentials, internal details, and flags were randomized. More importantly, the target had never been published or used as a CTF before the test. GLM-5.3 had no walkthrough to remember.

Some people will reasonably look at it and call it an easy target. And I agree. This wasn't intended to reproduce the sophistication of Anthropic's or CAISI's benchmarks or a sophisticated box challenge. I wanted to see how an agent running GLM-5.3 could reason independently through a completely unknown environment, discover the attack path, recover from mistakes, cross privilege boundaries, and ultimately obtain root access without me providing exploitation guidance.

The agent struggled badly at first. One of the biggest problems was actually my test harness. Claude Code was interacting with Kali through MCP, and GLM became extremely aggressive with enumeration and fuzzing. It launched enough concurrent activity that the Kali VM eventually became unresponsive. I had to force-stop and restart the VM.

The network topology also confused the agent considerably. Kali was running inside Lima, while the vulnerable application was running inside Docker on the macOS host. This meant the agent had to reason about multiple network perspectives and multiple meanings of `localhost`. At one point, it even encountered unrelated services exposed through the host, including the previous Juice Shop instance.

GLM-5.3 identified an SSRF possibility in the Bookshop application very quickly, but then spent a surprisingly long time failing to turn that finding into useful access. It tried multiple approaches, revisited enumeration, probed things like cloud metadata services, fuzzed additional paths, and generally started to look like it was spinning.

I was very close to terminating the assessment when the agent figured it out. It successfully used the SSRF to reach an internal-only diagnostic service, discovered a command-injection vulnerability, and obtained code execution. From there, the pace changed completely.

It independently enumerated the system, discovered the next stage of the attack path, obtained normal user access and the user flag, identified the local privilege-escalation path, and obtained root remarkably quickly. The final report and supporting test artifacts are available in [the same GitHub Gist](https://gist.github.com/soeirosantos/23e69d4f187b41b1557f7ae281eee5c4#file-b_final-report-md).

There are several caveats to this experiment. Bookshop VWA is deliberately modest compared with the exploit-development challenges used by Anthropic and CAISI. The infrastructure itself introduced significant noise. And this was one run, not a statistically meaningful benchmark. Still, I came away impressed. The most interesting part wasn't simply that GLM-5.3 eventually obtained root. Models have been able to solve vulnerable lab machines for some time. What impressed me was the degree of autonomy. It discovered the vulnerabilities itself, pursued multiple hypotheses, recovered after getting badly stuck, chained distinct weaknesses together, crossed privilege boundaries, and completed the objective without receiving exploitation guidance from me.

The only question the agent asked me during the assessment was to confirm the authorized scope, and my only manual intervention was operational: I had to recover the Kali VM after the fuzzing workload overwhelmed it. I did not provide hints about the vulnerabilities or the intended attack path.

Next, I want to explore running GLM-5.3 on infrastructure I control directly and, of course, run it against a more elaborate application in a cloud environment with multiple components. Another thing that I'm looking forward to is checking how it performs against other models.
