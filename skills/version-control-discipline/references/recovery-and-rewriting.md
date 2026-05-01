# Recovery and Rewriting

Reference for the parts of git that catch you when you fall — `reflog`, `--force-with-lease`, `filter-repo` — and the discipline around using them.

## The reflog: the local safety net

Every movement of `HEAD` is logged. `git reflog` shows the journal:

```
abc1234 HEAD@{0}: commit: Fix login race
def5678 HEAD@{1}: rebase finished: returning to refs/heads/feature
9ab0cde HEAD@{2}: rebase: Squash fixups
```

Default retention: **90 days for entries reachable from a branch**, **30 days for unreachable entries**. Configurable via `gc.reflogExpire` and `gc.reflogExpireUnreachable`.

A botched `git reset --hard`, a deleted branch, a rebase that ate work — usually recoverable.

```
git reflog                   # find the SHA you want back
git reset --hard HEAD@{4}    # restore HEAD to that point
git checkout -b recovery <sha>   # or branch from the lost commit
```

The reflog is local-only. Remote git servers do not have your reflog. This is fine; the discipline is to recover before you push.

If reflog entries have expired, `git fsck --lost-found` finds dangling commits not referenced by anything. Last resort.

## Force-push discipline

`git push --force` overwrites unconditionally. If a teammate pushed in the interim, their commit disappears.

`git push --force-with-lease` is the safer variant: the push only succeeds if the remote ref is at the SHA you last fetched. If someone else has pushed, your force-push is rejected and you re-pull and reconsider.

The edge case: if you `git fetch` between the rewrite and the push, `--force-with-lease` may succeed even if you haven't reviewed the new remote work, because your "last seen" is now the teammate's commit. Mitigate with `git config --global push.useForceIfIncludes true` (Git 2.30+) — this requires the local branch to actually contain the remote work as ancestry, not merely to know about it.

The rule: **never force-push to `main`, `master`, or release branches.** These are shared history; rewriting them breaks every clone, every CI cache, every fork. On your own feature branches, `--force-with-lease` is the default.

## Recovering committed secrets

The first step is **rotate the secret immediately**. The commit exists in:

- Every clone any teammate has pulled.
- Every fork on GitHub.
- Every CI cache, every artifact store, every backup that includes the repo state.
- Any indexer that scraped the public git URL.

Assume the secret is compromised the moment it leaves your machine. Rotation is the actual fix; history rewriting is a hygiene step *afterward* to keep fresh clones from re-introducing the secret.

### Rewriting history with `git filter-repo`

The current recommended tool, by Elijah Newren. `https://github.com/newren/git-filter-repo`.

Typical use:

```
git filter-repo --replace-text replacements.txt
```

where `replacements.txt` contains literal strings or regexes for the secrets to redact. `filter-repo` is fast (orders of magnitude faster than `filter-branch`), safer, and actively maintained.

### Why not `git filter-branch`

The official Git docs explicitly deprecate it. From `https://git-scm.com/docs/git-filter-branch`:

> *"git filter-branch has a plethora of pitfalls that can produce non-obvious manglings of the intended history rewrite (and can leave you with little time to investigate such problems since it has such abysmal performance). These safety and performance issues cannot be backward compatibly fixed and as such, its use is not recommended. Please use an alternative history filtering tool such as git filter-repo."*

If you are looking at a Stack Overflow answer that recommends `filter-branch`, prefer `filter-repo`. The Stack Overflow answer is from before 2020.

### BFG Repo-Cleaner

`https://rtyley.github.io/bfg-repo-cleaner/`. A simpler alternative for the specific case of removing large files or strings. Less flexible than `filter-repo`; sometimes friendlier for one-shot cleanups.

### After rewriting

A history rewrite changes every commit SHA from the rewrite point forward. Consequences:

- Force-push the rewritten history (and only after rotating the secret).
- Tell teammates: they need to re-clone or carefully `git fetch && git reset --hard origin/main` after coordinating. Their old clones still have the secret in reflog and packed refs.
- Open forks still have the old history. Reach out to their maintainers if the data warrants.
- CI caches, artifact stores, and backups still have the old commits. Most of these expire on their own; if the secret is high-stakes, identify and purge.

The discipline is the same as for any incident: rotate, contain, communicate, monitor for misuse.

## Common recovery scenarios

| Symptom | Recovery |
|---|---|
| `git reset --hard` then realized I lost work | `git reflog`, find the SHA before the reset, `git reset --hard <sha>` |
| Deleted a branch that hadn't been merged | `git reflog`, find the tip SHA, `git checkout -b <name> <sha>` |
| Rebase corrupted my branch | `git reflog`, find HEAD@{N} before the rebase, `git reset --hard HEAD@{N}` |
| Force-pushed and teammate's work disappeared | Teammate has it in their local; have them push their version after rebasing onto your new history. Apologize. |
| Secret committed | Rotate the secret. Then `git filter-repo` to scrub from history. Then communicate to the team. |
| Botched merge | `git merge --abort` if still in progress. If completed, `git reset --hard HEAD@{N}` from reflog. |
| Pushed to main accidentally and tests fail | `git revert <sha>` to roll forward (preferred — preserves history). Force-push to main only as a last resort, and only with explicit authorization. |

## What to flag in review

- A history-rewriting fix to a leaked secret with no rotation step.
- A `git push --force` invocation in a CI script targeting a shared branch.
- A guide or runbook recommending `git filter-branch` (deprecated) instead of `git filter-repo`.
- Repo policy that allows force-push to main without a multi-party review.
- Pre-commit hooks bypassed via `--no-verify` as a workflow norm.
- Reflog-based recovery instructions that don't mention the 90-day window.

## Sources

- `git reflog` documentation. `https://git-scm.com/docs/git-reflog`.
- `git filter-branch` deprecation note. `https://git-scm.com/docs/git-filter-branch`.
- `git filter-repo` (Elijah Newren). `https://github.com/newren/git-filter-repo`.
- BFG Repo-Cleaner. `https://rtyley.github.io/bfg-repo-cleaner/`.
- Atlassian on `--force-with-lease`. `https://www.atlassian.com/blog/it-teams/force-with-lease`.
