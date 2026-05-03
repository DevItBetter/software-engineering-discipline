# Supply Chain Attacks and Defenses

Reference for the operational discipline behind defending the supply chain. The lessons are recent and specific; cite them by name.

## The incident library

These are the canonical examples. They are useful precisely because they are concrete — a senior engineer reading them knows exactly what failure mode is being described.

### Log4Shell — CVE-2021-44228 (December 2021)

A remote-code-execution vulnerability in Apache Log4j 2's JNDI lookup feature, present in the codebase since 2013. Privately disclosed by Chen Zhaojun (Alibaba Cloud) on 24 November 2021; exploit went public 9 December 2021; CVE assigned 10 December 2021.

The lesson is the **transitive-dependency depth**. Most affected teams did not directly depend on Log4j. They depended on a framework that depended on a library that depended on Log4j. SBOMs and dependency scanning would have reduced the manual hunt and identified many affected artifacts, but exposure still requires validation against packaged/runtime contents, shaded dependencies, vulnerable configuration, and exploitability.

### xz-utils backdoor — CVE-2024-3094 (March 2024)

A multi-year social engineering campaign by a contributor using the pseudonym "Jia Tan" (`JiaT75`). Timeline:

- 29 October 2021: first contribution.
- 21 January 2022: first authored commit.
- Mid-2022: co-maintainership granted to Jia Tan after a sock-puppet pressure campaign on the `xz-devel` mailing list using accounts "Jigar Kumar," "Dennis Ens," and others, all asking why the original maintainer was so slow.
- 24 February 2024: `xz` 5.6.0 released with malicious release-tarball content hidden in test fixtures.
- 9 March 2024: 5.6.1 released with the backdoor still present and exploitability/detection details changed.
- 29 March 2024: discovered by Andres Freund (Microsoft / PostgreSQL contributor) while investigating a 500ms vs. 100ms SSH login regression on Debian sid.

The 5.6.0 and 5.6.1 release tarballs were affected; the payload triggered only during specific Debian/RPM build contexts. The lessons:

1. **Sole-maintainer projects are a supply-chain risk.** The original `xz` maintainer was burned out; the social-engineering campaign exploited that. Critical infrastructure depending on overworked individuals is a structural problem.
2. **Test fixtures are code that runs.** Hiding the payload in `.test` files passed casual review.
3. **Build-context-conditional behavior is hard to detect.** The backdoor only activated in specific package-build contexts; manual review of the source would have missed it.
4. **The detection was accidental.** A 400ms login regression. Discovery is not a defense plan.

### Dependency confusion — Alex Birsan (February 2021)

*Dependency Confusion: How I Hacked Into Apple, Microsoft and Dozens of Other Companies* (Medium, 9 February 2021). The mechanic: an internal package name with no public-registry counterpart can be claimed by an attacker; many resolvers default to public-registry priority, so the attacker's package is fetched into the build with the build's privileges.

Birsan demonstrated dependency confusion against 35 organizations (Apple, Microsoft, PayPal, Shopify, Netflix, Yelp, Tesla, Uber, etc.) through responsible disclosure, showing code execution and data-exfiltration paths in internal build environments and earning over $130k in bug bounties. Defense:

- Scope or namespace internal packages (`@mycompany/foo` rather than `foo`).
- Configure resolvers to refuse public-registry fallback for private names. npm: scoped internal package names plus `.npmrc` scope-to-registry mappings and private-registry enforcement; pip: `--index-url` plus explicit `--extra-index-url` policy.
- Maintain a private registry proxy that mediates all package fetches.

### event-stream (November 2018)

The original maintainer (Dominic Tarr) granted publishing rights to a contributor calling themselves `right9ctrl`, who added `flatmap-stream` as a dependency on 9 September 2018 in event-stream 3.3.6. The payload targeted the Copay Bitcoin wallet (only in their build, by detecting "description" strings). Active for ~2.5 months before discovery.

