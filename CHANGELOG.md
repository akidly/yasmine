# Changelog

Les évolutions de l'API Yasmine visibles côté intégrateur : endpoints, champs,
formats, événements, erreurs et comportements observables.

Le format suit [Keep a Changelog](https://keepachangelog.com/fr/1.1.0/) et la
politique de versioning décrite dans `docs/versioning.md`.

## [Unreleased]

### Added

- **Nouvelle ressource catalogue `/v1/shops/{shop_external_id}/products` —
  déclarez vos produits, puis commandez-les par référence.** Cinq routes,
  calquées sur celles des boutiques : `POST`, `GET` (paginé), `GET /{id}`,
  `PATCH`, `DELETE` (archive). Le minimum tient en deux champs :
  `external_id` et `name`.

  **Deux niveaux, comme dans tout système de commerce** : le produit porte
  l'identité, la déclinaison porte le prix. Un produit sans choix n'y
  échappe pas — si vous n'envoyez pas de `variants`, une déclinaison unique
  est créée pour lui et `unit_price` s'y dépose. Vous envoyez un prix, vous
  relisez un prix ; le reste ne vous concerne pas.

  Points de contrat à connaître avant d'intégrer :

  - **Plusieurs déclinaisons ⇒ chacune porte son `external_id`.** Sans cela
    une commande ne pourrait pas désigner laquelle. Refus en `422` nommant
    les entrées fautives.
  - **`PATCH` sur `variants` remplace la liste entière.** Une déclinaison
    absente de la nouvelle liste est supprimée du catalogue. Pour n'en
    changer qu'une : relire, modifier, renvoyer la liste complète. Ne pas
    envoyer `variants` du tout les laisse intactes.
  - **`unit_price` seul est refusé sur un produit à plusieurs
    déclinaisons** (`422`) : il y serait ambigu.
  - **`compare_at_price` doit être strictement supérieur à `unit_price`** —
    un prix barré inférieur au prix est forcément une erreur de saisie.
  - **`external_id` est unique par boutique, pas par compte.** Le même
    identifiant reste libre dans une autre boutique.
  - **`DELETE` archive.** Les appels qui ont mentionné le produit gardent ce
    qui a été dit au client.

  Trois champs du produit priment sur ceux de la boutique quand ils sont
  renseignés : `return_policy` **remplace** celle de la boutique,
  `current_promotions` **s'ajoute** à la sienne, et `max_discount` obéit à
  une règle explicite décrite plus bas.

  Deux nouveaux slugs : `product_not_found` (404) et
  `product_external_id_already_exists` (409). Le 404 est **identique** que
  le produit soit inexistant, archivé, ou dans une autre boutique.

- **La demande d'appel s'allège : `order.items` accepte des références de
  catalogue.** Deux nouveaux champs par ligne de commande,
  `product_external_id` et `variant_external_id`. Une ligne qui référence
  un produit déclaré n'a plus à répéter son nom, sa description, son prix
  ni ses déclinaisons disponibles : tout est relu depuis le catalogue.

  Ce qui devient déductible et n'a plus à être transmis :

  | Champ | D'où il vient désormais |
  |---|---|
  | `shop_info` | La fiche déclarée par `POST /v1/shops` |
  | `country` | L'indicatif du numéro **du client** |
  | `order.currency` | Le pays de la boutique (`MA` → MAD, `DZ` → DZD, `TN` → TND) |
  | `name`, `description`, `unit_price`, `available_variants` d'une ligne | La fiche produit et la déclinaison référencées |

  **Ce que vous envoyez malgré tout l'emporte** — un prix promotionnel pour
  cette commande-là reste possible. **Une ligne décrite sur place avec
  `name` reste valide** : c'est la voie pour un article sur mesure, des
  frais de port ou un cadeau. Un produit absent du catalogue ne doit jamais
  bloquer un appel.

  `variant_external_id` est inutile sur un produit à déclinaison unique.
  S'il en a plusieurs et qu'aucune n'est désignée, la requête est refusée
  (`order_item_variant_required`) **en listant celles qui existent** :
  aucune n'est choisie au hasard, sans quoi l'appel annoncerait au client un
  article qu'il n'a pas commandé.

  Deux nouveaux slugs : `order_item_unknown_product` (422) et
  `order_item_variant_required` (422).

- **`order.max_discount` — la remise maximale accordable sur une commande.**
  Trois niveaux, du plus précis au plus général : la commande l'emporte sur
  les produits, qui l'emportent sur la boutique. Sur une commande à
  plusieurs produits, c'est **la plus haute** des remises déclarées qui
  vaut, jamais leur somme ; et une valeur à `0` ne bloque rien — elle dit
  seulement que ce produit-là n'apporte pas de marge, un autre article de
  la même commande pouvant la couvrir.

  Champ **enregistré, pas encore exploité pendant l'appel**. Il est
  documenté dès maintenant pour que vos intégrations puissent le remplir.

- **Nouvelle ressource `/v1/shops` — gérez vos boutiques explicitement.**
  Cinq routes : `POST` (créer), `GET` (lister, paginé), `GET /{external_id}`
  (consulter), `PATCH` (modifier), `DELETE` (archiver). Une boutique se
  déclare **une fois**, avec votre propre identifiant, au lieu d'être
  retransmise dans le `shop_info` de chaque demande d'appel.

  Le minimum tient en trois champs : `external_id`, `name`, `country`. Les
  vingt-six autres s'ajoutent quand vous voulez, par `PATCH`.

  Points de contrat à connaître avant d'intégrer :

  - **`PATCH` efface sur `null`.** Champ absent = inchangé, champ à `null` =
    effacé, chaîne vide = `422`. Cette sémantique **diffère de `shop_info`**
    dans `POST /v1/calls`, où rien n'est jamais effacé. Si vous gérez vos
    boutiques par cet endpoint, cessez d'envoyer `shop_info` dans vos
    demandes d'appel — sinon les deux se contredisent.
  - **`DELETE` archive, ne supprime pas.** L'historique d'appels reste
    intact et consultable via `?status=archived`.
  - **`external_id` est immuable** — il porte l'historique de la boutique.
    L'envoyer dans un `PATCH` retourne `422`.
  - **`phone` et `whatsapp` sont validés contre le `country` déclaré.**
    Format local (`0522123456`) ou international, au choix ; mais un numéro
    d'un autre pays est refusé. Stockés et renvoyés en E.164.
  - Pas d'`Idempotency-Key` requise, contrairement à `POST /v1/calls` :
    l'unicité de l'`external_id` rend déjà la création rejouable.

  Deux nouveaux slugs d'erreur : `shop_not_found` (404) et
  `shop_external_id_already_exists` (409). Le 404 est **identique** que la
  boutique soit inexistante, archivée, ou détenue par un autre compte —
  une réponse différenciée permettrait de deviner ce qui existe ailleurs.

- **`merchant_external_id` — regrouper des boutiques sous un commerçant.**
  Champ optionnel sur la boutique, et filtre sur `GET /v1/shops`. Plusieurs
  boutiques partagent la même valeur : c'est ainsi que l'on sait combien de
  boutiques possède un commerçant. C'est une clé de regroupement, pas une
  ressource — le commerçant n'a pas d'attributs propres.

- **`has_physical_store`** — la boutique a-t-elle un point de vente où le
  client peut se présenter ? `true`, `false`, ou absent si non déclaré.
  Distinct de `address`, qui dit *où* vous êtes sans dire si l'on *peut
  venir*.

- **`inspection_before_payment`** — le client peut-il ouvrir le colis avant
  de payer ? Trois états, l'absence signifiant « non déclaré ».

- **`max_discount`, `delivery_companies`, `city_coverage`, `agent_name`** —
  enregistrés sur la fiche boutique, **pas encore exploités pendant
  l'appel**. Vous pouvez les renseigner dès maintenant pour préparer votre
  intégration ; leur activation suivra.

### Changed (BREAKING)

- **`shop_info` : cinq champs renommés, un retiré.** Le préfixe `shop_`
  était redondant dans un objet déjà nommé `shop_info`, et la distinction
  service après-vente / ligne principale n'avait pas de réalité pour la
  plupart des boutiques.

  | Avant | Après |
  |---|---|
  | `shop_phone_sav` | `phone` |
  | `shop_whatsapp_sav` | `whatsapp` |
  | `shop_email_sav` | `email` |
  | `shop_address` | `address` |
  | `shop_sector` | `sector` |
  | `shop_sav_hours` | `customer_service_hours` |
  | `size_exchange_policy` | `exchange_policy` |

  `exchange_policy` élargit aussi le sens : un échange couvre la couleur,
  le modèle et l'article défectueux, pas seulement la taille.

  Les anciens noms retournent `422 extra_forbidden`.

- **`merchant_external_id` à la racine de `POST /v1/calls` devient
  `shop_external_id`.** Il désignait une boutique, il porte désormais le nom
  qui convient. ⚠️ Le nom `merchant_external_id` **existe toujours**, mais
  dans `shop_info`, et il y désigne le **commerçant propriétaire** — pas la
  boutique. L'ancien usage à la racine retourne `422`.

- **`loyalty_program` retiré de `shop_info`.** Champ quasi inutilisé en
  appel de confirmation. L'envoyer retourne `422` ; l'information a sa place
  dans `faq_free`.

- **`merchant_id` devient `shop_id`** dans le filtre de `GET /v1/calls` et
  dans les réponses.

- **`country` sur les appels reflète désormais le client appelé.** Il était
  renseigné avec le pays du **premier** client jamais appelé par la boutique
  et ne changeait plus ensuite : une boutique appelant un Marocain puis un
  Français annonçait `MA` sur les deux. Les webhooks `call.ended` et
  `GET /v1/calls` renvoient maintenant le vrai pays de chaque appel. Les
  appels antérieurs conservent leur ancienne valeur.

### Changed

- **`order.currency` et `country` deviennent facultatifs sur
  `POST /v1/calls`**, de même que `name` sur une ligne de commande qui
  porte une référence de catalogue. Les requêtes existantes, qui envoient
  ces champs, continuent de fonctionner à l'identique : une valeur fournie
  prime toujours sur ce qui serait déduit. Aucune action requise.

### Deprecated

- **`industry`, `active`, `logo_url` dans `shop_info`.** Toujours acceptés et
  enregistrés, mais absents de `/v1/shops` et sans effet pendant l'appel.
  `industry` fait doublon avec `sector` ; `active` est remplacé par
  l'archivage. Retrait après préavis `Sunset` — ne construisez pas dessus.

- **`call_params.voice_id` et `call_params.max_call_duration_seconds`** sont
  désormais documentés pour ce qu'ils sont : **enregistrés, sans effet
  aujourd'hui**. Ils étaient acceptés depuis des mois sans que leur inertie
  soit écrite nulle part. Leur comportement n'a pas changé — seulement sa
  description.


### Changed (BREAKING)

- **`order.items[].variant` remplacé par `variants` structurés + `description`.**
  Chaque article accepte désormais `variants` — une liste de paires
  `{name, value}` (max 3, ex. `{"name": "Taille", "value": "XL"}`), citées au
  client sous la forme « Taille : XL » — et `description` (texte libre
  ≤ 500 caractères) que l'agent utilise pour répondre aux questions du client
  sur l'article (matière, contenu, usage…) sans la réciter d'office. L'ancien
  champ libre `variant` est retiré : un payload qui l'envoie encore reçoit
  `422` avec un rejet `extra_forbidden` explicite. Détails :
  `docs/getting-started.md` §4, `docs/examples.md` §3.

- **Classification d'appel enrichie.** `result` est simplifié à 3 valeurs
  (`confirmed` / `cancelled` / `requires_action`, en minuscules) et les champs
  `result_detail`, `customer_mood`, `flags`, `preferences`, `next_action` et
  `summary` sont exposés dans `CallOut` (`POST /v1/calls`, `GET /v1/calls`,
  `GET /v1/calls/{call_id}`) ainsi que dans le payload `data` du webhook
  `call.ended`. Un appel modifié (« oui mais en bleu ») est désormais classé
  `result=confirmed` + `result_detail=modified` et confirme la commande — il
  était auparavant rangé en `UNCLEAR` sans confirmation. Un faux numéro ou un
  client qui nie la commande est désormais explicitement `result=cancelled` +
  `result_detail=wrong_number|denied_order` (avant : `UNCLEAR`, indistinguable
  d'un audio inaudible). Le filtre `?result=` est ajouté à `GET /v1/calls`.
  Détails et exemples : `docs/webhooks.md` §7, `docs/getting-started.md` §4.5,
  `docs/examples.md` §3.5.

- **Schéma `CallOut` refondu** (réponse `POST /v1/calls`,
  `GET /v1/calls/{call_id}`, `GET /v1/calls`) : ajout de
  `call_duration_seconds`, `meta_error_code` et `meta_error_title` ; renommage
  `billed_seconds` → `billable_duration_seconds` ; retrait du champ legacy
  `status` — utiliser `request_status` + `call_status` à la place. Ferme l'écart
  où le polling `GET /v1/calls/{call_id}`, présenté comme fallback aux webhooks
  perdus, ne renvoyait pas le résultat métier de l'appel.

- **Payload `POST /v1/calls` restructuré en sous-objets.** `shop_ref` à la
  racine devient `shop_external_id` ; `amount` / `currency` passent sous
  `order` ; `purpose` passe sous `call_params`. Nouveaux sous-blocs
  `shop_info`, `order` et `call_params`. Un payload plat legacy renvoie `422`
  avec des rejets `extra_forbidden` sur les anciens champs et `missing` sur les
  nouveaux.

- **Sous-objet `customer` obligatoire** sur `POST /v1/calls` : `customer_name`
  et `phone_number` à la racine deviennent `customer.{name, phone_number}`. Un
  payload legacy renvoie `422 extra_forbidden` + `missing customer`.

### Changed

- **`order.delivery_address` devient optionnelle** sur `POST /v1/calls`
  (assouplissement rétro-compatible). Absente — ou réduite à un nom de ville
  nue — l'agent collecte l'adresse auprès du client pendant l'appel (ville +
  quartier, avec confirmation) et la remonte dans `preferences` du compte
  rendu. Les payloads qui envoient déjà une adresse complète sont inchangés.

- **`GET /v1/me/webhooks` renvoie `secret_prefix` sur 12 caractères** + `…`
  (6 auparavant), cohérent avec le `key_prefix` des clés API.

### Added

- **Délai de livraison boutique** : nouveau champ optionnel
  `shop_info.delivery_delay` (≤ 100 caractères, ex. « 8 heures », « 24-48h »).
  L'agent répond à « quand vais-je recevoir ma commande ? » selon cette
  priorité : `order.estimated_delivery_date` (date propre à la commande,
  annoncée avec son jour de semaine) > `shop_info.delivery_delay` >
  formulation générale par défaut. Distinct de `delivery_policy`, qui reste
  dédié aux conditions de livraison (gratuite, contre remboursement…).
  Rétro-compatible.

- **Disponibilité des variantes par article** : nouveau champ optionnel
  `order.items[].available_variants` — liste (max 60) de **combinaisons**
  réellement disponibles, chaque combinaison étant 1 à 3 paires
  `{name, value}` sur les mêmes axes que `variants` (ex. `[[Taille XL, Couleur
  rouge], [Taille 2XL, Couleur noir]]`). Le format par combinaisons exprime les
  croisements : une couleur disponible seulement dans certaines tailles. Quand
  le champ est présent, l'agent propose et n'accepte **que** ces combinaisons
  lors d'une demande de changement ou d'ajout, et annonce toute autre valeur
  comme indisponible ; absent, il s'appuie sur `description` et les infos
  boutique, sinon sur la seule variante commandée. Rétro-compatible.

- **Téléchargement de l'enregistrement audio** via
  `GET /v1/calls/{call_id}/recording`. Stream le mix audio (client + agent) au
  format WAV mono 16 kHz, 16-bit PCM (~2 Mo pour 60 s). Auth Bearer identique
  aux autres endpoints `/v1`. Rate limit dédié 60 req/min. `CallOut` et le
  payload `data` du webhook `call.ended` exposent le champ `recording_url`
  (URL relative). Quatre cas renvoient un `404 call_not_found` indistinct
  (identifiant invalide, appel inexistant, appel d'un autre compte, audio pas
  encore produit). **Rétention 30 jours**, puis `410 Gone` avec le slug
  `recording_gone` — distinct de `call_not_found` pour différencier « n'a
  jamais existé » de « a existé mais a été purgé ». Détails et exemples curl :
  `docs/getting-started.md` §4.6, `docs/examples.md` §3.5, `docs/webhooks.md` §7.

- **Valeur `unclear` ajoutée à l'énumération `customer_mood`**
  (`CallOut.customer_mood`, `data.customer_mood` du webhook `call.ended`).
  Posée quand l'audio du client est globalement incompréhensible (corrélé
  généralement avec `result_detail=unclear`). Aucun changement de format JSON :
  les intégrations qui valident strictement l'énumération doivent ajouter
  `unclear` à leurs valeurs attendues, sinon la traiter comme valeur inconnue.

