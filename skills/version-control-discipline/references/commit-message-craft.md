# Commit Message Craft

Reference for writing commits that earn their keep — that explain *why* the code looks the way it does, in a form that survives the team that wrote them.

## The two canonical sources

- **Tim Pope, *A Note About Git Commit Messages* (2008-04-19).** `https://tbaggery.com/2008/04/19/a-note-about-git-commit-messages.html`
- **Chris Beams, *How to Write a Git Commit Message* (2014-08-31).** `https://chris.beams.io/posts/git-commit/`

Beams credits Pope; Pope was first.

### The 50 / 72 rule (Pope, 2008)

- Subject ≤ 50 characters.
- Body wrapped at 72 characters.

The 72 number isn't arbitrary: an 80-column terminal minus a 4-column indent on each side leaves 72. `git log` indents the body by 4; output that wraps at 72 stays within 80 visual columns.

### The seven rules (Beams, 2014)

1. Separate subject from body with a blank line.
2. Limit the subject line to 50 characters.
3. Capitalize the subject line.
4. Do not end the subject line with a period.
5. Use the imperative mood in the subject line.
6. Wrap the body at 72 characters.
7. Use the body to explain *what* and *why* vs. *how*.

The load-bearing line, from Beams: *"A diff will tell you what changed, but only the commit message can properly tell you why."*

## Why imperative

The imperative matches the language of git itself: `git revert` produces *"Revert XYZ"*; `git merge` produces *"Merge branch …"*; `git cherry-pick` retains the original message. A subject of *"Fix authentication race"* composes naturally with these auto-messages. *"Fixed authentication race"* and *"Fixes authentication race"* compose awkwardly.

A useful test (from Beams): the subject completes the sentence *"If applied, this commit will __."* If "fix the authentication race" reads naturally, you have the right mood. If "fixed the authentication race" reads better, you've written past tense.

## What goes in the body

The body explains:

- **The constraint.** What invariant, requirement, deadline, or upstream contract drove this change.
- **The bug being fixed.** Specific input, sequence, environment. A future debugger reading `git blame` should be able to confirm whether the bug is back.
- **Trade-offs rejected.** "I considered using X, but it would have required Y" — saves the next person the same investigation.
- **Upstream issues / discussions.** Link to the issue, the design doc, the linked PR. Not as a substitute for explanation, as additional context.

What does *not* go in the body: a restatement of the diff (the reader already has the diff); a summary of which files changed (the reader has that); ceremony like "this commit makes the following changes" before listing them.

## Worked examples

### Bad

```
Fix bug
```

The reader has no idea what bug, where, or why. `git log` becomes uninspectable.

### Better but still bad

```
Fix authentication bug

Fixed the authentication bug in the login handler.
Updated the tests to cover the new case.
```

The body restates the subject and the diff. It adds nothing.

### Good

```
Fix race in token refresh during concurrent renewals

Two concurrent refreshes could both observe an expired token,
each issue a new one, and the second's write would clobber the
first's session record. The fix: take an advisory lock on the
session id before reading the token.

Considered: making the refresh idempotent by storing a
nonce-keyed cache of recent refreshes. Rejected because the
cache TTL would have to exceed the refresh interval, which is
configurable per tenant; the lock is simpler.

Reproduces with the new test in test_auth_concurrency.py.

Fixes #4421.
```

The reader learns: the precise failure mode, the fix, the alternative that was considered and rejected (and why), how to reproduce, and the upstream issue. Two years later, when someone is asking "do we still need this lock?", the body answers "what would change if we removed it."

## Conventional Commits — when it earns its keep

Conventional Commits (`feat:`, `fix:`, `refactor:`, `docs:`, `BREAKING CHANGE:`) is a structured prefix layered on top of the message. It earns its keep when downstream tooling consumes it:

- Automated CHANGELOG generation.
- Semver-driven release tools (Changesets, semantic-release, release-please).
- Monorepo release frameworks that need to know which packages get a major / minor / patch bump.
- Library publishing where consumers expect semver-shaped releases.

It is bureaucratic theater on internal SaaS apps with continuous deployment where releases aren't versioned, no CHANGELOG is generated, and no tool reads the prefix. The prefix becomes ritual without semantic value.

The default: write Beams-quality messages. Add Conventional Commits *only* when something in your pipeline reads them.

## Local commit hygiene

Pope and Beams describe what *lands on main*. While developing on your own branch, do whatever serves your thinking — `wip`, `fix`, half-baked attempts, debugging breadcrumbs. Then before you push for review:

- `git rebase -i` to squash fixups into the commit they belong to.
- `git commit --fixup <SHA>` and `git rebase --autosquash` for the cleanest workflow when fixing a commit you made earlier.
- Reword to give each commit the message it deserves.
- Drop dead-end attempts.

The history that lands on main should be readable. The history of how you got there is your own.

## What to flag in review

- Subject line over 50 characters (especially when it could trivially be shortened).
- Subject in past tense ("Fixed bug X") when imperative was the convention.
- Subject ending with a period.
- Body that restates the diff.
- Body that has no "why."
- Commits titled `wip`, `fix`, `address PR comments`, `asdf` — scaffolding leaked into the historical record.
- Conventional Commits prefix on a project that doesn't read them — cargo-cult formatting.
- A commit that bundles refactor + behavior change.
- A commit that should reference an issue or design doc and doesn't.

## Sources

- Tim Pope. *A Note About Git Commit Messages* (2008). `https://tbaggery.com/2008/04/19/a-note-about-git-commit-messages.html`.
- Chris Beams. *How to Write a Git Commit Message* (2014). `https://chris.beams.io/posts/git-commit/`.
- Conventional Commits 1.0.0. `https://www.conventionalcommits.org/en/v1.0.0/`.
- Peter Hutterer's framing on commit messages as collaboration signal (quoted in Beams).
