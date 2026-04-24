# Webhooks Yasmine

Guide pour configurer et recevoir les notifications sortantes Yasmine.

> Ce document décrit les webhooks **sortants** émis par Yasmine vers l'endpoint reseller. Les webhooks entrants (opérateurs de messagerie vers Yasmine) sont une implémentation interne et ne sont pas documentés ici.

> **Statut M3.6 C6** :
> - **Phase 1 livrée (2026-04-20)** : `POST/GET/DELETE /v1/me/webhooks` self-service. Un reseller peut enregistrer son endpoint et récupérer son secret HMAC.
> - **Phase 2 livrée (2026-04-20)** : émission réelle des events (10 events `call.*`) via dispatcher async + retry 3× (0s / 30s / 5min), timeout 10 s, SSRF re-check à chaque POST.

---

## 1. Configuration (live)

### 1.1 Créer

```bash
curl -X POST https://api.yasmine.akidly.com/v1/me/webhooks \
  -H "Authorization: Bearer yk_..." \
  -H "Content-Type: application/json" \
  -d '{"url":"https://monshop.com/yasmine-webhooks"}'
```

Réponse `201` :
```json
{
  "url": "https://monshop.com/yasmine-webhooks",
  "secret": "AbCdEfGhIj...1234",
  "created_at": "2026-04-20T16:00:00Z"
}
```

**Le `secret` n'est affiché qu'une seule fois.** Le stocker immédiatement — aucune route ne permet de le récupérer ensuite (cf `GET` ci-dessous qui n'expose qu'un préfixe).

### 1.2 Consulter

```bash
curl https://api.yasmine.akidly.com/v1/me/webhooks \
  -H "Authorization: Bearer yk_..."
```

Réponse `200` :
```json
{
  "url": "https://monshop.com/yasmine-webhooks",
  "secret_prefix": "AbCdEf…",
  "active": true,
  "created_at": "2026-04-20T16:00:00Z"
}
```

### 1.3 Désactiver

```bash
curl -X DELETE https://api.yasmine.akidly.com/v1/me/webhooks \
  -H "Authorization: Bearer yk_..."
```

Réponse `204 No Content`. Soft-delete : la config passe en `active=false`, l'historique `webhook_deliveries` reste intact.

### 1.4 Rotation du secret (P1-8)

Utiliser l'endpoint dédié — **pas besoin** de delete/recreate :

```bash
curl -X POST https://api.yasmine.akidly.com/v1/me/webhooks/rotate-secret \
  -H "Authorization: Bearer yk_..."
```

Réponse `200` :
```json
{
  "secret": "whsec_tAunISmhee9BvK-PpHE5GfXZp_0WnMJwZB6SRbZUpng",
  "secret_prefix": "whsec_tAunIS…",
  "rotated_at": "2026-04-21T12:00:00.000Z"
}
```

