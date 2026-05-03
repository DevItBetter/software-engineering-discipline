# Verification Recipes for AI-Generated Code

Concrete, executable recipes for catching the AI antipatterns. Use these when you have AI-authored code in front of you.

## Recipe 1: Verify every external call

For each function/method called from a library that you didn't add:

```bash
# In Python — find the actual signature
python -c "import library; help(library.module.function)"

# Or search the library source
rg "def function_name" path/to/library --glob '!vendor/**' --glob '!dist/**'

# Or check the official docs
# (Don't trust the AI to tell you what the signature is.)
```

Common signatures to verify:
- HTTP libraries (`requests`, `httpx`, `axios`, `fetch`).
- Database / ORM (`sqlalchemy`, `prisma`, `gorm`, `entity framework`).
- Cloud SDKs (`boto3`, `google-cloud-*`, `@aws-sdk/*`).
- Data libraries (`pandas`, `numpy`, `scikit-learn`).

If even one call is wrong, treat every call as suspect.

## Recipe 2: Verify every new dependency

```bash
# Python
python -m pip index versions package_name
# Also check the PyPI project page or JSON metadata for source, license, release history, and maintainers.

# JavaScript
npm view package_name name version description maintainers repository license time dist.integrity

# Rust
cargo info package_name

# Go
go list -m -versions module/path
```

Check:
- Does it exist?
- Maintainer / author looks legitimate?
- Last update recent (or appropriately old for stable libraries)?
- Repository link works and points to actual source?
- Download count consistent with the project's apparent maturity?
- License compatible with your project?

For each dependency the AI added, paste the output into the PR. Build the habit.

## Recipe 3: Run the "revert the production change" test

For each AI-generated test:

1. Read the production code under test.
2. Read the test.
3. Mentally remove the production change (or git stash it).
4. Run the test in your head with the unmodified production code.
5. Does the test fail?

If the test passes against unmodified production code, the test isn't testing the change. Either the test is wrong or the production change is dead code.

For a real check: actually revert the production code and run the test. Watch what happens. (Best done in a scratch branch.)

## Recipe 4: Search for reinvented utilities

Before accepting a new utility function:

```bash
# Search the codebase for similar names
rg "function_name_or_similar" --glob '!node_modules/**' --glob '!vendor/**' --glob '!dist/**'

# Search for similar functionality
rg "format.*phone" --glob '!node_modules/**' --glob '!vendor/**' --glob '!dist/**'   # if you're adding a phone formatter
rg "validate.*email" --glob '!node_modules/**' --glob '!vendor/**' --glob '!dist/**' # if you're adding an email validator
rg "deep.*merge" --glob '!node_modules/**' --glob '!vendor/**' --glob '!dist/**'     # if you're adding deep merge

# Search the project's documented utilities folder if there is one
ls path/to/utils/
```

If a similar utility exists, the new code should use it, not reinvent it.

## Recipe 5: Audit `try/except` blocks

```bash
# Find all broad excepts in the diff
git diff | rg -A 3 "except Exception"
git diff | rg -A 3 "except:"
git diff | rg -A 3 "catch \(.*\)"  # JS/Java
```

For each one:
- What specific exceptions are expected here?
- Why doesn't the recovery handle the specific exception type?
- What's the actual recovery — meaningful, or "log and continue"?

Replace catch-alls with specific catches and real recovery, or remove the try entirely.

## Recipe 6: Check comments

```bash
# Find new comments in the diff
git diff | rg '^\+\s*(#|//)'
```

For each new comment:
- Does it explain *why*, or restate the code?
- Is it correct (not contradicting the code)?
- Is the example in the docstring runnable as-is?

Delete comments that narrate the next line.

## Recipe 7: Verify deletions

When the AI says "I removed X":

```bash
# Did X actually go away?
rg "X" --glob '!node_modules/**' --glob '!vendor/**' --glob '!dist/**'

# What remains?
git log --diff-filter=D --all -- path/to/file/X
```

If X is still there (commented out, moved, hidden behind a flag), it wasn't deleted. Either delete it or update the description to say what really happened.

## Recipe 8: Verify the scope matches the request

For an AI-authored PR:

1. Read the user's original request.
2. Read the diff.
3. Count: how many things does the diff do?

If the request was "fix the timezone bug" and the diff also renames three functions, restructures a class, and adds type hints — that's four changes, three of which weren't requested. Split the PR (or push back).

## Recipe 9: Check that tests cover what matters

For a feature, look for tests in these categories:

- [ ] Happy path with realistic data
- [ ] Empty input
- [ ] Boundary values (max, min, off-by-one)
- [ ] Invalid input (each kind of invalidity)
- [ ] Authorization failure (unauthorized caller)
- [ ] Downstream failure (dependency unavailable)
- [ ] Concurrency (if the path is concurrent)
- [ ] Idempotency (if the operation might be retried)

Missing categories: the test suite is incomplete. Add the missing tests, or note them as residual risk.

## Recipe 10: Cross-check with the actual library docs

For any non-trivial framework / library claim the AI made:

```bash
# Pull up the official docs in a browser (or via fetch)
# Verify the claim against the source of truth
```

Don't trust the AI's summary of how a library works. The model's training data may be old, may be wrong, may be misremembered. The docs and the source code are the truth.

## Recipe 11: Pin AI assertions to commits / lines / docs

When the AI says "the existing code does X":

- "Show me the line that does X."
- "Which commit added that?"
- "Which docs page describes that behavior?"

If the AI can't ground the claim in something verifiable, the claim is suspect. Most AI hallucinations come unstuck under direct grounding requests.

## Recipe 12: Sanity-check the diff size

A useful heuristic: if the AI's diff is much larger or much smaller than you'd expect for the task, something is off.

- Much larger: scope creep (it did extra things). Split.
- Much smaller: probably didn't do enough. Verify all the requirements were actually addressed.

## Habits to build

- **Run tests locally** before approving. Not just CI.
- **Read every line of the diff.** AI diffs are not LGTM material.
- **Check at least one call site of every changed public function.**
- **Verify at least one library call you don't recognize.**
- **Notice when the AI is being unusually agreeable** to your suggestions — push back to test whether it has independent evaluation or is sycophantic.

## What to write in the review

For suspected or disclosed AI-authored code, write the review around verified risk, not provenance guesses:

```
For this change, I verified:
- [list of library calls / APIs you confirmed exist]
- [list of dependencies you confirmed in the registry]
- [tests that fail when the production change is reverted]
- [scope matches the request: yes / no — if no, what extra]

Outstanding concerns:
- [anything you couldn't verify and why]
- [antipatterns flagged]
```

This protects the team: the next reviewer or the author later knows what was checked and what wasn't. Mention AI authorship only when it is disclosed or policy-relevant.
