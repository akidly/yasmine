---
description: Guide Claude to use the Yasmine voice AI API effectively — auth, country/language codes, E.164 phone format, error slugs.
---

# Yasmine API — usage guide for AI agents

Yasmine is a voice AI API that triggers outbound calls to confirm orders for resellers in Morocco, Algeria, Tunisia and France.

## Authentication

All endpoints require `Authorization: Bearer yk_<40chars>` (your reseller API key).

The plugin stores it via `userConfig.api_token` (keychain OS, prompted at first enable). Never paste the key in chat or commit it.

## Country and language codes

| Country | ISO | Default prompt language |
|---------|-----|-------------------------|
| Morocco | MA  | ar (darija)             |
| Algeria | DZ  | ar (darija)             |
| Tunisia | TN  | ar (darija)             |
| France  | FR  | fr                      |

The `country` field on `create_call` selects an internal prompt variant. Use the ISO code, never the language directly.

## Phone number format

**Strict E.164 only**. Examples : `+212612345678` (MA mobile), `+33612345678` (FR mobile). Validation rejects anything else with `validation_error` (422).

## Idempotency-Key

`POST /v1/calls` requires the header `Idempotency-Key`. Use a fresh UUID v4 per logical request.

For network retries (504, timeout) **reuse the same key** — the API replays the stored response with `X-Idempotent-Replay: true`. Different body + same key → `idempotency_key_conflict` (409).

## Common error slugs (RFC 7807)

| Slug | Status | When |
|------|--------|------|
| `validation_error` | 422 | Body fails schema |
| `rate_limit_exceeded` | 429 | Per-key bucket exhausted |
| `insufficient_balance` | 402 | Reseller balance < 10s |
| `idempotency_key_conflict` | 409 | Same key, different body |
| `invalid_cursor` | 400 | Pagination cursor malformed/expired |

For full Causes / Remediation / Example call the MCP tool `explain_error(slug)`.

## Discovery

The `yasmine` MCP server (declared in this plugin) exposes 8 tools :

- **Pass-through** (4) : `get_account`, `create_call`, `get_call`, `list_calls`
- **Introspection** (4) : `list_endpoints`, `get_endpoint_spec`, `get_changelog`, `explain_error`

Use `list_endpoints()` to discover the live API surface, `get_endpoint_spec(operation_id)` for full schemas, `get_changelog()` for what shipped recently.
