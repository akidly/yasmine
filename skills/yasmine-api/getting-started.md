# Getting Started — API Yasmine

Yasmine est une API qui déclenche des **appels vocaux IA sortants** (confirmation de commande, relance, livraison, annulation) vers vos clients finaux. Vous envoyez un `POST /v1/calls`, un agent conversationnel appelle le numéro, et vous recevez le résultat.

---

## 1. Prérequis

### Obtenir une clé API

**Bootstrap de la 1ère clé** : le self-service `POST /v1/me/api-keys` est live (P1-4) mais il exige une clé existante pour s'authentifier. La 1ère clé se demande donc manuellement — contactez `contact@akidly.com` en précisant :

- Nom de votre société / projet
- Volume estimé d'appels par mois
- URL d'endpoint webhook si vous en avez déjà un prêt (optionnel)

Vous recevrez :

- Une **clé API** au format `yk_...`. À traiter comme un mot de passe — elle n'est **jamais** retransmise.
- Un **crédit initial** en secondes pour vos tests.

**Ensuite, self-service complet** : dès que vous avez cette 1ère clé en main, vous gérez vous-même vos clés supplémentaires (CI, rotation, backup) via `POST /v1/me/api-keys`, `GET /v1/me/api-keys`, `DELETE /v1/me/api-keys/{key_id}`. Voir `§4.8` ci-dessous et la recette complète dans `docs/examples.md §3.2`.

### Base URL

```
https://api.yasmine.akidly.com
```

Toutes les requêtes passent par HTTPS. Le certificat est Let's Encrypt, valide.

### Authentification

Toutes les routes sous `/v1/*` exigent un header `Authorization: Bearer <votre_clé>` :

```bash
curl https://api.yasmine.akidly.com/v1/calls \
  -H "Authorization: Bearer yk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

Sans clé ou avec une clé invalide → **401** au format RFC 7807 (`application/problem+json`).

---

## 2. Premier appel de bout en bout

> ⚠️ Un `POST /v1/calls` déclenche un **vrai appel téléphonique** vers le numéro indiqué et **consomme du crédit réel**. Pour vos premiers tests, utilisez **votre propre numéro**.

Remplacez la clé et le numéro, puis collez :

```bash
curl -X POST https://api.yasmine.akidly.com/v1/calls \
  -H "Authorization: Bearer yk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{
    "customer": {
      "name": "Ahmed Bennani",
      "phone_number": "+212612345678"
    },
    "merchant_external_id": "ma-boutique-test",
    "shop_info": {
      "name": "Ma Boutique Test"
    },
    "order": {
      "delivery_address": "12 Rue du Test, Casablanca",
      "amount": "249.00",
      "currency": "MAD"
    },
    "country": "MA",
    "call_params": {
      "purpose": "confirmation"
    }
  }'
