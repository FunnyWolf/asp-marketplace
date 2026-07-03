This repository is an ASP skill registry.

## Repository rules

- Skills live under `skills/<bucket>/<skill-name>/SKILL.md`.
- Skill names are canonical English names without `-zh` or `-en`.
- Every skill must respond in the user's language.
- Every operational skill depends on `asp` CLI and uses `asp ... --output json`.
- Do not reintroduce legacy server-specific tool dependencies or environment-variable authentication setup.
- Write operations must only run when the user explicitly asks to create, update, upload, comment, enrich, or run automation.
- Investigation workflow skills are user-invoked orchestrator skills, not platform-specific agents.
- Keep `.claude-plugin/plugin.json` as a light compatibility manifest that lists the promoted skills.

## Invocation policy

Use `disable-model-invocation: true` for setup, creator, rule-authoring, and investigation workflow skills. Record and integration lookup skills may be model-invoked when their description matches the user request.
