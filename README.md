# Yasmine — Claude Code plugin

MCP plugin pour interagir avec l'API Yasmine (agent vocal IA pour confirmation de commandes, marché MA/DZ/TN/FR).

## Installation

    /plugin marketplace add akidly/yasmine
    /plugin install yasmine@akidly

Configurer ensuite le token API dans l'environnement :

    export YASMINE_API_TOKEN="yk_..."

## Contenu

- **MCP server** `yasmine` (HTTP streamable) connecté à `https://mcp.yasmine.akidly.com/mcp/`
- **Skill** `yasmine-api` — docs OpenAPI, errors (RFC 7807), examples curl, getting-started

## Documentation

- Portail : https://docs.yasmine.akidly.com
- API source : https://api.yasmine.akidly.com

---

> ⚠️ **Auto-generated repo.** This repository is automatically generated from [cbensassi2/yasmine](https://github.com/cbensassi2/yasmine) via `.github/workflows/sync-plugin.yml`. **Do not commit directly here** — any manual change will be overwritten at the next sync.
