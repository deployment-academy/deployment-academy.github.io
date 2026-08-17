---
title: "Autonomous Agentic Coding With Deterministic Verification Using Claude Code"
description: "This post describes an experiment in which Claude Code is used to build a nontrivial system without human assistance."
date: 2026-08-16
lastmod: 2026-08-16
draft: false
sidebar: "right"
widgets:
  - "ddg-search"
  - "recent"
  - "social"
tags:
  - "artificial intelligence"
  - "software engineering"
---

After I wrote [How AI Is Changing Software Engineering Work](https://deployment.properties/posts/eng/how-ai-is-changing-software-engineering-work/), I kept returning to one part of the argument: whether deterministic verification steps would help increase trust in AI-generated output. When I considered which languages and platforms would favor that, Go and Rust came to mind first. While other languages can, of course, also offer good verification controls, Go and Rust came to mind as good candidates because both ship standardized toolchains with strong support for automated, deterministic verification. Tests, formatting, vetting, and compilation can all be expressed as repeatable commands with clear pass/fail outcomes — exactly the kind of signal an agent can be held to.

Between the two, Rust offers stronger compile-time guarantees through its type system, ownership model, and borrow checker, which can prevent entire classes of memory-safety and concurrency errors before the code runs. That rigor comes at a cost in verbosity and compile latency, both of which matter more than usual when an agent is iterating through a generate–compile–verify cycle.

I decided to run it as an experiment, starting with Go. As I worked through the setup, another requirement became clear. If I wanted to make a stronger case that deterministic verification can increase trust in AI-generated code, I needed to let the agent operate as autonomously as possible. Every change I made to the code along the way would be my contribution to the result, and it would get harder to tell whether the verification was helping.

That created another problem. Giving the agent more autonomy also meant giving it more opportunities to consume time, tokens, and budget without producing useful progress. I needed an execution harness that would give the agent enough freedom to work independently while still giving me control over its boundaries and visibility into what it was doing.

<!--more-->

## The machinery

The approach is straightforward: give an agent a written specification, have it decompose the work into bounded tasks, delegate each task to a subagent, and refuse to let any task close until a deterministic check passes. Everything else — the sandbox, the telemetry, the escalation rules — exists to make that loop safe to run unattended.

This methodology isn't perfect, nor is it intended as a blueprint for production adoption. It's an experiment designed to explore how realistic an approach like this would be in day-to-day software development. I recognize that many teams are already running autonomous agentic workflows. My premise is that trust in AI-generated output remains limited, and that human verification still represents significant overhead across the industry. To be clear, I'm not looking for — or proposing — a way to eliminate human review. The goal is narrower: to see whether deterministic verification can raise the floor on what reaches a human reviewer, and reduce how much of it has to be taken on faith.

### Specification

The specification is the starting point. Nothing about it is specific to this machinery, though — a specification like the one proposed here would be just as useful for any agentic coding approach.

It is a technical design document describing what the agent needs to build. It stays on the *what* and avoids the *how*. It should be as detailed as you can make it and avoid ambiguity, covering functional requirements along with the non-functional ones that matter — logging, caching, access control, and so on. It doesn't need to describe an entire component; most of the time it won't. A specification can scope a single task or unit of effort just as well, though nothing requires it to be focused on small tasks.

The quality of that document determines how the rest goes. A good specification makes the planning phase smooth. A poor one turns planning into a back-and-forth with the planning agent, which is exactly the cost you're trying to avoid by delegating in the first place. The most reliable way I've found to produce one is to work with an agent in a preliminary session — discussing and brainstorming the feature or system in scope, then writing the specification from that conversation.

Both specifications used in this experiment are in the companion repository: a [simple one for a health check endpoint](https://github.com/soeirosantos/taskforge/blob/experiment/warmup-go-healthcheck-3/SPEC.md), and a [considerably more detailed one for the job processing service](https://github.com/soeirosantos/taskforge/blob/experiment/go-job-processing-service/SPEC.md).

### Task planning

Planning is a separate phase with its own instructions, run before any implementation begins. The agent reads the specification, inspects the repository, and produces a plan that breaks the work into bounded tasks — each with an objective, its dependencies, acceptance criteria, and the verification command that proves it. I review that plan before anything is built — the design assumes a human does.

The planning instructions are a somewhat opinionated document covering workflow behavior, the types of subagents available, implementation guidance, and how acceptance criteria should be written. It's mostly stable across runs, though it can be refined as needs change — mine reflects how I wanted this experiment to run, not a general recommendation. The [full planning instructions document is in the repository](https://github.com/soeirosantos/taskforge/blob/main/.claude/TASK_PLANNING_INSTRUCTIONS.md), and there is considerably more in there than this section covers.

You can see a [concrete plan output here](https://github.com/soeirosantos/taskforge/blob/experiment/go-job-processing-service/experiment/PLAN.md). The plan is the outcome of a prompt that says `Check the @TASK_PLANNING_INSTRUCTIONS.md against @SPEC.md and write it to the workspace`.

### Agent orchestration

The work is split between two kinds of agent. The main Claude Code session — the orchestrator from here on — reads the plan, dispatches one subagent per task, and decides what happens when each one returns. The subagents do the implementation. I sit outside both, reviewing at checkpoints.

Breaking work up this way adds complexity, so it needs to earn its place. It does, for two reasons.

The first is context. A single agent carrying an entire implementation accumulates everything it has read and written, and that context is re-sent on every turn. Handing a task to a subagent gives it a clean context containing only what that task needs, and returns only a report to the orchestrator, which stays the one place that holds the whole picture.

The second is containment. A subagent that misunderstands its task, or gets stuck investigating a detail, does so inside a boundary. It cannot spend the whole budget, and it cannot quietly widen its own scope. When it finishes — successfully or not — control returns to the orchestrator, which decides what happens next.

Containment also depends on the graph staying flat. Workers cannot delegate further — only the orchestrator dispatches — so the work can't branch unboundedly into subagents spawning subagents. That also keeps the task lifecycle in one place: the orchestrator opens each task, hands it out, runs the verification when the worker returns, and closes it. A worker closing its own task would be self-certifying, which is the whole thing this is built to avoid.

Tasks run sequentially by default. Parallel execution is available where the plan identifies genuinely disjoint work, but it is treated as an opportunity rather than a goal.

### Definition of done and stop conditions

This is where the premise from the beginning of the post becomes a mechanism. A task is complete when its acceptance criteria are satisfied and the complete unit-test suite passes — both, not either.

The important part is that the second half is not an instruction. Telling an agent to run the tests before it finishes is a request, and a request can be skipped, misremembered, or reported optimistically. Instead, the check is wired into the moment of completion itself: a [blocking hook](https://github.com/soeirosantos/taskforge/blob/main/.claude/hooks/verify-unit-tests.sh) runs the suite when the agent tries to close a task, and can refuse. The agent cannot opt out of it, cannot substitute its own account of the result, and cannot close the task by asserting it is done — which replaces a claim with a check. The gate script itself lives in the repository the agent can write to, so "cannot" is a property to be verified rather than assumed.

It fails closed. A missing test command, a suite that cannot run, a suite that exceeds its timeout, a command that isn't found — all of them refuse completion rather than waving it through, on the principle that a verification step passing while broken is worse than no verification step at all. The mirror image of that risk is a suite that exits successfully while running no tests, which would technically satisfy the gate; that case is detected and recorded separately so it can't quietly count as a pass.

What the gate deliberately does *not* do is certify the work. It cannot see the acceptance criteria — it only knows whether the tests passed. So it can refuse completion, but it can never authorize it. The script says as much when it succeeds: the suite passed, and that is not a statement about whether the task was actually done. Judging the criteria remains the orchestrator's job, and during the run that occasionally meant sending a task back to a worker even though the gate had passed. Any experienced engineer already knows that green tests aren't proof of correct software. That's precisely why the gate is positioned as a veto rather than a stamp of approval: it removes a category of failure, it doesn't replace review.

That veto is what makes stop conditions necessary. If a gate can refuse indefinitely, something has to decide when to quit. Each worker gets a hard turn limit, so no single attempt runs forever. When a worker exhausts its budget without satisfying its acceptance criteria, the task gets exactly one further attempt on a stronger model. If that also fails, autonomous work on that task stops, dependent tasks do not proceed, and the task waits for a human. There is no tier after the escalation tier, deliberately — an unbounded retry ladder is the failure mode this is meant to prevent, not a fallback to add later.

{{< notice type="tip" id="verification" title="The verification can go beyond unit tests" >}}
One thing worth making explicit: the gate is a slot, and for this experiment I filled it with the unit-test suite because that was the simplest thing that could work. Nothing about the design requires that. Any automated check that exits non-zero on failure fits — linters, security scans, integration or smoke tests that exercise the system at a more functional level. The tradeoff is latency. Every check added runs on every completion attempt, and a slower gate means a slower loop for an agent that may hit it many times.
{{< /notice >}}

### Sandboxing

Running an agent with permission prompts disabled is only reasonable if it cannot reach anything that matters. Rather than adopt existing sandboxing tooling, I put together a quick Docker Compose setup that was good enough for this purpose.

The container mounts the repository and nothing else — no host home directory, no credentials, no other projects. Language toolchains and the Claude Code version are pinned in the image, so a run is reproducible and the agent's own version is part of the recorded apparatus rather than whatever happened to be installed. Inside that boundary the agent runs with permissions skipped, which is what allows it to work for long stretches without stopping to ask. The blast radius is a directory that can be restored from git.

The [sandbox setup is in the repository](https://github.com/soeirosantos/taskforge/tree/main/.claude/sandbox) for anyone who wants the details.

### Telemetry

The telemetry I put together is just enough to serve the experiment. It uses Claude Code's capability to export OpenTelemetry metrics, so the setup is a small Docker Compose stack — an OTLP collector, Prometheus, and Grafana — collecting cost, token usage, and session data, labelled per experiment run.

It served two purposes. During the run itself it let me watch progress as it happened rather than waiting for it to finish, which matters when an agent is working unattended for long stretches. Afterward it provided the numbers used to evaluate the run, independent of anything the agent reported about itself.

The [telemetry stack is in the repository](https://github.com/soeirosantos/taskforge/tree/main/.claude/metrics).


{{< notice type="warning" id="note" title="A note on reinventing things" >}}
If you use Claude Code regularly, you will have noticed that a good part of what I described above overlaps with what the tool already does. It has planning modes, it dispatches subagents, it runs work in parallel, and it tracks tasks on its own. I did not need to specify most of that to get an agent to build something.

I made it explicit anyway, for a reason specific to running an experiment rather than shipping work. Built-in behavior is a moving target: it improves, it changes between versions, and it is not mine to pin. Writing the decomposition, the boundaries, and the escalation rules down as documents in the repository gave me predictable control over how the work was broken up and executed, run after run.

The completion gate is the part with no built-in equivalent, and it's the reason the rest exists in the form it does. Refusing to let a task close is a policy decision, not a capability — no tool ships an opinion about what *done* means in your repository. Once that refusal is the center of the design, the surrounding pieces have to be explicit enough to hang it on.
{{< /notice >}}

## Initial tests

Before running anything real, I put the machinery through several cycles with a deliberately small use case: a [health check specification](https://github.com/soeirosantos/taskforge/blob/experiment/warmup-go-healthcheck-3/SPEC.md), small enough that any problem I hit would be a problem with the machinery rather than with the work. The [resulting code is on this branch](https://github.com/soeirosantos/taskforge/tree/experiment/warmup-go-healthcheck-3), and there is [a Rust version](https://github.com/soeirosantos/taskforge/tree/experiment/warmup-rust-healthcheck) too, if you want to see the same setup pointed at a different toolchain.

Those cycles turned up a couple of small but consequential infrastructure problems in the sandboxing and telemetry, along with a series of adjustments to the planning instructions and the other guidance documents. I won't detail them all — the [commit history on `main`](https://github.com/soeirosantos/taskforge/commits/main/) has the details if you're curious. Two lessons came out of it, and they're the reason this section exists at all.

**Broken machinery pulls the agent into fixing it.** Until the infrastructure was working exactly as intended, the agent kept drifting into it. It would notice something off in the sandbox, the telemetry, or the orchestration, start reasoning about the cause, and propose fixes for the machinery that was supposed to be constraining it. It wasn't malfunctioning; it was doing what a capable engineer would do when the tooling misbehaves. But the effect is that infrastructure noise becomes indistinguishable from the task, and time gets spent on the setup instead of the specification. Getting it genuinely quiet was a precondition for autonomy, not a nice-to-have.

**The specification carries more weight than anything else.** I said earlier that the quality of that document determines how the rest goes; this is where I learned it. Early iterations turned into long back-and-forth clarification with the agent — precisely the cost delegation is supposed to avoid. Ambiguity in the specification doesn't stay in the specification. It surfaces later as questions, or worse, as confident guesses.

Once both were addressed, the loop worked the way it was meant to: the agent stayed on the task, and my involvement dropped to reviewing output and confirming the flow at checkpoints.

## The main experiment

I wanted something past a "Hello World", but time and budget ruled out building an entire product. Something non-trivial would be enough to show what I was looking for. What I settled on was a job processing service: an HTTP API that accepts jobs, persists them, runs them on a worker pool, and supports listing, cancellation, and retry. The jobs themselves are deliberately dummy — hash a string, sleep, fail on purpose — but the system around them is designed properly, with atomic state transitions, startup recovery, cancellation of jobs already running, and graceful shutdown. The [full specification is here](https://github.com/soeirosantos/taskforge/blob/experiment/go-job-processing-service/SPEC.md).

During the entire execution I gave no technical instructions and never told the agent how to implement anything. I could have, and I intentionally didn't. My interaction was limited to answering flow questions, confirming checkpoints, and resolving the one escalation that reached me.

### The task breakdown and execution

Planning produced nine tasks. The dependency structure is mostly linear — domain, then persistence, then transitions, then the worker pool, then HTTP, then composition — with the README and cross-cutting race tests able to run in parallel at the end.

| Task | Scope | Dispatches | Notes |
|---|---|---:|---|
| T1 | Domain model, executors, module init | 4 | 2 turn exhaustions, 1 human escalation |
| T2 | Persistence foundation | 2 | 1 send-back after review |
| T3 | Atomic transitions, startup recovery | 1 | rated highest-risk in planning |
| T4 | Worker pool, cancellation coordination | 1 | dispatched directly to Opus |
| T5 | HTTP foundation | 1 | |
| T6 | List, cancel, retry endpoints | 2 | escalated to Opus |
| T7 | Composition root | 1 | |
| T8 | Cross-cutting race tests | 1 | ran in parallel with T9 |
| T9 | README | 1 | ran in parallel with T8 |

Most of it ran smoothly. Three tasks are worth a closer look.

**T1 cost the most and taught the least about the code.** It took four dispatches and produced the run's only human escalation. What's notable is *why* it failed: both the initial Sonnet attempt and the Opus escalation ran out of turns rather than getting the problem wrong. The first spent its budget investigating whether Go's `json.Number` was needed to reject a non-integer field; the second consumed that answer, wrote 1,208 correct lines across nine files, and stopped one undefined test helper short of a compiling package. Neither produced a bad design. The ladder had escalated a *configuration* problem to a stronger model, which a stronger model cannot fix. The escalation record came to me, and the resolution was to raise the turn budgets — Sonnet 15 → 40, Opus 15 → 50 — and re-run T1 from the normal tier. That is exactly what a human escalation is for, and it is also the run's clearest cost: a badly calibrated budget is expensive, and no amount of model quality compensates for it.

**T2 exposed a hole in the policy I had written.** The worker passed the gate, but the orchestrator's own review found one acceptance criterion unmet — a persistence test that didn't actually verify what it claimed across a reopen. The worker hadn't failed and hadn't exhausted its budget, so the escalation ladder didn't apply; escalating would have been the wrong instrument. What the situation needed was to hand the specific defect back to the same agent with its context intact, which took four tool uses and about twenty-six seconds. It worked, but this step wasn't in the written policy — the orchestrator improvised it because the case fell between the rules. It's since been recorded as a gap rather than quietly folded in.

**T6 is the case that justifies the design.** A Sonnet worker exhausted its budget and left behind a two-line compile error — two references to an undefined symbol in a test file. The policy said escalate to Opus. The orchestrator noted at dispatch time that this looked disproportionate: a cheap, narrow repair would obviously fix two undefined symbols, and spending an escalation on it seemed wasteful. It followed the ladder anyway, on the grounds that substituting its own judgment for the policy is precisely the drift the machinery exists to catch.

That judgment was wrong and the policy was right. `go vet` reports only the first failure, so the compile error was *masking* everything behind it. Once Opus made the package compile, two more problems surfaced — a test that contradicted the specification's ordering guarantee, and, far more seriously, **acceptance criteria 2–5 had no tests at all.** The cancel and retry handlers were entirely unexercised; the first attempt had spent its whole budget on the list endpoint.

Consider what the cheap fix would have done. Patch the two symbols, and the suite goes green with two handlers completely untested — and **the gate passes it**, because the gate runs the tests that exist, not the tests that should exist. This is the argument from the definition-of-done section arriving as a fact instead of a claim: the gate removes a category of failure, and it is not a certificate of correctness. What caught this was not the gate. It was a bounded escalation policy that the orchestrator followed even when it looked like the wrong call.

Worth noting what the gate log shows across the whole run: nine closures, nine passes, no refusals. That isn't the gate proving itself — it's the orchestrator running the suite before attempting to close anything, so a failing task never reached the gate in the first place. The refusals happened earlier, during the calibration runs, which is where I learned what a failing gate looks like. In this run the gate worked as a standing constraint rather than as an event.

The [full execution record](https://github.com/soeirosantos/taskforge/blob/experiment/go-job-processing-service/experiment/EXECUTION_NOTES.md) has all nine tasks at this level of detail, including the mutation tests used to check that new assertions actually fail when the behaviour they cover is broken.

### What it built

| | Files | Lines |
|---|---:|---:|
| Production Go | 23 | 2,473 |
| Test Go | 16 | 4,832 |
| **Total** | **39** | **7,305** |

A test-to-production ratio of **1.95 : 1**, across five packages: domain, store, worker pool, HTTP API, and the composition root. **135 top-level tests, 207 counting table-driven subtests.** The build is a single 14 MB static binary with one direct dependency, and it compiles clean under `go vet`, `gofmt`, and the race detector.

Beyond code, the run produced a 389-line README covering the twenty items the specification asked for, plus its own execution notes and escalation record — about 9,800 lines authored in total.

The parts I'd point at as non-trivial are the concurrency guarantees: jobs transition state atomically under contention, a job already running can be cancelled mid-flight, the service recovers jobs left in-flight by an unclean shutdown, and there are forced-race tests that deliberately collide cancellation with completion to prove exactly one of them wins.

### Stats

| | Value |
|---|---|
| Wall clock | 11 h 36 m |
| **Active time** | **~2 h 35 m** |
| Subagent execution within that | ~1 h 26 m |
| Subagent invocations | 15 (14 distinct agents) |
| Tool uses | 413 |
| Subagent tokens | 972,842 |
| Dispatches per task (mean) | 1.56 |
| Tasks completed in one dispatch | 5 of 9 |
| Opus invocations | 3 (2 escalations, 1 planned dispatch) |
| Human escalations | 1 |
| Tests weakened, skipped, or deleted | **0** |

The gap between wall clock and active time is almost entirely me waiting for subscription usage limits to refresh — roughly nine hours of the eleven and a half. Budget set the calendar here, not capability.

One caveat on the table above: orchestrator usage isn't in it. Those figures come from the subagent records, which the orchestrator can see, and it cannot see its own consumption — so 972,842 tokens covers workers only, and the real total is considerably higher.

The telemetry stack is where that gap gets filled, and the split it shows is the most surprising number in the run. Total API-equivalent cost was **$4.21** across **3.09 million tokens** — and the orchestrator accounts for $2.97 of the $4.21, roughly three times all fifteen worker dispatches combined. That is the direct price of making the orchestrator re-run verification itself after every task instead of trusting a worker's report. Prompt caching is what keeps the absolute number small: 85 % of those tokens were cache reads and only 0.3 % were fresh input. Both totals are floors rather than exact figures — the collector's counters reset between windows, so the sums understate — and this ran on a subscription, so $4.21 is an API-equivalent price rather than anything I was billed.

Against that, I estimated what the same deliverable would take a solo mid-to-senior Go developer working from the same specification and holding the same quality bar: **42–67 focused engineering hours**, or 5–8.5 working days. That puts the ratio somewhere around 16× to 26×.

That number needs handling with care, so here is the version I'll actually defend. It is *not* "the agent is 20× a developer." The specification was exceptionally detailed and writing it was real work counted on neither side. A human stayed in the loop, approving the plan and making the turn-budget call. The run needed one human escalation and a mid-run configuration change to get past T1, and without them it would have stalled. The narrower claim is the one worth making: **a specified, race-tested, 7,300-line Go service with 135 passing tests and no unresolved requirements was produced in about two and a half hours of active time, with one human decision point during the run.**

The [full statistics](https://github.com/soeirosantos/taskforge/blob/experiment/go-job-processing-service/experiment/STATS.md) label every number as measured, derived, or estimated, and state what couldn't be measured.

### Code quality

Everything so far is the harness grading its own homework. The tests were written by the same process that wrote the code, and a passing suite is exactly the evidence I've spent this post arguing is necessary but not sufficient. So after the run finished, I put the code through analysis tools that had no part in producing it — `golangci-lint`, `gosec`, `gocyclo`, coverage, and the race detector.

Across 7,305 lines, they returned nine findings.

| Check | Result |
|---|---|
| `golangci-lint`, 13-linter extended set | 9 findings total |
| `gosec` | 8 findings: 6 LOW, 2 MEDIUM |
| Test coverage | **83.0 %** of statements |
| Cyclomatic complexity | no production function above 12 |
| `go vet` / `gofmt` / race detector | clean |

Across both tools, **one is a real defect** — the HTTP server is constructed without timeouts, a one-line fix. Everything else is an intentional idiom or a false positive on inspection. Notably, the one real finding isn't a violation of the brief either: the specification never asked for server timeouts, and the run was under standing instructions not to add what wasn't specified. The agent held that line closely — across the whole run there was exactly one departure from the specification, an error code added for an otherwise unreachable branch, and it was flagged and documented as an addition rather than slipped in. The gap here is in the specification, not in the execution of it.

`gosec` raised two MEDIUM findings and six LOW ones; the six LOWs are unchecked error returns, the same ones the linter reports. Of the two MEDIUMs, one is the server-timeout defect above. The other is a false positive — a flagged SQL string concatenation that builds the query template from string literals, with every user-supplied value bound as a parameter. Given that the store layer is hand-written SQL throughout, that was the finding most worth checking, and it held up.

I went through each finding and wrote up the reasoning in the [full analysis](https://github.com/soeirosantos/taskforge/blob/experiment/go-job-processing-service/experiment/CODE_QUALITY.md), including the commands to reproduce it.

The coverage distribution is the detail I find most convincing, and the hardest to fake. It isn't uniform: 92.7 % on the domain package, 89.8 % on the worker pool, down to 68.3 % on the composition root, where the uncovered remainder is signal handling and error paths that are awkward to exercise in-process. That's the shape a careful engineer aims for — heaviest where a defect costs most — rather than a single number chased for its own sake.

| Package | What lives there | Coverage |
|---|---|---:|
| `internal/jobs` | domain model, state transitions, executors | 92.7 % |
| `internal/worker` | worker pool, cancellation coordination | 89.8 % |
| `internal/api` | HTTP handlers, validation, error envelope | 84.1 % |
| `internal/store` | SQLite persistence, atomic claims | 79.5 % |
| `.` | composition root, config, shutdown | 68.3 % |
| **Total** | | **83.0 %** |

Which brings me to what I take from this. Given a solid specification, deterministic verification wired into the point of completion, and bounded escalation with a human at the end of it, the output was not merely plausible-looking code. It was code I would put through review with a straight face — and, importantly, the places where it fell short were places the *specification* fell short, not places the agent wandered off.

## Conclusion

This started as a thought exercise about how a language and its toolchain could favor building code with AI through deterministic verification. Building the machinery to support that premise turned out to be most of the work on my end, and it sharpened the question along the way into something simpler and more demanding: can I get something genuinely solid built this way?

The answer, within limits worth stating, is yes. It's one run, of one specification, in one language, with one model provider — not a benchmark. Within those limits: given a solid specification, verification wired into the moment of completion rather than requested politely, and bounded escalation with a human at the end of the ladder, an agent produced a 7,300-line concurrent service in about two and a half hours of active time, with one human decision point during the run. It stands up to tools that had no hand in writing it. Not proven correct — checked, independently, and the checks are reproducible.

The parts that fell short are the ones I find most useful. The one real defect static analysis found was a missing server timeout the specification never asked for. The closest call in the run was T6, where a green test suite would have concealed two entirely untested handlers, and what caught it was a policy the orchestrator followed against its own better judgment. Neither is the agent wandering off. Both are gaps in what I specified and how I bounded the work — a far more tractable problem than the one I expected to be writing about, and one that puts the burden back where engineers can actually do something about it.

That's what I'd carry into real work. The gate never certified anything; refusing was the only power it had. What made the output trustworthy was the combination — a specification precise enough to be checkable, a check that couldn't be skipped or talked around, and a hard stop with a human at the end of it. Take any one of those away and I'd be back to reading every line with the suspicion I started with.

Everything is in [the repository](https://github.com/soeirosantos/taskforge) — the code, the commit history, the plan, the execution notes, the escalation record, the statistics, and the quality analysis. I've tried to state throughout which numbers are measured and which are judgment.

## A note on the Rust execution

My initial goal of comparing Go and Rust never happened. The main reason is that the Go run alone was enough to answer what I was after — whether I could get something genuinely solid built this way. If you're curious how it would go with Rust, or any other language of your choice, you're invited to clone the repository and follow the instructions in [AGENT_CONTROL_README.md](https://github.com/soeirosantos/taskforge/blob/main/.claude/AGENT_CONTROL_README.md), though you may need some adjustments.

> **AI assistance acknowledgment**: This article and code supporting it was produced with AI assistance. Including the machinery research and development, experiment execution, etc. During writing multiple rounds of drafting and review involved AI. The experiment planning, core ideas, arguments, editorial decisions, and final responsibility are mine.
