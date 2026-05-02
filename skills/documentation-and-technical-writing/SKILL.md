---
name: documentation-and-technical-writing
description: "Write documentation, design docs, ADRs, READMEs, commit messages, and other engineering communication that holds up under reading. Use this skill whenever the task involves writing for engineers (or being read by engineers): READMEs, design docs, technical specs, RFCs, ADRs, runbooks, API documentation, commit messages, PR descriptions, postmortems, technical blog posts, onboarding docs, or asking \"how should I structure this writeup.\" Bad docs cost more than bad code; good docs scale knowledge across the team and across years. Built on the Diátaxis framework (Daniele Procida), Google's developer documentation style guide, the design-doc culture from Google and others, and Conventional Commits."

---

# Documentation and Technical Writing

Documentation is one of the highest-leverage activities in software engineering. A well-written design doc saves dozens of meetings; a clear README saves hundreds of newcomer hours; a sharp commit message lets a future engineer understand a decade-old change. Bad documentation is worse than no documentation: it lies, ages without telling you, and becomes evidence the system is well-understood when it isn't.

This skill is the discipline of writing for engineers. It covers the kinds of docs you actually write (READMEs, design docs, ADRs, commits, runbooks, postmortems), the framework for organizing larger documentation systems (Diátaxis), and the writing principles that hold up under skeptical reading.

This skill covers the *writing* discipline. For specific artifacts owned by other skills, defer to those: ADRs are detailed in `systems-architecture` (architecture decisions live with architecture); postmortems are detailed in `debugging-and-incident-response` (postmortems live with the incident process). This skill links to both with brief inline guidance below.

## The framework: Diátaxis (Daniele Procida)

Most documentation problems trace to mixing four different *kinds* of documents that have different audiences and serve different purposes. Diátaxis names the four:

1. **Tutorials** — *learning-oriented*. The reader is new and following along. Goal: produce a working result by the end. Tutorials handhold; assume nothing; accept some lack of generality. ("Build your first thing.")
2. **How-to guides** — *task-oriented*. The reader has a goal and needs the steps. Goal: solve a specific problem. How-tos assume baseline knowledge; aren't comprehensive; address one thing well. ("How to authenticate against the API.")
3. **Reference** — *information-oriented*. The reader is looking something up. Goal: provide accurate, complete facts. Reference is comprehensive, austere, no story. ("API: every endpoint, every parameter.")
4. **Explanation** — *understanding-oriented*. The reader wants to know *why*. Goal: build mental models. Explanation discusses, frames, considers alternatives. ("Why we chose CRDTs for the sync engine.")

The framework's central insight: **trying to do two of these in one document produces a worse version of both.** A "tutorial" that's also reference becomes a slog. A "how-to" that's also explanation distracts the reader who just wants the steps.

When organizing a docs site or planning a doc, ask: which of the four am I writing? Then commit to that mode and refer out to the others.

For most code review and design work in this skills suite, you're writing **explanation** (design docs, ADRs) or **reference** (API docs, runbook procedures). Tutorials and how-tos come up less, but the frame still applies.

## READMEs

The README is the front door. Most readers will not go further. Optimize for the reader's first 30 seconds.

A useful default structure:

```markdown
# [Project name]

[One sentence: what this is and why it exists. Concrete, not aspirational.]

[2-3 sentences: who this is for and what problem it solves. Skip if obvious from context.]

## Quick start

[The minimum commands to get a working result. ~5 lines. If this takes more, the
project has friction worth addressing — and a "Quick start" that's a 50-step procedure
isn't a quick start.]

```bash
git clone ...
cd ...
make install
make run
# You should see X.
```

## What's here

[Brief tour of the directory structure or key entry points. ~100-200 words.]

## Documentation

- Tutorial: building your first foo — `docs/tutorial.md`
- How-to: deploying to production — `docs/deploy.md`
- API reference — `docs/api/`
- Architecture — `docs/architecture.md`

## Development

[How to set up for development; how to run tests; how to contribute. Or link to CONTRIBUTING.md.]

## License

[License name. Link to LICENSE file.]
```

Common README failures:

- **Too long.** The 5,000-word README has buried everything important. Move depth into other files.
- **Aspiration-as-description.** "An elegant, blazing-fast solution to your problems" tells me nothing. What does it *do*?
- **No quick-start.** Reader has to dig to discover whether this thing is even runnable.
- **Stale install instructions.** Run them. Do they still work?
- **Missing the obvious questions.** What language? What runtime? What dependencies? License?
- **Linking to non-existent docs.** Every link in the README must work.

## Design docs

A design doc is the artifact of designing something non-trivial. It captures: what problem, what's the proposed solution, what alternatives were considered, what's the impact.

Done well, design docs **scale knowledge of senior engineers into the organization**, **form organizational memory**, and **prevent re-litigating the same decisions** when the same questions come up later.

