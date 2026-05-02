---
name: build-and-dependencies
description: "Build systems, dependency management, lockfiles, reproducibility, and supply-chain hygiene. Use this skill whenever the task involves adding or upgrading a dependency, choosing a package manager or build system, evaluating a manifest/lockfile policy, judging supply-chain risk, designing CI dependency-scan or SBOM generation, deciding monorepo vs. multi-repo build strategy, or asking \"is this dependency safe / pinned / reproducible.\" Use it when reviewing changes to manifests, lockfiles, build configuration, or CI pipelines that fetch dependencies. Built on the lessons of Log4Shell (CVE-2021-44228), the xz-utils backdoor (CVE-2024-3094), Alex Birsan's dependency confusion, the npm left-pad incident, Ken Thompson's *Reflections on Trusting Trust*, the Reproducible Builds project, SLSA, Sigstore, and SBOM standards (SPDX, CycloneDX)."

---

# Build and Dependencies

Every direct dependency owes the project a license check, a security review, a maintenance assumption, and an audit-log entry. Every transitive dependency is something you depend on without choosing. The discipline of build and dependency management is acknowledging this honestly: dependencies are debt that accrues silently, and building software reproducibly is the precondition for any serious supply-chain story.

The lessons are recent and concrete:

- **Log4Shell** (CVE-2021-44228, public 9 December 2021) — a remote-code-execution vulnerability in Apache Log4j 2's JNDI lookup feature, which had existed since 2013, that affected most of the JVM-based internet. Most affected teams did not know they used Log4j; it was a transitive dependency of a transitive dependency.
- **xz-utils backdoor** (CVE-2024-3094, discovered 29 March 2024) — a multi-year social-engineering campaign by a contributor using the pseudonym "Jia Tan" inserted a backdoor into `xz`/`liblzma` 5.6.0 and 5.6.1, hidden in test fixtures and triggered only during specific Debian/RPM build contexts. Discovered by Andres Freund (Microsoft / PostgreSQL) while investigating a 500ms vs. 100ms SSH login regression on Debian sid. The attacker built trust over years before the payload landed.
- **Dependency confusion** (Alex Birsan, *Dependency Confusion*, 9 February 2021) — internal package names that don't exist on the public registry can be claimed by an attacker; if the build tool prefers the public registry, the attacker's code runs with the build's privileges. Birsan breached 35 companies including Apple, Microsoft, PayPal, Shopify, Netflix, Yelp, Tesla, and Uber.
- **left-pad** (March 2016) — Azer Koçulu unpublished 273 npm packages including `left-pad`, an 11-line string-padding utility, after a dispute with npm Inc. The unpublishing broke installs at Babel, Webpack, Meta, PayPal, Netflix, Spotify, and Kik. The lesson: a single 11-line dependency was a single point of failure for half the JS ecosystem.

The discipline that follows is the response to these.

## Manifests vs. lockfiles

A **manifest** declares *intent*: "I depend on `lodash ^4.17`" — a constraint, expressing the consumer's tolerance band. A **lockfile** records *resolution*: "Last time we resolved, `lodash` was exactly 4.17.21 at sha512:abc..., and its transitive `inherits` was 2.0.4." The manifest is reviewed by humans; the lockfile is reviewed by machines.

**The cardinal rule: commit the lockfile.**

This is universal advice for **applications** — anything you deploy. For **libraries** published to a registry, the picture is contested. The lockfile doesn't constrain consumers (they re-resolve on install), but committing it gives CI a deterministic baseline for your tests and contributor workflows. The honest summary: always commit for applications; for libraries, commit for CI determinism but never assume it constrains or predicts downstream consumers' full graphs.

By ecosystem (early 2026):

