# yasmine

> Official Claude Code plugin for the [Yasmine](https://docs.yasmine.akidly.com) voice AI API — automated outbound call confirmations for reseller workflows in Morocco, Algeria, Tunisia and France.

## Status

Beta — `v0.1.0`. API surface stable on `/v1/`, plugin schema may iterate.

## Install

```
/plugin install yasmine@akidly/yasmine
```

Claude Code clones the default branch, registers the MCP server declared in `.claude-plugin/plugin.json`, and prompts for your Yasmine reseller API key on first enable. The token is stored in your OS keychain and forwarded to `https://mcp.yasmine.akidly.com/mcp/` as a `Bearer` header on each MCP call.

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

The plugin's `userConfig` declares two fields :

| Field | Default | Notes |
|-------|---------|-------|
| `api_token` | (prompted) | Your `yk_...` reseller key. Stored in OS keychain (`sensitive: true`). |
| `api_endpoint` | `https://mcp.yasmine.akidly.com/mcp/` | Override only for local dev (e.g. `http://127.0.0.1:9001/mcp/`). |

## Documentation

- API reference (Scalar) : https://docs.yasmine.akidly.com
- Source of the MCP server (private repo) : contact Akidly.

## License

MIT — see [LICENSE](LICENSE).
