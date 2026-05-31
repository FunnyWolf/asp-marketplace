# ASP Marketplace

Agentic SOC Platform (ASP) Claude Code Plugin Marketplace.

## Overview

This marketplace provides Claude Code plugins for operating the ASP platform, including agents and skills for security operations workflows.

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