| Language | Manifest | Lockfile | Frozen-install command |
|---|---|---|---|
| npm / yarn / pnpm | `package.json` | `package-lock.json` / `yarn.lock` / `pnpm-lock.yaml` | `npm ci` / `yarn install --immutable` / `pnpm install --frozen-lockfile` |
| Python (modern) | `pyproject.toml` | `poetry.lock` / `uv.lock` / `pdm.lock` / `Pipfile.lock` | `poetry check --lock && poetry install` / `uv sync --frozen` / `pdm install --frozen-lockfile` |
| Python (legacy) | `requirements.txt` | `requirements.txt --hash` (manual) | `pip install --require-hashes -r requirements.txt` |
| Ruby | `Gemfile` | `Gemfile.lock` | `bundle install --frozen` |
| Go | `go.mod` | `go.sum` (content hashes) | `go test -mod=readonly ./...` or clean-tree check after `go mod download` |
| Rust | `Cargo.toml` | `Cargo.lock` | `cargo build --locked` |
| JVM (Gradle) | `build.gradle[.kts]` | `dependencies.lockfile` (opt-in) | enable dependency locking; CI runs builds without `--write-locks` |
| .NET | `*.csproj` | `packages.lock.json` (opt-in via `RestorePackagesWithLockFile`) | `dotnet restore --locked-mode` |

A note on Python: `requirements.txt` is **not a lockfile by default**. It becomes one only when generated with hashes (`pip-compile --generate-hashes`) and installed with `--require-hashes`. The modern Python toolchain (uv, Poetry, PDM, Pipenv) all produce real lockfiles; prefer them for new projects.

## Version constraints and resolver behavior

**Semantic versioning** (`semver.org` 2.0.0): increment MAJOR for incompatible API changes, MINOR for backward-compatible additions, PATCH for backward-compatible fixes. Format `MAJOR.MINOR.PATCH` with optional pre-release / build metadata.

**Hyrum's Law in practice.** SemVer is aspirational. In real codebases, minor and patch releases regularly break consumers — Raemaekers et al.'s empirical study of the Maven ecosystem (*Semantic Versioning versus Breaking Changes: A Study of the Maven Repository*, SCAM 2014, expanded EMSE 2017) found that roughly **one-third of releases introduce at least one breaking change**, and **14.5% of releases violated SemVer** by introducing breaking changes in minor or patch versions. The practical implication: SemVer is a contract the upstream maintainer hopes to keep, not a guarantee the consumer can rely on. Lockfiles exist because SemVer is not enough.

**The resolver problem.** Dependency resolution under SemVer constraints is in general NP-hard (Boolean SAT-equivalent). Real package managers approximate:

- **npm Arborist** — builds a logical graph, deduplicates where possible, nests conflicting copies under the requester. Tolerates diamond dependencies via duplication.
- **Maven** — "nearest definition first": the closest declaration in the dependency tree wins; ties between equally-near declarations are resolved by order of declaration in the POM. Fast but surprising — the tie-break by declaration order is usually the source of conflict-resolution surprise in real projects.
- **Cargo** — backtracking, prefers highest compatible. Permits multiple major-version copies because Rust's name-mangling allows it. Strong determinism.
- **pip** (modern, post-2020 resolvelib) — backtracking, can hit exponential worst cases ("resolution-too-deep").
- **Go modules** — Minimum Version Selection (MVS): pick the *lowest* version that satisfies all constraints. Unusually deterministic; trades aggressiveness for predictability.

**The constraint trade-off.** Over-tight constraints (pinned exact) prevent you from getting fixes; security patches stall. Over-loose constraints (`*`, `latest`) let the resolver pick anything, including something that broke compatibility at a "minor" or "patch" version. The healthy middle is caret/tilde range plus a committed lockfile — manifest expresses intent, lockfile pins resolution, lockfile is updated deliberately.

## Reproducible and hermetic builds

The Reproducible Builds project (`reproducible-builds.org`) defines reproducibility precisely:

> "A build is reproducible if given the same source code, build environment and build instructions, any party can recreate bit-by-bit identical copies of all specified artifacts."

Reproducibility is a foundation for serious supply-chain auditability:

1. **Auditability.** If you can't reproduce, you can't independently verify that the published artifact corresponds to the published source. SLSA Build L3 does not require bitwise reproducibility, but reproducible builds make SLSA provenance far more useful to consumers.
2. **Trusting Trust.** Ken Thompson's 1984 ACM Turing Award lecture, *Reflections on Trusting Trust* (CACM 27.8, August 1984), demonstrated that a malicious compiler can insert backdoors into binaries with no source-level evidence. The defense — David A. Wheeler's *Diverse Double-Compiling* (2005) — requires reproducible builds as a precondition.
3. **Cache effectiveness.** Non-deterministic outputs poison build caches and balloon CI cost.
4. **Security baselines.** Bit-identical means CVE patches can be verified against an exact prior state.

