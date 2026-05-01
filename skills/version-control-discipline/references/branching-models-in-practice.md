# Branching Models in Practice

Reference for choosing among trunk-based, GitHub Flow, GitLab Flow, and GitFlow. The right answer depends on what you ship and how often, not on tribal preference.

## Trunk-based development

Maintained at `https://trunkbaseddevelopment.com` by Paul Hammant. Definition: a source-control branching model where developers collaborate on code in a single branch (`main` / `trunk`) and resist any pressure to create other long-lived development branches. Stated objective: avoid merge hell, do not break the build, live happily ever after.

**Mechanics.**
- Short-lived feature branches (hours to a day, occasionally two) for code review and CI.
- Feature flags and "branch by abstraction" hide incomplete work behind on/off switches.
- Commit to trunk multiple times per day. The team integrates on the same branch the same day.

**DORA correlation.** DORA's research consistently shows trunk-based development among the practices of elite-performing teams. Concretely, the threshold DORA describes:
- Three or fewer active branches.
- Merge to trunk at least daily.
- No code freezes or long integration phases.

The mechanism is batch size. Small changes are easier to review, easier to debug when they fail, and faster to roll back; trunk-based development *forces* small batches.

**When trunk-based is right.** Continuous-delivery shops; SaaS web apps and services; teams with strong CI and observability; teams that can put incomplete work behind release flags. Most modern web/SaaS work.

**When trunk-based is wrong.** Software with multiple supported major versions in production simultaneously (libraries, OS distributions, on-prem installers). The model assumes one production version; if you have v1 and v2 both maintained, the model breaks down.

## GitHub Flow

The lightest model in wide use: `main` + short-lived feature branches; merge via PR; main is always deployable. Documented in GitHub's quickstart: branch → commit → open PR → review → merge → delete branch.

GitHub Flow and trunk-based are close cousins. The difference is mostly cultural: trunk-based emphasizes daily merges and aggressive flag use; GitHub Flow is silent on cadence. In practice teams converge on the same shape if they ship continuously.

## GitLab Flow

A variant of GitHub Flow that adds **environment branches** for staged deployments — `main → pre-production → production`, or `main → staging → production`. Commits flow downstream only.

Also supports `release/v1`, `release/v2` for versioned-software cases. Sits between GitHub Flow (no environment branches at all) and GitFlow (multiple branch kinds, more ceremony). For teams with explicit staging environments and a need for downstream cherry-picks, this model formalizes practices already common.

## GitFlow

Vincent Driessen, *A successful Git branching model* (2010-01-05). The most-cited branching model of the 2010s. Five branch kinds:

- `master` — production-ready, tagged releases.
- `develop` — next-release integration.
- `feature/*` — branched off `develop`, merged back to `develop`.
- `release/*` — branched off `develop`, merged to both `master` and `develop` at release time.
- `hotfix/*` — branched off `master`, merged to both `master` and `develop`.

GitFlow was designed for software with discrete, versioned releases. It enforces explicit release-preparation periods and clean separation of in-flight features from stabilized release candidates.

### Driessen's 2020 reflection note (verbatim)

Driessen added a note at the top of the post on 2020-03-05 retiring GitFlow for continuous-delivery contexts:

> *"In those 10 years, git-flow (the branching model laid out in this article) has become hugely popular in many a software team to the point where people have started treating it like a standard of sorts — but unfortunately also as a dogma or panacea."*
>
> *"If your team is doing continuous delivery of software, I would suggest to adopt a much simpler workflow (like GitHub flow) instead of trying to shoehorn git-flow into your team."*
>
> *"If, however, you are building software that is explicitly versioned, or if you need to support multiple versions of your software in the wild, then git-flow may still be as good of a fit to your team as it has been to people in the last 10 years."*

The 2020 note is doing real work. GitFlow remains a defensible choice for the use cases it was designed for; it is over-engineering for SaaS in 2026. Reference Driessen himself when teams want to "adopt GitFlow" in a continuous-delivery context — the original author has retired it for that case.

## Choosing

| You ship... | Use |
|---|---|
| SaaS web app or service, continuous delivery | Trunk-based or GitHub Flow |
| Multi-environment service with explicit staging | GitLab Flow |
| Library with multiple maintained major versions | GitFlow or GitLab Flow's release branches |
| OS distribution, embedded firmware, enterprise on-prem product | GitFlow |

The decision is about your *delivery* shape, not about preference. A team forcing GitFlow onto a continuous-delivery codebase pays merge-conflict tax for nothing; a team forcing trunk-based onto a library with three maintained majors finds itself fighting the model.

## What to flag in review

- Adoption of GitFlow on a SaaS team — usually picked because the team has heard of it, not because it fits.
- A trunk-based team running a long-lived feature branch — the discipline has slipped; the branch will accumulate merge debt and probably die.
- A "develop" branch that has drifted from main without a release plan.
- A GitFlow team that has stopped doing release branches but kept everything else — the worst of both worlds.
- Branch policies that allow direct push to main (when the model expects PRs).
- Branch policies that require PRs to release/* but not to main.

## Sources

- Paul Hammant. `https://trunkbaseddevelopment.com`.
- DORA capability page on Trunk-Based Development. `https://dora.dev/capabilities/trunk-based-development/`.
- Vincent Driessen. *A successful Git branching model* (Jan 2010, retired Mar 2020 for CD contexts). `https://nvie.com/posts/a-successful-git-branching-model/`.
- GitHub Flow. `https://docs.github.com/en/get-started/quickstart/github-flow`.
- GitLab Flow. `https://about.gitlab.com/topics/version-control/what-is-gitlab-flow/`.