- **Pays et langue exposés sur les événements terminaux** (`call.ended`,
  `call.cancelled`, `call.failed`) : `country` (`MA` / `DZ` / `TN` / `FR`) et
  `language` (`ar` / `fr`) dans le payload `data`. Utile quand `language`
  n'a pas été spécifié à la création — vous récupérez la langue appliquée par
  défaut. Champs ajoutés en fin de payload, ordre des champs existants
  préservé : aucun handler existant ne casse. Détails : `docs/webhooks.md` §7.

- **Choix de la langue d'appel** : nouveau champ optionnel `language` à la
  racine de `POST /v1/calls`, valeurs `ar` ou `fr`. Un client qui préfère le
  français peut être appelé en français depuis un pays maghrébin. Si omis, la
  langue locale du pays s'applique automatiquement. Combinaisons supportées :
  `MA` / `DZ` / `TN` avec `ar` ou `fr` ; `FR` uniquement avec `fr`. La
  combinaison `country=FR` + `language=ar` est rejetée en
  `422 language_not_supported_for_country`.

- **Détail commande structuré** : champ optionnel `order.items` (max 50
  entrées) — chaque entrée expose `name` (requis), `quantity` (défaut 1) et
  `unit_price` (optionnel). Cohabite avec `order.items_text` (résumé libre,
  max 500 caractères) ; quand les deux sont fournis, la liste structurée prime.