**Caractéristiques** :
- Format secret : `whsec_` + 43 chars base64url (entropie 256 bits). Visible **une seule fois** dans la réponse — aucun endpoint ne la ré-expose. Perte = re-rotation.
- Atomique : l'ancien secret ne signe plus aucun event dès le retour 200. Le dispatcher relit `webhook.secret` à chaque POST, pas de cache in-memory.
- `404 webhook_not_configured` si aucun webhook actif (configurer d'abord via `POST /v1/me/webhooks`).
- Rate-limit **5/min** (aligné sur `POST /v1/me/api-keys`) — au-delà → 429 RFC 7807.

**Legacy** : si vous préférez le pattern delete/recreate historique, il reste supporté :

```bash
curl -X DELETE https://api.yasmine.akidly.com/v1/me/webhooks -H "Authorization: Bearer yk_..."
curl -X POST   https://api.yasmine.akidly.com/v1/me/webhooks -H "Authorization: Bearer yk_..." \
  -H "Content-Type: application/json" -d '{"url":"https://monshop.com/yasmine-webhooks"}'
```

mais `rotate-secret` est préféré (sans indisponibilité entre les 2 appels, préserve l'URL, pas de 409 à gérer).

### 1.5 URLs rejetées (SSRF guard)

Retour `400 webhook_url_rejected` pour :
- scheme autre que `https` (en prod) ;
- `localhost`, `*.local` ;
- IPs privées (RFC 1918 `10.*`/`172.16-31.*`/`192.168.*`, loopback `127.*`, link-local `169.254.*`, IPv6 `::1`/`fe80::/10`/`fc00::/7`) ;
- hostnames qui ne résolvent pas DNS.

Le payload d'erreur inclut `reason` parmi : `invalid_url`, `scheme_not_allowed`, `localhost_rejected`, `dns_resolution_failed`, `private_ip_rejected`.

---

## 2. Modèle d'emission (Phase 2)

À chaque événement, Yasmine enverra une requête `POST` vers l'URL configurée :

```http
POST /your/webhook/endpoint HTTP/1.1
Host: your-app.example.com
Content-Type: application/json
User-Agent: Yasmine-Webhooks/1.0
X-Yasmine-Signature: sha256=5257a869e7ecebeda32affa62cdca3fa51cad7e77a0e56ff536d0ce8e108d8bd
X-Yasmine-Event: call.ended
X-Yasmine-Event-Id: evt_01HTYE9Q4QK3M1B5X7Z2V8W6RF

{
  "id": "evt_01HTYE9Q4QK3M1B5X7Z2V8W6RF",
  "type": "call.ended",
  "created_at": "2026-04-19T14:40:00.123Z",
  "livemode": true,
  "data": {
    "object": {
      "id": "call_01HTYE9...",
      "status": "ended",
      "result": "CONFIRMED",
      "duration_s": 37,
      "billable_s": 37,
      "order": { "id": "ord_...", "external_id": "CMD-2026-001" },
      "merchant": { "id": "mer_...", "external_id": "boutique-x" }
    }
  }
}
```

**Réponse attendue** : `2xx` dans les **10 secondes**. Tout autre code ou timeout déclenche un retry (cf §4).

---

## 3. Enveloppe standard

| Champ | Type | Description |
|---|---|---|
| `id` | `string` | ID unique de l'event. Préfixé `evt_`. ULID. |
| `type` | `string` | Nom de l'event (cf catalogue Phase 2). |
| `created_at` | `string` (ISO 8601) | Date d'émission côté Yasmine. |
| `livemode` | `boolean` | `true` en prod. Sandbox séparé = dette future (P1). |
| `data.object` | `object` | Ressource principale concernée par l'event. |

L'ordre d'arrivée **n'est pas garanti** (retries réseau). Utilise `created_at` + `id` pour ordonner côté reseller.

---

## 4. Politique de retry (Phase 2)

Décision produit : 3 tentatives, cadence alignée Stripe/Twilio.

| Tentative | Délai depuis la précédente | Timeout HTTP |
|---|---|---|
| 1 | immédiat (à la transition d'état) | 10 s |
| 2 | +30 s | 10 s |
| 3 | +5 min | 10 s |

Après **3 échecs consécutifs**, l'event est abandonné (`error='max_retries_exceeded'` dans `webhook_deliveries`). Un endpoint de replay manuel est prévu en P2.

Un event est considéré **livré** dès qu'un `2xx` est reçu. Tout `3xx`, `4xx`, `5xx`, timeout, ou erreur DNS/TLS = échec → retry.

### Tester votre endpoint (P1-2)

Avant d'attendre un vrai event en prod, vous pouvez forcer Yasmine à POSTer un `webhook.test` synchrone vers votre URL :

```bash
curl -X POST -H "Authorization: Bearer $YK" \
  https://api.yasmine.akidly.com/v1/me/webhooks/test
# {
#   "test_id": "...",
#   "delivered": true,
#   "http_status": 200,
#   "latency_ms": 142,
#   "target_url": "https://votre-endpoint.com/yasmine",
#   "attempted_at": "2026-04-21T12:00:00Z",
#   "error_message": null
# }
```

Le payload reçu côté votre endpoint :

```json
{
  "id": "evt_test_<uuid>",
  "type": "webhook.test",
  "created_at": "2026-04-21T12:00:00Z",
  "livemode": true,
  "data": {
    "test": true,
    "test_id": "...",
    "request_id": "..."
  }
}
```

Filtrez côté votre code sur `data.test === true` pour ignorer ces événements en logique métier.

**Timeout 5 s, pas de retry**, **rate-limit 10/min** (cumulé avec 600/min global). Toutes les tentatives sont tracées dans `webhook_deliveries` avec `event_type='webhook.test'` (visibles via `GET /v1/me/webhooks/deliveries`).

### Auditer vos livraisons (P1-2)

```bash
# Liste paginée des dernières livraisons (tous events, défaut 50, max 200).
curl -H "Authorization: Bearer $YK" \
  "https://api.yasmine.akidly.com/v1/me/webhooks/deliveries?limit=20"

# Filtrer sur les échecs uniquement.
curl -H "Authorization: Bearer $YK" \
  "...?status=failed"

# Sur un type d'event spécifique.
curl -H "Authorization: Bearer $YK" \
  "...?event_type=call.ended&since=2026-04-01T00:00:00Z"

# Page suivante.
curl -H "Authorization: Bearer $YK" \
  "...?cursor=eyJkaXIi..."
```

11 champs exposés par ligne : `delivery_id`, `event_type`, `target_url`, `http_status`, `latency_ms`, `attempt_count`, `status` (`delivered`/`failed`/`pending`), `next_retry_at`, `created_at`, `last_attempted_at`, `error_message` (tronqué 200 chars). Anti-BOPLA strict : ni `response_body`, ni `signature_sent`, ni headers internes.

### Garde-fous techniques côté Yasmine (P0-5)

Transparent pour le reseller — documenté pour information.

- **Pas de suivi de redirect** (`follow_redirects=False` httpx). Un `3xx` est traité comme un échec → retry. Si votre endpoint répond en 3xx vers une autre URL, configurez l'URL finale directement via `POST /v1/me/webhooks` au lieu d'attendre que Yasmine la suive.
- **Body response borné à 1024 bytes**. Yasmine lit votre réponse en streaming et coupe le flux dès 1024 B reçus. Inutile de renvoyer du contenu volumineux — un `200 OK` vide suffit. Le body tronqué est stocké dans `webhook_deliveries.response_body` côté Yasmine pour debug ops.
- **Anti-GC sur retries** : nos tentatives 2/3 (à +30 s / +5 min) sont protégées par un `set[asyncio.Task]` module-level, donc elles arrivent même si le process est sous charge.

---

## 5. Signature HMAC

Yasmine signe chaque webhook avec le secret généré à la création.

### Header

```
X-Yasmine-Signature: sha256=<hex_hmac_sha256>
```

Où `<hex>` = `HMAC_SHA256(secret, raw_body_bytes)`, lowercase hex.

### Vérification — Python

```python
import hashlib
import hmac

WEBHOOK_SECRET = "..."  # récupéré une seule fois via POST /v1/me/webhooks

def verify(secret: str, body: bytes, sig_header: str) -> bool:
    expected = "sha256=" + hmac.new(
        secret.encode(), body, hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, sig_header)

# FastAPI
from fastapi import FastAPI, Request, HTTPException

app = FastAPI()

@app.post("/webhooks/yasmine")
async def receive(request: Request):
    raw = await request.body()
    sig = request.headers.get("X-Yasmine-Signature", "")
    if not verify(WEBHOOK_SECRET, raw, sig):
        raise HTTPException(status_code=401, detail="invalid signature")
    # traiter l'event de façon idempotente (cf §6)
    return {"received": True}
```

### Vérification — Node.js

```javascript
import crypto from "node:crypto";

const WEBHOOK_SECRET = "...";

function verify(secret, bodyRaw, sigHeader) {
  const expected =
    "sha256=" +
    crypto.createHmac("sha256", secret).update(bodyRaw).digest("hex");
  const a = Buffer.from(expected);
  const b = Buffer.from(sigHeader);
  return a.length === b.length && crypto.timingSafeEqual(a, b);
}

// Express — attention au raw body
import express from "express";
const app = express();
app.post(
  "/webhooks/yasmine",
  express.raw({ type: "application/json" }),
  (req, res) => {
    const sig = req.headers["x-yasmine-signature"] || "";
    if (!verify(WEBHOOK_SECRET, req.body, sig)) {
      return res.status(401).send("invalid signature");
    }
    const event = JSON.parse(req.body);
    // traiter de façon idempotente
    res.json({ received: true });
  }
);
```

---

## 6. Idempotence côté reseller (dédupliquer les retries)

Un event **peut arriver plusieurs fois** en cas de timeout/5xx côté votre endpoint : Yasmine retente jusqu'à 3 fois (backoff 0 s / 30 s / 5 min, cf §4). **Les retries portent le même `X-Yasmine-Event-Id`** — utilisez-le pour dédupliquer. L'`id` dans le body est la source de vérité (signé par HMAC §5, non-altérable par un MITM) ; le header `X-Yasmine-Event-Id` est la même valeur, répliquée pour les outils qui ne veulent pas parser le JSON.

**Option Redis** (production-safe, O(1)) :

```python
def handle_webhook(request):
    event_id = request.headers["X-Yasmine-Event-Id"]
    # SET NX EX 86400 : 1re écriture réussit, les suivantes retournent None.
    if redis.set(f"yasmine:event:{event_id}", "1", nx=True, ex=86400) is None:
        return Response(status=200)  # déjà traité, idempotent, pas de retraitement
    # ... traitement métier une seule fois
```

**Option DB** (sans Redis) — UNIQUE INDEX sur la colonne `event_id` :

```sql
CREATE UNIQUE INDEX ON webhook_events (event_id);
```

```python
try:
    db.insert(webhook_events, event_id=event_id, received_at=now())
except UniqueViolation:
    return Response(status=200)  # déjà reçu
```

**Toujours répondre `200 OK` rapidement** sur un duplicate — sinon Yasmine continuera de retenter et polluera vos logs. TTL recommandé : 24 h (> fenêtre maximale de retry 5 min × 3 tentatives, avec marge).

---

## 7. Catalogue d'événements (live depuis Phase 2)

10 events. Payloads `data.*` ci-dessous. Envelope standard cf §3.

### Rail DEMANDE (vie du template WhatsApp)

#### `call.request.accepted`
Déclenché quand l'utilisateur a cliqué **Autoriser** sur le template `call_permission_request`. La permission est acquise, l'appel va partir.

```json
{
  "call_id": "d5a97d2b-...",
  "permission_type": "permanent",
  "expires_at": null
}
```
- `permission_type` : `permanent` (autorisation permanente) ou `temporary` (7 j glissants).
- `expires_at` : ISO 8601 Z si `temporary`, `null` si `permanent`.

#### `call.request.refused`
L'utilisateur a cliqué **Ne pas autoriser**. Aucune facturation. Aucun event rail APPEL ne suivra.

```json
{ "call_id": "d5a97d2b-..." }
```

#### `call.request.expired`
Timeout template (180 s par défaut) sans réponse utilisateur. Aucun event rail APPEL ne suivra.

```json
{
  "call_id": "d5a97d2b-...",
  "reason": "user_timeout"
}
```

#### `call.request.quota_blocked`
Yasmine a bloqué l'envoi template pour protéger le compte Meta (guards proactifs §4.2 / §4.3).

```json
{
  "call_id": "d5a97d2b-...",
  "reason": "recipient_not_authorized",
  "limit_type": "1_per_24h"
}
```
- `limit_type` parmi :
  - `1_per_24h` : 1 `call_permission_request` déjà envoyé au même `wa_id` dans les 24 h.
  - `2_per_7d` : 2 dans les 7 j glissants.
  - `4_no_answer_revoked` : 4 `NO_ANSWER` consécutifs → Yasmine anticipe la révocation auto côté Meta.

**Contrainte privacy (M3.6 §1.0 P7)** : ce payload ne leak jamais quel marchand a consommé le quota, ni l'énumération des calls précédents. C'est une info interne Yasmine. Si le reseller demande pourquoi, la réponse est générique : « ce client n'a pas autorisé les appels automatisés sur notre réseau pour le moment ».

#### `call.request.permission_granted_late`
Cas spécial : l'utilisateur a cliqué **Autoriser** APRÈS que le rail DEMANDE soit déjà terminal (expired / refused / cancelled). L'appel actuel ne reprend pas, mais la permission est désormais acquise.

```json
{
  "call_id": "d5a97d2b-...",
  "permission_type": "temporary",
  "expires_at": "2026-04-27T14:40:00.000000Z",
  "hint": "re-submit POST /v1/calls for same customer to dial immediately"
}
```

### Rail APPEL (vie de l'appel téléphonique)

#### `call.started`
Transition `call_status → dialing`. Meta a accepté le POST `/calls` SDP offer, l'appel est en cours de composition.

```json
{ "call_id": "d5a97d2b-..." }
```

#### `call.ringing`
Transition `call_status → ringing`. Le téléphone du client sonne.

```json
{ "call_id": "d5a97d2b-..." }
```

#### `call.connected`
Transition `call_status → connected`. Le client a décroché, le pipeline IA démarre.

```json
{
  "call_id": "d5a97d2b-...",
  "accepted_at": "2026-04-20T16:28:20.485030Z"
}
```

#### `call.ended`
Transition `call_status → ended`. Fin d'appel normale. Facturation appliquée.

```json
{
  "call_id": "d5a97d2b-...",
  "result": "CONFIRMED",
  "duration_s": 37,
  "billable_s": 37
}
```
- `result` parmi `CONFIRMED`, `CANCELLED`, `NO_ANSWER`, `VOICEMAIL`, `BUSY`, `UNCLEAR`, `FAILED`.
- `duration_s` : durée totale de l'appel côté Meta.
- `billable_s` : secondes décomptées du solde reseller.

#### `call.cancelled`
**M3.6 C8** — émis quand le reseller appelle `POST /v1/calls/{id}/cancel`. Distinct de `call.ended` (fin naturelle) et `call.failed` (échec infra). Facturation ajustée sur le payload même.

```json
{
  "call_id": "d5a97d2b-...",
  "cancelled_at": "2026-04-20T14:00:16.123456Z",
  "cancelled_state": "connected",
  "billed_seconds": 25,
  "merchant_id": "a5797a0a-..."
}
```
- `cancelled_state` : `call_status` au moment où le cancel a été traité (`not_started`/`dialing`/`ringing`/`connected`). Permet au reseller de savoir à quel stade l'appel a été coupé.
- `billed_seconds` : 0 si cancel avant `connected` ; sinon `ceil(now - accepted_at)` avec plancher 10 s.

**Pas d'event ré-émis** si le cancel est idempotent (appel déjà terminal au moment de la requête).

#### `call.failed`
Transition `call_status → failed`. Échec infra, **aucune facturation**.

```json
{
  "call_id": "d5a97d2b-...",
  "failure_reason": "meta_500"
}
```
- `failure_reason` parmi (non exhaustif) :
  - `client_rejected` : client a `reject`é avant d'accepter (webhook Meta).
  - `meta_<status>[:<code>]` : erreur Meta (ex. `meta_500`, `meta_400:138008`).
  - `run_origination_unexpected:<ExcType>` : crash non-prévu dans le handler origination.
  - `resumed_after_restart` : call marqué échoué au reclaim lifespan (process a redémarré pendant que ce call était en cours).
  - `janitor_projection` : le janitor a réconcilié un état d'échec via un GET Meta.

---

## 8. Règle d'ordonnancement

Si le rail DEMANDE termine en échec (`call.request.{refused,expired,quota_blocked}`), **aucun event** du rail APPEL ne sera émis — la demande ne s'est jamais traduite en POST `/calls` Meta. Respecté structurellement côté Yasmine.

**Distinction `call.cancelled` vs `call.ended` vs `call.failed`** :
- `call.ended` = l'appel s'est terminé naturellement (client a raccroché, IA a conclu, timeout normal Meta). `result` défini (CONFIRMED/CANCELLED/NO_ANSWER/...).
- `call.cancelled` = le reseller a explicitement appelé `POST /v1/calls/{id}/cancel`. Toujours volontaire côté Yasmine.
- `call.failed` = échec infra (Meta 5xx, reclaim lifespan, projection janitor, crash origination). Aucune facturation.

Un call ne peut émettre **qu'un seul** de ces 3 events — ils sont mutuellement exclusifs (états terminaux distincts de `call_status`).

---

## 9. Adresses sources

Les webhooks sont émis depuis l'IP du VPS Yasmine. Recommandation : whitelister cette IP en entrée côté reseller (IP communiquée à la mise en service).
