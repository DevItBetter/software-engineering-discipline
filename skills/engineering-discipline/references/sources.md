# Sources and Canon

The wisdom in this skill suite is not invented. It is condensed from a body of work the field has converged on over decades. Cite these when an author asks "why."

## Foundational papers

- **David Parnas, "On the Criteria To Be Used in Decomposing Systems into Modules"** (CACM, 1972). The origin of information hiding. Decompose systems by hiding decisions likely to change, not by following the flowchart.
- **Fred Brooks, "No Silver Bullet"** (1986) and *The Mythical Man-Month* (1975). The distinction between essential and accidental complexity. The mythical-man-month effect.
- **Butler Lampson, "Hints for Computer System Design"** (SOSP 1983; expanded 2020 as *Hints and Principles*). STEADY goals, AID techniques. "Do one thing well." The robustness principle.
- **Hoare, "The Emperor's Old Clothes"** (Turing Award lecture, 1980). The "two ways of constructing a software design" passage on simplicity.
- **Hoare, "Null References: The Billion Dollar Mistake"** (QCon London, 2009). The retrospective on introducing the null reference into ALGOL W.

## Books that shaped the field

- **John Ousterhout, *A Philosophy of Software Design*** (2018, 2nd ed. 2021). Deep modules, complexity = dependencies + obscurity, tactical vs strategic programming, "define errors out of existence." The single most cited modern source on design.
- **Martin Fowler, *Refactoring*** (2nd ed., 2018). The canonical catalog of code smells and refactorings. Read this if you read nothing else.
- **Eric Evans, *Domain-Driven Design*** (2003) and **Vaughn Vernon, *Implementing Domain-Driven Design*** (2013). Bounded contexts (strategic), aggregates / entities / value objects (tactical), ubiquitous language.
- **Martin Kleppmann, *Designing Data-Intensive Applications*** (2017; 2nd ed. with Chris Riccomini, 2026). The current standard reference for data systems and distributed-systems trade-offs. Reliability, scalability, maintainability; consistency models; consensus.
- **Kent Beck, *Test-Driven Development*** (2002), *Tidy First?* (2024), *Extreme Programming Explained* (2nd ed.). TDD; "make the change easy, then make the easy change."
- **Sam Newman, *Building Microservices*** (2nd ed., 2021). Service boundaries, evolution, decomposition.
- **Andy Hunt & Dave Thomas, *The Pragmatic Programmer*** (20th anniversary ed., 2019). DRY (in its original meaning), orthogonality, broken windows.
- **Steve McConnell, *Code Complete*** (2nd ed., 2004). Empirical software construction at depth.
- **Sandi Metz, *Practical Object-Oriented Design in Ruby*** (2012). A usable take on OO that doesn't dogmatize. The "Sandi Metz rules" — ≤100 line classes, ≤5 line methods, ≤4 parameters per method, controllers instantiate one object and views know one instance variable — were proposed on Ruby Rogues podcast episode 87 (January 2013), not the *POODR* book.
- **Michael Feathers, *Working Effectively with Legacy Code*** (2004). The seam concept. How to add tests to untested code.
- **Robert Martin, *Clean Architecture*** (2017). The dependency rule, hexagonal/clean variants. Read with critical distance — much of "Clean Code" is contested; the dependency-direction ideas are durable.

## Talks worth watching

- **Rich Hickey, *Simple Made Easy*** (Strange Loop 2011). The simple/easy distinction. "Complecting." State complects everything.
- **Rich Hickey, *Hammock-Driven Development*** (2010). On thinking before coding.
- **Greg Young, *8 Lines of Code*** (2013). On unnecessary abstraction.
- **Joe Armstrong, *The How and Why of Fitting Things Together*** (2018). Erlang's "let it crash" and supervisor trees.
- **Sandi Metz, *All the Little Things*** (RailsConf 2014). The squint test, on cohesion.
- **Hyrum Wright, *Hyrum's Law*** (informal — see hyrumslaw.com). With enough users, every observable becomes a contract.

## Practitioner guides