- **6 nouveaux événements sortants** sur le cycle de vie de l'autorisation
  WhatsApp : `call.request.template_sent` / `_delivered` / `_read` /
  `_delivery_failed`, `call.request.permission_revoked` (révocation manuelle
  par le client) et `call.request.permission_auto_revoked` (révocation après
  4 appels sans réponse). Le catalogue passe à 18 événements `call.*`.

- **Nouvel événement `call.request.service_unavailable`** qui isole les
  indisponibilités temporaires côté Yasmine du `call.failed` générique. Inclut
  `retry_after_hint_minutes` (entier) — distingue un problème transitoire d'un
  échec définitif côté client.

- **Idempotence stricte des événements** : un envoi par changement d'état
  observé, même en cas de retransmission réseau. L'enveloppe
  `id=evt_<ULID>` reste l'identifiant de déduplication de référence.

- **`GET /v1/me` expose 10 champs de profil société** : `owner_name`,
  `owner_phone_e164`, `default_language`, `country`, `timezone`, `legal_name`,
  `tax_id`, `billing_address`, `business_type`, `website_url`. Tous nullables,
  retournés `null` tant qu'ils ne sont pas renseignés. Pas encore de mise à
  jour en self-service : les valeurs sont posées à la demande.

- **Header `X-Yasmine-Event`** ajouté aux webhooks sortants : permet de router
  ou dédupliquer les événements sans parser le corps JSON.

