# Intégrer Yasmine

Yasmine déclenche des **appels vocaux IA sortants** pour confirmer ou relancer des commandes clients (marchés MA/DZ/TN/FR). Vous envoyez une requête, un agent conversationnel appelle votre client, vous recevez le résultat via API ou webhook.

Trois chemins d'intégration au choix — ils partagent la même API sous-jacente, exposée différemment selon votre contexte :

- **[Mode A — API REST directe](#mode-a--api-rest-directe)** — pour un backend serveur ou un workflow n8n/Zapier.
- **[Mode B — MCP server](#mode-b--mcp-server)** — pour un agent IA (Claude Desktop, ChatGPT Desktop MCP, ou autre client compatible MCP).
- **[Mode C — Plugin Claude Code](#mode-c--plugin-claude-code)** — pour un·e dev·e qui travaille avec Claude Code et veut l'outil intégré dans son IDE.

---

## Mode A — API REST directe

Usage typique : déclencher des appels depuis votre backend e-commerce (Shopify, WooCommerce, Magento, custom).

- **Base URL** : `https://api.yasmine.akidly.com`
- **Auth** : header `Authorization: Bearer yk_<votre_clé>` sur toutes les routes `/v1/*`.
- **Format erreurs** : [RFC 7807](https://datatracker.ietf.org/doc/html/rfc7807) `application/problem+json` — routez sur le champ `type` (URI stable), pas sur le code HTTP seul.

### Exemple minimum — déclencher un appel

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
    "shop_info": {"name": "Ma Boutique Test"},
    "order": {
      "delivery_address": "12 Rue X, Casablanca",
      "amount": "249.00",
      "currency": "MAD"
    },
    "country": "MA",
    "call_params": {"purpose": "confirmation"}
  }'
```

Réponse `201 Created` avec `id`, `status: queued`, `created_at`, `customer_phone_masked`. L'agent compose, sonne, converse, raccroche (~40-60 secondes). Suivi via `GET /v1/calls/{id}` ou webhook sortant.

### Surface complète

25 opérations au total, réparties par tag OpenAPI :

| Tag | Endpoints principaux |
|---|---|
| `Calls` | `POST /v1/calls`, `GET /v1/calls`, `GET /v1/calls/{id}`, `POST /v1/calls/{id}/cancel` |
| `Account` | `GET /v1/me`, `GET /v1/me/balance`, `GET /v1/me/transactions`, `GET /v1/me/usage` |
| `API Keys` | `POST/GET /v1/me/api-keys`, `DELETE /v1/me/api-keys/{id}` |
| `Webhooks` | `POST/GET/DELETE /v1/me/webhooks`, `POST /v1/me/webhooks/test`, `POST /v1/me/webhooks/rotate-secret`, `GET /v1/me/webhooks/deliveries` |
| `Merchants` | Planifié (M4+) |

Spec complète et interactive : **[docs.yasmine.akidly.com](https://docs.yasmine.akidly.com)**.

---

## Mode B — MCP server

Usage typique : piloter Yasmine depuis un agent IA (Claude Desktop, Cursor, ou tout client compatible [Model Context Protocol](https://modelcontextprotocol.io)).

- **URL endpoint** : `https://mcp.yasmine.akidly.com/mcp/`
- **Transport** : Streamable HTTP (spec MCP `2025-11-25`)
- **Auth** : header `Authorization: Bearer yk_<votre_clé>` (la même clé que l'API REST directe).

### Tools exposés

8 tools sous le namespace `yasmine` :

**Actions sur votre compte** (pass-through API)
- `get_account` — infos reseller, solde, rate limits
- `create_call` — déclencher un appel
- `get_call` — statut + transcription d'un appel
- `list_calls` — historique paginé de vos appels

**Introspection** (lecture des docs directement par l'agent)
- `list_endpoints` — surface complète de l'API
- `get_endpoint_spec` — spec OpenAPI d'une opération précise
- `get_changelog` — évolutions récentes
- `explain_error` — aide structurée sur un slug RFC 7807

### Config Claude Desktop

Éditer `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS) ou `%APPDATA%\Claude\claude_desktop_config.json` (Windows) :

```json
{
  "mcpServers": {
    "yasmine": {
      "type": "http",
      "url": "https://mcp.yasmine.akidly.com/mcp/",
      "headers": {
        "Authorization": "Bearer yk_your_key_here"
      }
    }
  }
}
```

Redémarrer Claude Desktop. Le serveur `yasmine` apparaît dans les serveurs actifs, les 8 tools sont appelables via langage naturel.

### Config autres clients MCP

Tout client qui implémente la spec MCP 2025-11-25 avec transport Streamable HTTP peut se brancher. Le contrat réseau est standard : POST JSON-RPC vers l'endpoint, Bearer dans les headers.

---

## Mode C — Plugin Claude Code

Usage typique : vous développez votre intégration Yasmine dans Claude Code (CLI ou IDE), et vous voulez l'outil + un guide d'usage (skill) directement dans votre session.

### Install

Une fois la clé API obtenue (cf §[Authentification](#authentification)), exporter comme variable d'environnement **avant** de lancer Claude Code :

**macOS / Linux** (bash/zsh)
```bash
export YASMINE_API_TOKEN="yk_your_key_here"
```

**Windows** (PowerShell, persistent)
```powershell
[Environment]::SetEnvironmentVariable('YASMINE_API_TOKEN', 'yk_your_key_here', 'User')
```

Puis **relancer Claude Code** (fermer toutes les instances) et installer le plugin :

```
claude plugin marketplace add akidly/yasmine
claude plugin install yasmine@akidly
```

### Ce que vous obtenez

- Les **8 tools MCP** (identiques au Mode B, mais chargés automatiquement par Claude Code).
- Un **skill `yasmine-api`** — guide de référence chargé automatiquement quand Claude Code détecte une question Yasmine : auth, codes ISO pays, format E.164, idempotence, slugs d'erreur courants.

Source publique : [github.com/akidly/yasmine](https://github.com/akidly/yasmine). Versions : SemVer, `v0.1.3` actuellement.

---

## Authentification

Tous les modes utilisent la même clé API format `yk_<40 chars>`. À traiter comme un mot de passe — jamais retransmise après émission.

**Obtenir une clé** : contactez l'équipe Yasmine à `contact@akidly.com` en précisant :
- Nom de votre société / projet
- Volume estimé d'appels par mois
- URL de webhook prête si applicable (optionnel)

Vous recevrez la clé + un crédit initial en secondes pour vos tests. Le self-service via `POST /v1/me/api-keys` est disponible dès que vous avez une première clé (création de clés CI/CD supplémentaires, révocation, audit — cf §[Ressources](#ressources)).

---

## Quickstart 5 minutes

1. Obtenez une clé (voir ci-dessus).
2. Copiez-collez dans un terminal (remplacer la clé + votre numéro) :

```bash
export YK="yk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
curl -X POST https://api.yasmine.akidly.com/v1/calls \
  -H "Authorization: Bearer $YK" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{
    "customer": {
      "name": "Test",
      "phone_number": "+212612345678"
    },
    "merchant_external_id": "test",
    "shop_info": {"name": "Test Shop"},
    "order": {
      "delivery_address": "Rue Test, Casa",
      "amount": "100.00",
      "currency": "MAD"
    },
    "country": "MA",
    "call_params": {"purpose": "confirmation"}
  }'
```

> ⚠️ Un `POST /v1/calls` déclenche un **vrai appel téléphonique** et consomme du crédit réel. Pour votre premier test, utilisez **votre propre numéro**.

Réponse `201` en < 1 seconde. Appel effectif en 40-60 secondes.

---

## Ressources

- **Portail API complet (Scalar)** : [docs.yasmine.akidly.com](https://docs.yasmine.akidly.com)
- **Onboarding détaillé** : [docs.yasmine.akidly.com/getting-started.md](https://docs.yasmine.akidly.com/getting-started.md)
- **Recettes curl** : [docs.yasmine.akidly.com/examples.md](https://docs.yasmine.akidly.com/examples.md)
- **Catalogue erreurs RFC 7807** : [docs.yasmine.akidly.com/errors](https://docs.yasmine.akidly.com/errors)
- **Webhooks sortants** : [docs.yasmine.akidly.com/webhooks.md](https://docs.yasmine.akidly.com/webhooks.md)
- **Politique de versioning** : [docs.yasmine.akidly.com/versioning.md](https://docs.yasmine.akidly.com/versioning.md)
- **Plugin Claude Code (source)** : [github.com/akidly/yasmine](https://github.com/akidly/yasmine)
- **Support** : `contact@akidly.com` — joignez toujours le `request_id` (8 chars) renvoyé dans les headers / bodies d'erreur pour un retour rapide.

---

## Bon à savoir

- **Pays supportés** : `MA` (Maroc, darija), `DZ` (Algérie, darija), `TN` (Tunisie, darija), `FR` (France, français). Le profil conversationnel est choisi côté serveur via le champ `country`.
- **Numéros** : format E.164 strict uniquement (`+212612345678`, `+33612345678`).
- **Facturation** : en secondes billables `= max(duration, 10)`. Minimum 10 secondes par appel.
- **Idempotence** : header `Idempotency-Key` obligatoire sur `POST /v1/calls`. Même clé + même body dans les 24 h = réponse rejouée sans re-déclencher l'appel (`X-Idempotent-Replay: true`). Même clé + body différent = `409 idempotency_key_conflict`.
- **Rate limits** : par clé API — 60/min sur `POST /v1/calls`, 600/min sur les reads, headers `X-RateLimit-*` + `Retry-After` sur 429.
- **Sandbox** : pas de sandbox aujourd'hui. Chaque appel = crédit réel. Utilisez votre propre numéro pour tester.