- **Google Engineering Practices** — https://google.github.io/eng-practices/. The CL author's guide and the reviewer's guide. Source for "small CLs."
- **Google SRE Book and Workbook** — https://sre.google/. SLOs, SLIs, error budgets, alerting.
- **OWASP Top 10** — https://owasp.org/Top10/. Application security baseline.
- **OWASP Top 10 for LLM Applications (2025)** — https://genai.owasp.org/llm-top-10/. AI-system-specific risks.
- **OWASP Secure Code Review Cheat Sheet** — https://cheatsheetseries.owasp.org/cheatsheets/Secure_Code_Review_Cheat_Sheet.html
- **NIST Secure Software Development Framework (SSDF)** — https://csrc.nist.gov/projects/ssdf
- **CERT Secure Coding** — https://wiki.sei.cmu.edu/confluence/display/seccode
- **The Twelve-Factor App** — https://12factor.net/. Cloud-native operational discipline.
- **DORA / Accelerate research** — https://dora.dev/. Empirical research on what makes high-performing software teams (deployment frequency, lead time, MTTR, change failure rate).

## Foundational concepts referenced throughout

- **Hyrum's Law** — "with a sufficient number of users of an API, it does not matter what you promise in the contract: all observable behaviors of your system will be depended on by somebody." (hyrumslaw.com)
- **Conway's Law** — system architecture mirrors the communication structure of the organization. (Melvin Conway, 1968.)
- **Brooks's Law** — adding people to a late project makes it later.
- **Fallacies of Distributed Computing** — the network is reliable, latency is zero, bandwidth is infinite, the network is secure, topology doesn't change, there is one administrator, transport cost is zero, the network is homogeneous. All false. The first four were articulated by Bill Joy and Tom Lyon at Sun in the early 1990s; Peter Deutsch added fallacies 5–7 in 1994; James Gosling added the eighth around 1997.
- **CAP / PACELC** — under partition, choose consistency or availability; absent partition, latency or consistency. (Brewer; Abadi.)
- **Postel's Law / Robustness Principle** — be conservative in what you do, liberal in what you accept. Increasingly contested for security-sensitive code; cited for context.
- **Knuth on premature optimization** — the full quote includes "yet we should not pass up our opportunities in that critical 3%." Don't use the partial quote to dismiss optimization, and don't use it to justify carelessness about performance on hot paths.

## Sources by domain — the specialty skills

The orchestrator's foundational citations are above. Each specialty skill anchors additional canon for its domain; the most load-bearing of those:

### Observability
- **Beyer, Jones, Petoff, Murphy, *Site Reliability Engineering*** (O'Reilly 2016) and ***The Site Reliability Workbook*** (2018). SLIs/SLOs/error budgets, multi-window multi-burn-rate alerting, the four golden signals.
- **Cindy Sridharan, *Distributed Systems Observability*** (O'Reilly Report, 2018). The three-telemetry-types treatment.
- **Charity Majors and the Honeycomb school**. Wide structured events; observability vs monitoring.
- **Sigelman et al., *Dapper*** (Google, 2010). The trace-of-spans data model.
- **W3C Trace Context** (Recommendation, Feb 2020; revised Nov 2021). `traceparent` / `tracestate`.
- **OpenTelemetry**. CNCF project standardizing API, SDK, semantic conventions, OTLP.
- **Brendan Gregg, *The USE Method*** (ACM Queue, 2012). Utilization / Saturation / Errors for resources.
- **Tom Wilkie, *The RED Method*** (Weaveworks, ~2015). Rate / Errors / Duration for services.
- **Rob Ewaschuk, *My Philosophy on Alerting*** (Google SRE, ~2014). Symptoms over causes.

### Deployment and release engineering
- **Jez Humble & David Farley, *Continuous Delivery*** (Addison-Wesley, 2010). The CD-as-discipline canon.
- **Forsgren, Humble, Kim, *Accelerate*** (IT Revolution, 2018). The four DORA metrics.
- **Pete Hodgson, *Feature Toggles (aka Feature Flags)*** (martinfowler.com, 2017). Release / experiment / ops / permissioning taxonomy.
- **Danilo Sato, *Parallel Change*** (martinfowler.com, 2014). Expand / migrate / contract.
- **James Governor, *Towards Progressive Delivery*** (RedMonk, 2018).
- **Werner Vogels, *A Conversation with Werner Vogels*** (ACM Queue, 2006). "You build it, you run it."

### Version control discipline
- **Tim Pope, *A Note About Git Commit Messages*** (2008). 50/72; imperative subject.
- **Chris Beams, *How to Write a Git Commit Message*** (2014). The seven rules.
- **Vincent Driessen, *A successful Git branching model*** (2010, retired for CD contexts in 2020 note).
- **Paul Hammant, *trunkbaseddevelopment.com***. The trunk-based canon.
- **Jonathan Corbet, *Rebasing and merging: some git best practices*** (LWN, 2009). "Thou Shalt Not Rebase Trees With History Visible To Others" (Corbet's distillation of Linus's view).
- **Linux kernel maintainer documentation** on rebasing and merging.

### Build and dependencies
- **Ken Thompson, *Reflections on Trusting Trust*** (ACM Turing lecture, CACM 27.8, August 1984). Compiler-inserted backdoors; reproducibility as defense.
- **Reproducible Builds project** — `reproducible-builds.org`. The bit-for-bit-identical-artifacts definition.
- **SLSA** (Supply-chain Levels for Software Artifacts) — `slsa.dev`. v1.0 (April 2023); Build track L1–L3.
- **Sigstore** (OpenSSF graduated 2024) — `sigstore.dev`. cosign / fulcio / rekor / gitsign.
- **SPDX** (Linux Foundation; ISO/IEC 5962:2021) and **CycloneDX** (OWASP). SBOM standards.
- **Karger et al., *Consistent Hashing and Random Trees*** (STOC '97). The sharding canon.
- **Raemaekers et al., *Semantic Versioning versus Breaking Changes: A Study of the Maven Repository*** (SCAM 2014, EMSE 2017). Empirical SemVer compliance.

### Caching strategies
- **Phil Karlton** — "There are only two hard things in Computer Science: cache invalidation and naming things." Folkloric; never documented in print (earliest internet trace: Tim Bray, 23 December 2005).
- **RFC 9111** *HTTP Caching* (June 2022). The current spec; obsoletes RFC 7234.
- **RFC 5861** *Cache-Control Extensions for Stale Content* (May 2010). `stale-while-revalidate`, `stale-if-error`.
- **RFC 9211** *Cache-Status* (June 2022). Cross-cache annotation.
- **RFC 8246** *HTTP Immutable Responses* (September 2017). Content-hashed asset URLs.
- **Vattani, Chierichetti, Lowenstein, *Optimal Probabilistic Cache Stampede Prevention*** (PVLDB 2015). XFetch.
- **Megiddo & Modha, *ARC: A Self-Tuning, Low Overhead Replacement Cache*** (USENIX FAST '03).
- **Einziger, Friedman, Manes, *TinyLFU*** (ACM TOS 13:4:35, 2017). Used by Caffeine and Ristretto.
- **Yang et al., *FIFO Queues are All You Need for Cache Eviction*** (S3-FIFO, SOSP 2023).
- **DHH, *How key-based cache expiration works*** (Signal v. Noise, 19 February 2012). Russian-doll caching.
- **AWS Builders' Library, *Caching challenges and strategies***. Cache addiction.

## Why a sources section matters

When an AI reviewer cites a principle, the author can rightly ask "says who?" Naming the source — Fowler on Long Parameter List, Ousterhout on shallow modules, Hyrum on observable behaviors — does three things:

1. It pre-empts arguments-from-authority on the author's side. They are no longer arguing with the reviewer; they are arguing with the field.
2. It teaches. The author learns the vocabulary and can apply it next time.
3. It signals epistemic discipline: the reviewer is not making it up.

The point is never to win on credentials. It is to make the reasoning checkable.
