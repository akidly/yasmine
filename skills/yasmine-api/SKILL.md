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

## Shops — declare once, reference later

`POST /v1/shops` registers a shop with **your** identifier. Minimum:
`external_id`, `name`, `country` (`MA` | `DZ` | `TN` — no `FR`, we do not
sell French shops).

Declaring a shop here means you no longer resend its full `shop_info`
profile on every call request.

- `PATCH /v1/shops/{external_id}` — partial update. **Absent = unchanged,
  `null` = cleared, `""` = 422.** This differs from `shop_info` on
  `POST /v1/calls`, where nothing is ever cleared. Pick one door per shop.
- `DELETE /v1/shops/{external_id}` — **archives**, does not delete. Call
  history stays intact. Find it again with `?status=archived`.
- `GET /v1/shops?merchant_external_id=M-4471` — all shops of one merchant.
  `merchant_external_id` is a grouping key, not a resource: several shops
  share it, and the merchant has no attributes of its own.

`external_id` is immutable — it carries the shop's call history. Sending
it in a `PATCH` returns 422.

**Phone fields are validated against the shop country.** Local
(`0522123456`) or international (`+212522123456`) both accepted, but the
number must belong to the declared country. Stored and returned as E.164.

Errors: `409 shop_external_id_already_exists` on a duplicate create (use
`PATCH` to modify), `404 shop_not_found` — byte-identical whether the shop
is missing, archived, or owned by another account.

## Products — the same idea, one level down

`POST /v1/shops/{shop_external_id}/products` adds a product to a shop's
catalogue. Minimum: `external_id`, `name`, and a price. A product declared here can be
ordered by reference alone — `POST /v1/calls` reads its name, description,
price and variants from the catalogue instead of having them repeated.

**Two levels, as in any commerce system**: the product carries identity,
the variant carries the price. A product with no choices is no exception —
omit `variants` and one nameless variant is created for it, holding
`unit_price`. You never have to think about it: send a price, read a price
back.

- **The price is mandatory** — `unit_price` on a product with no choices,
  or on every entry of `variants`. A product without a price is rejected
  (422), and so is `unit_price: null` on a `PATCH`: a price is edited,
  never erased.
- **Several variants ⇒ each needs its own `external_id`**, otherwise an
  order cannot say which one. Rejected with 422 naming the offending
  entries.
- `PATCH` — same three-case semantics as shops. **But `variants` is
  replace-whole-list**: a variant missing from the new list is deleted.
  To change one, read the product, edit the entry, send the full list
  back. Omitting `variants` entirely leaves them untouched.
- `unit_price` alone is only valid on a single-variant product; on a
  multi-variant one it is ambiguous and returns 422.
- `DELETE` — archives. Past calls keep what they were told.

**Ordering from the catalogue.** In `order.items`, send
`product_external_id` (plus `variant_external_id` when the product has
several). Anything you also send wins over the catalogue — a promotional
price for that one order stays possible. An item with `name` and no
reference is still valid: that is the door for a custom item, shipping
fees or a gift, and an undeclared product must never block a call.

Errors: `409 product_external_id_already_exists` (**`external_id` is
unique across the whole account, all shops included** — the same id is
NOT free in another shop), `404 product_not_found`,
`422 order_item_unknown_product`, `422 order_item_variant_required` whose
`detail` lists the variants that do exist.

## Orders as a resource (declare, then call by reference)

`POST /v1/shops/{shop_external_id}/orders` declares an order under a shop
— the only call that names the shop, since a fresh reference cannot say
which shop it belongs to. Everything else lives at the root:
`GET/PATCH/DELETE /v1/orders/{order_external_id}`, and
`GET /v1/orders?shop_external_id=…` to filter a list. Products follow the
same shape
with **your** identifier. Minimum: `external_id`, `customer`, `amount`, and
either `items` or `items_text`. Same five routes as products: `POST`, `GET`
(paginated, `?order_status=` filter), `GET /{id}`, `PATCH`, `DELETE`
(archives).

