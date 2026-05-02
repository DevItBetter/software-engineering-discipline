# Rock Solid Engineering Skills

Teach AI assistants to engineer software, not just generate code. This suite of language-agnostic software-engineering skills is designed to hold up under expert scrutiny, so work by humans or AI assistants meets a senior-engineer bar: evidence-backed, named in the canonical vocabulary, honest about what was checked and what wasn't, and resistant to the failure modes (sycophantic acceptance, stylistic nitpicking, AI hallucination) that produce code that *looks* correct but isn't.

## Where to start

**`skills/engineering-discipline/`** is the orchestrator and the entry point. It owns the workflow, the triage, the standards, and the output format. When the work is broad enough to need cross-domain triage, start here. The orchestrator routes to specialty skills.

The other specialty skills trigger **independently** when their domain is in play — you don't have to go through the orchestrator. Ask "should I add an index?" and `database-and-data-modeling` triggers directly. Ask "is this a good abstraction?" and `software-design-principles` triggers directly. The orchestrator coordinates when the work spans domains; the specialties stand alone for focused questions.

## The structure

```
software-engineering-discipline/
├── skills/
│   ├── engineering-discipline/              ← orchestrator skill
│   ├── software-design-principles/
│   ├── code-smells-and-antipatterns/
│   ├── refactoring/
│   ├── testing-discipline/
│   ├── error-handling-and-resilience/
│   ├── api-and-interface-design/
│   ├── secure-coding-fundamentals/
│   ├── concurrency-and-state/
│   ├── performance-engineering/
│   ├── systems-architecture/
│   ├── database-and-data-modeling/
│   ├── debugging-and-incident-response/
│   ├── observability/
│   ├── deployment-and-release-engineering/
│   ├── version-control-discipline/
│   ├── build-and-dependencies/
│   ├── caching-strategies/
│   ├── documentation-and-technical-writing/
│   └── ai-coding-antipatterns/
├── README.md
└── LICENSE
```

Each skill is a directory with:
- `SKILL.md` — the entry point (frontmatter + instructions)
- `references/` — deeper material loaded on demand
- `agents/openai.yaml` — universal adapter for non-Claude models

## Which skill for which question

| If the question is about… | Use… |
|---|---|
| "Review this PR / design / RFC" | `engineering-discipline` |
| "Is this a good abstraction / module / structure?" | `software-design-principles` |
| "What's wrong with this code? Name the smell." | `code-smells-and-antipatterns` |
| "How do I refactor this safely?" | `refactoring` |
| "What should I test? Are these tests any good?" | `testing-discipline` |
| "How should I handle this error / retry / failure?" | `error-handling-and-resilience` |
| "Is this API design sound? Will it evolve well?" | `api-and-interface-design` |
| "Is this secure? Trust boundaries? OWASP?" | `secure-coding-fundamentals` |
| "Race condition? Atomicity? Distributed state?" | `concurrency-and-state` |
| "Is this fast enough? Where's the bottleneck?" | `performance-engineering` |
| "Should we use microservices? Bounded contexts?" | `systems-architecture` |
| "Schema? Index? Query plan? Migration?" | `database-and-data-modeling` |
| "Why is this broken? How do I find the bug?" | `debugging-and-incident-response` |
| "Logs / metrics / traces / SLOs / can on-call diagnose this?" | `observability` |
| "How do we ship this safely? Canary? Feature flag? Migration?" | `deployment-and-release-engineering` |
| "Commit message? PR size? Branching? `git bisect`?" | `version-control-discipline` |
| "Lockfile? Reproducible build? Supply chain? Dependency safe?" | `build-and-dependencies` |
| "Should I cache this? TTL? Stampede? Invalidation?" | `caching-strategies` |
| "Help me write this design doc / README / postmortem" | `documentation-and-technical-writing` |
| "Reviewing AI-generated code" | `ai-coding-antipatterns` (alongside the orchestrator) |

## The ethos

The work is built on three convictions:

1. **Engineering judgment is teachable.** The vocabulary already exists — Fowler's smells, Ousterhout's deep modules, Hickey's complecting, Hyrum's Law, the fallacies of distributed computing, OWASP's threat categories, Allspaw's blameless postmortems. These skills make the canon usable.

2. **Verification beats assertion.** AI assistants (and tired humans) drift toward "this looks right" without evidence. Every skill enforces verification gates: read the symbol, not the name. Run one adversarial input. Check the test would fail without the change. Read the actual library to confirm the API exists.

3. **Specific beats general.** "Make this cleaner" is not feedback. "This is a Long Parameter List with a Boolean Trap; replace with a parameter object" is. The skills name failures sharply so authors can act on them.

## Sources

The canon this suite is built on: Fowler (*Refactoring* 2nd ed.), Ousterhout (*A Philosophy of Software Design*), Hickey (*Simple Made Easy*), Parnas (*On the Criteria…*), Kleppmann (*DDIA*), Evans / Vernon (DDD), Lampson (*Hints*), Newman (*Building Microservices*), Beck (*TDD*, *Tidy First?*), Feathers (*Working Effectively with Legacy Code*), Metz (POODR), Allspaw (Etsy postmortems), Dekker (Safety II), Zeller (*Why Programs Fail*), Agans (*Debugging: 9 Indispensable Rules*), Procida (Diátaxis), Winand (*SQL Performance Explained*), Google Engineering Practices, Google SRE Workbook, OWASP Top 10 (and OWASP Top 10 for LLM Applications). Full citations in `skills/engineering-discipline/references/sources.md`.

## Using the suite

**With Claude (or another agent that supports skills):** the skills are designed to be invoked by their description. Install the suite with:

```bash
npx skills add DevItBetter/software-engineering-discipline --all
```

Or install individual skills:

```bash
npx skills add DevItBetter/software-engineering-discipline --skill engineering-discipline
```

The orchestrator and specialties trigger when their domain is in play. The orchestrator's description is deliberately broad ("any non-trivial engineering work") so it triggers as the entry point; specialty descriptions are scoped to their domains.

**Without an agent:** the SKILL.md files are also designed to be readable as standalone references. A senior engineer onboarding to a team can read through them in roughly the order an engineer reaches for the topics across a career.

**Adapting to your team:** the skills are written language-agnostic and stack-agnostic. They use Python / TypeScript / SQL examples for clarity but the principles apply to any stack. For domain-specific skills (frontend, mobile, ML systems, embedded, data pipelines), this suite is the foundation; specialty skills can be added on top.

## Maintenance

The suite was built with two rounds of adversarial red-team review per addition. The pattern: write, red-team, fix the substantive findings, ship. When extending the suite, follow the same pattern — bare assertions don't survive expert scrutiny, and the suite's authority depends on every skill holding up to it.

Most `SKILL.md` files include a "What to flag in review" section that doubles as a checklist for self-review. If you're modifying a skill and that section exists, run it on your changes.

## Status

Active. The orchestrator (`engineering-discipline`) was renamed and broadened from the original `code-review-best-practices` skill to reflect the cross-mode role (review, design, debug, implement, document, refactor); the deprecated redirect has been removed. New specialty skills are added with the same two-round adversarial pattern that produced the originals.