### A working template

```markdown
# Design doc: [What this is about]

**Author(s):** [Names]
**Status:** Draft / In review / Approved / Implemented / Superseded
**Created:** [Date]
**Last updated:** [Date]
**Reviewers:** [Names of expected reviewers]

## Summary

[One paragraph: what we're proposing, the problem it solves, the recommended approach.
A reader should be able to read just this and know whether to keep going.]

## Background

[The problem we're solving. Why it matters. The current state. Constraints (technical,
organizational, regulatory). Anything the reader needs to evaluate the design.]

[Cite earlier docs, RFCs, ADRs, related projects. Don't re-litigate settled decisions
beyond a brief reference.]

## Goals

[Concrete, verifiable. "Reduce p99 latency for endpoint X to 200ms" not "make it faster."]

## Non-goals

[Things we are deliberately not solving. Pre-empts "but what about X?" objections by
saying X is out of scope. Often more important than the goals section.]

## Proposed solution

[The design itself. Architecture, components, data model, control flow, key algorithms.
Diagrams where they help. Specific enough that a reader could build it; abstract enough
that implementation details aren't pinned down prematurely.]

[Sub-sections for each major aspect. For example:
- Data model
- API design
- Storage and persistence
- Authentication and authorization
- Failure modes
- Migration / rollout plan
- Observability]

## Alternatives considered

[Each significant alternative considered, with reasons for rejection. This section is
non-negotiable; a design doc with no alternatives wasn't designed, it was advocated.]

### Alternative A: [Name]
[What it was. Why we considered it. Why we rejected it. Conditions under which we'd reconsider.]

### Alternative B: [Name]
...

## Risks and open questions

[What we're uncertain about. What could go wrong. What will need follow-up.]

## Cross-functional concerns

[Security review needed? Privacy review? Compliance? Operational readiness? Capacity?
Cost? Customer comms? Infrastructure dependencies?]

## Rollout plan

[How we'll ship this. Stages. Feature flags. Rollback. What "done" looks like.]

## Success metrics

[How we'll know it worked. What we'll measure. What targets.]

## Appendix

[Detailed designs, data tables, mocks, anything that supports the body but doesn't
fit in the flow.]
```

### What makes a design doc good

- **It has a problem statement, not just a solution.** Most bad design docs jump straight to a solution. Without a sharp problem, you can't evaluate the solution.
- **It considers alternatives.** "We chose X" is meaningless without "instead of Y, Z, W." If only one option was considered, the doc is advocacy.
- **It's specific.** Vague language ("we'll handle that with appropriate caching") is where bad designs hide. "We'll use a write-through cache with a 5-minute TTL on the user-profile table" is reviewable.
- **It's appropriately scoped.** A 50-page design doc gets read by no one. A 2-page design doc on a major architectural decision is too thin. Match scope to stakes.
- **It captures the "why."** A reader two years later should be able to understand not just what we did but why we did it that way. The why prevents re-litigation.
- **It has explicit non-goals.** Pre-empts scope creep in review.
- **It's revisable.** Design docs should be living documents during the design phase. They become artifacts after.

### What makes a design doc fail

- **Pure narrative with no decision.** Reads as a tour of considerations; doesn't actually commit to anything.
- **Implementation manual, not design doc.** Goes deep on code structure and shallow on the design choice. Design happens at the level of architecture, not the level of class hierarchies.
- **Foregone conclusion.** The "alternatives" exist only to be obviously dismissed. No real consideration.
- **Anchored on solution.** Starts from "we'll use Kafka" and works backward. The problem section is contorted to fit.
- **Too late.** Written after the work is done as a formality. The point is to drive design *during* the design phase.
- **Approval theater.** Heavy review process; never updated; never read after approval.

## ADRs (Architecture Decision Records)

ADRs are short documents — usually 1-2 pages — capturing one significant decision: what we chose, why, what we considered, what we accept as consequences.

ADRs and design docs complement each other. A design doc may produce several ADRs as it converges. ADRs outlast design docs, which often go stale; they are the durable record of "why is it this way?"