```

Les sous-objets `customer`, `shop_info`, `order`, `call_params` acceptent d'autres champs facultatifs (genre client, politiques SAV, delivery_method, voice_id, etc.). La liste complète et leurs contraintes sont dans `docs/openapi.yaml` (schémas `CustomerInput`, `ShopInfoInput`, `OrderInput`, `CallParamsInput`).

Réponse attendue (HTTP **201 Created**) :

```json
{
  "id": "dcaebcdd-a81a-4386-a0df-85d9aaafe862",
  "status": "queued",
  "created_at": "2026-04-19T18:36:09.496546Z",
  "merchant_id": "2671d5d2-efff-4e14-ba2a-3bfe80bd7840",
  "customer_phone_masked": "+212 6** *** *53",
  "purpose": "confirmation",
  "amount": "249.00",
  "currency": "MAD",
  "country": "MA",
  "metadata": null
}
```

Header `Location: /v1/calls/dcaebcdd-a81a-4386-a0df-85d9aaafe862` en prime.

---

## 3. Comprendre la réponse 201

- **`id`** : identifiant UUID de l'appel. À conserver pour le retrouver plus tard.
- **`status`** : `queued` au moment du 201. L'appel est asynchrone — l'agent va composer, sonner, converser, puis raccrocher (typiquement ~60 secondes). Les statuts traversés : `queued → dialing → ringing → in_progress → ended` (ou `failed`).
- **`merchant_id`** : le merchant a été upserté depuis `merchant_external_id`. Si cet identifiant existait déjà pour votre reseller, c'est le même ID — sinon un nouveau merchant a été créé.
- **`customer_phone_masked`** : **jamais le numéro brut** dans nos réponses (principe anti-BOPLA). Le numéro complet reste disponible côté opérations pour debug.
- **`amount`** est une **string** pour préserver la précision décimale (évite les pièges de `Number` en JavaScript).

**Le statut final et le résultat** (confirmé / annulé / pas de réponse) seront disponibles via `GET /v1/calls/{id}` et les webhooks sortants — **planned M4**. En attendant, contactez l'équipe si vous avez besoin d'exploiter les résultats côté dashboard Yasmine.

---

## 4. Gérer les erreurs

Toutes les erreurs `/v1/*` sont renvoyées au format RFC 7807 `application/problem+json`. Routez côté client sur le champ `type` (URI stable), **pas** sur le `status` seul.

### 401 — Clé invalide ou absente

```json
{
  "type": "https://docs.yasmine.akidly.com/errors/invalid_api_key",
  "title": "Unauthorized",
  "status": 401,
  "detail": "Authorization: Bearer <reseller_api_key> requis",
  "instance": "/v1/calls",
  "request_id": "f7d9a06a"
}
```

→ Vérifiez le header `Authorization` côté client. Si la clé a été générée en M1, elle reste valide.

### 402 — Solde insuffisant

```json
{
  "type": "https://docs.yasmine.akidly.com/errors/insufficient_balance",
  "title": "Payment Required",
  "status": 402,
  "detail": "Solde insuffisant (minimum 10 secondes requis).",
  "instance": "/v1/calls",
  "request_id": "b3c4d5e6"
}
```

→ Contactez l'équipe pour un top-up. Vous pouvez consulter votre solde courant via `GET /v1/me/balance` (P0-2, live).

### 422 — Validation

```json
{
  "type": "https://docs.yasmine.akidly.com/errors/validation_error",
  "title": "Unprocessable Entity",
  "status": 422,
  "detail": "Un ou plusieurs champs du payload sont invalides.",
  "instance": "/v1/calls",
  "request_id": "5312821c",
  "errors": [
    { "loc": ["body", "customer", "phone_number"], "msg": "Value error, phone_number invalide — format attendu E.164 (ex: +212612345678)", "type": "value_error" },
    { "loc": ["body", "order", "amount"], "msg": "Field required", "type": "missing" }
  ]
}
```

→ Lisez `errors[]` pour retrouver les champs fautifs avec leur `loc` (chemin dans le payload).

### Liste complète des codes d'erreur

Voir [`docs/errors.md`](https://docs.yasmine.akidly.com/errors.md) pour la table complète des slugs stables (auth, balance, validation, rate-limit, call_not_found, webhook_*, etc.).

Un `request_id` (8 caractères) est toujours présent — **incluez-le dans tout ticket support**.

---

### 429 — Rate limit dépassé

L'API applique des quotas **par clé API** sur tous les endpoints `/v1/*` (M3.6 C7) :

| Endpoint | Seuil |
|---|---|
| `POST /v1/calls` | 60 / minute |
| `POST /v1/calls/{id}/cancel` | 120 / minute |
| `GET /v1/calls/{id}`, `GET /v1/me/balance`, `GET /v1/me/webhooks` (reads `/v1/*`) | 600 / minute |
| `POST` / `DELETE /v1/me/webhooks` | 10 / minute |

Chaque réponse inclut les headers :
- `X-RateLimit-Limit` : plafond appliqué.
- `X-RateLimit-Remaining` : compteur restant dans la fenêtre courante.
- `X-RateLimit-Reset` : timestamp Unix du prochain reset.

Sur dépassement : **429 Too Many Requests** (RFC 7807 slug `rate_limit_exceeded`) avec header `Retry-After` en secondes + champ `retry_after` dans le body.

**Règle côté client** : respecter `Retry-After` avec backoff exponentiel si plusieurs 429 consécutifs.

```python
import time, httpx

def call_with_retry(method, url, **kwargs):
    for attempt in range(5):
        r = httpx.request(method, url, **kwargs)
        if r.status_code != 429:
            return r
        wait = int(r.headers.get("Retry-After", "1")) * (2 ** attempt)
        time.sleep(min(wait, 300))
    return r  # dernière tentative
```

Les webhooks Meta entrants (`POST /webhooks/whatsapp`) et les probes santé (`GET /healthz`) **ne sont jamais rate-limités**.

---

## 4.4. Tracer vos requêtes avec `X-Request-ID` (P1-5)

Chaque réponse Yasmine inclut un header `X-Request-ID` (UUID v4) unique à la requête. Ce même identifiant apparaît dans le body des erreurs RFC 7807 (champ `request_id`) et dans nos logs serveurs.

**Attribuer votre propre ID** (optionnel) :

```bash
curl -X POST "$BASE/v1/calls" \
  -H "Authorization: Bearer $YK" \
  -H "X-Request-ID: $(uuidgen)" \
  -H "Idempotency-Key: $(uuidgen)" \
  -H "Content-Type: application/json" \
  -d '{...}'
# Response : X-Request-ID echoed tel quel si format UUID v4 strict valide.
```

Si vous envoyez un ID non-UUID v4 (format invalide, chaîne arbitraire), Yasmine en régénère un et renvoie le nouvel ID — jamais la valeur que vous avez soumise (protection anti-log-injection).

**Pourquoi c'est utile** :
- **Corréler vos logs client avec les nôtres** : envoyez un UUID corrélable à votre trace métier interne (ordre, job ID hashé, etc.) et retrouvez-le instantanément dans nos logs.
- **Tickets support** : joignez le `request_id` → nous retraçons la requête exacte côté serveur sans besoin de timestamps ou de payload copy.
- **Fonctionne même sur les 401** : le middleware tourne avant l'authentification, donc les erreurs d'auth ont aussi un `request_id`.

---

## 4.8. Gérer vos clés API (P1-4)

CRUD self-service : créer une clé pour votre CI, en révoquer une après fuite, suivre l'usage via `last_used_at` et `last_used_ip` :

```bash
# Créer une clé — secret visible UNE SEULE FOIS.
curl -X POST -H "Authorization: Bearer $YK" \
  -H "Content-Type: application/json" \
  -d '{"label":"ci-prod"}' \
  "$BASE/v1/me/api-keys"

# Lister (sans secret).
curl -H "Authorization: Bearer $YK" "$BASE/v1/me/api-keys"

# Révoquer (soft delete, 204).
curl -X DELETE -H "Authorization: Bearer $YK" "$BASE/v1/me/api-keys/<key_id>"
```

Rate-limit création : 5/min/reseller. Recette complète dans `docs/examples.md §3.2`.

---

## 4.7. Intégrer vos webhooks (P1-2)

Flow recommandé pour votre intégration webhook :

1. **Configurer l'URL** : `POST /v1/me/webhooks` avec votre URL publique HTTPS. Stockez le `secret` retourné — il sert à vérifier la signature HMAC-SHA256 (`X-Yasmine-Signature: sha256=<hex>`).
2. **Tester l'endpoint** : `POST /v1/me/webhooks/test` → POST synchrone vers votre URL avec `data.test=true`. Retourne `delivered: true/false`, `http_status`, `latency_ms` immédiatement. Pas besoin d'attendre un vrai event.
3. **Auditer en continu** : `GET /v1/me/webhooks/deliveries?status=failed` pour repérer les échecs et debug. Chaque ligne expose `error_message` (tronqué 200 chars), `attempt_count`, `next_retry_at`.

Détails complets dans `docs/webhooks.md`.

---

## 4.6. Suivre mon compte (P1-1)

Trois endpoints self-service pour la supervision :

- **`GET /v1/me`** : identité du compte + profil société (propriétaire, langue par défaut, pays, raison sociale, identifiant fiscal, adresse de facturation, archétype, etc.). Liste complète : voir schéma `MeOut` dans `docs/openapi.yaml`.
- **`GET /v1/me/transactions`** : historique paginé du ledger (mêmes `next_cursor`/`has_more` que `/v1/calls`). Filtres `?since=&until=`.
- **`GET /v1/me/usage?period=YYYY-MM`** : consommation mensuelle (défaut = mois courant UTC). Breakdown par `call_status` (`completed`/`failed`/`cancelled`).

Rappel : le solde vit sur `/v1/me/balance` (P0-2), séparé pour des raisons anti-BOPLA.

Recette curl complète dans `docs/examples.md §3.3`.

---

## 4.5. Réconciliation après webhook perdu (P0-2)

Si un webhook sortant est perdu (crash mid-retry — edge case accepté produit), polling `GET /v1/calls/{id}` quelques secondes plus tard donne **l'état complet** de l'appel — y compris le résultat métier (`result`) et la durée (`call_duration_seconds`, `billable_duration_seconds`) — sans dépendre du webhook. Cet endpoint est scopé tenant : un 404 `call_not_found` est renvoyé avec un body **byte-identique** que le call soit inexistant, propriété d'un autre reseller, ou que le path contienne un UUID syntaxiquement invalide.

```bash
curl -H "Authorization: Bearer $YK" "$BASE/v1/calls/$CALL_ID" \
  | jq '{call_status, result, ended_at, call_duration_seconds, billable_duration_seconds, failure_reason, meta_error_code, meta_error_title}'
```

Champs clés à attendre une fois le call terminé :

- `call_status` parmi `ended` / `failed` / `cancelled` (rail APPEL).
- `result` parmi `CONFIRMED` / `CANCELLED` / `NO_ANSWER` / `UNCLEAR` / `FAILED` — état métier de la conversation. `null` tant que l'appel n'est pas terminé.
- `failure_reason` + (si `result=FAILED`) `meta_error_code` / `meta_error_title` — diagnostic technique côté opérateur de messagerie.

---

## 4.9. CORS — connecter un front web (P1-7)

Vous pouvez appeler `api.yasmine.akidly.com` depuis un navigateur (front JavaScript) **uniquement si l'URL de votre site est dans notre liste blanche CORS**. Par défaut, aucune origine n'est autorisée — il faut nous demander l'ajout via le support.

**Comment demander l'ajout** : envoyez-nous l'URL exacte (scheme + host + port si non standard) de votre front, par exemple `https://portal.monreseller.com` ou `https://dashboard.example.com:8443`. Nous ajoutons l'origine dans `/opt/yasmine/.env` sur le VPS puis redémarrons l'API (zéro downtime côté API reseller). Compte typique : 1 origine par environnement (prod / staging).

**Contrat technique** pour votre front une fois l'origine ajoutée :
- Méthodes autorisées : `GET`, `POST`, `DELETE`, `OPTIONS`.
- Headers client : `Authorization` (Bearer), `Content-Type`, `Idempotency-Key`, `X-Request-ID`.
- Headers lisibles par votre JS : `X-Request-ID`, `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, `Retry-After`. Utile pour pacer vos requêtes (éviter 429) et tracer les tickets support.
- `credentials: "include"` requis côté `fetch()` pour envoyer le Bearer cross-origin.
- Preflight `OPTIONS` mis en cache 10 min côté navigateur.

Exemple d'appel depuis un front React/Vue/etc. :

```javascript
const res = await fetch("https://api.yasmine.akidly.com/v1/me/balance", {
  method: "GET",
  headers: { "Authorization": `Bearer ${yk}` },
  credentials: "include",
});
// Lire les headers de rate-limit pour éviter un 429 imminent.
const remaining = res.headers.get("X-RateLimit-Remaining");
```

## 5. Prochaines étapes

- [`docs/examples.md`](https://docs.yasmine.akidly.com/examples.md) — recettes pratiques : appel France, metadata reseller, gestion fine des erreurs.
- [`docs/webhooks.md`](https://docs.yasmine.akidly.com/webhooks.md) — catalogue des events sortants (planned M4+ — spec cible à lire pour préparer votre intégration).
- [`docs/errors.md`](https://docs.yasmine.akidly.com/errors.md) — table complète des erreurs.
- [`docs/versioning.md`](https://docs.yasmine.akidly.com/versioning.md) — politique de versioning de l'API.

## FAQ courte

**Y a-t-il un environnement de test / sandbox ?**
Pas encore. Chaque appel consomme du crédit réel et déclenche un appel téléphonique.

**`Idempotency-Key` est-il obligatoire ?**
Oui — depuis P0-1 le header est obligatoire sur `POST /v1/calls`, sinon `400 missing_idempotency_key`. Format libre, 1-255 chars (UUID v4 recommandé), TTL 24 h, scope par reseller. Une 2e requête identique (même clé + même body) renvoie la réponse stockée bit-for-bit avec `X-Idempotent-Replay: true`. Une 2e requête avec la même clé mais un body différent retourne `409 idempotency_key_conflict`. Voir `docs/examples.md §4`.

**Quels pays sont supportés ?**
Pour l'instant : `MA` (Maroc), `DZ` (Algérie), `TN` (Tunisie), `FR` (France). Le choix du profil conversationnel (darija / français) est fait côté serveur selon le `country`, transparent pour vous.

**Comment suis-je facturé ?**
En **secondes billables** = `max(duration_s, 10)`. Chaque appel consomme au minimum 10 secondes même s'il est plus court.