- **`webhook.test` documenté** dans le catalogue (`docs/webhooks.md` §7).
  Payload `{test: true, test_id, request_id}` — la clé `test=true` le distingue
  d'un événement métier.

- **`POST /v1/me/webhooks/rotate-secret`** : rotation atomique du secret de
  signature du webhook, sans passer par `DELETE` + `POST`. Retourne
  `{"secret": "whsec_…", "secret_prefix": "whsec_xxxxxx…", "rotated_at": "…"}`.
  **Secret visible une seule fois** — aucun endpoint ne le ré-expose.
  Rate limit 5/min. `404 webhook_not_configured` si aucun webhook actif.

- **Format de secret `whsec_<43 caractères>`** pour les secrets générés à la
  création ou à la rotation. Facilite la détection automatique de secrets dans
  vos logs et votre CI. Les secrets antérieurs (sans préfixe) restent
  fonctionnels.

- **`GET /v1/me/api-keys/{key_id}/events`** : journal paginé des événements
  d'une clé API (`created`, `revoked`, `rate_limited`). Enveloppe
  `{data, next_cursor, has_more}`, `limit` max 50. `404 api_key_not_found`
  indistinct si la clé n'existe pas ou appartient à un autre compte ;
  `400 invalid_key_id` si l'identifiant est malformé.

