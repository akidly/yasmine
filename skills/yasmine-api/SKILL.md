---
name: yasmine-api
description: Skill expert de l'API Yasmine — auth, codes pays/langue, format E.164, classification d'appel enrichie, slugs d'erreur. Référence complémentaire pour les agents IA qui codent une intégration reseller.
---

# Yasmine API — usage guide for AI agents

Yasmine is a voice AI API that triggers outbound calls to confirm orders for resellers in Morocco, Algeria, Tunisia and France.

## Authentication

All endpoints require `Authorization: Bearer yk_<40chars>` (your reseller API key).

The plugin reads the key from the `YASMINE_API_TOKEN` environment variable (set once at install time, before invoking `/plugin install yasmine@akidly`). Never paste the key in chat or commit it to a repo.

## Country and language codes

| Country | ISO | Default call language |
|---------|-----|------------------------|
| Morocco | MA  | ar (darija)            |
| Algeria | DZ  | ar (darija)            |
| Tunisia | TN  | ar (darija)            |
| France  | FR  | fr                     |

The `country` field on `create_call` selects the localized model variant. Use the ISO code, never the language directly. The optional `language` field overrides the default (e.g. `country=MA` + `language=fr` is supported). The combination `country=FR` + `language=ar` is rejected with `language_not_supported_for_country` (422).

## Phone number format

**Strict E.164 only**. Examples : `+212612345678` (MA mobile), `+33612345678` (FR mobile). Validation rejects anything else with `validation_error` (422).

## Idempotency-Key

`POST /v1/calls` requires the header `Idempotency-Key`. Use a fresh UUID v4 per logical request.

For network retries (504, timeout) **reuse the same key** — the API replays the stored response with `X-Idempotent-Replay: true`. Different body + same key → `idempotency_key_conflict` (409).

## Call result model (3 main values + detail)

After a call ends, the `CallOut` and the `call.ended` webhook payload expose:

- **`result`** : `confirmed` / `cancelled` / `requires_action` (3 lowercase values).
  - `confirmed` = the order is confirmed (bill normally). May include a `result_detail=modified` if the customer asked for a change.
  - `cancelled` = the customer cancelled, OR `result_detail=wrong_number|denied_order` (treated as cancellation server-side).
  - `requires_action` = no automatic decision — the merchant must handle manually (callback, postponed date, unintelligible audio, no answer, etc.). Always check `result_detail` for the precise reason.
- **`result_detail`** : free-text slug. Common values : `modified`, `wrong_number`, `denied_order`, `human_requested`, `price_dispute`, `postponed`, `callback`, `unconfirmed`, `unclear`, `no_answer`, `failed`. `null` when `result` is `confirmed` or `cancelled` without nuance.
- **`customer_mood`** : `positive` / `neutral` / `negative` / `frustrated` / `unclear`, or `null`. `unclear` signals that the audio was globally unintelligible (typically paired with `result_detail=unclear`).
- **`flags`** : array of qualitative tags (e.g. `confirmed_by_relative`, `address_incomplete`, `audio_quality_bad`).
- **`preferences`** : array of customer demands (e.g. `["delivery Tuesday 2pm", "call before"]`).
- **`next_action`** : suggested follow-up for the merchant, or `null`.
- **`summary`** : 1-3 sentence summary of the conversation. May contain customer PII (name, address) as evoked during the call.
- **`recording_url`** : relative URL to download the call audio (mix of customer + agent). Path : `/v1/calls/{call_id}/recording`. Same Bearer auth as other `/v1` endpoints. `null` until the call ends, or after the 30-day retention window. Format : WAV mono 16 kHz 16-bit PCM (~32 KB/s, ~2 MB for 60s). Rate limit 60 req/min on this endpoint. Returns `410 Gone` with slug `recording_gone` when the audio has been purged.

## Common error slugs (RFC 7807)

| Slug | Status | When |
|------|--------|------|
| `validation_error` | 422 | Body fails schema |
| `language_not_supported_for_country` | 422 | `country=FR` + `language=ar` (only invalid combination) |
| `rate_limit_exceeded` | 429 | Per-key bucket exhausted |
| `insufficient_balance` | 402 | Reseller balance < 10s |
| `idempotency_key_conflict` | 409 | Same key, different body |
| `invalid_cursor` / `cursor_expired` | 400 | Pagination cursor malformed/expired (TTL 24h) |
| `call_not_found` | 404 | UUID invalid OR call belongs to another reseller OR recording not yet produced (anti-enum, byte-identical) |
| `recording_gone` | 410 | Call recording purged after the 30-day retention window |

For full Causes / Remediation / Example call the MCP tool `explain_error(slug)`.

## Webhooks (outbound notifications to your endpoint)

Yasmine signs every webhook with HMAC-SHA256 (header `X-Yasmine-Signature: sha256=<hex>`). Always verify before processing. Idempotency : the envelope `id` (`evt_<ULID>`) is unique per event ; deduplicate on it.

18 `call.*` event types organized in 3 rails (DEMAND 7 + TEMPLATE 4 + CALL 7) + `webhook.test` utility. Full catalog and payload schemas in `webhooks.md`.

## Discovery via MCP

The `yasmine` MCP server (declared in this plugin) exposes 8 tools :

- **Pass-through** (4) : `get_account`, `create_call`, `get_call`, `list_calls`
- **Introspection** (4) : `list_endpoints`, `get_endpoint_spec`, `get_changelog`, `explain_error`

Use `list_endpoints()` to discover the live API surface, `get_endpoint_spec(operation_id)` for full schemas, `get_changelog()` for what shipped recently, `explain_error(slug)` for structured error explanation.
