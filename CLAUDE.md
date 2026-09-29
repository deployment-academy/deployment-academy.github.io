# Writing style guide

This is the style guide for the deployment.properties Hugo blog. It sets the voice, structure, and formatting rules for drafting and editing posts. For anything it does not cover (grammar, numbers, abbreviations, tables, images, and so on), follow the [Google developer documentation style guide](https://developers.google.com/style).

## Site and front matter

- Hugo static site. Posts live in `content/posts/<category>/<slug>.md`; categories are directories: `eng`, `networking`, `devsecops`, `k8s-ops`, `hugo`. `content/about/_index.md` is the About page.
- Start each post from `archetypes/default.md` (`hugo new`). It sets this front matter:
  ```yaml
  ---
  title: "Title Case Title"
  description: "changeme"
  date: 2026-09-27
  lastmod: 2026-09-27
  draft: true
  sidebar: "right"
  widgets:
    - "ddg-search"
    - "recent"
    - "social"
  tags:
  ---
  ```
  - Replace `description` with 1 to 3 plain-prose sentences summarizing the post's arc.
  - Fill `tags` with lowercase, specific technology or topic names (`"dns"`, `"lima"`, `"tcpdump"`), not vague themes.
  - Add a `categories` field only for the "Key Concepts" series or another named series; a category is a taxonomy distinct from `tags`, rendered next to the date. Do not invent categories casually, and omit the field entirely otherwise.
  - Do not change the front matter of existing posts unless asked.
- `<!--more-->` marks the teaser cutoff. Place it after the opening paragraph or paragraphs, before the first `##` section.
- Set `toc: true` for long or reference-heavy posts. It is optional, not default.
- Publish or unpublish a post by toggling `draft: true`/`false` and bumping `lastmod` to the current date. Do not restructure content as part of a publish toggle.
- Commit message convention: an initial add is `"add <topic> post (#N)"`; a later `"publish (#N)"` or `"unpublish <topic> posts (#N)"` commit only flips the `draft` flag.

## Voice and person

- Tutorials address the reader directly as "you" and use the imperative mood for instructions ("Start from the plainest possible fact...", not "We will start from..."). Reserve "we" for cases that unambiguously mean the author as an organization, and keep those rare.
- In essays, use "I" to give your own view or experience ("I decided...", "I do not know exactly where this settles"). Use "you" whenever the text addresses the reader. Don't carry "I" or "we" into tutorials.

## Tone

- Write in a formal-but-conversational engineering register: approachable, like a knowledgeable colleague, not stiff or academic.
- No slang and no exclamation points. No emoji or emoticons.
- State opinions plainly and own them, without hedging into mush: "That judgment was wrong and the policy was right." When genuinely uncertain, say so directly rather than softening with qualifiers: "I do not know exactly where this settles, and I do not think anyone does."
- Dry, understated humor is fine in small doses, delivered as honest asides rather than jokes: "which sounds like an interesting topic for another blog post." Never slapstick, never exclamation-driven, and never at the expense of clarity.
- Do not describe a step as "simply," "just," "easy," or "quick," and do not use "please" in instructions.

## Timeless writing

- Avoid words that anchor a post to a moving present: "currently," "recently," "at present," "as of this writing," "soon," and "new" or "newer" describing a tool's state. Describe what a feature does instead of flagging its age.
- An essay that is deliberately dated may anchor itself to a point in time, since its premise is a snapshot of a moment. When a date matters, state it explicitly ("the September 2026 release adds...") rather than leaning on a relative-time word.

## Structure

- Open a post by naming and linking what it continues from, then stating what it will do, in one or two sentences, followed by `<!--more-->`. Do not open a post in isolation: if it continues a series or a prior post's argument, say so explicitly and link it in the first sentence. Example pattern: "Continuing the series of [Linux Networking](https://deployment.properties/tags/networking/), this post refreshes DNS."
- Open an essay with a concrete anecdote, data point, or personal trigger for the post, not a throat-clearing "In this post I will discuss...".
- Use `##` for major sections and `###` for subsections, in sentence case ("The port 53 conflict," not "The Port 53 Conflict"). Name sections concretely and specifically ("Recursion and the hierarchy"), not generically ("Overview," "Introduction"). Keep heading punctuation minimal, and do not bury a link or code element in a heading.
- End a tutorial with a "## Wrapping up" (or "Wrap up") section: a bulleted recap of the concepts covered, each bullet leading with a bolded key term, followed by a note on what does not persist or how it maps to production, then a teardown code block.
- End an essay with a "## Conclusion" section that restates the thesis without repeating it verbatim. Long-form essays add a "## References" section after it: `- Author, [*Title*](url). Publisher, date.`
- When AI assisted in drafting a post, end it with this acknowledgment blockquote:
  > **AI assistance acknowledgment**: This article was produced with AI assistance. [what AI did]. The thesis, argument, editorial decisions, and final responsibility are mine.
- Let length match the material: tutorials run long (150 to 950 lines) and thorough, not padded, with each section earning its length by doing something (a command, an observation, a failure reproduced on purpose). Essays run 100 to 130 lines.

## Sentences and paragraphs

- Favor active voice: make the actor the subject ("The resolver queries the root server," not "The root server is queried"). Use passive voice only when the actor is unknown or irrelevant.
- Write medium-to-long sentences, with an aside set off by an em dash or a semicolon joining two closely related clauses. Break a long thought into two sentences rather than nesting further.
- Keep paragraphs short: 2 to 5 sentences, one idea per paragraph, with the most important point first. A paragraph running past 5 or 6 sentences is usually carrying two ideas and should split.
- Introduce every list with a complete sentence, not a fragment finished by the first bullet. Use bullets for enumerable parallel facts or options, and numbered lists only for sequential steps or genuinely ordered reasons. Keep list items parallel in grammatical form. Bold a lead-in term inside a bullet when the bullet functions as a recap or reference entry.
- Use plain, functional transitions ("So," "Notice," "Before you test,") rather than literary connectives ("Furthermore," "Moreover"). Let a "## Conclusion" heading do the job of "In conclusion."

## Punctuation

- Use an em dash (`—`) for an aside or break in sentence flow, with no space on either side: `topology in prose—no diagrams.`
- Do not use en dashes, and do not use a spaced hyphen (` - `) as a dash.
- For a range, use "to" in prose ("2 to 5 sentences") or a hyphen in compact contexts such as tables and front matter (`2020-2022`).
- When separating a short label from its description, such as in a recap bullet, use a colon rather than a dash after the bold label: `**Network namespace**: an independent copy of the network stack.`

## Code and commands

- Fence code blocks with a language hint (```bash, ```shell), even for a one-liner. Show raw command output in an unlabeled ``` block immediately after.
- Precede every non-obvious command block with a sentence saying what it does, and follow it with a sentence explaining what the output means or what to look for in it.
- Use inline code (backticks) for every command, filename, flag, config key, environment variable, and command-line utility name. Never describe a flag in prose without also giving its literal form.
- Mark context switches inside shell blocks with a comment naming the machine: `# on client`, `# on dnsserver`, `# ssh into node1`. State which machine a command runs on whenever more than one is involved.
- Narrate topology in prose and tables rather than diagrams.
- Use GFM pipe tables for structured comparative data (task breakdowns, stats, coverage by package).
- Use the `{{< notice type="note|tip|warning" id="..." title="..." >}}...{{< /notice >}}` shortcode sparingly, for an aside that would otherwise interrupt flow: a caveat, an optional deep dive, a design note.
- Do not embed tweets.

## Links

- Link a tool, spec, or RFC on first mention, pointing to its primary docs (official docs, RFCs, GitHub) over a secondhand explainer.
- Write link text that is descriptive on its own, the tool or spec name or a description of the destination. Never use "click here" or a bare URL as visible text.
- Use the live site URL form for cross-post links, `https://deployment.properties/posts/<category>/<slug>/`, not a relative path.

## Jargon and inclusive language

- Define jargon in plain language on first use, in parentheses or a short clause, or link a trusted definition, rather than assuming the reader already knows the term.
- Avoid ableist idioms and figurative language even when common in casual engineering speech: no "crazy" or "insane" as an intensifier.
- Use "allowlist"/"denylist," not "whitelist"/"blacklist." Use "primary"/"replica," not "master"/"slave," when describing database or node roles.
- Prefer precise technical wording over metaphor or idiom generally.

## Word choice

Replace the word or phrase on the left with the one on the right.

| Instead of | Write |
|---|---|
| miserable | impractical |
| classic (misconfiguration) | common misconfiguration |
| famously, genuinely (as filler) | (omit) |
| the single most common | a common |
| handy | useful |
| dangerous by accident | can cause unexpected behavior |
| whitelist / blacklist | allowlist / denylist |
| sanity check | final check |
| dummy value | placeholder value |

Cut these outright rather than trimming them:

- Phrases that tell the reader how to feel about a fact instead of stating it, such as "worth sitting with" or "internalize this now."
- Throat-clearing lead-ins to a fact, and hedges once the point is made plainly.
- Sentences that restate something already said.
- Tangential asides that don't serve the current step.
- Meta-commentary about the post's own teaching approach.

## Examples

**Series opening** (names and links the prior post, addresses the reader as "you"):
> Continuing the series of [Linux Networking](https://deployment.properties/tags/networking/), this post refreshes DNS. Start from the plainest possible fact—a machine is reachable by its address—and work up through `/etc/hosts`, `nsswitch.conf`, `/etc/resolv.conf`, a real DNS server you run yourself, the recursive hierarchy that answers queries for the rest of the internet, the record types you actually meet in the wild, and the tools you use when any of it goes wrong.

**Tutorial explanation** (bolded key term, em dash aside, active voice):
> A **network namespace** is an independent copy of the network stack—its own interfaces, routing table, ARP table, iptables rules, and port space. A fresh one contains only a down `lo` and can't even ping itself. Isolation is the default; connectivity is what you build.

**Essay first-person judgment** (owned opinion, no hedging):
> That judgment was wrong and the policy was right. `go vet` reports only the first failure, so the compile error was *masking* everything behind it.

**Essay first-person uncertainty** (stated directly, not softened):
> I do not know exactly where this settles, and I do not think anyone does. Based on what we can observe today, this is the direction I would prepare for.