- **Gestion self-service des clés API** :
  - `GET /v1/me/api-keys` — liste des clés (actives et révoquées, avec
    `status`), triées par date de création décroissante. Ni le secret ni son
    empreinte ne sont jamais renvoyés — seul `key_prefix` (12 caractères).
  - `POST /v1/me/api-keys` — création. Le `secret_key` (`yk_<40 caractères>`)
    est visible **une seule fois** dans la réponse `201`. Corps
    `{"label": "optionnel ≤ 64 caractères"}`. Rate limit 5/min.
  - `DELETE /v1/me/api-keys/{key_id}` — révocation. `204` en cas de succès,
    `400 invalid_key_id`, `404 api_key_not_found` indistinct sur clé inconnue
    ou appartenant à un autre compte. La requête en cours se termine ; la
    suivante reçoit `401`.
  - Pas de scopes, pas d'expiration programmée : le seul levier de sécurité est
    la révocation manuelle.

- **Support CORS pour les fronts web cross-origin.** Liste blanche d'origines
  gérée à la demande. `allow_credentials` activé (Bearer cross-origin).
  Méthodes autorisées : `GET`, `POST`, `DELETE`, `OPTIONS`. En-têtes acceptés :
  `Authorization`, `Content-Type`, `Idempotency-Key`, `X-Request-ID`. En-têtes
  exposés au front : `X-Request-ID`, `X-RateLimit-Limit`,
  `X-RateLimit-Remaining`, `X-RateLimit-Reset`, `Retry-After` — de quoi lire les
  compteurs de rate limit et cadencer les requêtes. Cache de préflight 10 min.
  Communiquez-nous les URLs de vos fronts pour les ajouter à la liste.