**Hermetic** is related but distinct: a hermetic build is one whose result depends *only* on declared inputs, with no leakage from the host environment (system locale, installed compilers, ambient env vars, network). Hermeticity makes reproducibility easier, but it does not by itself guarantee bit-identical outputs.

**Tools, with honest assessment.**

- **Bazel** (Google, open-sourced March 2015) — rule-based, hermetic in the sense that rules must declare all inputs; sandboxing on Linux. Hermetic guarantees are strongest with Remote Build Execution; local builds are sandboxed but not fully isolated from the host. Configuration cost is real.
- **Buck2** (Meta, open-sourced April 2023) — Rust rewrite of the original Buck. Hermetic with Remote Execution; local execution sandboxing is in progress.
- **Pants v2** (active) — Python-and-friends focus; lower ceremony than Bazel.
- **Nx / Turborepo** — JavaScript monorepo tools. Caching is the focus; hermeticity is partial.
- **Nix / Nixpkgs** — the most rigorous reproducibility model in widespread use; builds are functions of declared inputs. Even Nix is not 100% bitwise reproducible across all packages, but it sets the bar.
- **Cargo, Mix, uv** — strong dependency determinism via lockfiles, but not hermetic in Bazel's sense; the host toolchain still influences output.

**Sources of non-determinism** (the Reproducible Builds project tracks these): embedded timestamps in archives, filesystem iteration order (`readdir`), parallel-build ordering, build paths embedded in binaries, locale and timezone, randomness in compression dictionaries, and compiler-flag-dependent code generation (LTO, PGO).

The honest bottom line: most teams should aim for *deterministic* builds (same lockfile + same toolchain + same source = same artifact) rather than full bitwise reproducibility, which has a real configuration cost.

## Supply chain attacks and defenses

### The taxonomy

- **Typosquatting.** Malicious package whose name resembles a popular one (`requests` vs. `requesst`). Always present at scale on npm and PyPI. Defense: organizational allowlist or private registry proxy.
- **Dependency confusion.** Birsan's 2021 attack: an internal package name with no public-registry counterpart can be claimed by an attacker; many resolvers default to public-registry priority, fetching the attacker's code. Defense: scope/namespace internal packages; configure resolvers to refuse public fallback for private names.
- **Maintainer compromise.** The original maintainer's account is hijacked or transferred; a malicious update is published to legitimate downloads.
  - *event-stream* (November 2018): Dominic Tarr granted publishing rights to a contributor (`right9ctrl`) who added a dependency (`flatmap-stream`) that targeted the Copay Bitcoin wallet. Active for ~2.5 months.
  - *ua-parser-js* (October 2021): account hijacked; malicious 0.7.29 / 0.8.0 / 1.0.0 published to ~8M weekly downloads, payload was a Monero miner plus credential stealer.
  - *xz-utils* (March 2024): the most sophisticated case. Multi-year social engineering by "Jia Tan," sock-puppet pressure on the original maintainer, payload camouflaged in test fixtures.

### Defenses

- **SLSA** (Supply-chain Levels for Software Artifacts; `slsa.dev`). Introduced by Google in June 2021, now an OpenSSF project. SLSA v1.0 (April 2023) defines a Build track with three levels (older v0.1 had four; the v1.0 restructure split L4 into separate future tracks):
  - **L1**: automated build with provenance generated.
  - **L2**: hosted build platform with signed provenance.
  - **L3**: hardened build platform; provenance non-forgeable; isolated builds.
  When citing SLSA levels, anchor to v1.0 — older articles citing "L1–L4" predate the restructure.
- **Sigstore** (`sigstore.dev`). OpenSSF graduated project (graduated 2024). Sign and verify artifacts:
  - `cosign` for signing containers and binaries.
  - `fulcio` issues short-lived certs bound to OIDC identities.
  - `rekor` is the append-only transparency log.
  - `gitsign` signs git commits with sigstore identities.
