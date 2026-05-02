# Architecture Decision Records (ADRs)

A short document capturing one significant decision: what we chose, why, what we considered, what we accept as a consequence.

ADRs are not a heavy ceremony. They are a discipline that pays off years later when someone asks "why did we build it this way?" and the answer is in version control rather than in the head of someone who left.

## When to write one

For decisions that:
- Are expensive to undo (architecture, technology choice, public API shape, schema decisions).
- Will affect more than one team or component.
- Have non-obvious trade-offs (the chosen path is not the obvious path).
- You'd want a future engineer to understand the reasoning behind.

Don't write one for: small implementation choices, code-style decisions, decisions that can be reversed in a day.

## The standard template (Michael Nygard, 2011)

```markdown
# ADR NNNN: [Title]

## Status
[Proposed | Accepted | Deprecated | Superseded by ADR-NNNN]

## Context
[What is the situation? What problem are we solving? What forces apply?
Include: the technical landscape, the team situation, the deadlines, the
alternatives that exist in principle.]

## Decision
[What we decided. State it clearly and unambiguously.]

## Consequences
[What becomes easier as a result of this decision?
What becomes harder?
What new constraints does it introduce?
What follow-up work does it trigger?]

## Alternatives considered
[For each significant alternative:
- What it was.
- Why we didn't pick it.
- (Optional) Conditions under which we'd reconsider.]
```

## A worked example

```markdown
# ADR 0017: Use PostgreSQL row-level security for tenant isolation

## Status
Accepted (2026-03-15)

## Context
We're a multi-tenant SaaS. Each customer organization is a tenant; tenants
share infrastructure. The risk we cannot accept: tenant A reads or modifies
tenant B's data.

Tenant filtering in application code (`WHERE tenant_id = ?` on every query)
has worked so far but has produced two near-misses in code review: a query
that forgot the filter, caught at review; a join that included the filter
on one side but not the other, also caught at review. Both bugs would have
been catastrophic.

We have ~200 unique queries across the codebase, growing.

## Decision
Adopt PostgreSQL row-level security (RLS). Every multi-tenant table gets
an RLS policy that restricts rows to the current tenant, identified via a
session variable set per-request. Application code no longer needs to add
`WHERE tenant_id = ?`; PostgreSQL enforces it.

The session variable is set in a connection-pool middleware that runs at
the start of every request, sourced from the authenticated user's tenant.

## Consequences
Easier:
- Tenant filtering enforced by the database for ordinary application roles; hard to bypass accidentally.
- Reviewers no longer need to verify `WHERE tenant_id = ?` on every query.
- Bugs from missing application-level tenant filters are largely eliminated as a class.

Harder:
- Database-level enforcement requires the session variable to be set
  correctly. Connection pool reuse can leak state across requests; we need
  to RESET ROLE / RESET tenant_id at the end of every request.
- Background jobs and cron tasks need a different mechanism (no request
  context). For these, we explicitly elevate to a "system" role that has
  bypass — and we audit every use.
- Migrations / DDL run as a superuser, also bypassing RLS. Acceptable.
- Performance: RLS adds one predicate per query. Measured as <1% impact
  in our benchmarks.
- Onboarding: developers new to the codebase need to understand RLS;
  unfamiliar pattern.

Follow-up:
- Connection pool middleware to set/reset the session variable.
- Tests that try cross-tenant queries; verify they 0-row.
- Runbook entry for the "system" elevation, including audit.

## Alternatives considered
1. **Continue with application-level filtering.** Rejected: track record of
   near-misses; risk profile not acceptable as we scale.
2. **Per-tenant database / schema.** Rejected: at our tenant count
   (~10,000), the operational burden is too high. Considered for top-tier
   customers with extra contractual isolation needs; not needed yet.
3. **Service-layer enforcement (a "tenant-aware" repository class).**
   Rejected: still relies on developers using the right repository for every
   query; doesn't catch raw SQL or new query paths.
4. **Database views per tenant.** Rejected: doesn't compose well with our
   ORM; explosion of views; harder to query across tenants for admin
   purposes (which we legitimately need).
```

## Status transitions

ADRs are append-only. When a decision changes, write a new ADR that **supersedes** the old one. Mark the old one's status as superseded, with a link to the new one. Don't edit the old one to reflect the new decision — the historical record matters.

This is important: the *reasoning* in the old ADR is the input to the new ADR. Without it, the new author re-litigates settled trade-offs.

## ADRs and design docs

ADRs are short and focused — one decision each. **Design docs** are longer and may cover multiple decisions for a single feature or system. Use both:

- Design doc when introducing a new system, with multiple decisions and trade-offs to walk through.
- ADRs for the specific decisions that emerge, so they're discoverable individually.

The relationship: a design doc may produce several ADRs as it converges. The ADRs outlast the design doc (which often goes stale and is forgotten); they are the durable record.

## Storage

ADRs live in version control, alongside the code. A `docs/adr/` directory with files numbered sequentially (`0001-initial-architecture.md`, `0002-postgres-vs-mysql.md`, etc.) is the convention. Several tools generate and manage them (`adr-tools`, MADR templates).

Make them findable: a README in `docs/adr/` listing the ADRs and their current status.

## Common pitfalls

- **No ADRs at all.** Decisions made in chat, in standups, in conversations. Lost. Re-litigated. The team feels constantly unsure why things are the way they are.
- **ADRs that are too long.** A 20-page ADR is a design doc in disguise. Split it, or write a design doc and have the ADR reference it.
- **ADRs without alternatives.** "We picked X because we like X" is not a decision; it's an assertion. The alternatives section forces you to articulate the trade-off.
- **ADRs that get stale.** The decision was reversed; nobody updated the ADR; readers are confused. Status discipline matters; superseding ADRs link to the new one.
- **Edit-in-place ADRs.** The history is destroyed. Append-only with supersession.
