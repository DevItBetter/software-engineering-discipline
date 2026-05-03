# Skill Quality Checklist

Use this checklist for any new skill or substantive edit. The goal is not more prose; the goal is guidance an agent can apply safely.

## Accuracy

- Verify factual claims against primary or authoritative sources.
- Check current standards, versions, and status pages for moving targets such as OWASP, SLSA, OpenTelemetry, RFCs, database behavior, package tooling, and cloud/security guidance.
- Avoid over-precise dates, numbers, or origin claims unless they are sourced.
- Distinguish canon from useful local shorthand. If a term is not canonical, say so.

## Sources

- Add new load-bearing sources to `skills/engineering-discipline/references/sources.md` or the skill's local `references/` file.
- Cite the source that supports the claim, not a secondary article that repeats it.
- Keep source claims aligned across `README.md`, `SKILL.md`, reference files, and adapters.

## Agent Usefulness

- State when the skill should trigger.
- Tell the agent what evidence to collect before giving advice.
- Include verification gates: commands, files, traces, plans, tests, schemas, logs, or source references as appropriate.
- Include stop or escalation conditions for unsafe uncertainty.
- Require output to say what was checked and what remains unverified.

## Examples

- Treat every command and code block as copyable unless clearly marked as pseudocode.
- Avoid examples that perform production side effects without prerequisites, approval, and verification.
- Prefer safe placeholders such as `<namespace>` over real-looking production targets.
- Check that fenced code labels match the language actually used.
- Avoid examples that contradict sibling skills, such as `SELECT *` in performance examples or raw sensitive data in logging examples.

## Nuance

- Strong rules are fine for safety boundaries, data loss, secrets, and destructive operations.
- Soften absolutes when guidance depends on scale, language, ecosystem, traffic volume, storage engine, maturity, or public vs internal consumers.
- Explain the condition that makes a rule true.

## Adapter Parity

For each `agents/openai.yaml`, preserve the full skill's operational contract:

- when to use the skill
- evidence to collect
- verification gates
- stop/escalation conditions
- common findings to flag

Adapters are not marketing summaries. For some agents, they are the only guidance loaded.

## Cross-Skill Consistency

- Check related skills for overlapping guidance before editing.
- Harmonize duplicated advice rather than letting two skills diverge.
- Common overlap pairs: deployment/database migrations, observability/debugging runbooks, testing/AI-generated tests, build/deployment supply chain, documentation/systems ADRs, caching/HTTP/API semantics.

## Final Checks

- Run YAML parsing for `skills/*/agents/openai.yaml`.
- Run a stale-phrase search for old wording you replaced.
- Re-index with `jdocmunch` and verify doc health.
- Review `git diff --check` before committing.