Lesson: **maintainer transfer is a supply-chain event**. A package's identity is not the npm name — it is the chain of trust over time. A new maintainer with no prior context is a new threat surface.

### ua-parser-js (October 2021)

Maintainer's npm account was hijacked. Malicious 0.7.29, 0.8.0, 1.0.0 published 22 October 2021 to ~8 million weekly downloads. Payload: XMRig Monero miner plus Windows credential stealer. Cleaned up to 0.7.30 / 0.8.1 / 1.0.1.

Lesson: **2FA on registry accounts is non-optional**. npm now mandates 2FA for high-impact packages. Verify your team's accounts are protected.

### left-pad (March 2016)

Azer Koçulu unpublished 273 npm packages including `left-pad` (an 11-line string-padding utility) on 22 March 2016 after a dispute with npm Inc. over the `kik` package name. The unpublishing broke installs at Babel, Webpack, Meta, PayPal, Netflix, Spotify, and Kik itself. npm restored 0.0.3 from backup ~2 hours later and changed unpublish policy: a package cannot be removed if it has been published >24 hours and any other public package depends on it.

Lesson: **a single 11-line dependency was a single point of failure for half the JavaScript ecosystem**. The dependency cost is independent of dependency size.

## SLSA — Supply-chain Levels for Software Artifacts

Introduced by Google in June 2021, now an OpenSSF project at `slsa.dev`. Use the current SLSA spec when citing levels; as of early 2026, SLSA v1.2 defines separate **Build** and **Source** tracks. The Build Track has three levels:

- **L1**: automated build process; provenance generated.
- **L2**: hosted build platform; provenance signed.
- **L3**: build platform hardened; provenance non-forgeable; builds isolated from one another and from provenance-signing material.

Older SLSA v0.1 had four levels (L1–L4); the v1.0 restructure changed the model. Do not mix pre-1.0 L4 language with current Build Track levels.

The point of SLSA is to make claims about your build *checkable*. A claim of "L3" can be verified by examining the provenance and the build platform's properties. Without that framework, "we have a secure pipeline" is unfalsifiable.

## Sigstore

OpenSSF graduated project (graduated 2024) at `sigstore.dev`. Components:

- **`cosign`** — sign and verify container images and binary artifacts.
- **`fulcio`** — short-lived X.509 certificate authority bound to OIDC identities. Signing key derives from your existing identity (GitHub, Google, etc.); no long-lived key material to manage.
- **`rekor`** — append-only transparency log; v2 GA reached 2025.
- **`gitsign`** — sign git commits with sigstore identities.

For keyless signing, the model is **sign with your identity, verify the artifact digest, certificate identity, issuer/OIDC claims, and Rekor inclusion or signed bundle/timestamp as applicable**. Keyless Sigstore reduces long-lived signing-key management; keyful or KMS-backed signing still needs normal key lifecycle controls. Verification policy matters as much as the signature.

## SBOM

A Software Bill of Materials lists every component in an artifact: name, version, license, supplier, hash. Common active standards:

- **SPDX 3.0** — Linux Foundation, originated 2010 for license compliance. SPDX 2.2.1 became ISO/IEC 5962:2021; SPDX 3.0 expands the supply-chain model.
- **CycloneDX 1.7** — OWASP, security-first focus, standardized as ECMA-424, strong for security/VEX and operational BOM use cases.

Either is fine; pick by what your downstream consumers expect. Tools that generate SBOMs at build time:

- **Syft** (Anchore) — multi-format, language-agnostic.
- **Trivy** (Aqua) — also a vulnerability scanner.
- **`cdxgen`** — CycloneDX-focused.
- **CycloneDX Maven and Gradle plugins** for JVM.

The point of an SBOM is that "what is in this artifact?" is a question with a precise answer. Without one, a vulnerability disclosure (Log4Shell, xz-utils) becomes a manual hunt across every artifact you've ever shipped.

