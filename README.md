# ASP Skills

Reusable Agentic SOC Platform skills for coding agents that support the open skills ecosystem.

The skills use the `asp` command line client as the runtime seam. Install and authenticate the CLI once, then let your agent call `asp ... --output json`.

## Quickstart

Install the skill package:

```bash
npx skills@latest add FunnyWolf/asp-marketplace
```

Select the skills and agent targets in the installer. Install `asp-setup` first, then run it in your agent:

```text
/asp-setup
```

`asp-setup` checks that `asp-cli` is installed, guides `asp auth login`, and verifies the connection with `asp doctor --output json`.

## Skills

### Setup

- `asp-setup`

### Records

- `asp-case`
- `asp-alert`
- `asp-artifact`
- `asp-comment`
- `asp-file`
- `asp-knowledge`
- `asp-enrichment`
- `asp-playbook`

### Integrations

- `asp-siem-search`
- `asp-siem-index-yaml`
- `asp-siem-rule`
- `asp-cmdb`
- `asp-threat-intelligence`

### Authoring

- `asp-module-creator`
- `asp-playbook-creator`

### Investigation workflows

- `asp-case-investigation`
- `asp-artifact-investigation`
- `asp-threat-hunting`

The investigation workflow skills replace the previous Claude Code agent definitions. They are user-invoked orchestrator skills so they work across Claude Code, GitHub Copilot CLI, Codex, and other skill-aware agents.

## Requirements

- `asp-cli` installed, usually with `pipx install asp-cli`
- ASP API URL and API key configured with `asp auth login`
- A valid `asp doctor --output json` result before using operational skills

Never store ASP API keys in skill files, repository files, or prompts.