- **Page d'erreurs lisible** : <https://docs.yasmine.akidly.com/errors> liste
  les 28 slugs émis par l'API, groupés par catégorie (authentification,
  validation, idempotence, facturation, rate limiting, introuvable, conflit,
  webhooks, limites de taille, serveur). Chaque slug a sa section avec
  explication, causes fréquentes, correctifs et exemple de réponse JSON. Les
  URLs `type` des réponses RFC 7807 (ex.
  `https://docs.yasmine.akidly.com/errors/insufficient_balance`) pointent
  directement sur la bonne section.

- **`POST /v1/me/webhooks/test`** : déclenche un `webhook.test` synchrone vers
  votre URL configurée. Retourne immédiatement `{test_id, delivered,
  http_status, latency_ms, target_url, attempted_at, error_message}`. Timeout
  5 s, sans retry. Rate limit 10/min. `400 webhook_url_not_configured` si aucun
  webhook actif.

- **`GET /v1/me/webhooks/deliveries`** (filtres `cursor`, `limit`, `status`,
  `event_type`, `since`, `until`) : historique paginé des livraisons. 11 champs
  par ligne — `delivery_id`, `event_type`, `target_url`, `http_status`,
  `latency_ms`, `attempt_count`, `status` (`delivered` / `failed` / `pending`),
  `next_retry_at`, `created_at`, `last_attempted_at`, `error_message` (tronqué
  à 200 caractères).

- **Endpoints de compte en lecture** :
  - `GET /v1/me` — identité du compte : `reseller_id`, `name`, `email`,
    `created_at`, `rate_limit_per_minute`.
  - `GET /v1/me/transactions` — historique paginé du crédit. Expose
    `transaction_id`, `type` (`topup` / `debit_call` / `refund` /
    `adjustment`), `seconds` (signé : positif = crédit, négatif = débit),
    `reference`, `reason`, `created_at`.
  - `GET /v1/me/usage?period=YYYY-MM` — agrégation mensuelle (mois courant UTC
    par défaut) : `total_calls`, `total_seconds` et répartition par statut
    (`completed_calls`, `completed_seconds`, `failed_calls`,
    `cancelled_calls`). Une période sans appel renvoie `200` avec des zéros,
    pas `404`. `400 invalid_period_format` si le format ne correspond pas.
  - `GET /v1/me/balance` — solde : `balance_seconds` (peut être négatif dans la
    limite de l'autorisation de découvert), `currency`, `updated_at`.

- **`GET /v1/calls`** — liste paginée de vos appels. Enveloppe
  `{data, next_cursor, has_more}`, 50 par défaut, 200 maximum. Curseur opaque
  signé, valable 24 h, en avant uniquement — à repasser tel quel en `?cursor=`,
  sans tenter de le parser. Quatre filtres combinables : `?status=`
  (`queued` / `dialing` / `in_progress` / `ended` / `failed`), `?shop_id=`,
  `?since=`, `?until=`. Erreurs : `400 invalid_cursor` (format ou signature
  invalide) et `400 cursor_expired` (> 24 h) — refaites la première requête
  sans curseur.

- **`GET /v1/calls/{call_id}`** — récupération d'un appel.
  `404 call_not_found` indistinct sur les trois cas : identifiant invalide,
  inconnu, ou appartenant à un autre compte. Expose `ringing_at`,
  `connected_at`, `ended_at`, `failure_reason` et `cancelled_state`, tous
  `null` tant que l'étape n'est pas atteinte.

- **Header `X-Request-ID`** sur toutes les requêtes. Votre valeur est reprise
  si c'est un UUID v4 valide, sinon une nouvelle est générée. Renvoyée en
  en-tête de réponse et présente dans le champ `request_id` des réponses
  d'erreur RFC 7807 — à citer pour tout support. Les réponses `401` en ont une
  aussi.

- **`Idempotency-Key` obligatoire sur `POST /v1/calls`.** TTL 24 h, format
  libre (1 à 255 caractères), portée par compte.
  - **Rejeu** transparent si la clé et le corps sont identiques : la réponse
    stockée est rejouée à l'identique avec `X-Idempotent-Replay: true`. Aucun
    nouvel appel, aucun nouveau débit.
  - **Conflit** `409 idempotency_key_conflict` si la clé est réutilisée avec un
    corps différent.
  - Erreurs `400` : `missing_idempotency_key`, `idempotency_key_empty`,
    `idempotency_key_too_long` (> 255 caractères).

- **Rate limiting** sur tous les endpoints `/v1/*` authentifiés, par clé API :
  `POST /v1/calls` 60/min, `POST /v1/calls/{id}/cancel` 120/min, lectures
  600/min, endpoints de configuration 10/min. En-têtes `X-RateLimit-Limit`,
  `X-RateLimit-Remaining` et `X-RateLimit-Reset` sur toutes les réponses `2xx`.
  Dépassement : `429 rate_limit_exceeded` en RFC 7807 avec en-tête
  `Retry-After` et champ `retry_after` dans le corps.

- **`POST /v1/calls/{call_id}/cancel`** — annulation d'un appel en cours.
  - `queued` / `dialing` / `ringing` → `200` avec `billed_seconds=0` (jamais
    connecté, jamais facturé).
  - `connected` → `200` avec facturation de la durée réelle (plancher 10 s).
  - `ended` / `failed` / `cancelled` → `200` sans effet, idempotent, aucun
    événement ré-émis.
  - `404 call_not_found` si l'appel appartient à un autre compte.
  - Émet l'événement `call.cancelled` (distinct de `call.ended`, qui signale
    une fin naturelle), avec `{call_id, cancelled_at, cancelled_state,
    billed_seconds, shop_id}`.

