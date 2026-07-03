# ASP Skills Context

## Language

**ASP CLI**: The `asp` command installed from the `asp-cli` Python package. It is the runtime seam for all skills.

**Operational skill**: A skill that reads or writes ASP platform data through `asp ... --output json`.

**Orchestrator skill**: A user-invoked investigation workflow that coordinates lower-level ASP skills and commands, then stops when it can answer the user's question.

**Authoring skill**: A user-invoked workflow that helps create ASP modules, playbooks, SIEM YAML, or SIEM rules.

## Relationships

- ASP skills depend on ASP CLI.
- ASP CLI talks to the ASP Agent Operations API.
- Orchestrator skills compose operational skills.
- Authoring skills may inspect local ASP project files, but still use ASP CLI for platform data and stream samples.
