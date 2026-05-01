# CL / PR Discipline

A review is only as good as the change being reviewed. A 2000-line PR with three concerns mixed in cannot be reviewed well no matter how skilled the reviewer is. The single most effective intervention for code quality is keeping changes small and focused.

This is borrowed and adapted from Google's engineering practices, which have stood up over many years and at large scale.

## What "small" means

A small change does **one thing**. The "one thing" can be small or moderately large in lines, but it is one coherent thing.

Rough sizes:

- **~100 lines** is a comfortable size to review carefully.
- **~400 lines** is the upper bound of what most reviewers can read in one sitting without their attention degrading.
- **~1000 lines** is too large unless the change is mechanical (rename, codemod, generated file).

These are guidelines, not rules. A 600-line PR that is one focused refactor of one module is fine. A 60-line PR that touches eight unrelated files probably is not.

## The "one thing" rule

A PR should be one of:

- One feature (or one slice of a feature behind a flag)
- One bug fix
- One refactor
- One dependency bump
- One mechanical movement (rename, file split, codemod)

If you find yourself writing "and also" in the description, you have two PRs.

Common "and also" anti-patterns to refuse:

- "Refactor this module **and also** fix the bug while I'm here." — Two PRs. Refactor first (no behavior change), then fix.
- "Add the new endpoint **and also** clean up the old one." — Two PRs. The cleanup can be a follow-up.
- "Bump the dependency **and also** use the new feature." — Two PRs. Bump and verify nothing broke; then a separate PR uses the new feature.
- "Format the file **and also** add a function." — Two PRs. The formatting change creates noise that hides the real diff.

The exception: when two changes truly cannot be split because one depends on the other and neither makes sense alone (e.g., a public-API rename plus all callers, in a monorepo). In these cases, structure the PR so the mechanical part and the judgment part are visually separable, and call out the boundary in the description.

## PR descriptions

A good PR description answers, in this order:

1. **What does this change do?** One sentence. If you cannot summarize in one sentence, you have multiple changes.
2. **Why?** What problem does it solve? What is the user impact? Link to the issue or design doc.
3. **How?** Brief description of the approach if it's not obvious from the diff. Mention alternatives considered if the choice was non-obvious.
4. **What was tested?** Specific tests added, manual verification steps if any, integration tested in which environment.
5. **What is the risk / blast radius?** What can break? What is the rollback plan? Is there a feature flag?
6. **What was deliberately not done?** Out-of-scope items, follow-ups planned, known limitations.

Bad PR descriptions:

- Empty or "fixes the bug"
- Repeats what the diff shows ("Adds a function called X that does Y")
- Buries the user impact under implementation detail
- Hides the real risk

## Splitting a big PR

When a PR is too big, the right move is to split it. Common splitting axes:

- **By layer**: data model → repository → service → handler → UI. Each layer in a separate PR (when the layers can be merged independently with feature flags or sentinel values).
- **By feature slice**: a thin end-to-end slice that works, then iterate.
- **By preparation vs feature**: refactor to make the change easy, then make the easy change. This is Kent Beck's "make the change easy, then make the easy change."
- **By risk**: high-risk piece first behind a flag (or in a separate PR with extra review), then low-risk pieces.
- **By concern**: separate the bug fix from the refactor from the cleanup.

For monorepo coordinated changes, use a stacked-PR or "soft launch" pattern: prep changes go in disabled, then the small atomic PR flips them on.

## Author responsibilities that make review possible

These are not the reviewer's job to enforce — but a review can call them out when missing:

- **Self-review the diff before requesting review.** Most embarrassing review feedback can be caught by reading your own diff with fresh eyes.
- **Run the full test suite locally** (or at minimum confirm CI green) before requesting review.
- **Respond to comments before pushing more changes** when possible — let the reviewer see what you addressed.
- **Don't squash-rebase mid-review** unless asked — it makes "what changed since last review" impossible to track.
- **Mark resolved comments resolved**, leave open ones open until they're agreed.
- **For substantive disagreements, move to a synchronous discussion.** Long comment threads with three rounds of pushback are a sign the conversation should leave the PR.

## Reviewer responsibilities that make review fast

- **Respond within one business day**, even if just to acknowledge and give an ETA. Stalled reviews are demoralizing and stale code rebases poorly.
- **Do the full review in one pass when possible**, not in trickle. Three rounds of one-comment-at-a-time is exhausting.
- **Distinguish your blocker from your taste.** Authors learn quickly which reviewers are credible and which are noise.
- **Approve when ready, comment when not.** Don't approve "with concerns" if the concerns are blocking.
- **For small uncontroversial PRs, a fast LGTM is better than a thorough review next week.** The cost of context switching for the author often exceeds the marginal value of nitpicking.

## When to escalate the review

Some PRs warrant a second reviewer or a senior reviewer:

- New external API or contract.
- New persisted schema.
- New cross-team dependency.
- Security-sensitive change (auth, crypto, secrets, user data).
- Performance-critical change with a benchmark to validate.
- A change that the original reviewer is uncertain about.
- A change that the author is uncertain about and is signaling it.

Don't be a hero. The cost of a second pair of eyes is small; the cost of a missed issue in any of the above is large.
