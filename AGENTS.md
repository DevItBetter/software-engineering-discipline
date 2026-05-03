# AGENTS.md

## Repo Shape

- This is a prose-heavy skills suite, not a runnable app. `package.json` has only metadata; there are no npm scripts, lockfiles, CI workflows, or pre-commit hooks.
- `skills/engineering-discipline/` is the orchestrator and entry point. Specialty skills under `skills/*/` also trigger directly for their domains; do not force every question through the orchestrator.
- Each skill bundle has three load-bearing parts: `SKILL.md`, `references/`, and `agents/openai.yaml`. Keep `README.md` as the public map and routing table.

## Editing Skills

- Preserve the existing `SKILL.md` frontmatter shape: `---`, `name`, quoted `description`, blank line, closing `---`.
- When editing a skill, keep `README.md`, `SKILL.md`, reference files, and `agents/openai.yaml` aligned. Put new load-bearing sources in `skills/engineering-discipline/references/sources.md` or the skill's local `references/` file.
- Treat `agents/openai.yaml` as operational guidance, not marketing copy. Keep `interface.display_name`, `interface.short_description`, and `interface.default_prompt`; preserve the prompt contract: Use when, Evidence to collect, Verification gates, Stop/escalate when, Report.
- Before changing overlapping guidance, check neighboring skills. Common overlap pairs: deployment/database migrations, observability/debugging runbooks, testing/AI-generated tests, build/deployment supply chain, documentation/systems ADRs, caching/HTTP/API semantics.
- For new skills or substantive edits, use `docs/skill-quality-checklist.md`. If a touched skill has `What to flag in review`, run that checklist against your own change.
- New or expanded skills should follow the repo's stated maintenance loop: write, red-team, fix substantive findings, then ship.

## Verification

- Do not run `npm install` as a verification step here; with no dependencies or scripts it only creates lockfile noise.
- Parse all OpenAI adapters after editing any `agents/openai.yaml`:
  `python3 -c 'import glob, yaml; files=sorted(glob.glob("skills/*/agents/openai.yaml")); [yaml.safe_load(open(f)) for f in files]; print(f"Parsed {len(files)} adapters")'`
- Search for stale wording after terminology changes across the relevant `README.md`, `SKILL.md`, `references/`, and `agents/openai.yaml` files.
- If changing docs substantially, re-index with `jdocmunch` and verify doc health, per `docs/skill-quality-checklist.md`.
- Before handing off changes, run `git diff --check`.

## Consumer Commands

- Install all skills with `npx skills add DevItBetter/software-engineering-discipline --all`.
- List available skills with `npx skills add DevItBetter/software-engineering-discipline --list`.
- Install one skill with `npx skills add DevItBetter/software-engineering-discipline --skill engineering-discipline`.
- `openai` is only the adapter filename under `agents/openai.yaml`; it is not a valid `npx skills --agent` target.
