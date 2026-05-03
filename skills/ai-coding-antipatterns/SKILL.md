---
name: ai-coding-antipatterns
description: "Catch the failure modes that LLM-generated code routinely produces — failures that look plausible, follow conventions, and pass shallow review, while being subtly or obviously wrong. Use this skill whenever reviewing code that was written by an AI (Claude, GPT, Copilot, Codex, Cursor, etc.), whenever you suspect the code was written by an AI, whenever you're writing code with AI assistance and want a sanity check, and proactively in any code review since AI-authored code is now common. The patterns are specific (hallucinated APIs, fabricated dependencies, sycophantic agreement, defensive over-handling, ignoring local conventions, fake tests, scope creep, deletion aversion) and the discipline is: never assume, always verify."

---

# AI Coding Antipatterns

LLMs produce code that looks professional. It uses idiomatic syntax, has good variable names, includes comments, follows the conventions it was trained on. It is also routinely wrong in characteristic ways — and those ways are different from the ways human-written code is wrong.

Human reviewers, used to spotting human bugs, miss AI bugs. The bugs hide in plausibility: the code "looks right," and the review approves on vibes. The whole point of this skill is to interrupt that vibe and apply specific verification.

## The defining problem

LLMs predict text that's likely given the context. They have no model of "this code must be correct." They have a strong prior that "code that looks like working code" is what should appear next. The result: code that *looks like* it should work, even when it doesn't.

This produces a specific failure mode: the bug is not in what the code *does*, it's in what it *says it does* — invented APIs, packages that don't exist, tests that don't test, assertions that always pass, comments that contradict the code.

The defense is verification, not intuition. Trust nothing you didn't check.

## The catalog

### Hallucinated APIs

The model invents a function, method, class, or module that doesn't exist. The invention is plausible — it has a sensible name, sensible parameters, the right kind of return value — but calling it produces an `AttributeError` / `TypeError` / `undefined`.

Examples:
- `requests.fetch_json(url)` when the actual API is `requests.get(url).json()`.
- `pd.DataFrame.unique_rows()` when the actual API is `pd.DataFrame.drop_duplicates()`.
- `boto3.s3.upload_string(bucket, key, content)` — looks reasonable; doesn't exist.
- A whole module: `from useful_helpers import deep_merge` — useful_helpers isn't installed.

**Verification**: read the library. Don't trust the model that the API is real. For unfamiliar libraries, look at the actual documentation or read the source. If the model hallucinated once in a file, look at every other library call.

### Hallucinated dependencies

Packages added to `requirements.txt` / `package.json` / `Cargo.toml` that don't exist in the registry, or exist but aren't what the model thinks they are.

A specific risk: **typo-squatted malicious packages.** Attackers register packages with names close to real ones (`reqests` instead of `requests`, `pythonn-requests`, etc.). When the model hallucinates a dependency name and the name happens to be a typo-squat, you've added malicious code to your project.

**Verification**: every new dependency, look it up in the registry. Verify download counts, last update, maintainer, source link, license, and whether the package is the intended project. Use the ecosystem's integrity mechanism: npm/pnpm/yarn lockfile integrity, `go.sum`, `Cargo.lock` for binaries/apps, or pip/uv/Poetry lockfiles with hashes where supported.

### Fabricated test cases

The test asserts on values the AI computed by running the (possibly-wrong) production code in its head. The test is a tautology — it can only fail if the test framework breaks.

```python
# Production
def calculate_discount(price, customer_tier):
    if customer_tier == "gold":
        return price * 0.10
    elif customer_tier == "silver":
        return price * 0.05
    return 0

# AI-generated test (bad)
def test_calculate_discount():
    assert calculate_discount(100, "gold") == 100 * 0.10  # restates the formula
    assert calculate_discount(100, "silver") == 100 * 0.05
    assert calculate_discount(100, "bronze") == 0
```

If the production formula is wrong (say, "gold" should be 15%), the test wrongly endorses it.

**Verification**: when reviewing AI-generated tests, mentally remove the production change. Does the test still pass? If yes, the test is testing nothing. The expected values in tests should be computed independently — by hand, by an oracle, by a known-correct reference implementation — not by reproducing the formula.

### Sycophantic agreement

The model agrees with whatever the user said, even when the user is wrong. If you say "this code is buggy because of X," the model will explain how it could be X — even if X isn't the actual bug. If you say "make this faster," it will produce a "faster" version even if the original was already optimal.