- **SBOM** (Software Bill of Materials). Two competing formats:
  - **SPDX** (Linux Foundation; standardized as ISO/IEC 5962:2021).
  - **CycloneDX** (OWASP; ECMA-424).
  Either is fine; pick by what your downstream consumers expect. The point is that you can answer "what is in this artifact?" precisely.

US Executive Order 14028 (May 2021) catalyzed federal SBOM guidance and procurement expectations, helping make SBOMs standard practice across the industry. The federal procurement landscape around SBOMs continues to shift; check current agency, OMB, and CISA requirements before claiming a binding procurement mandate.

### Practical defenses

- **Dependency scanning in CI.** OSV-Scanner (Google/OpenSSF), Trivy (Aqua), Grype (Anchore), Snyk, GitHub Advanced Security. Pick one and actually act on its findings.
- **Patch management SLAs by severity.** Critical CVE on a hot-path dependency: same-week patch. High: two-week. Routine non-security updates: monthly cadence at minimum.
- **Renovate / Dependabot.** Automation produces PRs; the discipline is *merging them*. A repo with 80 stale Dependabot PRs is worse than one with no automation, because the team has trained itself to ignore the signal.
- **Provenance.** Generate an SBOM at build time (Syft, Trivy, CycloneDX gradle/maven plugins). Sign artifacts with cosign. Attach SLSA provenance.
- **Signed commits.** `gitsign` or GPG-signed commits are weaker than artifact signing but raise the bar against impersonation.

## Container image hygiene

Container images are a special case of dependency management — every image is the transitive closure of a base image, OS packages, language runtime, and your application. The same supply-chain principles apply, with extra discipline:

- **Pin base images by digest, not tag.** `FROM ubuntu:22.04` floats; `FROM ubuntu@sha256:...` is reproducible. Tag-pinning is acceptable only when paired with a private registry that snapshots the resolved digest.
- **Prefer minimal base images.** Distroless (Google), Wolfi / Chainguard images, Alpine for size-sensitive cases, `scratch` for static binaries. Each line removed from the base image is one fewer CVE you inherit.
- **Multi-stage builds.** Compile in one stage with full toolchain; copy only the artifact into a slim runtime stage. The final image has no compiler, no package manager, no shell where unnecessary.
- **Drop privileges.** `USER` directive to a non-root user; drop Linux capabilities; read-only root filesystem where the app permits it.
- **`.dockerignore` matters.** A missing `.dockerignore` ships your `.git`, your `node_modules`, your `.env` files, and your build cache into the image. Audit what `COPY .` actually copies.
- **Sign images.** `cosign` against the image registry; verify signatures at deploy time. Attach and verify provenance/SBOM attestations (SLSA/in-toto style) if your platform supports it.
- **SBOM per image.** Syft or Trivy produces an SBOM at build time; attach to the image as an attestation.
- **Scan at build, scan periodically post-build.** A new CVE for a base-image package can land months after your image was built. Regular scans of in-production images catch this.

The mental model: an image is a frozen snapshot of a dependency graph. The disciplines for graphs apply (lockfiles, integrity verification, owner per layer); the additional disciplines exist because images are also a runtime environment with its own attack surface.

## Toolchain pinning

The lockfile pins your dependencies; the toolchain pinning pins the *resolver* and *compiler* that produced those dependencies' build outputs. Without it, your build is deterministic only on machines that happen to have the same toolchain version.

By ecosystem (early 2026):

- **Rust**: `rust-toolchain.toml` at the repo root pins compiler version, components, profile.
- **Node**: `.nvmrc`, `.node-version`, `package.json` `engines` field, or Volta's `volta` field. `corepack` (built into Node 16.10+) pins package-manager version.
- **Python**: `.python-version` (pyenv), `tool-versions` (asdf / mise), or specifying the interpreter version in `pyproject.toml` for tools that read it.
- **Cross-language**: `mise` (formerly `rtx`), `asdf`, devcontainers, Nix flakes — declare every tool's version in one file.
- **Container builds**: pin the base image by digest; the base image carries the system toolchain.