Then call it by reference — `POST /v1/calls` with `shop_external_id`,
`order_external_id` and `call_params` only. The server reads the customer,
the frozen items and the amounts from the order and attaches the call to
it. To retry the customer later, send the same request with a fresh
`Idempotency-Key`: calls stack on the order, there is never a duplicate
order. Prefer this form over describing the order inline.

- **Items are frozen at declaration.** A line referencing a catalogue
  product is resolved then (name, price, variants, ids); a product archived
  afterwards does not block calls on past orders. `PATCH` with `items`
  replaces the whole list and refreezes it.
- **`order_status` is read-only** — it comes from call results. The order
  also exposes `calls` and `call_in_progress`.
- **While a call is in progress** `PATCH`, `DELETE` and a second call return
  `409 order_call_in_progress`.
- **`customer`, `amount`, `items` cannot be set to `null`** (422).
- **Open / closed is the merchant's axis**, separate from `order_status`
  (which comes from calls). `POST /v1/orders/{id}/close` with `reason`
  (`delivered`, `returned`, `cancelled`, `other`) when the merchant is done
  with an order; `POST /v1/orders/{id}/reopen` to undo. The order exposes
  `closed`, `closed_at`, `closed_reason`. `GET /v1/orders` hides closed
  orders by default (`?lifecycle=closed|all`). A closed order cannot be
  called: `POST /v1/calls` returns `409 order_closed` until it is reopened.
- Do not send `customer` with `order_external_id` — it is the order's
  customer; change it with `PATCH` on the order. Do not send
  `order.previous_attempts` either: the server counts past calls.

Errors: `404 order_not_found`, `409 order_call_in_progress`,
`409 order_external_id_already_exists` — an order `external_id` is unique
across the whole account, all shops included, so one reference always
designates one order. Variant `external_id` stays unique within its
product.

**Calling hours.** `call_hours` on the shop (default) and on the order
(override), in OpenStreetMap opening-hours syntax:
`Mo-Fr 09:30-12:00,13:00-19:00; Sa 13:00-19:00`. Never before 09:00 nor
after 21:00 in the shop's timezone — anything outside is rejected (422).
Without any, every day 09:00-21:00. A call requested outside the window
is accepted and dials at the next opening; retries are re-scheduled the
same way.

## Idempotency-Key

`POST /v1/calls` requires the header `Idempotency-Key`. Use a fresh UUID v4 per logical request.

For network retries (504, timeout) **reuse the same key** — the API replays the stored response with `X-Idempotent-Replay: true`. Different body + same key → `idempotency_key_conflict` (409).

## Call result model (3 main values + detail)

After a call ends, the `CallOut` and the `call.ended` webhook payload expose:

- **`shop_external_id` / `order_external_id`** (webhook payload only) : *your* references, the ones you gave when declaring the shop and the order. Match the event to your own order without keeping a lookup table of our `call_id`. Also present on `call.cancelled` and `call.failed`.
- **`result`** : `confirmed` / `cancelled` / `requires_action` (3 lowercase values).
  - `confirmed` = the order is confirmed (bill normally). May include a `result_detail=modified` if the customer asked for a change.
  - `cancelled` = the customer cancelled, OR `result_detail=wrong_number|denied_order` (treated as cancellation server-side).
  - `requires_action` = no automatic decision — the boutique must handle manually (callback, postponed date, unintelligible audio, no answer, etc.). Always check `result_detail` for the precise reason.
- **`result_detail`** : free-text slug. Common values : `modified`, `wrong_number`, `denied_order`, `human_requested`, `price_dispute`, `postponed`, `callback`, `unconfirmed`, `unclear`, `no_answer`, `failed`. `null` when `result` is `confirmed` or `cancelled` without nuance.
- **`customer_mood`** : `positive` / `neutral` / `negative` / `frustrated` / `unclear`, or `null`. `unclear` signals that the audio was globally unintelligible (typically paired with `result_detail=unclear`).
- **`flags`** : array of qualitative tags (e.g. `confirmed_by_relative`, `address_incomplete`, `audio_quality_bad`).
- **`preferences`** : array of customer demands (e.g. `["delivery Tuesday 2pm", "call before"]`).
- **`next_action`** : suggested follow-up for the boutique, or `null`.
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