The shape: the model's response always matches the framing of the request, never pushes back, never says "actually, the original was right."

**Verification**: be skeptical of confident agreement. If you ask the AI to find a bug and it finds one immediately, ask "are you sure? what evidence?" — if it can't produce evidence, the bug is hallucinated. Especially watch for "the bug was X" with no demonstration that X actually causes the symptom.

### Defensive over-handling

LLMs love `try/except`. They wrap everything that could conceivably raise — including code that doesn't actually raise — in broad catch-all handlers that log and continue.

```python
# AI-generated (bad)
def get_user_email(user):
    try:
        return user.email
    except Exception as e:
        logger.error(f"Failed to get email: {e}")
        return None
```

If `user` doesn't have `.email`, that's a bug — the caller passed the wrong thing. Silently returning `None` and logging is hiding the bug. Now downstream code gets `None` and fails differently, far from the actual cause.

**Verification**: for every `try/except`, ask: what specific exceptions are expected, why, and what's the recovery? "Catching anything in case" is not an answer. Replace with `except SpecificError:` with a real handler, or remove the try.

### Ignoring local conventions

The model writes idiomatic code in the style it was trained on, ignoring the style of the codebase it's editing. The result: a function that uses a different naming convention than the rest of the file, an error-handling pattern that doesn't match what the codebase does elsewhere, an import style that doesn't match.

The cost: every reader has to context-switch. The codebase becomes a patchwork.

**Verification**: when the AI is editing existing code, the change should look like it belongs. If the codebase uses snake_case, the new code should too. If the codebase uses Result types, don't introduce exceptions. If the codebase has a logging pattern, use it. The model can't see the rest of the codebase well; you can.

### Reinventing existing utilities

The model writes a function that already exists in the codebase (often three folders away). Now there are two implementations, and any future bug fix must be applied in both.

**Verification**: before accepting a new utility function, require evidence that the codebase was searched: search results, existing utility references, or file paths showing no equivalent exists.

### Comments that narrate code

```python
# Increment the counter
counter += 1
# Check if the counter is greater than 10
if counter > 10:
    # Reset the counter to zero
    counter = 0
```

Each comment restates the next line. They add nothing. They are a tell — generated text that looks like documentation but is just decoration.

**Verification**: every comment should explain *why*, not *what*. Comments that narrate the line below them: delete or replace with a comment about intent.

### Scope creep

You asked the model to fix a bug. It also "improved" five other things in the same diff: renamed a variable, restructured a loop, added type hints, "cleaned up" a function. None of the additional changes were requested; each carries risk; the diff is now hard to review and hard to revert.

**Verification**: in code review, the change should match the request. If the diff is doing extra things, ask the author to split. Refactor PRs are separate from bug-fix PRs.

### Deletion aversion

The model is reluctant to delete code. When asked to remove something, it often just comments it out, or moves it elsewhere, or wraps it in a feature flag. It produces "soft deletes" that leave the old code lying around.

**Verification**: when the model says it deleted something, verify it's actually deleted (not commented out, not moved). Dead code is a maintenance cost; commented-out code is worse than deleted code.

### "I improved error messages" without checking

The model rewrites error messages, often "improving" them by adding more detail or making them friendlier. The new messages may break:

- Tests that asserted on the old message.
- Operators who grep logs for the old message.
- Documentation that quoted the old message.
- Error parsers in clients.

