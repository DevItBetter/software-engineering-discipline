# Review Checklist

> **This file is retained as a redirect.** Earlier drafts of this suite kept a single monolithic checklist here. The material has been substantially expanded and reorganized into focused references; load the one that matches the situation rather than reading this file.

For a code review, use:

- `triage-and-output.md` — how to classify findings and structure the report.
- `red-flag-patterns.md` — patterns that are nearly always wrong; skim on every non-trivial review.
- `diff-vs-architecture.md` — adjusting the lens for diff-sized vs. module-sized vs. system-sized review units.
- `review-by-change-type.md` — checklists tuned to feature / bug-fix / refactor / migration / dependency / infra / security / perf / revert.
- `cl-and-pr-discipline.md` — small-CL principles; how to refuse a too-large PR; PR-description discipline.
- `sources.md` — the canon and citations.

For the deeper domains, use the sibling skills:

- `software-design-principles` — cohesion / coupling / abstraction / SOLID-with-nuance / naming.
- `refactoring-and-code-smells` — Fowler's catalog, refactoring moves, legacy-code strategies.
- `testing-discipline` — test pyramid and alternatives, what to test, AI-generated test smells.
- `error-handling-and-resilience` — errors as values, retries, idempotency, fail-fast vs let-it-crash.
- `api-and-interface-design` — Hyrum's Law, contracts, evolution, defaults that are hard to misuse.
- `secure-coding-fundamentals` — OWASP Top 10 + LLM Top 10, trust boundaries, secrets, crypto rules.
- `concurrency-and-state` — races, atomicity, distributed state, ordering, idempotency.
- `performance-engineering` — measure-first, hot paths, common pitfalls, latency budgets.
- `systems-architecture` — distributed-systems fallacies, bounded contexts, hexagonal/clean, evolutionary architecture, observability-as-design, progressive delivery, chaos engineering, technical debt.
- `ai-coding-antipatterns` — failure modes specific to LLM-generated code; consult on every AI-authored change.
