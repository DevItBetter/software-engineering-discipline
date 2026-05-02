# Lockfiles and Resolvers

Reference for the semantics of manifests, lockfiles, and dependency resolvers across ecosystems. The discipline is the same everywhere; the file names and tool flags vary.

## Manifest vs. lockfile, precisely

A **manifest** is the human-edited declaration of intent: "I depend on these packages with these constraints."

A **lockfile** is the resolver's record of what those constraints resolved to last time, including transitive dependencies, exact versions, and content hashes.

The discipline:

- Edit the manifest deliberately.
- Regenerate the lockfile with each manifest change (`npm install`, `cargo update <pkg>`, `poetry lock`, etc.).
- Commit both.
- In CI and production, install in **frozen mode** that uses the lockfile as the source of truth and fails if the manifest disagrees with it.

## Frozen-install commands by ecosystem

| Ecosystem | Frozen install |
|---|---|
| npm | `npm ci` |
| yarn (Berry / v2+) | `yarn install --immutable` |
| pnpm | `pnpm install --frozen-lockfile` |
| Poetry | `poetry check --lock && poetry install` |
| uv | `uv sync --frozen` |
| PDM | `pdm install --frozen-lockfile` |
| Pipenv | `pipenv install --deploy` |
| pip + hashes | `pip install --require-hashes -r requirements.txt` |
| Bundler | `bundle install --frozen` (or `--deployment`) |
| Cargo | `cargo build --locked` |
| Go | `go mod download` (verifies against `go.sum`) |
| Maven | dependency-locking plugin or enforce-plugin |
| Gradle | enable lockfiles in `build.gradle.kts`, then `./gradlew --offline` against locked versions |
| .NET | `dotnet restore --locked-mode` |

These commands are the contract: if the lockfile is missing, stale, or disagrees with the manifest, the install fails. Use them in CI without exception.

## Library vs. application

For applications (anything you deploy), commit the lockfile. Always.

For libraries published to a registry, the picture is contested:

- The lockfile **does not constrain consumers**. Consumers re-resolve when they install your library; their resolver picks versions that satisfy *their* constraint set, not yours.
- Committing the lockfile gives **CI determinism** — your tests run against a known graph rather than whatever resolved today.
- Some teams argue the lockfile shouldn't be committed for libraries because it can mislead contributors into thinking it constrains anything beyond CI. This is a real concern.

The defensible default: commit the lockfile for libraries, document explicitly that it is for CI determinism, and never assume it constrains downstream.

## Resolver behavior across ecosystems

### npm Arborist (npm 7+)

Builds a logical graph; deduplicates where possible; nests conflicting copies under their requesters. Tolerates diamond dependencies via duplication, which is why a `node_modules` tree can have multiple copies of the same package at different versions. Arborist's distinguishing feature is treating `peerDependencies` as part of a single SAT-style conflict-detection step, which is what makes peer-dep errors more honest than they were in npm 6 — and the most common source of "unsolvable graph" complaints from teams migrating up.

Strength: rarely produces unsolvable graphs. Weakness: can produce surprisingly large `node_modules` and exotic edge cases when constraints are tight.

### Maven nearest-definition-first

Maven's resolver picks the closest declaration in the dependency tree: if your project depends on A which depends on `Lib 2.0`, and also on B which depends on `Lib 1.5`, the version closer to your project wins. Ties between equally-near declarations are resolved by **order of declaration in the POM** — first wins. Fast and predictable in a narrow sense; surprising when transitive constraints disagree, and the tie-break-by-order is usually the source of real-world surprise.

Practical implication: pin direct dependencies' versions of contested libraries explicitly in your project's `dependencyManagement` to avoid being surprised by nearest-definition resolution.

### Cargo backtracking

Cargo's resolver backtracks across the constraint set, preferring the highest version that satisfies all constraints. Permits multiple major-version copies of a crate to coexist (because Rust's name-mangling makes them distinct types). Strong determinism: same `Cargo.toml` plus same `Cargo.lock` always produces the same graph.

Strength: rarely surprising. Weakness: backtracking is exponential in the worst case, but Cargo's resolver has improved (PubGrub-style algorithms, since around 2020); exponential cases are now rare in practice.