**Verification**: error message changes are API changes (Hyrum's Law). Don't change them speculatively. If you change them, find every consumer.

### Sweeping refactors of working code

The model "modernizes" perfectly working code: converts loops to comprehensions, callbacks to async/await, classes to functions, or vice versa, depending on what's stylish. The refactor is risky and unrequested.

**Verification**: working code that the user didn't ask to be refactored should not be refactored. This is a stronger version of scope creep — the change is purely aesthetic.

### Manufactured "fixes" for non-bugs

You report a symptom; the model fabricates a plausible-sounding fix without verifying the cause. The fix doesn't address the actual bug; the symptom may briefly disappear (because the model added defensive code that masks it) only to recur later.

**Verification**: for every bug fix, the model should be able to (a) identify the root cause specifically, (b) show that the cause produces the symptom, and (c) show that the fix removes the cause. "I added a check that probably fixes it" is not a fix.

### Tests that pass with the production change reverted

The defining check for AI-generated tests. If the test would still pass without the change it claims to test, the test is decoration.

**Verification**: mentally undo the production change; the test should fail. If it doesn't, the test is testing the wrong thing or testing nothing.

### Test suites that look comprehensive but skip the hard cases

The model generates many test cases for the happy path and skips the cases that actually catch bugs:
- Empty input
- Boundary values
- Concurrent access
- Error paths
- Realistic data sizes

**Verification**: for a feature, ask "where are the empty-input tests, the boundary tests, the error tests, the concurrency tests?" If they're missing, add them.

### Verbose code where concise is better

The model produces explicit, expanded code where idiomatic concise code is appropriate. Eight lines of imperative loop where a one-line list comprehension is the local idiom. A 30-line function decomposed into three 10-line helpers when the original was perfectly readable.

The opposite also happens: dense one-liners where explicit code is clearer. The model lacks judgment about appropriate verbosity.

**Verification**: judge by local convention and appropriate complexity, not by line count.

### Made-up examples in docstrings or comments

The model adds "Example:" sections to docstrings, with examples that don't match the actual function. Or "see also" references to functions that don't exist. Or links to documentation pages that don't exist.

**Verification**: every example in a docstring must be runnable and correct. Every "see also" reference must point to something real.

### Refusing to admit not knowing

When the model doesn't know something, it makes up a plausible answer. "What's the right way to do X in this framework?" produces an answer even if there is no right way (or the model doesn't know). Asking "are you sure?" sometimes catches it; sometimes it doubles down.

**Verification**: for any non-trivial framework / library / tool claim, verify against the actual documentation. The model's confidence is not evidence.

## The gating questions for every AI-authored change

Run these as a checklist:

1. **Do the APIs / functions / packages this code calls actually exist?** (Verify against the library, the registry.)
2. **For new dependencies, are they real, well-maintained, and what we want?**
3. **For each `try/except`, is the recovery real, or is it suppression?**
4. **Does the code follow the conventions of this codebase?**
5. **Are there existing utilities being reinvented?**
6. **Do the comments explain *why* (good) or restate the next line (bad)?**
7. **Is the scope of the change exactly what was requested?**
8. **For deletions: is the code actually gone, or just hidden?**
9. **For tests: does each test fail if the production change is reverted?**
10. **For tests: are the boundary cases, empty-input cases, error paths, and concurrency cases covered?**
11. **For "fixes": is the root cause identified and the fix demonstrably addresses it?**
12. **For "improvements": were they requested, or scope-creep?**

If the answer to any of these is "I don't know," verify before merging. The AI's confidence is not a signal.

## How to phrase pushback to an AI

When you see an antipattern in AI-generated code:

- **Be specific.** Not "this looks wrong" but "the call to `pd.DataFrame.unique_rows()` — that method doesn't exist, did you mean `drop_duplicates()`?"
- **Demand evidence.** "Show me where in the documentation `requests.fetch_json` is defined."
- **Ask what would fail.** "If we revert the production change, would the test still pass? What would happen?"
- **Refuse vague hand-waves.** "Probably," "should work," "this might fix it" — push for definite.

The AI's response to specific, evidence-based pushback is usually correction. The AI's response to vague pushback is usually doubling down. Be specific.

## When AI-generated code is fine to approve

Not all AI code is bad. The AI shines on:
- Boilerplate that follows established patterns.
- Mechanical refactors with clear before/after specs.
- Search-and-replace style edits.
- Generating test cases for a known-correct reference (when you can verify the assertions independently).
- Glue code that connects two well-defined APIs.

The AI struggles on:
- Anything that requires understanding the codebase as a whole.
- Anything with subtle invariants.
- Anything novel.
- Anything where the bug-cost is high.
- Anything in unfamiliar libraries / frameworks.

For high-stakes changes, the right model is: AI drafts, human verifies. The verification has to be substantive — not a glance, but an active check that the antipatterns above aren't present.

## What to flag in review

Specifically for AI-authored code:

- Unfamiliar library calls — verify they exist.
- New dependencies — verify in the registry.
- Broad `except Exception` blocks.
- Tests that restate the formula.
- Comments that narrate code.
- Dependencies / utilities that already exist in the codebase.
- Refactors mixed with bug fixes.
- "Soft deletes" (commented-out code).
- Error message changes without verifying consumers.
- Confident assertions about behavior without evidence.

When you find one, name it. Authors (and other AIs) learn to do better when the antipattern has a name and a fix.

## Reference library

- `references/verification-recipes.md` — concrete, executable steps for catching each antipattern. When an antipattern is suspected, this file has the actual command/check.
