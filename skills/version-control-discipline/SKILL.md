---
name: version-control-discipline
description: "Use version control as a craft — atomic commits, buildable history, useful PRs, bisect-friendly main, recoverable mistakes. Use this skill whenever the task involves writing commits or PRs, choosing a branching model, deciding rebase vs. merge, recovering from a force-push or accidentally-committed secret, debugging a regression with `git bisect`, structuring a long change as a series of small reviewable steps, or judging whether a repo's history is readable. Use it especially when reviewing commit messages, PR descriptions, branching strategies, or merge policies. Anchored in Tim Pope and Chris Beams on commit messages, Paul Hammant on trunk-based development, Vincent Driessen on GitFlow (and his 2020 note retiring it for SaaS), Linus Torvalds on never rebasing public commits, and the Google Engineering Practices CL guide."

---

# Version Control Discipline

Version control is the documentation of how the code came to be. The asymmetry: code is read more often than written, and commits and PRs are read by people who are not the author — reviewers today, debuggers tomorrow, archaeologists six months from now. Sloppy history is a tax that compounds.

The discipline has two halves: **what lands on the main branch** (atomic, buildable, well-described, bisectable) and **how you get it there** (small PRs, sensible branching, clean rebases on your own branches, never on shared ones).

## What main looks like, when discipline holds

