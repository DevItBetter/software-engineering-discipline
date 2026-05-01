# Review Checklists By Change Type

Different changes have different failure modes. Apply the lens that matches.

## Feature

The author is adding new behavior. Risk concentrates in: contract, edge cases, observability, and "what was missed."

- Is the **acceptance criterion** stated? If not, you cannot verify it. Ask.
- Is the **happy path tested with realistic data**, not just `assert add(1,2)==3` placeholders?
- Are the **negative paths covered**: invalid input, unauthorized caller, missing dependency, downstream failure, timeout?
- Are **edge cases** covered: empty, max-length, unicode, leading/trailing whitespace, duplicates, ordering, time-zone, locale, currency precision, off-by-one on boundaries?
- Is there a **public contract** added or changed? Document it. Hyrum's law applies the moment it ships.
- Is **observability wired in** for the new path: log on entry/exit/error with relevant context, metric for success/failure rate, trace span if distributed?
- Is the feature **behind a flag** if the rollout strategy needs one? Is the default safe?
- Is **authorization checked** before any sensitive operation in the new path?
- For new persisted state: **migration plan**, indexes if needed, default value, nullability, schema-evolution story.

## Bug fix

The author is changing behavior to match the spec (or the spec to match reality). Risk concentrates in: regression, root cause vs symptom, and the test.

- Is there a **test that fails before the fix and passes after**? This is the single most important verification.
- Did the fix address the **root cause** or a symptom? A `try/except` that swallows the exception "fixes" nothing. A `if x is None` that papers over an upstream bug pushes the issue down the road.
- Are there **other places with the same bug** that the fix didn't touch? A bug in a duplicated piece of code usually has siblings.
- Does the fix **introduce a regression** elsewhere? Look at all callers of changed functions.
- Is the **commit message / PR description** explicit about what was wrong, why, and the user impact?

## Refactor

The author is changing structure without changing behavior. The *only* thing that should not change is behavior. Risk concentrates in: silent behavior changes, lost test coverage, and partial migrations.

- Is **behavior preserved**? The test suite passing is necessary but not sufficient. Read the diff for: changed return values, changed error types, changed default values, changed ordering, changed nullability.
- Are **tests still meaningful** after the refactor? A test that was testing a deleted internal helper now tests nothing.
- Is the refactor **complete**? Half-migrations leave two patterns living together — the worst of both worlds.
- Is the refactor **separated from behavior changes**? "Refactor and also fix the bug" is two PRs hiding as one. Insist they be split unless impossible.
- Is the refactor **mechanical** (rename, move, extract) or **judgment-based** (new abstraction)? Mechanical refactors are low-risk and high-value. Judgment-based refactors deserve a design conversation.

## Migration / data change

The author is changing persisted state shape, content, or location. This is the highest-risk category in most systems. Risk concentrates in: data loss, downtime, and irreversibility.

- Is the migration **reversible**? If not, what is the rollback plan? "Restore from backup" is a plan; "we'll figure it out" is not.
- Does the migration **lock the table**? For how long? Can it run online?
- Is the migration **safe under concurrent writes**? Adding a non-null column with a default to a heavily-written table can be a multi-hour outage.
- Is the migration **idempotent**? Can it be safely re-run if it fails partway?
- Is there a **backfill strategy** for existing data? Is it batched? Is it monitored?
- Has the **read path been deployed first** (dual-read), then the write, then the cleanup? Migrations that switch atomically are usually wrong.
- For destructive migrations (drop column, drop table, delete data): is there **evidence the data is unused** (logs, queries, time-bounded), not just a grep?
- Is there a **dry-run** or staging-environment validation step before production?

## Dependency bump / addition

The author is changing what runs in production by adding or moving someone else's code. Risk concentrates in: supply chain, unintended behavior changes, and license issues.

- For a **new dependency**: does it actually exist? (Hallucinated packages are real.) Is it well-maintained? When was it last updated? How many maintainers? What's the download count? Is it pinned to a specific version?
- Does the dependency **actually need to be added**, or is the functionality available in the standard library or an existing dependency?
- For a **version bump**: read the changelog. Are there breaking changes? Behavior changes? Security fixes?
- Are **lock files / hashes** updated correctly?
- Does the dependency introduce **new transitive dependencies** that should be audited?
- License compatibility?

## Infrastructure change (IaC, deployment config, CI)

Risk concentrates in: blast radius, irreversibility, and "this only manifests in prod."

- What **environments** does this change touch? Dev, staging, prod?
- Is the change **reversible** (can the previous state be restored from version control alone)?
- Are there **secrets or credentials** in the diff? Should there be an external secret reference?
- For network/firewall changes: what is the **principle of least privilege** justification?
- For CI changes: does the change affect what *runs* on every commit going forward? (E.g., disabling a test, weakening a security check.)
- For Terraform/Pulumi/etc: was a **plan** reviewed, not just the source?

## Security fix

Special category. Risk concentrates in: incomplete fix, regression of the fix in a future change, and disclosure.

- Does the fix **address the root cause** across all instances of the vulnerability, not just the reported one?
- Is there a **regression test** that exercises the previously-vulnerable input?
- Is the **PR description** appropriately discreet if the vulnerability is undisclosed? (Public CVE descriptions can wait.)
- Are there **upstream advisories** to check (CVE, GHSA) for related issues?
- Has the **on-call / security team** been informed?

## Performance work

Risk concentrates in: optimizing the wrong thing, regressing readability, and accidental correctness changes.

- Is there a **benchmark** showing the change actually helps? "I think this is faster" is not enough.
- Is the **target a real bottleneck** (profiled), not a guessed one? See `performance-engineering` skill.
- Did **correctness change** under the optimization? Does the test suite still pass with realistic, not just micro-benchmark, inputs?
- Is the **complexity cost** of the optimization worth the speedup? A 5% speedup that adds 100 lines of cache invalidation logic is a bad trade.
- Is the **benchmark methodology sound**: warmed-up, realistic data sizes, multiple runs, controlled environment?

## Revert

Risk concentrates in: incomplete revert and "why are we reverting?"

- Does the revert **fully undo** the original change, including any follow-up commits, migrations, or feature flags?
- Is the **reason for the revert documented**? A revert with no reason will be re-applied by someone else.
- Are there **dependent changes** that need to be reverted too?
- Is there a **plan** for the original change (re-roll forward, redesign, abandon)?
