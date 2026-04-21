# yasmine

> Official Claude Code plugin for the [Yasmine](https://docs.yasmine.akidly.com) voice AI API — automated outbound call confirmations for reseller workflows in Morocco, Algeria, Tunisia and France.

## Status

Beta — `v0.1.1`. API surface stable on `/v1/`, plugin schema may iterate.

## Setup

1. Get your Yasmine reseller API key from https://docs.yasmine.akidly.com/getting-started (format: `yk_...`).

2. Export it as an env var **before launching Claude Code** :

   **Linux / macOS (bash/zsh)**
   ```
   export YASMINE_API_TOKEN=yk_your_key_here
   ```

   **Windows (PowerShell, persistent)**
   ```
   [Environment]::SetEnvironmentVariable('YASMINE_API_TOKEN', 'yk_your_key_here', 'User')
   ```
   Then **restart Claude Code** (close all instances) for the env var to be picked up.

3. Install the plugin :
   ```
   claude plugin install yasmine@akidly/yasmine
   ```

4. Verify : the `yasmine` MCP server should appear in your active servers list, exposing 8 tools.

## Prerequisites

- A Yasmine reseller API key (starts with `yk_`). Get one at https://docs.yasmine.akidly.com/getting-started.
- Claude Code with plugin support (2025+ release).

## What you get

8 MCP tools and 1 skill, exposed under the `yasmine` namespace.

### MCP tools

**Pass-through to the Yasmine API** :

- `get_account` — your reseller info, balance, rate limits
- `create_call` — trigger an outbound voice confirmation
- `get_call` — status + transcript of a single call
- `list_calls` — paginated history of your calls

**Introspection (read API docs from the server)** :

- `list_endpoints` — enumerate the live API surface
- `get_endpoint_spec` — full OpenAPI spec for one operation
- `get_changelog` — what changed lately on the API
- `explain_error` — structured help on RFC 7807 error slugs

### Skill

- `yasmine-api` — quick reference (auth, ISO country codes, phone format, error slugs). Auto-loaded by Claude when it sees Yasmine-related questions.

## Configuration

The MCP endpoint is hard-coded to production (`https://mcp.yasmine.akidly.com/mcp/`). For local dev you can override it by editing `.mcp.json` after install (in `~/.claude/plugins/cache/akidly/yasmine/<version>/`) — but normal users should not need to.

The Bearer token is read from the `YASMINE_API_TOKEN` env var at MCP server startup time. Aligned with the official Anthropic plugin pattern (cf. `github` plugin from `claude-plugins-official`).

## Documentation

- API reference (Scalar) : https://docs.yasmine.akidly.com
- Source of the MCP server (private repo) : contact Akidly.

## License

MIT — see [LICENSE](LICENSE).
