# Reproducible and Hermetic Builds

Reference for the spectrum from "deterministic enough" to "bitwise reproducible," with honest assessment of what each tool actually achieves.

## Definitions

The Reproducible Builds project (`reproducible-builds.org/docs/definition/`) gives the rigorous definition:

> "A build is reproducible if given the same source code, build environment and build instructions, any party can recreate bit-by-bit identical copies of all specified artifacts."

**Hermetic** is a related but distinct property: a hermetic build is one whose output depends *only* on declared inputs, with no leakage from the host environment. A hermetic build is a precondition for a reproducible build, but not sufficient — you also need to eliminate sources of intra-build non-determinism.

Most teams need *deterministic* builds — same lockfile + same toolchain + same source = same artifact. Full bitwise reproducibility is a stronger property with a real cost.

## Why reproducibility matters

Four reasons, in increasing severity:

1. **Cache effectiveness.** Non-deterministic outputs invalidate build caches. A CI step that produces different bytes on every run is a cache that never hits. At scale this is the dominant performance issue.
2. **Auditability.** If you can't reproduce, you cannot prove the deployed artifact corresponds to the published source. This is a precondition for SLSA L3+ and for any meaningful supply-chain story.
3. **Trusting Trust.** Ken Thompson's 1984 ACM Turing Award lecture, *Reflections on Trusting Trust* (CACM 27.8, August 1984, pp. 761–763), demonstrated that a malicious compiler can insert backdoors into binaries with no source-level evidence. The defense — David A. Wheeler's *Diverse Double-Compiling* (2005) — requires reproducible builds across multiple compilers as a precondition.
4. **CVE response.** "Is the deployed artifact patched?" is an answerable question only if you can reproduce a known patched build and compare.

## Sources of non-determinism

The Reproducible Builds project tracks these. The recurring offenders:

- **Embedded timestamps** in archives, binaries, and build metadata. Set `SOURCE_DATE_EPOCH` and configure tools to honor it.
- **Filesystem iteration order.** `readdir` returns entries in inode order on most filesystems; that order varies across runs and machines. Sort.
- **Parallel-build ordering** in builds that emit per-task output to a shared file in interleaved order.
- **Embedded build paths.** A binary that records `/home/jeff/projects/foo/src/main.rs` is not reproducible across machines. Use relative paths, or path-mapping flags (Rust's `--remap-path-prefix`, GCC's `-fdebug-prefix-map`).
- **Locale and timezone.** Different `LANG`, `LC_ALL`, or `TZ` produces different sort order, date format, error message text.
- **Randomness.** Compression dictionaries that depend on `rand`, build-time UUIDs.
- **Compiler flag-dependent code generation.** LTO and PGO routinely produce different bytes on different machines, even with identical source.

The diagnostic tool is `diffoscope` (Reproducible Builds project), which compares two artifacts at the byte level and reports differences in a human-readable form.

## Hermetic build tools — honest assessment

### Bazel

Google, open-sourced March 2015. Rule-based; rules must declare all inputs. Linux sandboxing restricts filesystem visibility during rule execution.

**Honest assessment.** Hermetic with Remote Build Execution; locally, rules run in a sandbox that does not fully isolate from the host (the system Java, system Python, system C compiler can leak in). For full hermeticity, use `--config=remote` or a tightly-controlled toolchain registration.

Strengths: scales to very large polyglot codebases (Google's monorepo, Twitter, Stripe). Strong remote-cache story. Weaknesses: configuration cost is real; the learning curve is steep.

### Buck2

Meta, open-sourced April 2023. Rust rewrite of the original Buck. Designed for remote execution; local execution sandboxing is in active development.

**Honest assessment.** Strong story when run with remote execution; less hermetic than Bazel by default for local builds. The Rust core is fast.

### Pants v2

Active project (2.28 released September 2025). Python-friendly polyglot focus.

**Honest assessment.** Lower ceremony than Bazel; supports Python, Go, Java, Scala, Kotlin, Shell, Docker, increasingly TypeScript. Used by Coinbase, IBM, Slack, Salesforce. Hermeticity is partial; the goal is "deterministic enough for a Python-and-friends shop."

### Nix / Nixpkgs

The most rigorous reproducibility model in widespread use. Builds are mathematical functions of declared inputs, expressed in the Nix language. Every input is content-addressed.

**Honest assessment.** The bar-setter for reproducibility, but even Nix is not 100% bitwise reproducible across all packages — see the Reproducible Builds tracker. The Nix learning curve is famously steep; adoption decisions should weigh that cost.

### Cargo, Mix, uv

Strong dependency determinism via lockfiles, but not hermetic in Bazel's sense. The host toolchain (Rust version, Erlang version, Python version) still influences output. With careful toolchain pinning (`rust-toolchain.toml`, `.tool-versions`, etc.) builds are deterministic in practice for most teams.

### Nx and Turborepo

JavaScript monorepo tools. Caching is the primary focus; hermeticity is partial. Nx introduced task sandboxing that flags undeclared inputs/outputs. Turborepo completed a Rust rewrite from Go in 2024.

**Honest assessment.** Pragmatic — they make a JS monorepo fast without demanding a full hermetic-build commitment. The trade-off is that you don't get cross-machine bitwise reproducibility for free.

## Choosing

For most teams the right answer is:

1. Pin the toolchain (`rust-toolchain.toml`, `.tool-versions`, container base image with a specific tag).
2. Commit the lockfile.
3. Set `SOURCE_DATE_EPOCH` or equivalent to eliminate timestamps.
4. Use frozen-install commands in CI.
5. Periodically rebuild-and-compare with `diffoscope` to catch drift.

This delivers *deterministic* builds at modest cost. Most teams do not need full hermetic Bazel-style guarantees, and adopting them has real configuration tax that doesn't pay back unless the codebase is large and polyglot.

The teams that should adopt Bazel/Buck2/Nix:

- Polyglot monorepos at scale (Google, Meta, Twitter scale).
- High-assurance regulated environments (finance, defense, healthcare with bitwise auditability requirements).
- Teams pursuing SLSA L3+ provenance.

## What to flag in review

- Build outputs that include the build path (binaries that mention `/home/...`).
- Builds without `SOURCE_DATE_EPOCH` for archives or container images.
- CI that doesn't pin the toolchain version.
- Container builds that `apt-get install` or `pip install` without a pinned version manifest.
- A claim that builds are "hermetic" or "reproducible" without periodic rebuild-and-compare evidence.
- A move to Bazel/Buck2/Nix without a clear rationale for paying the configuration cost.

## Sources

- Reproducible Builds project. `https://reproducible-builds.org/`.
- Reproducible Builds, *Definitions*. `https://reproducible-builds.org/docs/definition/`.
- Ken Thompson, *Reflections on Trusting Trust*. ACM Turing lecture, CACM 27.8, August 1984. `https://dl.acm.org/doi/10.1145/358198.358210`.
- David A. Wheeler, *Fully Countering Trusting Trust through Diverse Double-Compiling* (2005, expanded 2009). `https://dwheeler.com/trusting-trust/`.
- Bazel docs on hermeticity. `https://bazel.build/basics/hermeticity`.
- `diffoscope` — Reproducible Builds. `https://diffoscope.org/`.
