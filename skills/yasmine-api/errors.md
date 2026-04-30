# Format d'erreur API — RFC 7807

> 💡 **Référence détaillée** : chaque slug est documenté en détail (causes,
> correctifs, exemples) sur la page [errors.html](https://docs.yasmine.akidly.com/errors).
> Les liens `type` dans les réponses d'erreur pointent directement sur la bonne
> section (ex. `https://docs.yasmine.akidly.com/errors/insufficient_balance`
> redirige sur `#insufficient_balance`). Ce fichier-ci reste la **référence
> rapide** en Markdown (une ligne par slug).

Toutes les erreurs des endpoints `/v1/*` sont renvoyées au format
[RFC 7807 — Problem Details for HTTP APIs](https://datatracker.ietf.org/doc/html/rfc7807)
avec `Content-Type: application/problem+json`. Chaque erreur identifie sa
**classe** via un champ `type` stable — le SDK reseller route là-dessus,
pas sur le code HTTP seul.

## Exemple de payload

```http
HTTP/1.1 402 Payment Required
Content-Type: application/problem+json

{
  "type": "https://docs.yasmine.akidly.com/errors/insufficient_balance",
  "title": "Payment Required",
  "status": 402,
  "detail": "Solde disponible 3s < minimum facturable 10s.",
  "instance": "/v1/calls",
  "request_id": "a1b2c3d4",
  "balance_seconds": 3,
  "required_seconds": 10
}
```

## Champs standards

| Champ | Description |
|---|---|
| `type` | URI stable identifiant la classe d'erreur. Forme : `https://docs.yasmine.akidly.com/errors/{slug}`. Le slug est stable entre versions — router dessus côté SDK. |
| `title` | Titre court, lisible par un humain. |
| `status` | Code HTTP (identique à celui de la réponse). |
| `detail` | Message spécifique à cette occurrence. Peut inclure des valeurs réelles. |
| `instance` | Chemin URL de la requête qui a échoué. |
| `request_id` | Identifiant de la requête (8 caractères). À inclure dans tout ticket de support. |

## Extensions (champs optionnels par type)

Certains `type` ajoutent des champs dédiés :

- `validation_error` : `errors` = liste des erreurs Pydantic `[{loc, msg, type}]`.
- `insufficient_balance` : `balance_seconds`, `required_seconds`.
- `rate_limit_exceeded` : header `Retry-After` + champ `retry_after` (secondes). Aussi `X-RateLimit-Limit`, `X-RateLimit-Remaining: 0`, `X-RateLimit-Reset` (unix timestamp).

## Table des `type` actuels

| Slug | Status | Déclenché quand |
|---|---|---|
| `bad_request` | 400 | Slug par défaut pour tout `400` qui n'override pas son type. En pratique : la plupart des 400 spécifiques utilisent un slug ciblé (`missing_idempotency_key`, `idempotency_key_*`, `invalid_cursor`, etc.). Si vous voyez `bad_request` brut, c'est un cas non-mappé — lire `detail` pour le diagnostic. |
| `invalid_api_key` | 401 | Header `Authorization` absent, malformé, ou clé invalide/révoquée. Le header `WWW-Authenticate: Bearer` est propagé. |
| `insufficient_balance` | 402 | Solde reseller < minimum facturable (10 s). |
| `forbidden` | 403 | Clé valide mais sans droit sur la ressource. |
| `not_found` | 404 | Ressource absente ou non accessible au reseller courant. |
| `conflict` | 409 | Conflit générique (cf slugs spécifiques `idempotency_key_conflict`, `webhook_already_configured`). |
| `missing_idempotency_key` | 400 | Header `Idempotency-Key` absent sur `POST /v1/calls`. **Obligatoire** depuis P0-1. Générer une clé unique par requête (UUID v4 recommandé). |
| `idempotency_key_empty` | 400 | Header `Idempotency-Key` présent mais vide. Doit faire au moins 1 char. |
| `idempotency_key_too_long` | 400 | Header `Idempotency-Key` > 255 chars. Limite stricte côté serveur. |
| `idempotency_key_conflict` | 409 | Même `Idempotency-Key` réutilisée avec un body différent dans la fenêtre de 24 h. Générer une nouvelle clé pour une requête différente. Le body original n'est jamais ré-exposé. |
| `order_external_id_already_exists` | 409 | `POST /v1/calls` avec `order.external_id` déjà utilisé pour ce merchant. Les commandes sont créées par INSERT strict (pas d'upsert) — pour atteindre à nouveau le client sur la même commande, relancer avec une nouvelle `Idempotency-Key` et incrémenter `order.previous_attempts`. Pour créer une commande distincte, utiliser une `external_id` différente. |
| `api_key_not_found` | 404 | `DELETE /v1/me/api-keys/{key_id}` sur une clé inexistante OU appartenant à un autre reseller (P1-4). Body byte-identique sur les 2 cas — pattern anti-énum P0-2. |
| `invalid_key_id` | 400 | `DELETE /v1/me/api-keys/{key_id}` avec UUID malformé (P1-4). Se distingue du 404 pour aider le client à diagnostiquer un bug local. |
| `webhook_url_not_configured` | 400 | `POST /v1/me/webhooks/test` sans webhook actif (P1-2). Configurer d'abord via `POST /v1/me/webhooks`. |
| `invalid_status_filter` | 400 | `GET /v1/me/webhooks/deliveries?status=...` avec valeur hors `delivered`/`failed`/`pending` (P1-2). Le `detail` liste les valeurs acceptées. |
| `invalid_period_format` | 400 | `GET /v1/me/usage?period=...` avec format non-`YYYY-MM` ou mois hors 01-12 (P1-1). Exemple : `?period=abc`, `?period=2026-13`, `?period=2026-00`. Remédiation : utiliser le format strict ou omettre (défaut = mois courant UTC). |
| `invalid_cursor` | 400 | `GET /v1/calls?cursor=...` ou `GET /v1/me/transactions?cursor=...` avec curseur malformé, signature tampered, ou `CURSOR_SIGNING_KEY` tournée côté serveur (P1-3). Le client refait la 1re requête sans `cursor`. |
| `cursor_expired` | 400 | Curseur > 24 h (TTL fixe, P1-3). Même remédiation : refaire la 1re requête sans `cursor`. |
| `payload_too_large` | 413 | Émis sur `POST /webhooks/whatsapp` quand le body dépasse 256 KiB (#P0-6). En pratique Meta envoie ~50 KB. Si vous voyez ce code côté Meta retry storm, contacter le support — c'est probablement un payload falsifié ou un bug Meta inattendu. **Pas applicable aux endpoints `/v1/*`** (qui ont leur propre validation Pydantic). |
| `validation_error` | 422 | Payload mal formé. Voir `errors`. |
| `language_not_supported_for_country` | 422 | `POST /v1/calls` avec une combinaison `(country, language)` non disponible. Aujourd'hui : `country=FR` + `language=ar`. Combinaisons supportées : `MA`/`DZ`/`TN` avec `ar` ou `fr`, `FR` uniquement avec `fr`. Omettre `language` pour appliquer la langue par défaut du pays. |
| `rate_limit_exceeded` | 429 | Rate-limit par clé API dépassé (M3.6 C7). Seuils par endpoint : 60/min POST calls, 120/min cancel, 600/min reads, 10/min config webhooks. Respecter `Retry-After` avant de retenter. Cf `docs/getting-started.md` §Rate limits. |
| `rate_limited` | 429 | Slug par défaut pour les 429 qui ne passent pas par le rate-limiter principal `/v1/*` (cas marginal — un 429 émis directement via `HTTPException(429)` sans override). En pratique vous verrez surtout `rate_limit_exceeded`. Même remédiation : respecter `Retry-After`. |
| `call_not_found` | 404 | POST `/v1/calls/{id}/cancel` sur un `call_id` inexistant ou appartenant à un autre reseller (M3.6 C8). Pas de leak d'existence. Émis aussi par GET `/v1/calls/{id}/recording` quand l'audio n'a pas été produit (call jamais finalisé) ou pour les mêmes raisons de scoping. |
| `recording_gone` | 410 | GET `/v1/calls/{id}/recording` sur un appel dont l'enregistrement audio a été purgé après la fenêtre de rétention (30 jours). Distinct de `call_not_found` pour permettre au reseller de différencier « jamais existé » vs « a existé mais purgé ». |
| `webhook_url_rejected` | 400 | POST `/v1/me/webhooks` avec URL rejetée par le SSRF guard. Champ `reason` parmi `invalid_url`/`scheme_not_allowed`/`localhost_rejected`/`dns_resolution_failed`/`private_ip_rejected`. |
| `webhook_already_configured` | 409 | POST `/v1/me/webhooks` sur reseller déjà configuré. DELETE d'abord pour rotate. |
| `webhook_not_configured` | 404 | GET/DELETE `/v1/me/webhooks` sans config active. |
| `service_unavailable` | 503 | Dépendance externe indisponible ou config manquante côté Yasmine. |
| `internal_error` | 500 | Erreur inattendue (bug). Toujours loguée côté Yasmine avec `request_id`. |
| `http_error` | autres 4xx/5xx | Fallback générique si le code HTTP n'est pas mappé. Ne pas dépendre de ce slug. |

## Compatibilité

Les URLs `https://docs.yasmine.akidly.com/errors/{slug}` sont **résolvables
depuis P1-10** : elles redirigent (302) vers la bonne section de la page
[errors.html](https://docs.yasmine.akidly.com/errors). Le slug reste
l'identifiant stable à utiliser côté SDK pour router une classe d'erreur
— l'URL humaine est un bonus.

## Hors `/v1/`

Les endpoints hors `/v1/*` (actuellement uniquement les webhooks Meta
`POST /webhooks/whatsapp`) gardent le format legacy FastAPI :

```json
{"detail": "invalid signature"}
```

avec `Content-Type: application/json`. Ce format ne bouge pas — Meta
n'interprète pas le body, seul le code HTTP compte. Le changement
RFC 7807 est **scopé** au préfixe `/v1/` pour ne pas casser les
webhooks déjà en prod.

## Slugs enrichis pour usage agent (MCP `explain_error`)

Sections détaillées pour les 5 slugs les plus rencontrés côté reseller.
Le tool MCP `explain_error` parse ce bloc en priorité, et tombe sur la
ligne brute du tableau ci-dessus pour les autres slugs. Format stable,
parsé par regex (voir `yasmine_mcp/introspection.py`).

### validation_error
**Status** : 422
**Causes** :
- Champ obligatoire manquant dans le body JSON.
- Type incorrect (ex. `amount` envoyé en number au lieu de string).
- Valeur hors enum (ex. `country: "ZZ"` alors que seuls `MA`, `DZ`, `TN`, `FR` sont autorisés).
- Format incorrect (UUID malformé, ISO date invalide, regex pattern non matché).

**Remediation** :
- Valider le body côté client avant envoi (générateur OpenAPI, Pydantic, JSON Schema).
- Lire le tableau `errors[]` dans la réponse pour le détail par champ : `errors[i].loc` = chemin (ex. `["body", "customer", "phone_number"]` pour un champ nested), `errors[i].msg` = message Pydantic, `errors[i].type` = code Pydantic.
- Sur les enums, prendre la valeur exacte depuis `docs/openapi.yaml` (les `enum:` sont normatifs).

**Example** :
```json
{
  "type": "https://docs.yasmine.akidly.com/errors/validation_error",
  "title": "Unprocessable Entity",
  "status": 422,
  "detail": "Un ou plusieurs champs du payload sont invalides.",
  "instance": "/v1/calls",
  "request_id": "a1b2c3d4",
  "errors": [
    {"loc": ["body", "customer", "phone_number"], "msg": "field required", "type": "missing"},
    {"loc": ["body", "country"], "msg": "value is not a valid enumeration member", "type": "enum"}
  ]
}
```

### rate_limit_exceeded
**Status** : 429
**Causes** :
- Limite par minute dépassée sur la clé API. Seuils par groupe d'endpoint : `60/min` sur `POST /v1/calls`, `120/min` sur `POST /v1/calls/{id}/cancel`, `600/min` sur les reads (`GET /v1/calls`, `GET /v1/me/*`), `10/min` sur les configs webhooks.
- Bursts non lissés côté client (script qui itère sans pause).
- Plusieurs services partagent la même clé sans coordination.

**Remediation** :
- Respecter le header `Retry-After` (secondes) avant de retenter — c'est la valeur autoritaire.
- Lire `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` (timestamp Unix) à chaque réponse 2xx pour anticiper.
- Si la clé est partagée par plusieurs workers, créer une clé par worker (`POST /v1/me/api-keys`) — chaque clé a son bucket indépendant.
- Voir `docs/getting-started.md §Rate limits`.

**Example** :
```json
{
  "type": "https://docs.yasmine.akidly.com/errors/rate_limit_exceeded",
  "title": "Too Many Requests",
  "status": 429,
  "detail": "Limite de frequence depassee (60 per 1 minute). Reessayer dans 23 seconde(s).",
  "instance": "/v1/calls",
  "request_id": "a1b2c3d4",
  "retry_after": 23
}
```

### insufficient_balance
**Status** : 402
**Causes** :
- Solde reseller en secondes < minimum facturable (10 s).
- Premier appel après expiration d'un crédit.
- Top-up oublié sur compte de prod.

**Remediation** :
- **Ne pas retry aveuglément** : le solde reste vide, le 402 sera identique. Aucun appel n'est créé, donc pas de double-facturation à craindre — mais bouclage inutile.
- Consulter le solde courant : `GET /v1/me/balance` (P0-2, live).
- Top-up via le support Yasmine.
- La réponse contient les champs étendus `balance_seconds` et `required_seconds` pour logger le delta exact.

**Example** :
```json
{
  "type": "https://docs.yasmine.akidly.com/errors/insufficient_balance",
  "title": "Payment Required",
  "status": 402,
  "detail": "Solde disponible 3s < minimum facturable 10s.",
  "instance": "/v1/calls",
  "request_id": "b3c4d5e6",
  "balance_seconds": 3,
  "required_seconds": 10
}
```

### order_external_id_already_exists
**Status** : 409
**Causes** :
- `POST /v1/calls` avec un `order.external_id` déjà utilisé pour le même merchant. Les commandes sont créées en INSERT strict (pas d'upsert) depuis Phase 2-final.
- Retry d'une commande sans réutiliser la même `Idempotency-Key` (ce qui aurait déclenché un replay de la réponse stockée).
- Deux services du reseller qui partagent le même pool d'`external_id` sans coordination.

**Remediation** :
- Pour **retenter sur la commande existante** : soumettre un nouveau POST avec une nouvelle `Idempotency-Key` + incrémenter `order.previous_attempts`. La même `external_id` reste interdite — à l'avenir une route `POST /v1/calls/retry?order_id=...` dédiée est prévue (M5).
- Pour **créer une commande distincte** : générer une nouvelle `external_id` unique côté reseller.
- Pour un **retry réseau pur** : garder la **même** `Idempotency-Key` — déclenche un replay (bit-for-bit) sans nouvelle INSERT.

**Example** :
```json
{
  "type": "https://docs.yasmine.akidly.com/errors/order_external_id_already_exists",
  "title": "Order external_id already exists",
  "status": 409,
  "detail": "An order with this external_id already exists for this merchant.",
  "instance": "/v1/calls",
  "request_id": "a1b2c3d4",
  "hint": "To retry reaching the customer on this existing order, submit a new POST with a fresh Idempotency-Key and increment order.previous_attempts. To create a distinct order, use a unique external_id."
}
```

### idempotency_key_conflict
**Status** : 409
**Causes** :
- Même valeur `Idempotency-Key` réutilisée dans la fenêtre 24 h avec un body JSON différent.
- Clé hardcodée par erreur dans le code (ex. `"order-2026-001"` réutilisée pour tous les appels).
- Retry après modification du payload (changement de numéro, de montant) sans changer la clé.

**Remediation** :
- Générer une **nouvelle UUID v4** par requête fonctionnellement différente (`uuid.uuid4()` ou équivalent).
- Pour un retry réseau pur (timeout, 504 Cloudflare), **garder la même clé** — c'est exactement le cas d'usage idempotence (renvoie la réponse stockée bit-for-bit avec `X-Idempotent-Replay: true`).
- Le body original n'est jamais ré-exposé pour des raisons de sécurité (évite leak entre requêtes différentes).

**Example** :
```json
{
  "type": "https://docs.yasmine.akidly.com/errors/idempotency_key_conflict",
  "title": "Conflict",
  "status": 409,
  "detail": "Idempotency-Key 'order-cmd-2026-789' a deja ete utilisee avec un body different. Generer une nouvelle cle pour une requete differente.",
  "instance": "/v1/calls",
  "request_id": "a1b2c3d4"
}
```

### invalid_cursor
**Status** : 400
**Causes** :
- Curseur opaque malformé (caractères tronqués, copy-paste partiel).
- Signature HMAC tampered (curseur modifié manuellement côté client).
- `CURSOR_SIGNING_KEY` rotée côté serveur (tous les curseurs antérieurs invalidés).
- Confusion entre `invalid_cursor` (format) et `cursor_expired` (TTL > 24 h dépassé) — slug différent, même remédiation.

**Remediation** :
- Refaire la **première requête sans paramètre `cursor`** pour repartir d'un curseur frais.
- Ne **jamais parser** le contenu du curseur côté client — c'est un opaque token signé HMAC-SHA256.
- Si l'erreur arrive systématiquement après une certaine date, c'est probablement une rotation de clé serveur — re-pagineriser tout l'historique.

**Example** :
```json
{
  "type": "https://docs.yasmine.akidly.com/errors/invalid_cursor",
  "title": "Bad Request",
  "status": 400,
  "detail": "Cursor payload corrupted: invalid signature",
  "instance": "/v1/calls",
  "request_id": "a1b2c3d4"
}
```
