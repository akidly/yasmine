# Politique de versioning — API Yasmine

## 1. Version dans l'URL

L'API utilise un segment de version **majeur** dans le path :

```
https://api.yasmine.akidly.com/v1/calls
https://api.yasmine.akidly.com/v2/calls   # futur
```

- Une montée de majeure (`/v1` → `/v2`) signale une **breaking change** (changement de schéma, de code HTTP, de sémantique).
- Les évolutions **non-breaking** (nouveau champ optionnel, nouveau endpoint, nouveau code d'erreur documenté) restent sous `/v1`.
- Pas de versioning mineur dans l'URL. Les changements mineurs sont tracés dans `CHANGELOG.md`.

## 2. Header `X-API-Version`

Chaque réponse API expose la version qui l'a servie :

```http
HTTP/1.1 200 OK
X-API-Version: v1
Content-Type: application/json
```

Permet à un SDK reseller de vérifier contre quoi il parle, et de logger en cas de changement inattendu.

## 3. Semver appliqué au contrat API

Le contrat API suit un semver relâché, distinct de la version produit :

| Incrément | Déclenche | Exemple |
|---|---|---|
| MAJOR | Breaking : renommage/suppression de champ, code HTTP modifié, ressource retirée | `v1 → v2` |
| MINOR | Ajout rétro-compatible : nouveau champ optionnel, nouvel endpoint, nouvel event webhook | `[0.1.0] → [0.2.0]` dans `CHANGELOG.md` |
| PATCH | Correction de bug sans changement de contrat, clarification de docs | `[0.1.0] → [0.1.1]` |

## 4. Dépréciation et sunset

Les dépréciations suivent **RFC 8594** (`Sunset` header).

### Cycle de vie

1. **Annonce** : entrée `CHANGELOG.md` avec date de `Deprecated` + date cible de `Removed`.
2. **Phase deprecated** : l'endpoint répond normalement mais inclut :

   ```http
   HTTP/1.1 200 OK
   Deprecation: true
   Sunset: Wed, 19 Oct 2026 00:00:00 GMT
   Link: <https://docs.yasmine.akidly.com/migrations/v1-v2>; rel="deprecation"
   ```

3. **Preview minimum 6 mois** entre l'annonce et le `Sunset`. Aucune suppression plus rapide, quelle que soit la pression interne.
4. **Retrait** : après la date `Sunset`, l'endpoint répond `410 Gone` avec un body RFC 7807 pointant vers la nouvelle version.

### Notifications

- Entrée dans `CHANGELOG.md` sous `## [X.Y.Z] — Deprecated`.
- Event webhook `api.deprecation_notice` pour tous les resellers qui appellent l'endpoint déprécié dans les 30 jours (planned, roadmap P2).

## 5. Champ `x-status` dans OpenAPI

La spec `docs/openapi.yaml` annote chaque endpoint :

- `x-status: live` — implémenté et stable, appelable en prod aujourd'hui.
- `x-status: internal` — route technique hors SLA `/v1/*` (probe santé).

Si un nouvel endpoint est annoncé en preview avant implémentation, il sera marqué `x-status: planned` à ce moment-là — la valeur n'a pas d'occurrence dans la spec actuelle.

## 6. Statut de `/v1` en production

`/v1` est **live** et **commercialisable** depuis M3.3 (2026-04-19) — `POST /v1/calls` est le premier endpoint reseller mis en prod. Les livraisons ultérieures ont étendu la surface **live** :

- `GET /v1/calls/{call_id}` depuis P0-2 (réconciliation scopée reseller, anti-énumération).
- `/v1/me/*` (identité, balance, transactions, usage, webhooks, api-keys) depuis P1-1 à P1-4 — 7 routes self-service.
- Webhooks sortants signés HMAC-SHA256 depuis M3.6 C6 Phase 2 (`call.*` events).

L'intégralité des endpoints documentés dans `docs/openapi.yaml` est désormais `x-status: live` (à l'exception de `/healthz` annoté `x-status: internal`, hors SLA `/v1/*`).

## 7. Environnements

| Env | URL | Statut |
|---|---|---|
| Prod | `https://api.yasmine.akidly.com` | live |
| Sandbox | `https://api-sandbox.yasmine.akidly.com` | planned — à voir plus tard |

Tant que la sandbox n'existe pas, tous les appels de test consomment du crédit réel. Les resellers le savent via `docs/getting-started.md`.