### pip (modern, post-2020 resolvelib)

pip's resolver since the resolvelib rewrite uses backtracking with conflict learning. It can hit "resolution-too-deep" errors on dense graphs. The newer toolchain (uv, Poetry, PDM) all do better.

### Go MVS — Minimum Version Selection

Russ Cox's deliberately-different design. Pick the *lowest* version that satisfies all constraints (rather than the highest). Unusually deterministic: as long as no maintainer pulls a version, the resolution is stable across time. Trades aggressiveness (you don't get new features automatically) for predictability (your dependency graph doesn't change underneath you).

Worth understanding even if you don't use Go: MVS is a credible alternative to the "always take the latest compatible" default that npm, Cargo, and pip share.

## SemVer and the gap to reality

Semantic Versioning (`semver.org` 2.0.0):

- MAJOR for incompatible API changes.
- MINOR for backward-compatible additions.
- PATCH for backward-compatible fixes.

In practice, MINOR and PATCH releases regularly break consumers. Raemaekers et al.'s empirical study of the Maven ecosystem (*Semantic Versioning versus Breaking Changes*, SCAM 2014, expanded EMSE 2017) found that roughly one-third of releases introduce at least one breaking change, and 14.5% of releases violated SemVer by introducing breaking changes in minor or patch versions. Hyrum's Law applies more broadly: with enough users, every observable behavior is depended on by somebody — the Maven study is one symptom of that, not a proof of it.

The implication is that SemVer is a *contract the upstream maintainer hopes to honor*, not a guarantee. Lockfiles exist because SemVer is not enough. Caret/tilde constraints (`^1.2.0`, `~2.0`) plus a committed lockfile is the working middle ground:

- The constraint expresses your tolerance band — you'll accept any future minor or patch in this band.
- The lockfile pins the exact resolution at a given commit so today's CI matches today's prod.
- Updating the lockfile is a deliberate act with a PR and tests, not an accident at install time.

## When pinned-exact is wrong

A pinned-exact constraint (`"lodash": "4.17.21"` rather than `"^4.17.21"`) seems safer. It's not — it locks you out of patch releases including security fixes. The dependency receives `4.17.22` with a CVE patch; you're stuck on `4.17.21` until someone manually updates. The lockfile already records the exact resolution; the manifest's constraint should express the *band you accept future updates within*.

The exception: pinned-exact is appropriate when you have an active incompatibility (a known bug in `4.17.22` that affects you) and you've documented it. Pin with a comment saying why and when to revisit.

## When `*` or `latest` is wrong

Always.

A `*` or `latest` constraint means the resolver picks anything. A future major release with breaking changes can land in your tree without notice. Production code with `*` constraints is one upstream maintainer decision away from a regression.

`*` is occasionally defensible for dev-dependencies you don't deploy (`@types/*` packages, lint configs) where breakage hits CI rather than production. Even there, prefer a tilde constraint.

## What to flag in review

- A manifest change without a lockfile change.
- A pinned-exact constraint on a package with active CVE patches.
- `*`, `latest`, or unbounded constraints in production code.
- An install command that doesn't use frozen mode in CI (`npm install` instead of `npm ci`).
- A library project that doesn't commit its lockfile *and* doesn't document that decision.
- A new dependency whose resolver-chosen transitive deps doubled `node_modules` size — has anyone looked at what was added?
- A "we don't have a lockfile" workflow at all.

## Sources

- Cargo Book on Dependency Resolution. `https://doc.rust-lang.org/cargo/reference/resolver.html`.
- pip's Dependency Resolution docs. `https://pip.pypa.io/en/stable/topics/dependency-resolution/`.
- Russ Cox on Go MVS. `https://research.swtch.com/vgo-mvs`.
- Hyrum's Law. `https://www.hyrumslaw.com/`.
- Hynek Schlawack, *Semantic Versioning Will Not Save You*. `https://hynek.me/articles/semver-will-not-save-you/`.
- Snyk, *Why npm lockfiles can be a security blindspot*. `https://snyk.io/blog/why-npm-lockfiles-can-be-a-security-blindspot-for-injecting-malicious-modules/`.
- Semantic Versioning 2.0.0. `https://semver.org/`.
