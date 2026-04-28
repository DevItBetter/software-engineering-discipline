# Diff vs Module vs System: Adjusting the Lens

Reviewing a 30-line bug fix and reviewing a proposed system design require different attention. Misapplying the lens is a common review failure: nitpicking a system design for variable names, or hand-waving a bug fix because "the architecture looks fine."

## Three review units

### Unit 1: The Diff (≤ ~200 lines, single concern)

This is the most common review. The change is local. The author has a specific intent. Risk is high *per unit of change* because diffs accumulate fast.

**Primary lens:** correctness, testing, contract preservation, security, local complexity.

**What you should always do:**
- Read every line of the diff. At this size you have no excuse to skim.
- Read at least one caller of every changed public symbol.
- Verify that the test exercises the change (mentally remove the change; does the test fail?).
- Run one adversarial input mentally.

**What you should not do:**
- Litigate the surrounding architecture. The author cannot fix it in this PR. Tag it as out-of-scope and file it.
- Demand a refactor of the existing code "while you're here." Scope creep harms reviewability and bisectability.
- Demand premature abstractions for hypothetical future variations.

### Unit 2: The Module / Substantive Change (~200–1500 lines, new component or significant refactor)

A new file/module/package, a non-trivial refactor, a feature that touches multiple files cohesively. Risk shifts toward **design** and **fit with the existing system**.

**Primary lens:** module boundaries, cohesion, coupling to existing code, public API shape, test design, naming of new domain concepts.

**What you should always do:**
- Read the description and the design context (linked doc, issue, RFC). If there isn't one for a change this size, that's itself a finding.
- Build a mental model of the module's responsibility *before* reading the code. If the responsibility is hard to state in one sentence, the module is incoherent.
- Ask: does this fit the codebase, or fight it? New patterns introduced when an existing pattern would have worked are usually a regression.
- Apply Ousterhout's "deep modules" check: is the public surface small relative to the work the module does? Shallow modules (lots of public methods, little hidden complexity) are usually a bug.
- Verify the dependency direction. Does the new module depend on stable abstractions, or on volatile details?
- For each public function/method/endpoint, look at the test for it.

**What you should not do:**
- Demand that the author refactor unrelated existing code into the new pattern. That's a follow-up.
- Approve based on "the diff is clean" — you have to evaluate the *thing*, not the typography.

### Unit 3: System / Architecture Review (RFC, design doc, or large structural PR)

Often the unit is a *document* with little or no code. The decision being made is binding for years. The cost of getting it wrong is high; the cost of catching it now is the cheapest it will ever be.

**Primary lens:** problem framing, alternatives considered, fit with existing systems, distributed-systems hazards, data ownership, evolution and versioning, operability.

**What you should always do:**
- Confirm the **problem statement** is sharp. A vague problem produces a vague design that will be re-litigated forever.
- Look for **alternatives considered and rejected, with reasons**. A design with no alternatives section was not designed; it was advocated. (See `systems-architecture` skill.)
- Identify the **data model and its ownership**: who is the system of record for each piece of state? What are the consistency requirements? Where are the trust boundaries?
- Identify the **failure modes**: what happens when each dependency is slow / unavailable / returns wrong data? What does recovery look like? Where is the on-call playbook?
- Identify the **evolution story**: how do you version this? How do you migrate off this if it's wrong? What's the rollback?
- Identify the **observability story**: what SLOs apply? What signals will tell you it's working?
- Compare against the **fallacies of distributed computing** if anything crosses a network. Latency, reliability, ordering, partitioning, security, topology — assume all hostile.
- Apply **Conway's Law**: does the architecture match the team boundaries? Mismatches manifest as constant cross-team friction.

**What you should not do:**
- Get hung up on naming or syntax in design docs. The point is the *decision*.
- Approve a design without an explicit operability section.

## How to switch lenses mid-review

Sometimes a "small diff" is actually an architectural change in disguise — a new persisted field, a new public endpoint, a new dependency, a new failure mode. When you notice this, switch to the larger lens and say so:

> This is a 30-line diff but it adds a new persisted field that becomes part of the public API contract. Reviewing under the architecture lens because the design decision is locked in by this change.

Conversely, sometimes a "big PR" is mostly mechanical movement (rename, file split, codemod) — review with the smaller lens and verify the mechanics, not the design.

## When a PR is too big to review well

If you cannot hold the change in your head, the change is too big. Common signs:

- More than ~10 files changed and they are not all the same kind of change (e.g., not a rename).
- More than ~400 lines of substantive (non-test, non-generated) code.
- Multiple concerns mixed: feature + refactor + dependency bump + formatting.
- The description has multiple "and also" clauses.

Refuse the review and ask the author to split it. Quote Google's guidance: 100 lines reasonable, 1000 lines too large. The goal is reviewable units, not heroic reviewers.

The exception: a coordinated change across many files that cannot be split (a rename, a codemod, a framework upgrade). For these, verify the mechanics (the rename is consistent, the codemod was applied correctly, no semantic changes slipped in) rather than reviewing each file.