The minimum viable ADR (Michael Nygard's structure):

```markdown
# ADR NNNN: [Decision title]

**Status:** Proposed | Accepted | Superseded by ADR-NNNN
**Date:** YYYY-MM-DD
**Author(s):** [Names]

## Context
[Situation, constraints, forces.]

## Decision
[What we decided. Stated unambiguously.]

## Consequences
[Easier; harder; trade-offs accepted; follow-up triggered.]

## Alternatives considered
[For each: what it was, why we didn't pick it.]
```

ADRs are append-only. When a decision changes, write a new ADR that supersedes the old; mark the old one's status `Superseded by ADR-XXXX`. Don't edit history; the prior reasoning is the input to the new decision.

Store in `docs/adr/` numbered sequentially. For the full template, worked examples, and operational guidance, see the `systems-architecture` skill (decisions live with the architecture they decide).

## Commit messages

A commit is a story. Good commit messages help future engineers (including you) understand a change years later. The cost of writing them well is small; the value is enormous.

### The "Tim Pope" / "Beams" structure

```
Short subject line (50 chars or less, imperative mood)

Longer body if needed. Wrap at 72 characters. Explain *why* this change
was made, not just what. Reference issues, link to design docs, mention
anyone who deserves credit.

If the change is non-obvious, walk through the reasoning. If alternatives
were considered, mention them.

- Bullet points are fine
- For longer changes

Refs: PROJ-123
Co-authored-by: Alice <alice@example.com>
```

Specifically:

- **Subject line: imperative mood.** "Add user validation" not "Added user validation" or "Adds user validation." (Convention from Linus / git itself: "if applied, this commit will [subject line].")
- **Subject line: ≤50 chars.** Most tools display a truncated subject; over 50 gets cut off.
- **Subject line: capitalize, no period.** "Fix race in retry handler" not "fix race in retry handler." or "Fix the race in the retry handler."
- **Blank line between subject and body.** Required for many tools to render correctly.
- **Body wrapped at ~72 chars.** Reads cleanly in `git log` and most diff tools.
- **Body explains *why*.** The diff shows what changed; the message tells you why.

### Conventional Commits (when the team uses them)

Conventional Commits adds a structured prefix that makes commits machine-parseable for automated changelogs and version bumping:

```
type(scope): subject
```

Examples:

```
feat(auth): add OAuth login flow
fix(api): correct off-by-one in pagination
docs(readme): update install instructions
refactor(db): extract connection pool config
test(orders): add integration test for cancel flow
chore(deps): bump axios to 1.6.0
```

Common types: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `chore`, `ci`, `build`. The two that affect semver are `feat` (minor bump) and `fix` (patch bump). `feat!` or `BREAKING CHANGE:` in the body indicates a major bump.

When the team uses Conventional Commits, follow the convention. When they don't, the Tim Pope structure above is universally fine.

### What to flag in commit messages

- **Subject line is "fix bug" or "wip" or "asdf".** Useless to anyone who isn't you, today.
- **No body for a non-trivial change.** Reader can't understand the why.
- **References by URL only with no context.** "See ticket: https://..." doesn't help readers without access to the ticket system.
- **Multiple unrelated changes in one commit.** The commit can't be understood as a unit; bisection is poisoned.
- **Subject in past tense or describing.** "Added user validation"; convention is "Add user validation."
- **Body that just restates the diff.** Pointless.

## PR descriptions

A PR description is the cover letter for your commit (or commits). It serves the reviewer:

```markdown
## What

[1-2 sentences. What does this change do? Match the title.]

## Why

[Why is this needed? Link to issue / design doc / RFC. The motivation.]

## How

[Brief description of the approach if not obvious from the diff. Mention alternatives
considered if the choice was non-obvious. Call out any subtle bits.]

## Testing

[What was tested. Specifically: what tests added, what manual verification, what
integration tested in which environment.]

## Risk and rollout

[What can break? What's the blast radius? Is there a feature flag? Rollback plan?]

## Out of scope

[What deliberately wasn't done. Follow-ups. Known limitations.]

## Screenshots / output

[If relevant.]
```

For tiny PRs (one-line fix, typo), this is overkill — title + brief description is fine. For substantive PRs, the structure pays off in faster, better reviews.

## API documentation (reference docs)

API documentation is reference, not tutorial. It's looked up, not read.

For each public function / endpoint / type:

- **Name and signature.** Exactly.
- **What it does.** One sentence preferred.
- **Parameters.** Name, type, meaning, constraints, default if any.
- **Return value.** Type, meaning, possible values.
- **Errors / exceptions.** Each one possible, with the condition that triggers it.
- **Side effects.** What this changes besides the return value.
- **Concurrency / threading notes.** Thread-safe? Reentrant?
- **Examples.** A canonical usage; an edge case if helpful.
- **Since-version.** When this was added (for libraries with semver).
- **Deprecation notes.** If applicable, the replacement and the timeline.

Common API doc failures:

- **Restating the signature in prose.** `def add(a, b)` documented as "Adds a to b and returns the result." Adds nothing the signature didn't say. Document what the *behavior* is — edge cases, side effects, errors.
- **Missing edge cases.** What happens on null? Empty? Negative? Maximum?
- **No examples.** Reader has to guess how to actually use the thing.
- **Stale.** Docs say one thing; code does another. The code wins; the docs are now actively misleading.
- **Generated and ignored.** Auto-generated API docs that nobody reads are not useful. They satisfy a "we have docs" check; they don't help anyone.

## Runbooks

A runbook is a how-to guide for a specific operational task. The reader is on-call at 3am; optimize for that.

For runbook structure and quality, see `skills/debugging-and-incident-response/references/incident-roles-and-runbooks.md`.

The writing-specific notes:

- **Specific commands.** "Restart the service" → `kubectl rollout restart deployment/order-service -n production`.
- **Expected output.** What "success" looks like — so the reader can tell if it worked.
- **Time estimates.** "This takes about 2 minutes" — so the reader knows when something's wrong.
- **Branch on observed conditions.** "If you see X, do Y; if you see Z, do W."
- **Escalation paths.** When to wake someone up; who.

Runbooks rot fast. Date them. Review after every incident.

## Onboarding docs

For a new engineer joining the team, the docs they need:

- **Setup**: how to get a working development environment, with verified install steps.
- **Architecture overview**: the system's main components, how they fit together. Link to deeper docs.
- **Codebase tour**: where things are, naming conventions, important entry points.
- **Workflow**: how PRs are reviewed, how releases happen, how on-call works.
- **People and ownership**: who owns what, who to ask about what.
- **Reading list**: the design docs, ADRs, and postmortems they should read.
- **First task**: a small, well-scoped task with a clear definition of done. Sets up early success.

The test of onboarding docs: a new engineer can productively contribute within their first week without burning a senior engineer's time on basic questions.

## Writing style for engineers

### Be concrete

Replace abstractions with specifics. "We had performance issues" → "p99 latency for `/orders` went from 200ms to 1500ms." "It scales well" → "Throughput is linear up to 10,000 RPS."

### Be honest

If the design has trade-offs, say so. If you don't know something, say so. If the work isn't done, say so. Pretending otherwise burns trust when the truth surfaces.

### Lead with the answer

Engineers read with intent. Put the bottom line at the top. The pattern: "We propose X. The reasons are Y, Z. Background and details follow." Not "We've been investigating X. Here's a long history. After much deliberation, we propose…"

### Cut

Most technical writing is too long. Cut filler ("very", "actually", "quite"). Cut sentences that don't move the document forward. Cut sections that are bookkeeping rather than content. Strunk and White's "Omit needless words" applies.

### Use the right tense and voice

- Decisions and decisions made: past tense, active voice. "We chose X." Not "X was chosen."
- Procedures and how-tos: imperative. "Run X. Verify Y." Not "You should run X."
- Reference docs and behavior: present tense. "Returns the user's id."
- Avoid passive voice except where the actor is genuinely irrelevant.

### Don't editorialize in technical docs

"Unfortunately we had to use X." "The amazing new feature." Save it for blog posts. Technical docs read better when they're matter-of-fact.

### Define your audience

A doc for senior engineers familiar with the codebase reads differently from a doc for new engineers, which reads differently from a doc for non-engineering stakeholders. Know who you're writing for; don't try to serve all three at once.

### Diagrams

Use a diagram when it shows something prose can't. Architecture diagrams for component relationships. Sequence diagrams for protocols. State diagrams for state machines. Don't use a diagram as decoration; an unhelpful diagram costs more attention than it gives.

For text-renderable diagrams (works in version control, easy to update): Mermaid, PlantUML, ASCII art. For polish: Excalidraw, draw.io, etc., committed alongside.

### Specific words to use carefully

- **"Should" vs "must" vs "may".** Per RFC 2119: must = required, should = recommended (with reason for deviation), may = optional. Engineers reading specs expect this.
- **"Will" vs "would".** Will = factual claim about future. Would = conditional or hypothetical.
- **"We" vs passive.** "We chose X" beats "X was chosen" — more direct, more honest.
- **"Probably" / "I think".** Use sparingly in formal docs; engineers read these as hedging when the writer isn't confident. Either be confident or explicit about uncertainty.

## What to flag in review

For documentation in PRs:

- A new public function / endpoint / class with no documentation.
- Documentation that contradicts the code (the code wins; the docs are misleading).
- A README that hasn't been updated to reflect significant changes.
- A design doc with no alternatives section (or with alternatives that are clearly straw men).
- An ADR that doesn't capture the *why* — only the *what*.
- A commit message that's "fix bug" / "wip" / "..."
- Stale TODO comments with no owner / date / trigger.
- A runbook that hasn't been updated since a relevant incident.
- A "how-to" that's actually three documents in a trench coat (intermixing tutorial / how-to / reference / explanation).
- API docs that just restate the signature in prose.

For project-level documentation:

- No CONTRIBUTING.md when the project accepts external contributions.
- No way to find the answer to "what is this and why does it exist."
- No clear mapping from "I have problem X" to "read doc Y."
- Documentation that depends on tribal knowledge.

## Reference library

- `references/diataxis-applied.md` — applying Diátaxis to a specific docs system, with worked examples.