- **Webhooks sortants signés.**
  - Configuration self-service : `POST` / `GET` / `DELETE /v1/me/webhooks`.
    Le secret est affiché **une seule fois** à la création ; `GET` ne renvoie
    que `secret_prefix`. `409 webhook_already_configured` si un webhook existe
    déjà. `DELETE` désactive sans effacer l'historique des livraisons.
  - Les URLs privées, locales ou non-HTTPS sont refusées, avec une raison
    explicite : `invalid_url`, `scheme_not_allowed`, `localhost_rejected`,
    `dns_resolution_failed`, `private_ip_rejected`.
  - Signature `X-Yasmine-Signature: sha256=<hex>` (HMAC-SHA256 du corps brut),
    en-tête `X-Yasmine-Event-Id` pour la déduplication, identifiant
    `evt_<ULID>` stable à travers les tentatives.
  - Livraison : 3 tentatives (immédiate, +30 s, +5 min), timeout 10 s, pas de
    suivi de redirection.
  - Catalogue initial de 10 événements : `call.request.accepted`,
    `call.request.refused`, `call.request.expired`,
    `call.request.quota_blocked`, `call.request.permission_granted_late`,
    `call.started`, `call.ringing`, `call.connected`, `call.ended`,
    `call.failed`. Si la demande d'autorisation échoue, aucun événement d'appel
    n'est émis.
  - Recette de déduplication côté intégrateur dans `docs/webhooks.md` §6
    (TTL recommandé : 24 h).

### Fixed

- **Détail de commande désormais transmis intégralement à l'agent** : adresse
  de livraison, articles, mode et zone de livraison, montants (sous-total,
  frais de livraison, remise), date estimée, numéro de commande et notes
  étaient acceptés et enregistrés mais ignorés pendant l'appel — l'agent
  appelait sans ce contexte. Si seul `order.items_text` est fourni, il est cité
  tel quel.

- **Code de diagnostic propagé dans `call.failed.data.failure_reason`** :
  auparavant `meta_<status>` seul (ex. `meta_400`), désormais
  `meta_<status>:<code>` quand le code est connu (ex. `meta_400:131030`).

- **Statut `REJECTED` traité correctement** : transition vers
  `call_status=failed` avec `failure_reason=client_rejected` et émission de
  `call.failed`. L'appel restait auparavant figé dans un état intermédiaire.

- **Notifications de progression sérialisées** (`RINGING`, `ACCEPTED`) : deux
  notifications concurrentes pouvaient auparavant s'exécuter en parallèle.