A build that runs `apt-get install build-essential` without a pinned base-image digest is not reproducible. Treat the toolchain as a dependency in its own right.

## Vendoring vs. registry vs. submodule

- **Vendoring** — copy dependencies into your repo (Go's `vendor/`, Cargo vendor, `npm pack`-bundled deps). Pros: build is offline-capable; supply-chain attack surface frozen at vendor-update time; historical builds remain buildable. Cons: large repo, manual cherry-picking of upstream fixes, license-audit overhead.
- **Registry-based** — the default. Lockfile is the trust anchor. Cheap and fast; depends on registry availability and integrity at install time.
- **Git submodules** — a constrained form of vendoring. Notoriously fiddly: forgotten `git submodule update --init --recursive`, detached-HEAD confusion, contributors who don't realize they exist. Rarely the right answer in 2026; prefer subtree, monorepo, or proper vendoring.

The honest pattern for most teams: registry + lockfile + dependency scanning + automated bot-driven updates that someone actually merges. Vendoring is justified for high-assurance, regulated, or air-gapped environments, and for small forks of unmaintained dependencies.

## Build system architecture

The shape of any modern build system: a declared dependency graph between targets; inputs content-hashed (not timestamped); outputs addressed by hash of inputs; cache lookups (local and remote) before execution; parallel scheduling across the graph; optional sandboxing for hermeticity.

**Make**'s strengths: ubiquitous, simple mental model. Weaknesses: timestamp-based not content-hash; clock-skew and `touch`-based invalidations are fragile; no cross-machine cache; recursive Make spreads the graph across files (Peter Miller, *Recursive Make Considered Harmful*, 1998).

**Modern build systems** (early 2026 status):

| Tool | Domain | Strength |
|---|---|---|
| Bazel | polyglot, large monorepos | rule-based, hermetic-with-RBE |
| Buck2 | polyglot, Meta-style | Rust, designed for remote execution |
| Pants v2 | Python-friendly polyglot | lower ceremony than Bazel |
| Nx | JS/TS monorepos | rich plugin ecosystem |
| Turborepo | JS monorepos | lightweight; Rust rewrite landed 2024 |
| Gradle | JVM | incremental, configuration cache |
| Cargo | Rust | exemplary single-language toolchain |
| uv | Python (Astral, 2024–) | the emerging modern Python default |

**Caching is the lever.** Local cache makes individual developers fast; *remote cache* (Bazel Remote Build Execution, Nx Cloud, Turborepo Remote Cache, Earthly) makes the team fast. Remote-cache poisoning is a real attack surface — verify cache isolation and authentication.

## Dependency policy and hygiene

**The "do you really need it" question.** Every direct dependency carries cost. Most teams underweight that cost — the canonical reminders are the left-pad incident and the long tail of micro-dependencies any large npm tree contains. Every direct dependency owes:

- A license check (compatible with your license? viral?).
- A security review (recent CVEs? maintained? sole-maintainer risk?).
- A long-term maintenance assumption.
- An audit-log entry — someone on the team owns understanding why this is a dependency.
- Bytes on disk and in your build cache.

Most teams underweight this. The left-pad incident is the canonical reminder that an 11-line dependency is still a dependency.

**Minimal sufficient set.** Direct dependencies should be (a) clearly load-bearing, (b) actively maintained, (c) known to a human on the team, (d) reviewed at adoption.

**Vendored fork.** When a small dependency is unmaintained but essential: fork, vendor, document why, accept ownership. Healthier than pretending an abandoned package will return.

**Updating discipline.**

- **Critical CVE on hot-path dep:** patch within one week.
- **High CVE:** two weeks.
- **Routine non-security updates:** monthly cadence at minimum.
- **Major-version bumps:** scheduled, behind a feature flag if necessary, never on a Friday.

## CI integration

- **Dependency scanning** as a pipeline stage. OSV-Scanner, Trivy, Snyk, GHAS — pick one, fail builds on critical, ticket high.
- **Lockfile drift detection.** `npm ci`, `pnpm install --frozen-lockfile`, `yarn install --immutable`, `cargo --locked`, `pip install --require-hashes` — CI fails if a manifest changed without regenerating the lockfile.
- **Reproducibility checks.** Periodic rebuild-and-compare in CI (using `diffoscope` from the Reproducible Builds project) catches non-determinism early.
- **SBOM generation** as a build step. Sign artifacts with cosign. Attach SLSA provenance via slsa-github-generator or equivalent.
- **Privilege isolation.** Split build into a "fetch deps" stage with no secrets and an "execute" stage with the secrets it needs. A compromised dep in the fetch stage cannot exfiltrate `GITHUB_TOKEN` or `AWS_ROLE` from the runner because they aren't in scope.

## Common antipatterns

- **No lockfile, or lockfile not committed.** Determinism gone; new contributor gets a different graph.
- **"Works on my machine."** Different versions resolved locally vs. CI vs. prod; symptom of the above.
- **`latest` tags or unbounded constraints in production.** A future minor or patch can break you; you have no record of what you were running.
- **Direct download from random URLs without integrity.** `curl | bash`, unsigned tarballs, GitHub releases with no checksum.
- **`--force` / `--ignore-engines` / `npm i --force` to make CI green.** Converts a real signal (incompatible peer or version conflict) into hidden risk.
- **Auto-merging Dependabot PRs without test gating.** Inverts the security model: the bot now runs new code in production with no review.
- **200 transitive dependencies for one utility function.** The left-pad lesson; also `is-odd` and friends.
- **Maintenance debt: 3+ years stale, known CVEs.** Each year of staleness compounds the upgrade cost.
- **"We forked it" without a written reason or upstream-diff.** Future maintainers cannot tell what was changed or whether to upstream.
- **Build pulls from public registry inside a CI runner with prod secrets in the environment.** The runner becomes the attack target.
- **Build that requires network and breaks offline.** Vendoring, prefetch caches, or registry mirrors fix this.
- **Trusting "verified publisher" badges as a security control.** Both npm and PyPI have lost maintainer accounts (event-stream, ua-parser-js); the badge means "same person we verified once," not "this code is safe today."

## What to flag in review

- A new direct dependency with no rationale, owner, or license check.
- A manifest change without a corresponding lockfile update.
- A pinned-exact constraint on a dependency receiving security patches (locks you out of the patches).
- A `*` or `latest` constraint in production code.
- A new CI step that runs network-fetched scripts (`curl | bash`).
- A build step that pulls from the public registry while production secrets are in scope.
- A Dependabot/Renovate PR auto-merged without tests passing.
- A long-stale dependency with known CVEs.
- A "vendored fork" without a README explaining what was changed and why.
- A build that depends on the developer's local toolchain (Java version, glibc version, etc.) without declaring it.
- An SBOM- or provenance-claiming process that produces nothing parseable downstream.
- A "verified publisher" or "trusted source" badge cited as the security control.

## Reference library

- `references/lockfiles-and-resolvers.md` — manifest/lockfile semantics across ecosystems; resolver algorithms (Arborist, Maven nearest-wins, Cargo backtracking, Go MVS); when to commit a lockfile for a library.
- `references/supply-chain-attacks-and-defenses.md` — the Log4Shell / xz-utils / dependency-confusion / event-stream taxonomy; SLSA, Sigstore, and SBOM in operational depth.
- `references/reproducible-and-hermetic-builds.md` — Reproducible Builds project, sources of non-determinism, Bazel/Buck2/Nix in honest comparison, the *Trusting Trust* defense.

## Sibling skills

- `secure-coding-fundamentals` — OWASP Top 10:2025 A03 (Software Supply Chain Failures) is the policy view of the same territory; this skill is the operational view.
- `deployment-and-release-engineering` — hermetic builds and signed artifacts feed into deployment integrity; SBOM and SLSA provenance are release-engineering artifacts.
- `version-control-discipline` — committed lockfiles, atomic commits including manifest+lockfile changes, monorepo build-system trade-offs.
- `engineering-discipline` — the orchestrator; new dependencies are a review category in their own right.
- `systems-architecture` — monorepo vs. multi-repo as a system-design decision, not just a tooling one.
