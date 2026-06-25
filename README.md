# ASP Marketplace

Agentic SOC Platform (ASP) Claude Code Plugin Marketplace.

## Overview

This marketplace provides Claude Code plugins for operating the ASP platform, including agents and skills for security operations workflows.

The bundled MCP configuration targets the current ASP backend MCP server:

- Transport: Streamable HTTP
- Endpoint: `${ASP_MCP_URL}` such as `https://asp.example.com/api/mcp`
- Authentication: `Authorization: Api-Key ${ASP_MCP_API_KEY}`

Create an ASP user API key in the platform and set both environment variables before using the plugin. The backend MCP endpoint is served by the ASGI application; in production, route `/api/mcp` to the Uvicorn process.

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
    ├── asp-siem-index-yaml-*
    ├── asp-siem-search-*
    └── asp-threat-intelligence-*
```

Each agent and skill is available in both English (`-en`) and Chinese (`-zh`) variants.

## Documentation

Installation and usage guide: https://asp.viperrtp.com/asp/PLUGINS/ClaudeCode/

## License

See [LICENSE](LICENSE) for details.