US Executive Order 14028 (May 2021) catalyzed federal SBOM guidance and procurement expectations. The federal compliance landscape continues to change; cite EO 14028 for the *concept* and check current agency, OMB, and CISA requirements before claiming a binding procurement mandate.

## Operational defenses

The discipline that connects the framework to the day-to-day:

- **Dependency scanning in CI**, blocking on critical findings. OSV-Scanner (Google/OpenSSF, free), Trivy, Grype, Snyk, GitHub Advanced Security, Sonatype Lifecycle. The choice matters less than the act of doing it.
- **Patch SLAs by severity.** Critical CVE on a hot-path dep: same week. High: two weeks. Routine non-security updates: monthly cadence at minimum.
- **Renovate / Dependabot.** Automation produces PRs; **the discipline is merging them**. A repo with 80 stale Dependabot PRs is worse than one with no automation.
- **Pre-merge build provenance.** Every released artifact has a SLSA provenance attestation, signed via cosign, queryable via rekor.
- **Private registry proxy.** All package fetches go through a registry your team controls; the proxy enforces allowlists, scans, and pinning policies. Nexus, Artifactory, GitHub Packages, AWS CodeArtifact.
- **Privilege isolation in CI.** Do not expose production secrets to dependency resolution, install scripts, PR builds, or untrusted code execution. Disable lifecycle scripts where feasible, build in isolated workers, use short-lived OIDC credentials only in deploy/promote jobs after artifact creation and policy checks, and promote immutable artifacts instead of rebuilding with secrets.
- **Dangerous workflow review.** Flag `pull_request_target` or `workflow_run` workflows that check out untrusted PR code, untrusted GitHub context interpolated into shell, missing least-privilege `permissions`, or write tokens available before tests/builds of untrusted code.
- **2FA on every registry account.** Verify periodically.
- **License audit gating.** A new dependency with a license incompatible with your distribution policy must be caught before it ships. License-scanning tools (FOSSA, ScanCode, Black Duck) integrate at PR time.

## What to flag in review

- A new dep with no SBOM-trackable identity (vendored from an unknown origin).
- An unpinned `latest` in production code.
- A "verified publisher" badge cited as the security control.
- A CI runner that pulls from a public registry while production credentials are in scope.
- An auto-merge bot for Dependabot PRs without test gating.
- A "we forked it" without a README of what changed and why.
- An artifact promoted to production with no provenance attestation.
- A dependency with a sole maintainer and no organizational sponsorship for a load-bearing path.

## Sources

- NVD CVE-2021-44228 (Log4Shell). `https://nvd.nist.gov/vuln/detail/cve-2021-44228`.
- Wikipedia: XZ Utils backdoor. `https://en.wikipedia.org/wiki/XZ_Utils_backdoor`.
- Sam James, *xz-utils backdoor situation* gist. `https://gist.github.com/thesamesam/223949d5a074ebc3dce9ee78baad9e27`.
- Alex Birsan, *Dependency Confusion* (Medium, 9 February 2021). `https://medium.com/@alex.birsan/dependency-confusion-how-i-hacked-into-apple-microsoft-and-dozens-of-other-companies-4a5d60fec610`.
- npm Blog, *Details about the event-stream incident*. `https://blog.npmjs.org/post/180565383195/details-about-the-event-stream-incident`.
- npm Blog, *kik, left-pad, and npm*. `https://blog.npmjs.org/post/141577284765/kik-left-pad-and-npm`.
- SLSA. `https://slsa.dev/`.
- Sigstore (OpenSSF). `https://openssf.org/community/sigstore/`.
- Sigstore Blog, *Rekor v2 GA - Cheaper to run, simpler to maintain* (2025-10-10). `https://blog.sigstore.dev/rekor-v2-ga/`.
- SPDX. `https://spdx.dev/`.
- CycloneDX (OWASP). `https://cyclonedx.org/`.
- US EO 14028. `https://www.whitehouse.gov/briefing-room/presidential-actions/2021/05/12/executive-order-on-improving-the-nations-cybersecurity/`.