- **Doublon de `call.started` supprimé** en cas de course serrée.

- **Valeurs vides normalisées** dans `call.ended` et `CallOut` : la chaîne
  `"null"` parfois renvoyée pour `next_action`, `summary`, `customer_mood` ou
  `result_detail` devient un `null` JSON propre. Idem pour `"none"`, `"n/a"`,
  `""` et les espaces seuls (insensible à la casse). Les éléments vides sont
  retirés de `flags` et `preferences`. Aucun changement de schéma.

- **Documentation des webhooks alignée sur le comportement réel** :
  suppression du wrapper `data.object` dans les exemples (le corps émis a `data`
  à plat) ; énumération des slugs d'erreur réellement émis (`timeout`,
  `connect_error:*`, `http_error:*`, `unknown:*`).

- **Paramètre de chemin renommé `{id}` → `{call_id}`** sur `getCall` et
  `cancelCall` dans la spec OpenAPI. Les SDK générés exposeront désormais
  `call_id`.

- **Slugs d'erreur manquants ajoutés** à la documentation : `bad_request`
  (400), `rate_limited` (429), `language_not_supported_for_country` et
  `http_error` — ils pouvaient être émis sans entrée documentée.

### Removed

- **5 opérations non implémentées retirées de la spec OpenAPI**
  (`/v1/calls/{id}/events`, `/v1/calls/{id}/transcript`, `/v1/shops` GET et
  POST, `/v1/shops/{id}` PATCH) : leur présence poussait les SDK clients à
  les générer pour rien. Elles réapparaîtront lorsqu'elles seront livrées.
  Les schémas associés (`CallEventList`, `Transcript`, `Merchant*`) sont
  supprimés en conséquence.

- **Anciens endpoints de démonstration et de développement supprimés** :
  `/dev/lab_check`, `/summaries`, `/summary/{label}`, `/api/events`,
  `/api/offer`, `/call/twilio`, `/call/whatsapp`, `/api/calls/{id}/bundle`.

### Security

- **Rétention des données** : les charges utiles brutes des webhooks et les
  données personnelles associées sont purgées automatiquement au-delà de
  30 jours. Les enregistrements audio suivent la même rétention de 30 jours.
  Le journal d'événements des clés API est conservé 180 jours.

- Surface d'exposition réduite au strict minimum : la documentation
  auto-générée de l'API n'est pas publiée, la seule référence publique est la
  spec servie sur `docs.yasmine.akidly.com`.

### Breaking (historique — validation stricte des entrées)

- `POST /call/whatsapp` supprimé, remplacé par `POST /v1/calls`.
- `total` (chaîne libre) **remplacé** par `amount` (décimal requis,
  `0 < x ≤ 1 000 000`) + `currency` (ISO-4217, `^[A-Z]{3}$`).
- `country` devient un **code ISO 3166-1 alpha-2** : `MA`, `DZ`, `TN`, `FR`
  (défaut `MA`). Le profil conversationnel correspondant est choisi côté
  serveur — transparent pour l'intégrateur.
- `phone_number` validé puis renormalisé en E.164 canonique.
- `shop_ref` : `^[a-zA-Z0-9_.-]+$`, 128 caractères maximum.
- `customer_name` : 200 caractères maximum, espaces retirés aux extrémités,
  caractères de contrôle rejetés, normalisation Unicode NFC.
- `metadata` : 2 Ko maximum une fois sérialisé en JSON.
- `CallOut` : `total_amount` (chaîne) remplacé par `amount` (décimal) +
  `currency` ; champ `country` ajouté.

## [0.1.0] — 2026-04-19 — Spécification initiale de l'API `/v1`

### Added

- Spécification OpenAPI 3.1 de l'API `/v1` — 22 opérations, erreurs RFC 7807,
  pagination par curseur, authentification Bearer `yk_`.
- Politique de versioning `/v1 → /v2`, en-tête `X-API-Version`, `Sunset`
  RFC 8594 (`docs/versioning.md`).
- Catalogue des événements webhook et signature `X-Yasmine-Signature`
  HMAC-SHA256 (`docs/webhooks.md`).
- Guide de démarrage avec exemples curl de bout en bout
  (`docs/getting-started.md`).
- Recettes curl détaillées : pagination, clés, webhooks (`docs/examples.md`).
- Référence interactive de l'API sur `docs.yasmine.akidly.com`.

### Notes

- À ce stade, le contrat était figé mais les endpoints pas encore
  implémentés — la livraison est retracée dans le bloc `[Unreleased]` ci-dessus.
