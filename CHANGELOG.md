# Changelog

Toutes les évolutions visibles côté reseller de l'API Yasmine.

Le format suit [Keep a Changelog](https://keepachangelog.com/fr/1.1.0/) et la politique de versioning de `docs/versioning.md`.

## [Unreleased]

### Fixed
- `.github/workflows/sync-plugin.yml` : `DRY_RUN` fallback corrigé (le ternaire GHA `A && B || C` court-circuitait à `C` quand `B = false`, donc une case `dry_run` décochée en `workflow_dispatch` retombait sur `'true'` via le fallback au lieu de rester `false`). Fix : tester d'abord `github.event_name == 'push'` (toujours truthy string `'true'`), laisser `inputs.dry_run` passer tel quel en `workflow_dispatch`.

### Added (Lot 4 — CI auto-sync plugin)
- `.github/workflows/sync-plugin.yml` : auto-sync docs sources → `akidly/yasmine` on `push master` (paths-filtered : `docs/openapi.yaml`, `docs/errors.md`, `docs/examples.md`, `docs/getting-started.md`, `CHANGELOG.md`) ou `workflow_dispatch`.
- Bump patch auto des versions dans `plugin.json` et `marketplace.json` ; bump `minor`/`major` disponible via `workflow_dispatch` input.
- Copies : 4 docs vers `skills/yasmine-api/*` + `CHANGELOG.md` à la racine publique.
- Commit + tag `vX.Y.Z` poussés par le bot `akidly-sync-bot`.
- Skip automatique si `git diff-index --quiet HEAD` (pas de commit vide).
- Auth : PAT fine-grained `AKIDLY_PUBLISH_PAT` (scope `contents: write` sur `akidly/yasmine` uniquement, rotation 90j).
- Premier run forcé en `dry_run=true` — cutover à `false` par défaut dans un commit ultérieur après validation.

### Fixed (Lot 3 — plugin MCP activation)
- `akidly/yasmine` bumped to `v0.1.1` (commit `5627f92`, tag `v0.1.1`) : pattern `userConfig` + `${user_config.api_token}` (documenté dans `plugins-reference.md`) **non implémenté** en Claude Code CLI 2.1.116 — MCP server skippait silencieusement à l'init. Remplacé par le pattern officiel des plugins Anthropic (cf. `github` plugin) : `.mcp.json` séparé à la racine + substitution standard `${YASMINE_API_TOKEN}` (env var). Plugin minimal `plugin.json` (name/version/description/author). README mis à jour avec étape `export YASMINE_API_TOKEN=yk_...` avant install. v0.1.0 gardée comme artefact historique.

### Added (Lot 3 — Claude Code plugin `yasmine@akidly`)
- Public plugin released at [akidly/yasmine](https://github.com/akidly/yasmine) `v0.1.0`. Install via `/plugin install yasmine@akidly/yasmine`.
- Manifest in `.claude-plugin/plugin.json` with `mcpServers` inline (declares the prod `https://mcp.yasmine.akidly.com/mcp/` endpoint) + `userConfig.api_token` (`sensitive: true`, prompts user once, stored in OS keychain) + `userConfig.api_endpoint` (override-able for local dev).
- Skill `yasmine-api` bundled (auth + ISO country/language codes + E.164 phone format + 5 RFC 7807 error slugs + pointer vers `explain_error()` for full detail).
- README, LICENSE (MIT 2026 Akidly), .gitignore standard Python+IDE.
- `plan-mcp-plugin.md §4.3` re-aligné sur la spec officielle Claude Code 2026 (audité Phase 0 du Lot 3).

### Added (Lot 2 — MCP introspection tools)
- `yasmine_mcp/introspection.py` : 4 new MCP tools for API self-discovery by AI agents :
  - `list_endpoints` — enumerates 20 live API operations (filters `x-status: planned`)
  - `get_endpoint_spec(operation_id)` — returns full OpenAPI spec for an operation
  - `get_changelog(version=None)` — returns parsed changelog entries (default: Unreleased + latest released)
  - `explain_error(slug)` — returns structured error explanation with Causes/Remediation/Example
- `yasmine_mcp/_openapi.py` : shared OpenAPI loader + `x-status: planned` filter (de-duplicates logic between `server.py` and `introspection.py`)
- `docs/errors.md` : 5 slugs enriched with structured sections (`validation_error`, `rate_limit_exceeded`, `insufficient_balance`, `idempotency_key_conflict`, `invalid_cursor`). Markdown table untouched, section appended.
- 14 new tests in `tests/yasmine_mcp/test_introspection.py`.

### Changed
- `yasmine_mcp` server name : `Yasmine MCP (Lot 1)` → `Yasmine MCP (Lot 1+2)`.
- `tests/yasmine_mcp/test_server.py` tool inventory : 4 → 8 expected.

### Fixed
- `ops/deploy.sh` self-reload : guard `YASMINE_DEPLOY_RELOADED` env var pour casser la boucle infinie qu'introduisait l'`exec bash "$0"` du commit `9aaf0b2`. Sans guard, le script post-reset relançait `exec bash` indéfiniment (chaque relance lit la version disque qui ne change plus, mais re-exec quand même). Découvert sur le push Lot 2 (`093b7c5`) qui n'a jamais déclenché le restart de `yasmine-mcp.service`. Pattern `if [[ -z "${YASMINE_DEPLOY_RELOADED:-}" ]]; then export ...; exec bash "$0" "$@"; fi` — la 1re exec set la var puis exec, la 2e voit la var posée et continue à `pip install`.

### Chore
- `ops/deploy.sh` : self-reload after `git reset` to pick up script updates from the same pull.
- `ops/mcp-bootstrap.sh` : archive VPS one-shot bootstrap for reproducibility.
- Docs cleanup : align `plan-mcp-plugin.md` + `CLAUDE.md` on post-Lot 1 state (`yasmine_mcp/` path, `fastmcp>=3.2.0`, `yk_` key format).
- Remove `CLAUDE.md.bak.2026-04-19` (git history is the source of truth).

### Added
- `respx` to `requirements.txt` (test dependency, was transitively installed).

### Added (Lot 1 — MCP remote `mcp.yasmine.akidly.com`)
- `yasmine_mcp/` — MCP remote FastMCP (PyPI 3.x) pass-through vers l'API Yasmine, exposant **4 tools live** : `get_account` (→ `GET /v1/me`), `get_call` (→ `GET /v1/calls/{id}`), `create_call` (→ `POST /v1/calls`), `list_calls` (→ `GET /v1/calls`). Transport **Streamable HTTP** (spec MCP 2025-11-25), bind `127.0.0.1:9001`, **stateless** (pas de `MCP-Session-Id`). Auth = **passthrough Bearer** du client MCP vers l'API Yasmine via `httpx` event hook + `fastmcp.get_http_headers(include={"authorization"})`. `OriginGuard` middleware (anti DNS rebinding, spec MCP Security). Response hook convertit les 4xx/5xx RFC 7807 en `ToolError` lisible (`title: detail`, pas de stack trace). 2 préprocesseurs in-memory sur `docs/openapi.yaml` : `_strip_planned_operations` (filtre `x-status: planned` → masque `listMerchants`/`createMerchant` jusqu'à M4, réactivation non-breaking) et `_normalize_nullables` (convertit récursivement `nullable: true` OpenAPI 3.0 → `type: [X, null]` JSON Schema draft-07+, nécessaire pour `PaginatedCalls.next_cursor` entre autres). Le fichier `docs/openapi.yaml` sur disque reste en forme OpenAPI 3.0 canonique servable à Scalar.
- `tests/yasmine_mcp/test_server.py` — 16 tests verts : inventaire (4 tools), 1 par tool via `respx` mock + `fastmcp.Client` in-process, régression nullable `next_cursor: None`, preuve que les planned ne fuient pas, forwarding Bearer (unit), Origin guard (3 cas), RFC 7807 → ToolError, masking header dans logs. `respx` ajoutée en dep de dev.
- `ops/yasmine-mcp.service` — systemd unit (user `deploy`, `EnvironmentFile=/opt/yasmine/.env`, `ExecStart=/opt/yasmine/venv/bin/python -m yasmine_mcp.server`, `Restart=on-failure`).
- `ops/nginx-mcp.conf` — bloc Nginx `mcp.yasmine.akidly.com` : port 80 + ACME challenge + proxy_pass `http://127.0.0.1:9001`, `proxy_buffering off` / `chunked_transfer_encoding on` / timeouts 3600 s pour supporter les SSE streams de Streamable HTTP. SSL posé par certbot `--nginx`.
- `ops/deploy.sh` étendu : après le restart `yasmine.service`, le script copie la unit MCP en `/etc/systemd/system/`, `daemon-reload`, `enable`, `restart yasmine-mcp.service`, puis snapshot status.
- `requirements.txt` : `fastmcp>=3.2.0`, `pyyaml>=6.0.1`.

### Fixed
- `docs/openapi.yaml` : 2 plain scalars non-conformes YAML 1.2 strict (ligne 620 `description` commençant par `` ` ``, ligne 1180 `detail` contenant ` : `) encadrés en `"..."`. Scalar les tolérait, ni PyYAML ni ruamel.yaml ne les chargeaient. Sémantique inchangée.
- `.claude/context/plan-mcp-plugin.md` + `roadmap.md` : rename `mcp/` → `yasmine_mcp/` (collision avec le package PyPI `mcp`, dépendance transitive de FastMCP — un dossier `mcp/` à la racine fait shadow sur l'import `mcp.types`).
- `.claude/context/roadmap.md` : mention `FastMCP 2.x` → `FastMCP 3.x` (PyPI publie en 3.x désormais, la doc gofastmcp.com unifie sous "2.x+" mais les imports canoniques sont `fastmcp.server.providers.openapi.{MCPType, RouteMap}` / `fastmcp.server.dependencies.get_http_headers`).

### Known issues
- 11 test failures in `tests/test_finalize_whatsapp.py` (pre-existing, introduced by `0d14a21` — signature mismatch on `_resolve_result`). Unrelated to Lot 1. Tracked separately.

### Changed (ops — source unique `ops/deploy.sh` + bascule GitHub Action)
- `ops/deploy.sh` enrichi avec les 4 traces timestamps `echo "[$(date -Iseconds)] <étape>"` et format `set -a` / `. .env` / `set +a` sur 3 lignes — matche désormais byte-pour-byte le script live `/opt/yasmine/deploy.sh` qui tournait avant versionnage. Fidélité des logs GitHub Action préservée.
- `.github/workflows/deploy.yml` : la step finale appelle `bash /opt/yasmine/ops/deploy.sh` au lieu de `bash /opt/yasmine/deploy.sh`. Aucune autre modif (secrets, triggers, concurrency identiques).
- Ancien `/opt/yasmine/deploy.sh` racine supprimé côté VPS (`sudo rm`) — source unique canonique désormais **tracée en Git** sous `ops/deploy.sh`. Toute modif future du script live doit passer par un commit.
- `docs/infrastructure.md` section "Script de déploiement" reformulée : fichier live VPS = `/opt/yasmine/ops/deploy.sh`, propagé par `git reset --hard` interne au script.

### Changed (doc infra — post-vérif VPS 2026-04-21)
- `docs/infrastructure.md` : les blocs Nginx M3.4 (aliases `.md` bruts `getting-started`/`examples`/`webhooks`/`versioning`) et P1-10 (surface `/errors` + `/errors.html` + redirection `^/errors/([a-z_]+)$`) sont reformulés en **historique déployé** (plus de diff "à appliquer"). La vérification VPS `2026-04-21 14:42 UTC` confirme qu'ils servent depuis le déploiement correspondant. Fallback `location /` corrigé : `try_files $uri $uri/ =404` (état prod réel, plus strict que le `/index.html` documenté).
- `docs/infrastructure.md` nouvelle section `## Script de déploiement` : pointe vers `ops/deploy.sh` comme source canonique, liste les 5 étapes (fetch+reset hard, pip, charge `.env`, `alembic upgrade head`, restart `yasmine.service`). Documente explicitement que les migrations Alembic tournent à chaque push — ne jamais merger sur `master` une migration non-réversible.

### Added (ops — deploy.sh versionné)
- `ops/deploy.sh` (534 o, exécutable) : copie à l'identique du script live `/opt/yasmine/deploy.sh` qui n'était **pas versionné** auparavant. Toute modif future côté VPS est à reporter dans ce fichier pour code review. `CLAUDE.md` section Production pointe désormais dessus.

### Added (contexte — audits v3 + plan MCP/plugin)
- `.claude/context/audit-2026-04-v3.md` — nouvel audit API 3 volets reflétant l'état post-P1 (P0 v2 tous fermés 2026-04-21 + 10 des 12 items P1 livrés même journée). Diff explicite avec l'audit v2 pour chaque item (statut ✅/⚠️/❌/➖). Surface live : 8 → 21 routes authentifiées. Nouveaux vecteurs identifiés : cursor HMAC clé globale sans `reseller_id`, DDoS amplifier via `/webhooks/test`, `last_used_ip` mono-valeur, URL webhook en logs + DB non purgée, collision de naming "P1-8" (webhook-rotate ≠ HMAC key hash).
- `.claude/context/roadmap.md` refondue : P0 v3 vide (aucun bloquant vente externe restant), P1 v3 resserré à 7 items (HMAC key hash vrai, timestamp signature webhook, retention `webhook_deliveries`, `api_key.compromised` proactif, découpage `me.py`, OpenAPI YAML ré-aligné, DNS async dispatcher), P2 v3 à 20 items. Ancienne roadmap v2 conservée en section `## Archive — Roadmap v2`.
- `.claude/context/docs-audit-2026-04.md` — audit AI-readability (sections 1-4-bis livrées) : état doc Markdown, infra Nginx vs VPS réel (drift levé), gap analysis 3 couches (llms.txt absent, aliases `.md` brut tous déployés, skill Claude Code à construire), pipeline update avec drift par artefact. §5 roadmap à générer.
- `.claude/context/plan-mcp-plugin.md` — plan vivant du chantier MCP remote + plugin Claude Code public. Référence avant tout travail sur le dossier `mcp/` ou un repo de distribution.
- `CLAUDE.md` section Documents de référence : pointe désormais `audit-2026-04-v3.md` comme source active (v2 + v1 en archive), ajoute les 2 nouveaux pointeurs `docs-audit-2026-04.md` et `plan-mcp-plugin.md`.
- `.claude/context/docs-audit-2026-04.md` §5 — roadmap actionnable rédigée : synthèse gaps, vision cible, 5 lots MCP/plugin priorisés, scénarios de dégradation. Clôt l'audit AI-readability.
- `.claude/context/roadmap.md` — sous-bloc `### Distribution AI-native (MCP remote + plugin Claude Code)` ajouté en P2 avec items `#P2-v3-MCP-1` à `#P2-v3-MCP-5` (total ~6 j). Légende volets étendue avec `Audit docs`.
- `CLAUDE.md` section Production — correction pointeur `/opt/yasmine/deploy.sh` → `/opt/yasmine/ops/deploy.sh` aligné sur l'état live post-`6d81b7a`.

### Added (P1-8-bis — Dédup events côté reseller)
- **Documentation explicite** que `X-Yasmine-Event-Id` (déjà émis depuis M3.6 C6) reste stable à travers les retries du dispatcher — usable côté reseller pour dédupliquer. `docs/webhooks.md §6` enrichi avec recette Redis `SET NX EX 86400` en plus de l'approche DB UNIQUE INDEX existante. TTL recommandé 24 h (> fenêtre max retry 5 min × 3 tentatives).
- 3 tests dédiés (`tests/test_dispatcher_event_id.py`) verrouillent les invariants : header `X-Yasmine-Event-Id` == body.id, event_id préservé sur 3 POST retry (500, 500, 200), 2 dispatch successifs produisent 2 event_id distincts. Le 4e test du brief (signature couvre l'event_id) est déjà couvert par `test_webhook_dispatch.py::test_dispatch_success_happy_path`. Suite : 329 → 332 passed.
- **Aucun code applicatif modifié** : l'infrastructure (`envelope["id"] = f"evt_{ulid.new().str}"` généré **hors** boucle retry, header + body top-level, colonne `webhook_deliveries.event_id` TEXT NOT NULL + index) existait depuis la migration 0006 (M3.6 C6 Phase 1) et le dispatcher Phase 2.

### Added (P1-8 — Rotation du signing secret webhook)
- **`POST /v1/me/webhooks/rotate-secret`** (#P1-8) : nouvel endpoint qui rotate atomiquement le secret HMAC-SHA256 du webhook actif sans passer par `DELETE + POST`. Retourne 200 `{"secret": "whsec_...", "secret_prefix": "whsec_xxxxxx…", "rotated_at": "..."}`. **Secret visible une seule fois** dans la réponse — aucun endpoint ne la ré-expose. Rate-limit **5/min** (aligné sur `POST /v1/me/api-keys`). 404 `webhook_not_configured` si aucune config active.
- **Format secret `whsec_<43 chars>`** (Stripe-like) pour les nouveaux secrets générés à la création ou à la rotation (`secrets.token_urlsafe(32)` + préfixe repère). Facilite le grep / secret scanning dans les logs et CI. Les secrets legacy pré-P1-8 (sans préfixe) restent fonctionnels — la fonction `sign_payload` est prefix-agnostique.
- 7 nouveaux tests (`tests/test_webhook_rotate.py`) : format `whsec_`, unicité, endpoint 200/404, rate-limit 5/min, 2 rotations consécutives produisent 2 secrets distincts, invariant "l'ancien secret ne valide plus après rotate".

### Changed (P1-8 — Preview secret 12 chars au lieu de 6)
- `GET /v1/me/webhooks` renvoie désormais `secret_prefix` sur **12 chars** + `…` (était 6 chars jusqu'au P1-8). Cohérent avec `key_prefix` des clés API (P1-4). Pour les nouveaux secrets format `whsec_xxxxxx…` (6 chars utiles après le préfixe). Pour les secrets legacy, 12 chars bruts — plus informatif qu'avant.
- **Pas de migration DB** : la colonne `reseller_webhooks.secret` existait déjà depuis M3.6 C6. Le HMAC signing, le dispatcher et le header `X-Yasmine-Signature: sha256=<hex>` sont live en prod depuis M3.6 — seule la rotation atomique et le format `whsec_` sont nouveaux.
- `docs/webhooks.md §1.4` réécrit pour pointer sur `POST /v1/me/webhooks/rotate-secret` en premier (pattern DELETE + POST documenté en legacy fallback). `docs/openapi.yaml` ajoute le path + schéma `WebhookSecretRotated`.

### Added (P1-9 — Rétention auto + audit trail `api_key_events`)
- **Purge automatique de `api_key_events` après 180 jours** (#P1-9). Audit trail (events `created`, `revoked`, `rate_limited` écrits par P1-6) purge via `DELETE` — standard OAuth2/Stripe. Évite la croissance illimitée de la table (worst-case ~43k rows/mois/clé estimé en rapport P1-6).
- Script `scripts/retention_purge.py` refondu en 3 fonctions indépendantes (`purge_webhook_raw`, `purge_call_events`, `purge_api_key_events`), chacune dans sa propre transaction. Une erreur sur une table ne bloque plus les autres.
- **Logs JSON structurés** : une ligne INFO par table (`{"table":"<n>","retention_days":<N>,"purged":<N>,"dry_run":<bool>}`) + une ligne summary en fin de run. Consommable direct par journalctl + tooling d'observabilité (Grafana Loki, etc.).
- **Exit code 1** si au moins une table a raisé (visible dans `systemctl status yasmine-retention.service`). Exit 2 sur argument CLI invalide.
- Nouveau flag CLI `--api-key-events-days` (défaut 180) pour overrider le seuil audit trail indépendamment du seuil court `--older-than-days` (webhook_raw + call_events, défaut 30).
- 9 tests ajoutés (total retention : 2 → 11). Suite : 312 → 321 passed.

### Changed (P1-9 — Rétention désormais automatique)
- Rétention PII `webhook_raw.payload` et `call_events.data.raw` désormais **quotidienne automatique** via systemd timer (`yasmine-retention.timer`, 02:00 UTC) au lieu d'un run mensuel manuel. Pas de migration de données — le passage de "mensuel manuel" à "quotidien auto" n'affecte que la fréquence du cleanup, pas le seuil 30 j.
- Les 2 fichiers systemd (`yasmine-retention.service` + `.timer`) sont documentés dans `docs/infrastructure.md §Rétention auto — systemd timer` — à déposer manuellement sur le VPS (config système, hors repo). User `deploy` (identique à `yasmine.service`).
- `webhook_deliveries` (P1-2) reste **hors scope** P1-9 — volume actuel faible, à revoir en P2 si un reseller live génère du trafic.

### Added (P1-7 — CORS middleware)
- **Support CORS pour fronts web cross-origin** (#P1-7). `CORSMiddleware` de FastAPI enregistré en dernier dans la stack (donc le plus extérieur à l'exécution — les preflight `OPTIONS` sont interceptés avant tout autre traitement, y compris le rate-limit). Politique stricte :
  - **Liste blanche statique** via env `CORS_ALLOWED_ORIGINS` (URLs absolues séparées par virgules). Défaut : vide (aucune origine acceptée). Ajouter les URLs des fronts reseller au cas par cas côté `/opt/yasmine/.env`.
  - **Dev auto** : si `ENV=dev`, regex `^http://(localhost|127\.0\.0\.1)(:\d+)?$` autorise `localhost`/`127.0.0.1` sur n'importe quel port. Verrouillé strictement — aucune autre IP privée acceptée.
  - `allow_credentials=true` (Bearer Authorization cross-origin OK). Incompatible avec `*` côté spec, d'où la liste blanche explicite.
  - **Methods** : `GET`, `POST`, `DELETE`, `OPTIONS`.
  - **Headers client acceptés** : `Authorization`, `Content-Type`, `Idempotency-Key`, `X-Request-ID`.
  - **Headers exposés au front** : `X-Request-ID`, `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, `Retry-After`. Permet au front de lire les en-têtes de rate-limit pour pacer ses requêtes.
  - **Preflight cache 10 min** (`max_age=600`).
- Nouvelle variable `CORS_ALLOWED_ORIGINS` dans `.env.example` avec commentaire explicatif. **Aucune URL committée** (liste blanche reste vide dans le repo — les origines réelles vivent en `/opt/yasmine/.env`, hors git).
- **Aucun header Nginx ajouté** côté `api.yasmine.akidly.com` : CORS géré entièrement côté application Python. Pas de risque de double injection.
- 8 tests (`tests/test_cors.py`) : preflight whitelist/rejet, GET avec/sans origine, dev localhost accepté, prod localhost rejeté, garde-fou spec credentials+allow_origins spécifique. Suite : 304 → 312 passed, 0 régression.

### Added (P1-10 — Page d'erreurs humaine)
- **`https://docs.yasmine.akidly.com/errors`** (#P1-10) : nouvelle page HTML listant les 28 slugs d'erreur émis par l'API, groupés par catégorie (Authentication, Validation, Idempotency, Billing, Rate limiting, Not found, Conflict, Webhooks, Size limits, Server). Chaque slug a sa propre `<section id="<slug>">` avec : explication, causes fréquentes, correctifs actionnables, exemple de réponse JSON. TOC latérale sticky, style cohérent avec Scalar dark theme.
- **Rewrite Nginx `/errors/<slug>` → `/errors#<slug>`** (302) : les URLs `type` RFC 7807 émises par l'API (ex. `https://docs.yasmine.akidly.com/errors/insufficient_balance`) atterrissent désormais sur la bonne section de la page, sans changer le code back. Diff Nginx documenté dans `docs/infrastructure.md` — à appliquer manuellement sur le VPS (`nginx -t` + `systemctl reload nginx`). Compatible avec l'alias existant `/errors.md` (non-régression).
- Test garde-fou `tests/test_errors_page_coverage.py` (4 tests) : parse le HTML et compare avec `api.v1.errors._KNOWN_V1_SLUGS`. Bloque toute dérive (slug émis sans section doc, section orpheline). Suite : 300 → 304 passed.
- **4 slugs ajoutés à `_KNOWN_V1_SLUGS`** (étaient émis mais absents de l'allow-list documentaire) : `rate_limit_exceeded` (M3.6 C7), `webhook_url_rejected`, `webhook_already_configured`, `webhook_not_configured` (P1-2). Aucun impact runtime — allow-list strictement documentaire, `build_problem` accepte tout slug.
- Note ajoutée en tête de `docs/errors.md` pointant vers la page détaillée ; section "Compatibilité" mise à jour (URLs désormais résolvables).

### Changed (P1-12 — Strict body validation `POST /v1/calls`)
- **Audit** : `CallCreate` (schema public du POST) était **déjà** en `extra="forbid"` depuis M3.3 — les champs inconnus sont rejetés 422 avec body RFC 7807 `validation_error` + `errors[].type=extra_forbidden` + `loc` explicite. Smoke prod confirmé (`{"custommer_name":"typo", ...}` → 422). Aucun slug dédié ajouté (le body existant expose déjà `type=extra_forbidden` côté Pydantic, le gain clarté d'un slug custom est marginal).
- **Défense en profondeur** : `CallRequest`, `ShopInfo`, `Product` (tous dans `api/schemas.py`, legacy interne jamais désérialisés depuis un body HTTP) patchés avec `model_config = ConfigDict(extra="forbid")`. Bloque un kwarg typo dans l'instanciation interne du handler `api/v1/calls.py::create_call` — exposition publique future protégée par défaut.
- 10 tests ajoutés (`tests/test_schemas_extra_forbid.py`) couvrant CallCreate (typo top-level, extras multiples, optionnel omis, metadata dict libre, payload valide) + schemas legacy. Suite : 290 → 300 passed, 0 régression.
- OpenAPI `docs/openapi.yaml` : déjà `additionalProperties: false` sur `CallCreate`, aucune modif nécessaire.
- **Pas de breaking change effectif** : ce comportement est actif en prod depuis M3.3. Le chantier P1-12 consolide la couverture tests + documente le contrat strict.

### Added (P1-6 — Audit log clés API)
- **`GET /v1/me/api-keys/{key_id}/events`** (#P1-6) : journal paginé des événements d'une clé API. Enveloppe `{data, next_cursor, has_more}`, réutilise le helper pagination P1-3 (curseur HMAC TTL 24 h, forward-only, tie-break stable sur `id DESC`). `limit` max 50 (Pydantic). Anti-énum : **404 `api_key_not_found` byte-identique** si la clé n'existe pas OU appartient à un autre reseller (pattern P0-2). 400 `invalid_key_id` si UUID malformé. Rate-limit 600/min global (pas de limite spécifique).
- **3 events captés en fire-and-forget** depuis le code serveur :
  - `created` — POST `/v1/me/api-keys` réussi, `metadata={"label": "<label>"}` si le reseller en a fourni un.
  - `revoked` — DELETE `/v1/me/api-keys/{id}` réussi (*jamais* fire sur 404 ou cross-tenant, cf anti-énum P1-4).
  - `rate_limited` — chaque réponse 429 émise par `api/rate_limit.py`. **Dédup in-memory 60 s** par `api_key_id` pour éviter qu'un attaquant qui sature `/v1/*` ne remplisse la table. `metadata={"path": "/v1/...", "limit": "<N> per <unit>"}`.
- Repository `db/repositories/api_key_events.py` : `record_event` (INSERT simple), `list_for_key` (SELECT paginé curseur), `key_belongs_to_reseller` (pré-check 404 anti-énum).
- Helper `api/v1/api_key_events_hooks.py::fire_event` : ouvre une session DB dédiée (comme P0-5 dispatcher) pour que l'event ne soit pas perdu en cas de rollback de la transaction métier, et utilise `fire_and_forget` (module `api/webhooks_out/tasks.py`) pour ne pas bloquer la réponse HTTP.
- `db/repositories/api_keys.py::resolve_readonly` : variante stricte lecture seule de `resolve()`, utilisée par le handler 429 pour ne **pas** bumper `last_used_at` / `last_used_ip` sur une requête rejetée (un 429 est un échec client, pas un usage légitime).
- Schemas Pydantic `ApiKeyEventOut` (extra="forbid", Literal `event_type`), `ApiKeyEventListOut` (enveloppe curseur). Anti-BOPLA : **pas** de `api_key_id` ni `reseller_id` dans la réponse (redondant avec l'URL et le scope tenant).
- Tag OpenAPI : `API Keys` (réutilise le tag créé en P1-4). Aucun nouveau slug RFC 7807 (réutilise `api_key_not_found`, `invalid_key_id`, `invalid_cursor`, `cursor_expired`, `service_unavailable`).
- 10 tests intégration ajoutés, 0 régression. Suite : 275 → 290 passed.
- **Out-of-scope explicite** : pas d'event `used_from_new_ip` aujourd'hui (pas de cas produit concret). Le champ `event_type` reste ouvert côté DB (TEXT libre) pour permettre l'ajout sans migration.

### Added (P1-4 — CRUD clés API self-service)
- **`GET /v1/me/api-keys`** (#P1-4) : liste les clés du reseller (actives + révoquées avec `status` computed). Tri `created_at DESC`. 11 champs exposés par ligne, **jamais** de `key_hash` ni de `secret_key` — seul `key_prefix` (12 chars) pour l'UI.
- **`POST /v1/me/api-keys`** : crée une nouvelle clé. Pattern Stripe/GitHub — `secret_key` visible **UNE SEULE FOIS** dans la réponse 201 (`yk_<40 chars base62>`). Body `{"label": "optional ≤ 64 chars"}` avec `extra="forbid"`. Rate-limit **5/min** spécifique (cumulé avec 600/min global) pour éviter un abus type "régénération en boucle".
- **`DELETE /v1/me/api-keys/{key_id}`** : soft-delete (`revoked_at = NOW()`). 204 sur succès, 400 `invalid_key_id` si UUID malformé, **404 `api_key_not_found` byte-identique** sur clé inexistante OU cross-tenant (pattern anti-énum P0-2). Auto-révocation safe : la requête courante termine, la suivante 401.
- **Tracking `last_used_at` + `last_used_ip`** : helper `touch_used()` isolé (`db/repositories/api_keys.py`) appelé après chaque `resolve()` dans `get_current_reseller`. Best-effort : un UPDATE qui plante n'empêche pas l'auth de réussir. IP = `request.client.host` (INET natif PostgreSQL, IPv4/IPv6).
- Migration Alembic `0011_p1_4_api_keys_last_used_ip_and_events` : ajoute `last_used_ip INET NULL` sur `reseller_api_keys` + crée la table `api_key_events` (vide, structure livrée pour #P1-6 audit log). 2 index couvrants. Zero-downtime.
- Schemas Pydantic `ApiKeyOut`, `ApiKeyListOut`, `ApiKeyCreatedOut`, `ApiKeyCreateIn` — tous `extra="forbid"`.
- Slugs RFC 7807 : `api_key_not_found` (404), `invalid_key_id` (400).
- Tag OpenAPI dédié `API Keys` (détaché de `Account`). 3 endpoints en `x-status: live`.
- 11 tests intégration, 0 régression. Suite complète 275 passed.
- **Décisions simplificatrices tranchées** (cf `plan-p1.md §Décision 2`) : pas de scopes, pas d'`expires_at`, pas de rotation programmée. Seul levier sécurité côté reseller = révocation manuelle.

### Fixed (P0-3-ter — httpx/httpcore logger silencing)
- **Logs** (#P0-3-ter) : `httpx` et `httpcore` rabaissés à `WARNING` dans les deux points d'entrée applicatifs (API + composant téléphonie sortante). Ferme le volet PII P0-3 identifié lors du smoke P0-3-bis : `httpx` loggue au niveau INFO chaque requête avec l'URL complète (query-string inclus), or l'API Meta Graph impose `user_wa_id` en query-string sur `GET /call_permissions` → le numéro fuitait dans `journalctl` via `INFO httpx._client HTTP Request: GET .../?user_wa_id=212...`. Les erreurs réseau (timeouts, connect refused) au niveau WARNING passent toujours. 2 tests ajoutés (setLevel + integration MockTransport). Suite complète 264 passed.

### Fixed (P0-3-bis + P1-2-bis — correctifs post-audit)
- **Dispatcher webhook** (#P1-2-bis) : `target_url`, `latency_ms`, `next_retry_at` renseignés aussi pour les events réels (`call.ended`, `call.failed`, etc.), plus seulement `POST /v1/me/webhooks/test`. Latence mesurée via `time.monotonic()` (pas de saut d'horloge NTP) autour de chaque tentative, y compris sur timeout/connect_error. `next_retry_at` = `now + _BACKOFF_SECONDS[next_idx]` si retry prévu, `None` si succès 2xx ou dernière tentative atteinte. Le cas SSRF-reject garde `target_url=url, latency_ms=None, next_retry_at=None`. 4 tests intégration ajoutés.
- **Masquage PII** (#P0-3-bis) : `redact_wa_id` appliqué aux logs et messages d'exception des modules `channels/voice/whatsapp_permission.py` (10 sites : 8 `logger.*` tags `[WHATSAPP perm]`/`[QUOTA]` + 2 `raise PermissionTimeout`/`PermissionRefused`) et `channels/voice/whatsapp_permission_waiter.py` (5 sites tag `[WA-PERM-WAITER]`). Complète P0-3 qui n'avait ciblé que `api/routes.py`. Valeurs métier (arg DB, payload Meta `to=wa_id`) inchangées — masquage strictement log-side. 1 test intégration `caplog` ajouté.

### Added (P1-2 — observabilité webhooks)
- **`POST /v1/me/webhooks/test`** (#P1-2) : déclenche un `webhook.test` synchrone vers l'URL configurée du reseller. Retourne immédiatement `{test_id, delivered, http_status, latency_ms, target_url, attempted_at, error_message}`. Timeout 5 s (pas de retry, contrairement au dispatcher P0-5). `follow_redirects=False`. Signature HMAC-SHA256 réutilisée du dispatcher (`api/webhooks_out/signing.py`). **Rate-limit 10/min spécifique** (cumulé avec 600/min global). 400 `webhook_url_not_configured` si pas de webhook actif.
- **`GET /v1/me/webhooks/deliveries?cursor=&limit=&status=&event_type=&since=&until=`** : historique paginé des livraisons. 11 champs exposés par ligne (`delivery_id`, `event_type`, `target_url`, `http_status`, `latency_ms`, `attempt_count`, `status` computed `delivered`/`failed`/`pending`, `next_retry_at`, `created_at`, `last_attempted_at`, `error_message` tronqué 200 chars). Anti-BOPLA strict : pas de `response_body`, pas de `signature_sent`, pas d'`event_id` en clair. Réutilise helper pagination P1-3.
- Migration Alembic `0010_p1_2_webhook_deliveries_extras` : ajoute 3 colonnes nullables sur `webhook_deliveries` (`target_url VARCHAR(500)`, `latency_ms INT`, `next_retry_at TIMESTAMPTZ`). Zéro impact dispatcher P0-5 (colonnes optionnelles).
- Slugs RFC 7807 : `webhook_url_not_configured` (400), `invalid_status_filter` (400, liste des valeurs valides dans le `detail`).
- 23 tests intégration verts (8 endpoint test + 15 endpoint deliveries). Suite complète 257 passed, 0 régression.
- Filtre `?status=` valide manuellement (pas Pydantic Literal) → tolérant à une future extension sans breaking change client.

### Added (P1-1 — endpoints self-service `/v1/me/*` lecture)
- **`GET /v1/me`** (#P1-1) : identité du reseller, 5 champs `reseller_id`, `name`, `email` (col DB `contact_email`, nullable), `created_at`, `rate_limit_per_minute` (constante serveur 600/min aujourd'hui). Anti-BOPLA : ni `balance_seconds` (cf `/v1/me/balance` P0-2) ni `status`.
- **`GET /v1/me/transactions?cursor=&limit=&since=&until=`** : historique paginé du ledger `credit_transactions`. Réutilise le helper pagination P1-3 (curseur HMAC TTL 24 h, tri `created_at DESC` + tie-break `id DESC`, défaut 50 max 200). Expose `transaction_id`, `type` (`topup`/`debit_call`/`refund`/`adjustment`), `seconds` (signé — positif = crédit, négatif = débit), `reference`, `reason`, `created_at`.
- **`GET /v1/me/usage?period=YYYY-MM`** : agrégation mensuelle. Défaut = mois courant UTC. Fenêtre semi-open `[period_start, period_end)` sur `calls.created_at`. Retourne `total_calls` / `total_seconds` + breakdown par `call_status` (`completed_calls`/`completed_seconds`/`failed_calls`/`cancelled_calls`). Période sans appel → **200 avec zéros** (pas 404). Slug `invalid_period_format` (400) si le format ne matche pas `^\d{4}-(0[1-9]|1[0-2])$`.
- Helper pagination P1-3 réutilisé tel quel — pas de générique-isation forcée (principe YAGNI : P1-2 déterminera si une abstraction commune est justifiée).
- Aucune migration DB nécessaire : l'index `ix_credit_tx_reseller_created` (M3.6) couvre la pagination ; l'agrégation usage tourne sur les index existants `ix_calls_reseller_*` (P1-3) et filtre par `created_at` range + `call_status`.
- 21 tests intégration verts. Suite complète 234 passed, 0 régression.
- **Scope restreint** : CRUD `/v1/me/api-keys` déplacé en P1-4 (cf `.claude/context/plan-p1.md`). `/v1/me/balance` reste intact (livré P0-2).

### Added (P1-3 — pagination `GET /v1/calls`)
- **`GET /v1/calls`** : nouvelle liste paginée scopée tenant (#P1-3). Enveloppe `{data, next_cursor, has_more}`, défaut 50, max 200 (Pydantic valide → 422 `validation_error` au-delà).
- **Curseur opaque signé HMAC-SHA256** (`api/pagination.py`) : forward-only, TTL 24 h, tie-break stable sur `id DESC` quand deux rows partagent `created_at`. Résistant au forgeage client et au cross-reseller (le `WHERE reseller_id = :me` prime côté SQL, le `id` dans le curseur sert uniquement au tie-break).
- **4 filtres combinables** : `?status=` (`queued`/`dialing`/`in_progress`/`ended`/`failed`), `?merchant_id=<uuid>`, `?since=<iso>`, `?until=<iso>`.
- **Nouvelle variable env `CURSOR_SIGNING_KEY`** (requise en prod, `openssl rand -hex 32`). Endpoint renvoie `503 service_unavailable` avec log `[P1-3] CURSOR_SIGNING_KEY manquant` si vide — fail-closed explicite, pas crash au startup.
- **3 nouveaux index DB** (migration Alembic `0009`) : `idx_calls_reseller_created`, `idx_calls_reseller_status_created`, `idx_calls_reseller_merchant_created` — tous avec `created_at DESC` pour aligner avec le tri handler.
- Slugs RFC 7807 : `invalid_cursor`, `cursor_expired`, `invalid_pagination_param` (limit ≤ 0 / > 200 tombe sur `validation_error` Pydantic, cohérent avec le pattern `/v1/*`).
- 32 tests verts (15 unit helper + 15 intégration endpoint + 2 tiebreak/anti-énum tenant). Suite complète 213 passed, 0 régression.

### Added (P1-5 — middleware X-Request-ID)
- **Header `X-Request-ID`** sur toutes les requêtes `/` (#P1-5). Chaque requête reçoit un UUID v4 unique :
  - Respecte la valeur entrante si format UUID v4 strict valide (RFC 4122). Rejette et régénère si UUID v1/v3/v5, hex invalide, vide, ou injection log (`"abc\ndef"`, path traversal).
  - Echoed en header sur la response.
  - Injecté dans `request.state.request_id`, dans les réponses d'erreur RFC 7807 (champ `request_id` du body = header), et dans les logs applicatifs (ContextVar + `RequestIdLogFilter`). Format log mis à jour : `%(asctime)s %(levelname)s [%(request_id)s] %(name)s %(message)s`.
  - Middleware enregistré en premier dans la stack Starlette → tourne AVANT l'auth, donc même les 401 reçoivent un request_id.
- 9 tests unitaires (`_extract_or_generate`) + 6 tests intégration (TestClient flow). 0 régression sur 181 tests.

### Changed (P0-3 — masquage PII logs + rétention DB)
- **PII masquée dans les logs applicatifs** (#P0-3) : `api/_redact.py` expose `redact_wa_id` (SHA-1 hex tronqué 12 chars, préfixe `wa_`, consistant pour corréler events sans ré-identifier) et `redact_name` (3 premiers codepoints + ellipsis, unicode-safe pour darija arabe). 6 sites `logger.*` patchés dans `api/routes.py` (RAW dump body retiré, call event / status / permission from / dispatch wa_id / DB échec wa_id). **Aucun impact** sur les tables métier (clients, orders, calls, transcripts, merchants, resellers) — masquage strictement log-side.
- **`_debug_save_whatsapp_webhook` gardé derrière `ENV=dev`** : en prod, plus aucun fichier brut n'est écrit dans `/opt/yasmine/logs/whatsapp_webhooks/`. `webhook_raw` (DB) reste la source de vérité, purgée à 30j.

### Added (P0-3 — script rétention)
- **`scripts/retention_purge.py`** : purge `webhook_raw.payload` (`-> '{}'::jsonb` car NOT NULL) et `call_events.data.raw` (`data - 'raw'`) > 30 jours. UPDATE, **pas DELETE** : préserve l'idempotence `webhook_raw.dedup_hash` et l'intégrité des partitions mensuelles `call_events`. Flags `--dry-run` (COUNT seul) et `--older-than-days N`. Cron systemd suggéré : 1×/mois (cf `docs/infrastructure.md`).
- **`ops/logrotate.d/yasmine-whatsapp-webhooks`** : config logrotate (rotation 7j, gzip après 1j, mode 0640) pour `logs/whatsapp_webhooks/`. Garde-fou si `ENV=dev` activé par erreur en prod. Installation manuelle une fois (`sudo cp ... /etc/logrotate.d/`).

### Changed (P0-4 + P0-6 — sécurité hygiène)
- **Doc auto FastAPI fermée** (#P0-4) : `openapi_url=None`, `docs_url=None`, `redoc_url=None` dans `main.py`. `GET /openapi.json`, `/docs`, `/redoc` → 404 (prod et dev). Scalar (`docs.yasmine.akidly.com`) continue de servir `docs/openapi.yaml` curé via Nginx — surface publique unique. Pas d'override env volontairement (évite l'oubli en prod après debug).
- **Limite 256 KiB sur `POST /webhooks/whatsapp`** (#P0-6) : `read_request_body_bounded()` itère `request.stream()` chunk par chunk et raise `WebhookBodyTooLarge` dès `total > 262144`. Réponse `413 Payload Too Large` RFC 7807 (slug `payload_too_large`). **Vérification AVANT HMAC** : économise CPU + early-rejection des payloads suspects (Meta envoie ~50 KB en pratique, on a 5× de marge). Anti-DoS / anti-OOM si Meta falsifie un payload avec signature valide mais 100 MB.
- 9 tests unitaires verts (3 doc auto 404 + 6 body limit avec early-rejection).

### Changed (P0-5 — dispatcher webhook garde-fous techniques)
- **`follow_redirects=False`** explicité dans `httpx.AsyncClient` côté dispatcher (#P0-5). Defense-in-depth SSRF : un 3xx vers une IP privée bypasserait notre `validate_webhook_url` qui ne valide que l'URL initiale.
- **Body response lu en streaming**, borné à 1024 B via `_read_body_bounded` (`aiter_raw()` + break dès `total >= max_bytes`). Anti-OOM si reseller hostile / mal configuré renvoie plusieurs MB. Plus aucun load RAM complet du body.
- **`fire_and_forget(coro)`** helper (`api/webhooks_out/tasks.py`) : retient les `asyncio.Task` dans un set module-level + `add_done_callback(_pending_tasks.discard)`. Protection contre le GC Python 3.11+ qui peut reaper une task orpheline pendant les `asyncio.sleep` (jusqu'à 5 min entre attempts). Cf [doc Python officielle](https://docs.python.org/3/library/asyncio-task.html#asyncio.create_task) : "the event loop only keeps weak references to tasks".
- **2 sites patchés** : `api/webhooks_out/dispatcher.py:72` (le seul `create_task` réel pour `_run_with_retries` — couvre transitivement `_record_permission_reply` et `_dispatch_to_db` qui font tous deux `await dispatch_webhook(...)`) ; `api/v1/calls.py:493` (cancel C8). `agent/origination.py` et `main.py` lifespan janitor hors scope (cycles de vie distincts).
- 9 tests unitaires nouveaux (`test_webhook_guardrails.py`) + 16 tests C6 Phase 2 maintenus 100% verts (refactor `client.post` → `client.stream` reflété dans `_patch_httpx_post`).

### Added (P0-2 — endpoints read scopes tenant)
- **`GET /v1/calls/{call_id}`** : récupération scopée tenant d'un appel (#P0-2).
  - SELECT composite `WHERE id=? AND reseller_id=?` — anti-BOLA, jamais `session.get(Call, UUID)` qui ne filtre pas le tenant (audit v2 §3.1).
  - **404 `call_not_found` byte-identique** sur les 3 cas : UUID syntaxiquement invalide, UUID inconnu, UUID propriété d'un autre reseller. Aucune fuite d'existence ni de propriétaire (anti-énumération). Detail générique `"Aucun appel trouve pour cet identifiant."` partagé par les 3 branches.
  - Schéma `CallOut` enrichi : `ringing_at`, `connected_at` (= `accepted_at` DB), `ended_at`, `billed_seconds`, `failure_reason`, `cancelled_state`. Tous Optional, `None` tant que l'étape n'est pas atteinte.
  - `purpose`, `amount`, `currency`, `country` deviennent Optional dans `CallOut` (défensif pour le GET sur calls historiques sans données complètes). Pas de breaking pour le POST qui fournit toujours ces valeurs.
  - Rate-limit 600/min (hérité C7 reads).
- **`GET /v1/me/balance`** : consultation du solde reseller.
  - 3 champs : `balance_seconds` (int, peut être négatif jusqu'à `-MAX_OVERDRAFT_SECONDS`), `currency` (`null` aujourd'hui — balance en secondes pures), `updated_at` (`MAX(credit_transactions.created_at)` ou `reseller.created_at` en fallback).
  - Rate-limit 600/min.
  - Anti-BOPLA : pas de `overdraft_limit_seconds` (paramètre interne), pas de `estimated_minutes_remaining` (fausse précision).
- Repos : `db/repositories/calls.py::fetch_by_id_for_reseller`, `fetch_call_view_for_reseller` (JOIN order/customer/merchant + filtre tenant) ; `db/repositories/resellers.py::get_balance_snapshot` + dataclass `BalanceSnapshot`.
- 12 tests unitaires verts (8 GET call + 4 GET balance).

### Added (P0-1 — Idempotency-Key obligatoire)
- **`Idempotency-Key` obligatoire** sur `POST /v1/calls` (#P0-1).
  - TTL 24 h. Format libre, 1-255 chars, scope par reseller (PK composite `(reseller_id, key)`).
  - **Replay** transparent si (clé, fingerprint body) identiques : la réponse stockée (status_code + body + headers) est rejouée bit-for-bit avec `X-Idempotent-Replay: true`. Aucune ré-exécution de `prepare_call`, aucun nouveau débit, aucun template Meta supplémentaire.
  - **Conflit** `409 idempotency_key_conflict` si la clé est réutilisée avec un body différent.
  - **400 RFC 7807** : `missing_idempotency_key` (header absent), `idempotency_key_empty` (vide), `idempotency_key_too_long` (> 255 chars).
  - **Race-safe** : `pg_advisory_xact_lock(hashtext("idem:<reseller>:<key>"))` au début du handler — sérialise les requêtes concurrentes sur la même clé. Pattern aligné sur C5/C8. INSERT à la fin du handler dans la même transaction (commit unique via `get_db`) → aucune row zombie en cas de plantage.
  - **Fingerprint** = SHA-256 hex (64 chars) du body brut, déterministe pour les retries legit (Stripe-style).
  - **Cleanup** : pas de tâche applicative — purge cron systemd hebdomadaire (cf `docs/infrastructure.md`).
- Migration Alembic `0008_p0_1_idempotency_keys.py` : table `idempotency_keys` (PK composite, ON DELETE CASCADE sur reseller, index `ix_idempotency_keys_expires`).
- Repository `db/repositories/idempotency.py` : `lookup_or_reserve`, `persist_response`, `cleanup_expired`, enum `IdempotencyAction` (FIRST/REPLAY/CONFLICT).
- 11 tests unitaires (header missing/empty/too_long, FIRST persist, REPLAY same fingerprint, CONFLICT, expired→FIRST, scope reseller, séquence concurrente, Location preserved, cleanup DELETE).

### Added (audit v2 — post-M3.6)
- `.claude/context/audit-2026-04-v2.md` — nouvel audit API 3 volets reflétant l'état post-P0 (M2 suppression, M3.1-M3.4 reconstruction /v1, M3.6 C1-C8). Diff explicite avec l'audit du 2026-04-19 pour chaque item (statut ✅/⚠️/❌/➖).
- `.claude/context/roadmap.md` refondue : P0 resserré à 6 items bloquants (idempotency storage, endpoints read scopés tenant, fuites PII logs/disque, OpenAPI publique, dispatcher body-size/redirects, body-max webhooks Meta). Ancienne roadmap avril conservée en section `## Archive`.
- `CLAUDE.md` section Documents de référence : audit-2026-04-v2.md devient la source active, audit-2026-04.md passe en archive historique.

### Added (M3.6 C7 — rate-limit par clé API)
- **Rate-limit** sur tous les endpoints `/v1/*` authentifiés, scope = clé API (hash SHA-1 tronqué 16 chars du raw Bearer, déterministe pré/post-auth).
  - Seuils : `POST /v1/calls` = 60/min, `POST /v1/calls/{id}/cancel` = 120/min, `GET /v1/me/webhooks` (reads) = 600/min, `POST/DELETE /v1/me/webhooks` (config) = 10/min.
  - Headers `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` sur toutes les réponses 2xx (injectés par slowapi via le param `response: Response` des endpoints décorés).
  - **429 RFC 7807** `rate_limit_exceeded` avec header `Retry-After` + champ `retry_after` dans le body. Content-Type `application/problem+json`.
  - **Kill switch** `RATE_LIMIT_ENABLED` (env var, default `true`). Bascule à `false` → bypass total sans redeploy code.
  - **Backend** : in-memory (1 worker uvicorn en prod, cohérent). Bascule Redis automatique si `REDIS_URL` présent.
  - **Exclus** : `POST /webhooks/whatsapp` (webhooks Meta entrants hors router `/v1`), `GET /healthz` (nouvelle endpoint monitoring ops, léger `{"status":"ok"}`), `/docs`, `/openapi.json`.
  - **Architecture** : décorateurs `@limiter.limit(...)` seuls, **sans** `SlowAPIMiddleware`. Raison : le middleware slowapi tourne pré-Depends et hit le bucket en parallèle du décorateur → double-count (observé 30/60 au lieu de 60/60 en test). Décorateur seul = 1 hit + headers injectés automatiquement.
- `api/rate_limit.py` : `limiter`, `get_api_key_id` (key_func déterministe par hash court du Bearer), `rate_limit_exceeded_handler` (RFC 7807 avec headers).
- `GET /healthz` endpoint public minimaliste (200 OK toujours).
- `api/deps.py` : `get_current_reseller` pose `request.state.api_key_id` et `request.state.reseller_id` (cohérence logs, pas source de vérité rate-limit).
- 11 tests unitaires passants (happy path sous limite + 429 dépassement, buckets séparés par clé, headers 2xx et 429, seuils dédiés cancel/reads/config, exclusions webhooks Meta et healthz, kill switch bypass, format RFC 7807).
- Docs : CHANGELOG, `docs/errors.md` (entrées `rate_limit_exceeded`, `call_not_found`, `webhook_*`), `docs/getting-started.md` (§Rate limits avec table des seuils + snippet retry Python).

### Added (M3.6 C8 — cancel reseller)
- `POST /v1/calls/{call_id}/cancel` : le reseller annule un appel en cours. Verbe POST (pas DELETE) — action sur ressource vivante, pattern Stripe-like.
  - **Matrice** : `queued/dialing/ringing` → 200 `billed_seconds=0` (jamais facturé, jamais connecté). `connected` → 200 avec `billed_seconds = max(ceil(now - accepted_at), 10)` et `debit_for_call` immédiat (ledger idempotent, évite double-débit si le webhook `terminate` Meta arrive après). `ended/failed/cancelled` → **200 no-op idempotent**, aucune mutation, **aucun event** ré-émis.
  - **Scoping** : filtre `reseller_id = <caller>` dans la clause SELECT → 404 `call_not_found` si le call appartient à un autre reseller (pas de leak d'existence).
  - **Race-safe** : `pg_advisory_xact_lock(hashtext(call_id))` en tête de transaction + `SELECT ... FOR UPDATE` — protège contre un webhook Meta concurrent qui passerait ringing → connected pile pendant le cancel (même pattern que C5).
  - **Pipeline cleanup** : nouveau helper `cancel_call(call_id)` côté canal WhatsApp Business → cancel la task pipeline (timeout 2s), POST Meta `action=terminate` pour couper la sonnerie, nettoie les 3 dicts internes (`_pipeline_tasks`, `_connections`, `_meta_to_yasmine`). Idempotent.
  - **Event webhook** : `call.cancelled` émis uniquement sur cancel effectif (distinct de `call.ended` — fin naturelle). Payload : `{call_id, cancelled_at, cancelled_state, billed_seconds, merchant_id}`. 11 events au total dans le catalogue.
  - **Enum extension** : `call_status` ajoute `'cancelled'` (migration Alembic `0007_c8_cancel_state.py` — DROP/CREATE `ck_calls_call_status`). `TERMINAL_CALL_STATES` Python inclut désormais `cancelled`. Colonne legacy `status` GENERATED mappe `cancelled` → `'failed'` (sous-type d'échec côté surface M3.0-M3.5 inchangée).
  - **Métadonnées DB** : `calls.metadata.cancelled_state` (snapshot du call_status au moment du cancel), `cancelled_at` (ISO Z), `billed_seconds`.
  - 12 tests unitaires passants (queued, ringing, connected avec debit, min-10s floor, 3 idempotence terminaux, wrong reseller, unknown id, malformed uuid, failed terminal, advisory lock).
- `api/v1/errors.py` : slug `call_not_found` mappé 404.
- `docs/webhooks.md` §7 : entrée `call.cancelled` avec payload exemple + distinction explicite avec `call.ended`.
- `docs/examples.md` : recette curl `POST /v1/calls/{id}/cancel`.
- `docs/openapi.yaml` : opération `cancelCall` (tag `Calls`, `x-status: live`), schéma `CallCancelOut`.

### Added (M3.6 C6 Phase 2 — dispatcher + émission réelle)
- **Émission réelle** des webhooks sortants reseller. `api/webhooks_out/dispatcher.py` : `dispatch(event_type, call_id, data, reseller_id)` fire-and-forget, retry **3 tentatives / 0s · 30s · 5 min**, timeout httpx **10 s total** (DNS + connect + send + receive), SSRF re-check à chaque POST (DNS peut avoir été compromis depuis la config), signature `X-Yasmine-Signature: sha256=<hex>`, event id `evt_<ulid>`, header `X-Yasmine-Event-Id` pour l'idempotence côté reseller.
- Chaque tentative trace une row dans `webhook_deliveries` (status_code, response_body tronqué 1024 chars, error, delivered_at si 2xx). Si SSRF re-check rejette → 1 row `error=url_no_longer_valid:<reason>`, zéro retry.
- **10 events** émis :
  - Rail DEMANDE : `call.request.accepted`, `call.request.refused`, `call.request.expired`, `call.request.quota_blocked`, `call.request.permission_granted_late` (nouveau : accept Meta tardif alors que rail DEMANDE déjà terminal → metadata `late_accept_at` + event avec hint "re-submit POST /v1/calls").
  - Rail APPEL : `call.started` (→ dialing), `call.ringing`, `call.connected`, `call.ended` (payload = result + duration_s + billable_s), `call.failed` (infra : reject client, Meta 5xx, crash origination, reclaim lifespan, janitor projection).
- Règle d'ordonnancement : si le rail DEMANDE termine en échec (refused/expired/quota_blocked/cancelled_by_reseller), aucun event du rail APPEL n'est émis. Structurellement respecté : les hooks `call.*` vivent dans des chemins qui ne s'exécutent pas si la demande bail early.
- `api/webhook_dispatch.py::dispatch_whatsapp_event` et `db/finalize_whatsapp.py::finalize_whatsapp_call` retournent désormais `(result, pending_webhook_events)`. Le caller (`_dispatch_to_db`) consomme le buffer et fire **APRES commit** — pattern identique au bug-fix #4 du C4 (évite les tasks spawn sur DB stale).
- Hooks post-commit dans : `agent/origination.py` (call.started + call.request.{expired,quota_blocked,failed}), `api/routes.py::_record_permission_reply` (call.request.{accepted,refused,permission_granted_late}), `main.py::_reclaim_stuck_calls` (call.failed par row reclaimée au lifespan startup), `agent/reconciliation.py::_apply_projection` (call.failed quand le janitor projette un échec ; `call.ended` non fire côté janitor car payload complet result/duration/billable non disponible hors `finalize_whatsapp_call`).
- §1.0 P7 contraintes payload `quota_blocked` : `reason=recipient_not_authorized` + `limit_type` technique (`1_per_24h` / `2_per_7d` / `4_no_answer_revoked`). Pas de leak `blocking_merchant_id` / `last_sent_by` / énumération calls précédents.
- `ulid-py>=1.1.0` ajouté à `requirements.txt`.
- 16 tests unitaires (8 dispatcher happy path + retry + timeout + SSRF re-check + signature + backoff timings ; 8 hooks branches dans webhook_dispatch et finalize_whatsapp).
- `docs/webhooks.md` catalogue complet 10 events + payloads exemples + contraintes P7.

Crash mid-retry : les retries via `asyncio.sleep` sont perdus au crash process (décision produit 2026-04-20). Edge case rare, le reseller polle en fallback.

### Added (M3.6 C6 Phase 1 — fondations webhooks sortants reseller)
- `POST/GET/DELETE /v1/me/webhooks` : configuration self-service du webhook sortant unique par reseller.
  - Secret HMAC-SHA256 (256 bits d'entropie, base64url) généré à la création et **affiché une seule fois en clair** dans la réponse `201`. Stocké en clair côté DB (obligatoire pour signer, standard Stripe/Twilio). `GET` ne renvoie jamais que `secret_prefix` (6 chars + `…`).
  - `POST` sur un reseller déjà configuré → `409 webhook_already_configured`. Rotation volontaire = `DELETE` + `POST`.
  - `DELETE` = soft-delete (`active=false`). L'historique `webhook_deliveries` reste intact.
  - **SSRF guard** `api/webhooks_out/ssrf.py` : rejet des URLs vers IPs privées (RFC 1918 + loopback + link-local + IPv6 ULA), `localhost`, TLD `.local`, scheme non-`https` (hors `ENV=dev`). 5 reasons machine-readable : `invalid_url`, `scheme_not_allowed`, `localhost_rejected`, `dns_resolution_failed`, `private_ip_rejected`.
- Utilitaire `api/webhooks_out/signing.py` : `sign_payload(secret, raw_body) → "sha256=<hex>"` + `verify_signature` timing-safe. Format du header `X-Yasmine-Signature` documenté dans `docs/webhooks.md §5`.
- Nouvelle exception `agent/exceptions.py::ResellerWebhookAlreadyExists`.
- Repository `db/repositories/reseller_webhooks.py` (distinct de `webhooks.py` qui sert le sink Meta entrant).
- Migration Alembic `0006_c6_reseller_webhooks.py` : tables `reseller_webhooks` (PK = `reseller_id`, ON DELETE CASCADE) + `webhook_deliveries` (BIGSERIAL, 3 indexes). Models SQLAlchemy `ResellerWebhook` + `WebhookDelivery`.
- Tag OpenAPI `Webhooks` dédié aux endpoints `/v1/me/webhooks*` dans `docs/openapi.yaml` + schemas alignés sur l'impl live (`WebhookCreateInput`, `WebhookCreated`, `WebhookGet`, `WebhookUrlRejectedProblem`).
- `docs/webhooks.md` réécrit : §1 (live) décrit la config self-service ; §2-§7 documentent le contrat cible (payload, retry 3×/0s/30s/5min, signature, idempotence event_id) figé pour la Phase 2.
- 23 tests unitaires (`tests/test_webhook_ssrf.py` 12 + `tests/test_webhooks_config.py` 11) : endpoints happy-path + conflict + SSRF + get_prefix + delete soft + sign/verify round-trip.

**L'émission réelle** des events (dispatcher async + hooks dans les transitions d'état) arrive en **Phase 2** — la configuration aujourd'hui ne déclenche aucun trafic sortant.

### Added (M3.6 C5 — idempotence events webhook)
- Advisory lock PostgreSQL (`pg_advisory_xact_lock`) sur chaque event webhook Meta `calls[]` pour sérialiser le traitement par `call_id`. Protège contre les doubles spawns lors des retries Meta (>20 s response) et les events out-of-order. Clé : `biz_opaque_callback_data` (call_id Yasmine UUID) en priorité, fallback `event.id` (wacid Meta) si absent. Auto-release au COMMIT.
- `calls.metadata.pipeline_started_at` : timestamp ISO de décision de spawn pipeline, posé à la transition `accepted → connected`. COALESCE-safe (no-op si déjà présent, race résiduelle entre 2 webhooks `accepted` consécutifs ne l'écrase pas). Utile pour debug du délai accept→ready.
- `db/repositories/calls.py::get_status(session, call_id)` : helper SELECT `call_status` pour le check anti-double-spawn côté channel.
- `db/repositories/calls.py::mark_pipeline_started_at(session, call_id)` : UPDATE conditionnel JSONB avec `NOT (metadata ? 'pipeline_started_at')`.

### Changed (M3.6 C5)
- `api/routes.py` : inversion d'ordre pour les events `calls[]` — DB d'abord (sous advisory lock), channel side effects (spawn / cancel / cleanup) ensuite. Le channel lit désormais une DB à jour. `statuses[]` et `messages[]` conservent leur ordre historique (hors scope C5).
- Handler webhook côté canal WhatsApp Business, branche `accepted` : check anti-double-spawn via `calls_repo.get_status(yasmine_id)` (refuse si `connected`/`ended`). Ancien check RAM `if yasmine_id in self._pipeline_tasks` retiré. Dict `_pipeline_tasks` conservé pour son usage `task.cancel()` au `terminate/reject` (cf audit).

### Changed (M3.6 — plafond d'overdraft, C3 reporté)
- `check_balance` honore désormais `MAX_OVERDRAFT_SECONDS` (default 300 s) : refuse un POST `/v1/calls` si le solde descend sous le plancher. Protection anti-clé volée, n'empêche pas les dérives mineures en concurrence (C3 complet reporté post-M3.6, voir `roadmap.md`).
- Docstring de `check_balance` corrigé : le lock `FOR UPDATE` ne protège pas l'overdraft, il sérialise seulement les lectures (l'ancienne doc prétendait l'inverse — erreur factuelle).

### Fixed (M3.6 C4 — 5 hot-fix post-smoke prod)
- **Fix #1** `db/repositories/quotas.py` : CTE streak filtre `WHERE result IS NOT NULL` pour exclure le call en cours (fraîchement créé par POST, `result=NULL`) du `LIMIT 4`. Sans ce filtre, la streak était sous-estimée de 1 (3 observé au lieu de 4 en prod), guard `auto_revoke_imminent_no_answer_streak_4` jamais déclenché à 4 sans 5+ NO_ANSWER. Commit `4fb529f`.
- **Fix #2** `channels/voice/whatsapp_permission.py::_record_template_send` : SELECT `Call.merchant_id` + INSERT `template_sends` regroupés dans le même `async with session.begin()`. Avant, le SELECT hors begin déclenchait l'autobegin SQLAlchemy 2.0 → `begin()` raisait `A transaction is already begun` → INSERT silencieusement skippé → compteurs `templates_24h/7d` restaient à 0 en prod → guard quota templates fonctionnellement contourné. Commit `4fb529f`.
- **Fix #3** `api/routes.py::_record_permission_reply` : même pattern que #2. SELECT `TemplateSend.merchant_id, call_id` + `perm_repo.record` + `template_sends_repo.update_status` + `calls_repo.mark_request_status` regroupés dans le même `begin()`. Avant, INSERT `permission_history` + transition `waiting_user_permission → permission_accepted` silencieusement skippés → cascade `InvalidStateTransition` dans `run_origination` → flow accept/refuse via template cassé depuis `bc63503` (C2). Commit `c2ffa72`.
- **Fix #3-bis** `db/finalize.py::finalize_if_tracked` : même pattern préventif. Le canal WhatsApp Business actif n'utilise pas cette fonction (`finalize_whatsapp.py` en place), bug latent non déclenché en prod. Fix inclus par hygiène anti-whack-a-mole. Commit `c2ffa72`.
- **Fix #4** `api/routes.py` L.122-128 : inversion d'ordre `_record_permission_reply` puis `permission_waiter.deliver`. Avant, la Future résolue débloquait `ensure_call_permission` → `run_origination` poursuivait avec DB stale (transition `permission_accepted` pas encore commitée, ~50 ms d'écart) → `mark_request_status(ready_to_dial)` refusé par matrice. Fix post-#3 : DB commit d'abord, waiter ensuite. Commit `343228a`.
- **Fix #5** `db/finalize_whatsapp.py::_resolve_result` : discrimine `NO_ANSWER` vs `FAILED` via `call.accepted_at`. Avant, tout call sans `pending_result` LLM mappé en `FAILED` → **aucune** row n'atteignait `result='NO_ANSWER'` en prod → CTE streak toujours 0 → guard 4-NO_ANSWER **dead code en conditions réelles** (fonctionnait uniquement dans les tests avec INSERT artificiels). Règle post-fix : `pending_result=None` + `accepted_at IS NULL` → NO_ANSWER ; `accepted_at NOT NULL` → FAILED + `fallback_reason`. Commit `0d14a21`.
- Détails complets + 9 scénarios smoke (A/B/C/F/E/D-original/D-replay/D-replay2/G) documentés dans `.claude/context/smoke-history.md` (nouveau fichier, source de vérité sémantique code).

### Changed (M3.6 C4)
- Guards quotas Meta WhatsApp avant POST `call_permission_request` : `db/repositories/quotas.py::check_quota(wa_id)` retourne un `QuotaVerdict` avec 3 compteurs (`templates_24h`, `templates_7d`, `no_answer_streak`) calculés en une CTE combinée (1 round-trip, indexes existants `ix_template_sends_wa_sent` + `ix_calls_wa_id_created`).
- `channels/voice/whatsapp_permission.py::ensure_call_permission` : guard `no_answer_streak < 4` AVANT `GET /call_permissions` Meta (proactif, §4.2 révocation auto après 4 NO_ANSWER). Guard `templates_24h < 1` et `templates_7d < 2` DANS la branche `no_permission` seulement (§4.3). Logs `[QUOTA] wa_id=... call_id=... verdict counts=24h:N 7d:M streak:K` structurés.
- Nouvelle exception `agent/exceptions.py::QuotaExceeded(reason)`. `run_origination` la catche → `request_status='quota_blocked'` → chaîné `done` avec `failure_reason='quota:<reason>'`.
- `db/repositories/calls.py` : ajout `"quota_blocked"` à `_ALLOWED_REQUEST_TRANSITIONS["queued"]` (cas où `QuotaExceeded` raise AVANT `mark_request_status('waiting_user_permission')`). **Fix latent C2** : `TERMINAL_REQUEST_STATES` réduit à `{"done"}` — les pré-terminaux (`permission_refused`, `permission_expired`, `quota_blocked`, `cancelled_by_reseller`) doivent pouvoir chaîner vers `done` via l'escape hatch. L'ancien comportement bloquait la chaîne pré-terminal → done mais n'avait jamais été déclenché en prod (aucun des 4 cas n'était atteint avant C4).
- Aucune migration DB : table `template_sends` + index `ix_template_sends_wa_sent` présents depuis 0001.
- Race condition 2 POST concurrents sur même wa_id : assumée (Option B, arbitrage Phase 1). Coût estimé ~$0.005/occurrence (template UTILITY hors CSW). `# TODO: revisit with advisory lock if volume x100 or quality rating degraded`.
- `docs/openapi.yaml` : enrichissement description enum `request_status=quota_blocked` avec les 3 sous-cas et pointeur vers `calls.failure_reason`.
- Tests pytest : 15 nouveaux (10 repo matrix incluant frontières 24h/7j/streak + priorités + reset accepted_at, 2 origination quota_blocked, 3 integration permission guards streak/permanent/templates). Helpers `WA_ID`, `_insert_template`, `_insert_call`, `_isolate_wa_id_c4` centralisés dans `tests/db/conftest.py` (tech debt évitée). Fixture `db_session` dispose aussi l'engine global `db.database.engine` pour éviter `RuntimeError: Event loop is closed` quand `run_origination` ouvre ses propres sessions.

### Changed (M3.6 C2)
- `POST /v1/calls` passe en asynchrone : retour 201 en <500 ms avec `request_status=queued`, `call_status=not_started`. Permission WhatsApp + place_call Meta tournent en background task (`agent/origination.py::run_origination`). Plus de blocage 3 min côté reseller.
- State machine split : 2 enums applicatifs `request_status` (9 valeurs, rail DEMANDE côté Yasmine) + `call_status` (6 valeurs, rail APPEL côté téléphonie). Matrice de transitions + escape hatch vers `done` avec `failure_reason` requis depuis non-terminal. Gardes applicatives dans `db.repositories.calls.mark_request_status` / `mark_call_status` (raise `InvalidStateTransition`).
- Migration Alembic 0005 : ajout `calls.request_status`, `calls.call_status`, `calls.wa_id`, `calls.failure_reason` (TEXT + CHECK PG). Backfill depuis l'ancien enum `status` + jointure `orders→customers` pour `wa_id` (fallback `regexp_replace(phone_e164, '[+\s]', '', 'g')`). Ancien enum PG `call_status` droppé.
- `calls.status` devient une colonne PostgreSQL `GENERATED ALWAYS AS (...) STORED` dérivée des rails. Lecture seule — toute tentative d'UPDATE direct est rejetée côté PG. Compat GET /v1/calls inchangée pour les resellers M3.0–M3.5.
- Indexes ajoutés : `ix_calls_wa_id_created` (pour C4 futur + janitor), `ix_calls_request_status_open` partiel (reclaim lifespan + janitor), `ix_calls_active` recréé sur col générée.
- Scan lifespan reclaim (`main.py::_reclaim_stuck_calls`) : au démarrage, calls avec `request_status IN (queued, waiting_user_permission, permission_accepted, ready_to_dial)` ET `updated_at < NOW() - 10 min` sont marqués `done`/`failed` avec `failure_reason='resumed_after_restart'`. Aucun POST Meta au reclaim.
- Janitor GET-only (`agent/reconciliation.py::MetaReconciliationJanitor`) : tick 120 s, réconcilie via `GET /{phone_number_id}/calls?biz_opaque_callback_data=<call_id>`. Defensive sur 400 (filtre Meta non supporté → log + no-op). **Jamais de POST Meta**, jamais de retry.
- Exceptions typées centralisées dans `agent/exceptions.py` : `PermissionRefused`, `PermissionTimeout`, `MetaAPIError`, `InvalidStateTransition`. `ensure_call_permission` remplace les `RuntimeError` génériques par ces types.
- `CallOut` (POST /v1/calls) expose 3 champs : `status` (legacy dérivé), `request_status`, `call_status`. `docs/openapi.yaml` à jour.
- `pytest.ini` ajouté avec marker `migration` (tests d'invariants schema post-0005 lancés via `pytest -m migration`).
- Suite de tests : `tests/db/conftest.py` (fixture `db_session` + `call_fixture`), 18 tests DB-ancrés sur transitions + dérivation legacy + contraintes PG (25 passant dont 6 marker `migration`). 3 fichiers tests complexes (handler async, lifespan reclaim, janitor) skip avec raison → follow-up infra fixture async refacto.

### Changed (M3.6 C1)
- `db/call_store.py` supprimé. Le pipeline voice lit désormais son contexte depuis la DB via `db.repositories.calls.fetch_for_pipeline` retournant un `CallPipelineContext`. L'état d'appel n'est plus en RAM process-local : résilient au restart, visible multi-worker. Préalable à C2 (POST async) et C5 (idempotence events webhook).
- `agent/call_handler.CallHandler` : argument `store` retiré du constructeur. `CallStatus` déplacé dans le même module (ex-`db.call_store`). Méthode `mark_result` simplifiée — paramètre `transcript` retiré, délègue intégralement à `finalize_if_tracked`.
- Les deux pipelines vocaux (générique + canal WhatsApp Business) : signature `call_request: CallRequest` remplacée par `call_ctx: CallPipelineContext`.

### Tech-debt (M3.6 C1)
- `db/repositories/calls.py::fetch_for_pipeline` importe `_COUNTRY_TO_PROMPT_VARIANT` depuis `api/v1/calls.py` (inversion `api → db`, import paresseux pour éviter le cycle). Follow-up : déplacer la constante dans `agent/prompts/country.py` lors d'un refactor ultérieur.

### Removed (M3.5 — grand ménage code + doc obsolète)
- `tools/local_relay.py` (relay dev local — référençait les endpoints supprimés en M2).
- Canal dev local côté navigateur + sa branche dans la factory canaux (orphelin depuis suppression de `POST /api/offer` en M2).
- `client/index.html` + `client/dev.html` (UI lab consommant `/api/events`, `/api/call/whatsapp`, `/api/calls/{id}` — endpoints morts) + dossier `client/`.
- `main.py` : `StaticFiles` mount `/client` (plus de fichiers à servir).
- `docker-compose.dev.yml` : service `pgweb` (dépendait du lab).
- `.claude/rules/lab-local.md`, `.claude/commands/lab.md`, `.claude/REFACTOR_NOTES.md`.
- `doc/test_tools.md` (décrivait le lab UI + outils supprimés).

### Changed (M3.5)
- `.env.example` : ajout `WHATSAPP_APP_SECRET=` (critique depuis M1 fail-closed — était oublié).
- `.claude/rules/webhooks.md` : correction `WHATSAPP_WEBHOOK_VERIFY_TOKEN` → `WHATSAPP_APP_SECRET` pour la vérif HMAC (bug #doc-1 corrigé).
- `.claude/rules/call-flow.md` : mention `debug_bus` retirée (bus inerte sans subscriber depuis M2).
- Règles internes agent + skill de création de profils conversationnels pays : alignés sur `POST /v1/calls` + mapping country ISO interne.
- `api/v1/calls.py` : docstring réécrite (description du flow actuel, sans historique).
- `api/call_setup.py` : docstring pointant vers `/v1/calls`.
- `scripts/seed_reseller.py` : docstring allégée (plus de mention de l'ancien `seed_dev.py`).
- `doc/database.md` + `doc/call_flow_pipeline.md` + `doc/whatsapp.md` + `doc/ambient_sound.md` : bulk-replace `/call/whatsapp` → `/v1/calls`, suppression des mentions `dev_demo`, `YASMINE_ADMIN_TOKEN`, `seed_dev.py`, `/api/events`, TWILIO_*.
- `CLAUDE.md` : mise à jour architecture post-M3.4 (environnement local retiré, canaux alignés sur factory, docs de référence listées).

### Added
- Documentation reseller recréée et alignée sur l'API M3.3 : `docs/getting-started.md`, `docs/examples.md`, `docs/openapi.yaml` refondu.
- `POST /v1/calls` marqué `x-status: live` dans la spec publique (remplace l'ancien `planned + x-legacy-path`).
- Webhooks Meta (`GET/POST /webhooks/whatsapp`) documentés dans un tag `Internal` séparé — clairement marqués « pas destinés aux resellers ».
- Tags OpenAPI explicites par domaine pour meilleur rendu Scalar : `Calls`, `Account`, `Merchants`, `Internal`.
- Router `/v1` avec authentification par défaut (`Depends(get_current_reseller)` hérité par toute route). Route temporaire `GET /v1/ping` pour validation — retirée en M3.2.
- Format d'erreur RFC 7807 (`application/problem+json`) scope à `/v1/*`. Les webhooks Meta gardent leur format legacy.
- Documentation des erreurs dans `docs/errors.md`.
- `POST /v1/calls` — déclenchement d'appel IA. Remplace l'ancien `POST /call/whatsapp` supprimé en M2. Renvoie `201 Created` + header `Location: /v1/calls/{id}` + body `CallOut` (sans fuite de champs internes). Header `Idempotency-Key` accepté mais non persisté (storage en M5).

### Removed
- Suppression totale des endpoints de dev/démo/debug : /dev/lab_check, /summaries, /summary/{label}, /api/events, /api/offer, /call/twilio, /call/whatsapp, /api/calls/{id}/bundle, routes de démo, pipeline Twilio.
- Scripts et fichiers de démo : seed_dev.py, client/demo.html, api/demo.py, api/demo_scenarios.py.
- Documentation reseller obsolète : docs/getting-started.md, docs/examples.md (décrivaient des endpoints qui disparaissent ; seront recréés en M3 sur /v1).

### Security
- Surface d'attaque réduite au strict minimum : seuls les webhooks Meta signés restent exposés.
- Webhook Meta WhatsApp : fail-closed si `WHATSAPP_APP_SECRET` manquant (503 au lieu d'acceptation silencieuse).

### Breaking
- POST /call/whatsapp supprimé. Sera remplacé par POST /v1/calls en M3.
- `POST /v1/calls` — validation stricte des inputs (M3.3). Pas de transition gracieuse (aucune intégration client active).
  - `total` (str libre) **remplacé** par `amount` (`Decimal`, requis, `0 < x ≤ 1_000_000`) + `currency` (str, requis, ISO-4217 `^[A-Z]{3}$`).
  - `country` — désormais un **code pays ISO 3166-1 alpha-2** : `"MA"`, `"DZ"`, `"TN"`, `"FR"`. Défaut `"MA"`. Le choix du profil conversationnel interne (darija marocaine / française) est fait côté serveur via un mapping dédié — transparent pour le reseller. Plusieurs codes ISO peuvent pointer vers le même profil (ex. MA/DZ/TN → `maroc_02`) tant qu'il n'y a pas de profils localisés distincts.
  - `phone_number` : validé par libphonenumber (`is_valid_number`), renormalisé en E.164 canonique avant traitement.
  - `merchant_ref` : regex `^[a-zA-Z0-9_.-]+$`, max 128 chars.
  - `customer_name` : max 200 chars, strip auto, rejet caractères de contrôle ASCII, normalisation Unicode NFC.
  - `metadata` : max 2 KB sérialisé JSON.
  - `CallOut` : `total_amount` (str) remplacé par `amount` (Decimal) + `currency` (str). Champ `country` (ISO) ajouté.
- Nouvelle dépendance runtime : `phonenumbers>=8.13`.

### À venir (planned)
- P0 — `POST /v1/calls`, isolation tenant sur `GET /v1/calls/*`, fermeture des endpoints dev/admin, `Idempotency-Key` obligatoire.
- P1 — webhooks sortants signés + dispatcher, self-service `/v1/me/*`, RFC 7807 global, pagination cursor, rate-limit, scopes clés.

## [0.1.0] — 2026-04-19 — Initial API /v1 spec

### Added
- Spécification OpenAPI 3.1 **cible** `/v1` (`docs/openapi.yaml`) — 22 endpoints, RFC 7807, pagination cursor, auth Bearer `yk_`.
- Politique de versioning `/v1 → /v2`, header `X-API-Version`, `Sunset` RFC 8594 (`docs/versioning.md`).
- Catalogue d'événements webhook + signature `X-Yasmine-Signature` HMAC-SHA256, anti-replay 5 min, retry 3/6/12/24 h + DLQ (`docs/webhooks.md`).
- Guide *Getting Started* avec exemples curl de bout en bout (`docs/getting-started.md`).
- Recettes curl détaillées (pagination, clés, webhooks, merchants) (`docs/examples.md`).
- Documentation infrastructure (Nginx, DNS, setup Let's Encrypt) pour `docs.yasmine.akidly.com` (`docs/infrastructure.md`).
- Script `scripts/seed_reseller.py` pour provisionner un reseller + clé API + crédit initial.
- Page Scalar statique (`docs/site/index.html`) pointant sur `../openapi.yaml`.

### Notes
- Tous les endpoints `/v1/*` sont en `x-status: planned` — le contrat est figé, l'implémentation arrive via P0/P1 (cf `.claude/context/roadmap.md`).
- Les intégrations actuelles contre `/call/whatsapp`, `/call/{id}/status` restent valides — un champ `x-legacy-path` dans la spec indique la correspondance.