- Every commit on main builds.
- Every commit on main passes the test suite (or at minimum the regression test you'd use as a `git bisect` oracle).
- Each commit is one logical change. Refactor and behavior change land in separate commits — not separate paragraphs of one commit.
- Each commit message tells you *why*, not just what. The diff is the what.
- `git log --oneline` on a path reads like a reasoned changelog. `git log -p` answers "why does this line exist?"

The killer use case for this discipline is `git bisect`. Binary-searching across a clean history pinpoints a regression in *log₂(n)* commits. Across a history of `wip / fix / address PR comments` commits that don't individually build, bisect produces noise. The Linux kernel uses bisect on a multi-million-line codebase routinely — the practice scales as far as the discipline holds.

## Commit messages

Two canonical sources:

- **Tim Pope, *A Note About Git Commit Messages* (2008).** The 50 / 72 rule: subject ≤ 50 characters; body wrapped at 72. Imperative subject ("Fix bug," not "Fixed bug" or "Fixes bug") because that matches the language of `git revert`, `git merge`, and the rest of the tool surface. The blank line between subject and body is structural — `git rebase` and other tools rely on it.
- **Chris Beams, *How to Write a Git Commit Message* (2014).** Seven rules, expanding Pope's:
  1. Separate subject from body with a blank line.
  2. Limit the subject line to 50 characters.
  3. Capitalize the subject line.
  4. Do not end the subject line with a period.
  5. Use the imperative mood in the subject line.
  6. Wrap the body at 72 characters.
  7. Use the body to explain *what* and *why* vs. *how*.

Beams' load-bearing line: *"A diff will tell you what changed, but only the commit message can properly tell you why."*

The tactical version: subject is what a busy reviewer reads in a list. Body is what an investigator reads two years later trying to understand whether the constraint that produced this code still applies. The body is for *reasons* — the constraint, the bug being fixed, the trade-off rejected, the upstream issue. Not for restating the diff.

### Conventional Commits — when it helps, when it's theater

Conventional Commits (`feat:`, `fix:`, `refactor:`, etc.) are useful when downstream tooling consumes them: automated CHANGELOGs, semver-driven releases, monorepo release tools (Changesets, semantic-release), library publishing. They are bureaucratic theater on internal SaaS apps with continuous deployment where releases aren't versioned — the prefix becomes ritual without semantic value.

Default: Beams' seven rules. Layer Conventional Commits *only* when something in your pipeline reads them.

## Branching models

The honest take in 2026:

- **Trunk-based development** (Paul Hammant, `trunkbaseddevelopment.com`) is the default for continuous-delivery shops. Developers integrate to a single branch (`main` / `trunk`) at high frequency. Branches, when used, are short-lived (hours to a day or two). Incomplete features hide behind release flags. DORA's research consistently correlates trunk-based development with strong delivery performance through the small-batch mechanism (small changes are easier to review, debug, and roll back). Concretely, DORA's threshold: three or fewer active branches, merge to trunk at least daily, no code freezes.
- **GitFlow** (Vincent Driessen, *A successful Git branching model*, 2010) — five branch kinds (`master`, `develop`, `feature/*`, `release/*`, `hotfix/*`) — was the default for a decade. Driessen himself added a 2020 note retiring it for continuous-delivery contexts: *"If your team is doing continuous delivery of software, I would suggest to adopt a much simpler workflow (like GitHub flow) instead of trying to shoehorn git-flow into your team."* GitFlow remains appropriate for **versioned, on-prem, multi-version-supported software** — libraries, OS distributions, enterprise installers. For SaaS in 2026 it is overhead.
- **GitHub Flow** — the lightweight default. main + short-lived feature branches; merge via PR; main is always deployable. The simplest model that works for most webapps and services.
- **GitLab Flow** — variant of GitHub Flow with environment branches (`main → pre-production → production`) for staged deploys. Also supports `release/v1`, `release/v2` for the versioned-software middle ground.

Pick by what you actually ship. Continuous delivery ⟹ trunk-based or GitHub Flow. Versioned product with multiple maintained majors ⟹ GitFlow or GitLab Flow with release branches.

## Rebase vs. merge

The technical difference: merge preserves topology (a merge commit with two parents); rebase rewrites history into a linear sequence (each rebased commit gets a new SHA). Both are legitimate; the question is *when*.

The canonical advice: **rebase to clean up before merging; merge to integrate.** Use rebase to polish your private feature branch (squash fixups, reorder, reword); use merge or fast-forward to land it on main.

### Linus's rule

Jonathan Corbet's *Rebasing and merging: some git best practices* on LWN (April 2009) summarized the kernel-community rule as:

> **Thou Shalt Not Rebase Trees With History Visible To Others.**

The phrase is Corbet's distillation, not a verbatim Linus quote — Linus's own framing is closer to *"if you're still in the 'git rebase' phase, you don't push it out."* Either way, the rule stands. The Linux kernel maintainer documentation reinforces it: *"History that has been exposed to the world beyond your private system should usually not be changed. … If you have pulled changes from another developer's repository, you are now a custodian of their history. You should not change it."*

Practical consequence: never rebase `main`, `master`, release branches, or any branch others have based work on. Never force-push them. Force-push to your own feature branch before it merges is fine — *with* `--force-with-lease` and after coordinating with anyone who's pulled it.

### PR-time merge styles

- **Fast-forward merge.** Branch is a strict descendant; main pointer just moves. Cleanest history; no merge commit. Requires rebasing the branch on main before merging.
- **Merge commit (no-ff).** Explicit topology preserved. Useful when the branch boundary is meaningful (release integration, long-running feature work).
- **Squash merge.** Collapses the branch to a single commit on main. Pro: every main commit is a complete feature. Con: loses the incremental commits below feature granularity. Whether that hurts bisect depends on whether those underlying commits were individually buildable — squash is harmful when they were (you've lost real bisect granularity), helpful when they weren't (you've collapsed un-bisectable noise).
- **Rebase-and-merge.** Replays each branch commit linearly. Pro: keeps atomic commits with linear history. Con: requires every branch commit to be individually buildable.

Pick a default per repo and stick with it. The principle is consistency.

## Pull requests

Google's *Code Author's Guide* (`google.github.io/eng-practices/`) is the canonical source for sizing:

- **One self-contained change per CL.** Refactors separate from behavior changes; tests with the code they validate; experiment / config changes separate from code.
- **Small CLs.** Google's explicit numerical guidance: *"100 lines is usually a reasonable size for a CL, and 1000 lines is usually too large."*
- The reviewer-time argument: it's easier to find five minutes several times for small CLs than to set aside thirty minutes for one large one.

### PR descriptions

A useful template:

- **What** changed (one or two sentences; matches the commit subject).
- **Why** (the problem, the user impact, the constraint).
- **How tested** (specific commands run, environments verified, manual cases checked).
- **Risk** (what could go wrong, blast radius, who else is affected).
- **Rollback** (how to revert if it breaks; whether the migration is reversible).

The Google guide's bad-example list — `Fix bug`, `Fix build`, `Add patch` — is exactly the antipattern set to flag in review.

### Stacked PRs

Tools (Graphite, Sapling, ghstack) let one PR depend on another so a chain of changes can be reviewed independently. Useful when you have a genuinely-dependent chain that each deserves its own review. Overhead when the underlying SCM doesn't support stacks well, or when changes are independent enough to be parallel branches.

## Bisect

`git bisect start; git bisect bad; git bisect good <known-good-SHA>` walks a binary search through history. Mark good or bad at each step; git checks out the next midpoint. Automate with `git bisect run <script>` — exit 0 = good, 1–127 (except 125) = bad, 125 = "skip, can't test."

Bisect-friendliness has three preconditions:

1. Every commit on main builds.
2. Every commit on main passes the test suite (or at least the test you're using as the bisect oracle).
3. Commits are small enough that the bad commit localizes to a meaningful unit.

The Linux kernel routinely bisects across a multi-million-line codebase. If it scales there, it scales anywhere — provided the discipline holds.

## Recovering from mistakes

### Reflog — the local safety net

Every HEAD movement is logged. Default retention: 90 days for reachable entries, 30 days for unreachable. `git reflog` shows the journal. Prefer preserving work first with `git branch rescue-before-reset HEAD` or `git switch -c recovery <sha>`; use `git reset --hard HEAD@{n}` only after `git status` confirms no needed working-tree changes remain.

A botched `git reset --hard`, a deleted branch, a rebase gone wrong — usually recoverable from reflog. If reflog has expired, `git fsck --lost-found` finds dangling commits.

### Force-push discipline

`git push --force` overwrites unconditionally, destroying any commits pushed in the interim. `git push --force-with-lease` only succeeds if the remote is at the SHA you last fetched. Use `--force-with-lease` always; pair it with `push.useForceIfIncludes=true` (Git 2.30+) to defend against the edge case where a `git fetch` between rewrite and push silently re-baselines what `--force-with-lease` considers safe — without `useForceIfIncludes`, the lease passes even though you haven't actually reviewed the new remote work.

The rule: **never force-push to `main`, `master`, or release branches.** On your own feature branches, `--force-with-lease` is the default.

### Committed a secret

The actual fix is **rotate the secret immediately**. The commit exists in every clone, every fork, every CI cache, every backup. Assume it is compromised the moment it is pushed.

History rewriting (after rotation, to keep the secret out of fresh clones): use **`git filter-repo`** (Elijah Newren). The older `git filter-branch` is officially deprecated; the Git docs say plainly: *"its use is not recommended. Please use an alternative history filtering tool such as git filter-repo."* BFG Repo-Cleaner is also widely recommended for this specific use case.

History rewriting is not a substitute for rotation. It is a hygiene step *after* rotation.

## History rewriting

- `git rebase -i` — interactive; `pick` / `reword` / `edit` / `squash` / `fixup` / `drop`.
- `git commit --fixup <SHA>` plus `git rebase --autosquash` slots a fix into the commit it belongs to. The cleanest workflow for "this fixes the commit I made yesterday."
- Force-push your own feature branch with `--force-with-lease` before merge — fine, after coordinating with anyone who has pulled it.
- Never rewrite shared history. Linus's rule.

## What history is for

`git log path/to/file` and `git log -p path/to/file` should answer "why does this line exist?" `git blame` is for context, not for assigning blame — the right reframing is "who can I ask about this code?"

The canonical record is: PR description + commit messages + linked issue. Slack threads, hallway conversations, design docs in unlinked Google folders — all decay. The git history travels with the code and is the source of truth in the long run.

The implication: **clean what you push for review.** Do whatever you want on your local branch — squash fixups, reorder, drop dead-end attempts. What lands on main should be readable. Intermediate scratch commits are not the record.

## Monorepo vs. multi-repo, briefly

The trade-offs are real and context-dependent:

- **Monorepo** — atomic cross-cutting changes; unified versioning; code discoverability. Costs: build-system complexity (Bazel/Pants/Buck2/Nx/Turborepo); tooling demands at scale (Sapling/Scalar); coupling can mask boundaries.
- **Multi-repo** — clear ownership boundaries; independent release cadence; off-the-shelf tooling. Costs: cross-cutting changes require coordinated PRs; dependency upgrades become a manual chase across repos.

Public references at scale: Google's monorepo (~2 billion lines, Piper VCS); Meta's Sapling (open-sourced 2022); Microsoft's VFS for Git (now superseded by Scalar, upstreamed into Git). Pick by team scale, build-system maturity, and ownership clarity. There is no universally right answer.

## Merge queues

At trunk-based-development scale, the bottleneck shifts from "is this PR ready" to "is `main` still green after every merge." Naïve merge of N PRs against a moving `main` produces semantic conflicts (each individually green, none green together) that bisect later as mysterious failures.

**Merge queues** serialize and rebase merges through a common gate: GitHub's native Merge Queue (GA 2023), Bors, Mergify, Aviator. The queue rebases each PR onto the latest `main`, runs CI, and merges only if CI passes against the actual post-merge state. Trades latency (your PR waits in line) for the guarantee that `main` is always built-and-tested as it actually is, not as it was at PR time. For teams shipping dozens of PRs per day, this is the missing piece between trunk-based development and "main is always green."

## Signed commits

Commit signatures bind authorship to a verifiable identity. Two formats:

- **GPG-signed** — the older standard. `gpg.format=openpgp` (default).
- **SSH-signed** — supported since Git 2.34 (November 2021), `gpg.format=ssh`. Reuses your existing SSH identity. Increasingly the default for new setups.
- **Sigstore-signed** via `gitsign` — short-lived OIDC-bound certs, no long-lived key material. See `build-and-dependencies` for sigstore depth.

Signed commits are weaker than artifact signing (anyone with push access can author a malicious signed commit) but raise the bar against impersonation and are the foundation for branch-protection rules that require verified commits. For protected branches and release tags, require signatures.

## Pre-commit hooks

Pre-commit hooks run client-side before a commit lands locally. They are appropriate for **fast checks that catch obvious mistakes early** — lint, format, simple secret detection, basic tests on changed files. The discipline:

- **Sub-second.** Pre-commit hooks that take five seconds train developers to use `--no-verify`. Move slow checks to pre-push or CI.
- **Idempotent and deterministic.** A hook that randomly fails poisons the workflow.
- **Match what CI runs.** Pre-commit catching things CI doesn't is fine; CI catching things pre-commit also catches is duplication. CI catching things pre-commit *should* have caught is the right division of labor.
- **Frameworks**: pre-commit (`pre-commit.com`), husky (JS), lefthook (cross-language), simple-git-hooks. Pick by team.

`--no-verify` is occasionally legitimate (a known-broken hook, a doc-only commit). It must not be a habit. A pre-commit hook that gets bypassed routinely is a hook to fix or remove.

## Common antipatterns

- Commit messages: `wip`, `fix`, `asdf`, `more changes`, `address PR comments`, `final`, `final-2`.
- 1000+ line PRs (Google's "usually too large" threshold).
- Long-lived feature branches (weeks). They accumulate merge debt and die.
- Force-push to `main` / release branches.
- `--no-verify` on commits to skip pre-commit hooks "just this once" — habits form fast.
- Committed secrets without rotation.
- Mixing refactor + behavior change in one commit (un-bisectable, un-revertable independently).
- Gratuitous merge bubbles — merge commits when the branch was a strict descendant. Use fast-forward.
- Branch names like `dev-jeff-final-final-2`, `temp`, `test123`. Use the repo's convention (typically `<author>/<issue>-<slug>`) consistently.
- Empty commit body when the *why* isn't obvious.
- Squash-merging everything reflexively — collapses bisect granularity below feature size.
- Five commits to revert the previous five because there was no review discipline.
- Bypassing branch protection via admin override to push to main.
- Using `git pull` (which silently merges) where `git pull --rebase` was intended; producing surprise merge commits in personal branches.

## What to flag in review

- A PR with `wip` or `fix` commits as final history rather than as scaffolding.
- A commit that mixes refactor and behavior change.
- A commit message body that restates the diff instead of explaining the constraint.
- A long-lived feature branch (over a few days) without a tracking issue and merge plan.
- A force-push to a shared branch.
- A commit that introduces a hard-to-bisect failure (the change is so large bisect can't localize).
- A merge commit on a personal branch where fast-forward was the intent.
- A repo policy that allows direct push to main without review.
- A history-rewriting fix to a leaked secret with no rotation.
- A PR description without **why** or **rollback**.
- 1000+ lines split across 30 commits as if that compensates.

## Reference library

- `references/commit-message-craft.md` — Pope and Beams in depth, Conventional Commits trade-offs, examples of subject-and-body that earn their keep.
- `references/branching-models-in-practice.md` — trunk-based vs. GitHub Flow vs. GitFlow with the criteria for choosing, including Driessen's 2020 note in full context.
- `references/recovery-and-rewriting.md` — reflog, force-with-lease, filter-repo, BFG, and the discipline of rotating secrets that escaped into history.

## Sibling skills

- `engineering-discipline` — review and triage; how to handle PR feedback as a craft (overlaps with the orchestrator).
- `deployment-and-release-engineering` — trunk-based development as the upstream of small-batch deployment; CL size correlated with deployment frequency in DORA's research.
- `documentation-and-technical-writing` — commit messages and PR descriptions as documentation.
- `refactoring` — Beck's *Tidy First?* discipline of separating tidying from behavior change, which is the same discipline as one-logical-change-per-commit.
- `testing-discipline` — bisect requires every commit on main to pass tests; that is a property of the test suite, not just the commits.
