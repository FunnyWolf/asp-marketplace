# ASP Marketplace

Agentic SOC Platform (ASP) Claude Code plugin marketplace.

## Overview

This marketplace provides Claude Code plugins for operating ASP through the current ASP MCP server. It bundles investigation agents and task-focused skills for common security operations workflows.

The bundled MCP configuration targets the current ASP backend MCP server:

- Transport: Streamable HTTP
- Endpoint: `${ASP_MCP_URL}` such as `https://asp.example.com/api/mcp`
- Authentication: `Authorization: Api-Key ${ASP_MCP_API_KEY}`

Create an ASP user API key in the platform and set both environment variables before using the plugin. The backend MCP endpoint is served by the ASGI application; in production, route `/api/mcp` to the Uvicorn process.

## Bundled agents

- `asp-case-investigator-en` / `asp-case-investigator-zh`
- `asp-artifact-investigator-en` / `asp-artifact-investigator-zh`
- `asp-threat-hunting-en` / `asp-threat-hunting-zh`

## Bundled skills

- `asp-alert-en` / `asp-alert-zh`
- `asp-artifact-en` / `asp-artifact-zh`
- `asp-case-en` / `asp-case-zh`
- `asp-cmdb-en` / `asp-cmdb-zh`
- `asp-comment-en` / `asp-comment-zh`
- `asp-enrichment-en` / `asp-enrichment-zh`
- `asp-knowledge-en` / `asp-knowledge-zh`
- `asp-module-creator-en` / `asp-module-creator-zh`
- `asp-playbook-en` / `asp-playbook-zh`
- `asp-playbook-creator-en` / `asp-playbook-creator-zh`
- `asp-siem-index-yaml-en` / `asp-siem-index-yaml-zh`
- `asp-siem-rule-en` / `asp-siem-rule-zh`
- `asp-siem-search-en` / `asp-siem-search-zh`
- `asp-threat-intelligence-en` / `asp-threat-intelligence-zh`

## Structure

```
plugins/
├── .mcp.json          # MCP server configuration
├── agents/            # Claude Code agent definitions
│   ├── asp-artifact-investigator-*.md
│   ├── asp-case-investigator-*.md
│   └── asp-threat-hunting-*.md
└── skills/            # Claude Code skill definitions
    ├── asp-alert-*
    ├── asp-artifact-*
    ├── asp-case-*
    ├── asp-cmdb-*
    ├── asp-comment-*
    ├── asp-enrichment-*
    ├── asp-knowledge-*
    ├── asp-module-creator-*
    ├── asp-playbook-*
    ├── asp-playbook-creator-*
    ├── asp-siem-index-yaml-*
    ├── asp-siem-rule-*
    ├── asp-siem-search-*
    └── asp-threat-intelligence-*
```

Each agent and skill is available in both English (`-en`) and Chinese (`-zh`) variants.

## Documentation

- English: https://asp.viperrtp.com/asp/integrations/claude-code/
- Chinese: https://asp.viperrtp.com/zh/asp/integrations/claude-code/

## License

See [LICENSE](LICENSE) for details.
