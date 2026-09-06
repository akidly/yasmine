# Recettes curl — API Yasmine

Tous les exemples utilisent :

```bash
export YK="yk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
export BASE="https://api.yasmine.akidly.com"
```

Remplacez `$YK` par votre clé et collez directement dans un terminal.

> Les endpoints marqués `planned` renvoient **404** aujourd'hui — ils seront disponibles après M4. Le code côté reseller peut être écrit contre le contrat dès maintenant.

---

## 0. Gérer ses boutiques — les déclarer une fois

Sans cet endpoint, il faut renvoyer le profil complet de la boutique
(`shop_info`) dans **chaque** demande d'appel. En la déclarant une fois,
les appels suivants n'ont plus qu'à la référencer.

### Créer une boutique

Trois champs suffisent. Tout le reste s'ajoute ensuite.

```bash
curl -X POST https://api.yasmine.akidly.com/v1/shops \
  -H "Authorization: Bearer $YASMINE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "external_id": "shop-4471",
    "name": "GlowArt Casa",
    "country": "MA"
  }'
```

`external_id` est **votre** identifiant : celui de votre base, pas le
nôtre. C'est aussi l'adresse de la boutique dans l'URL, d'où le jeu de
caractères restreint (`[a-zA-Z0-9_.-]`).

Il est **immuable** — il porte tout l'historique d'appels de la boutique.

### Enrichir la fiche

```bash
curl -X PATCH https://api.yasmine.akidly.com/v1/shops/shop-4471 \
  -H "Authorization: Bearer $YASMINE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "phone": "0522123456",
    "whatsapp": "0661234567",
    "email": "contact@glowart.ma",
    "address": "12 Rue des Fleurs, Casablanca",
    "customer_service_hours": "Lun-Sam 9h-18h",
    "sector": "Cosmétique bio artisanale",
    "has_physical_store": true,
    "delivery_delay": "24-48h",
    "payment_policy": "Paiement à la livraison ou carte bancaire",
    "return_policy": "Retour accepté sous 14 jours",
    "inspection_before_payment": true
  }'
```

**Le téléphone accepte le format local.** `0522123456` sur une boutique
marocaine est valide et vous est renvoyé normalisé en `+212522123456`.
En revanche un numéro tunisien sur une boutique marocaine est refusé
(422) : le numéro doit appartenir au pays déclaré.

### Modifier, et effacer un champ

Trois cas à distinguer dans le corps d'un `PATCH` :

| Dans le corps | Effet |
|---|---|
| Champ **absent** | Inchangé |
| Champ à **`null`** | **Effacé** |
| Champ à **`""`** | Rejeté (422) |

```bash
curl -X PATCH https://api.yasmine.akidly.com/v1/shops/shop-4471 \
  -H "Authorization: Bearer $YASMINE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "delivery_delay": "48-72h",
    "current_promotions": null
  }'
```

Ici le délai est modifié, la promotion effacée, et les vingt-sept autres
champs restent tels quels.

> ⚠️ Cette sémantique **diffère de `shop_info`** dans `POST /v1/calls`, où
> un champ absent ou `null` n'efface jamais rien. C'est délibéré : une
> demande d'appel est du passage, elle ne détruit rien ; un `PATCH` est un
> geste explicite. **Si vous gérez vos boutiques par cet endpoint, cessez
> d'envoyer `shop_info` dans vos demandes d'appel** — sinon les deux se
> marchent dessus.

### Regrouper des boutiques sous un même commerçant

`merchant_external_id` porte l'identifiant du commerçant propriétaire.
Plusieurs boutiques partagent la même valeur — c'est son objet.

```bash
# À la création ou par PATCH
curl -X PATCH https://api.yasmine.akidly.com/v1/shops/shop-4471 \
  -H "Authorization: Bearer $YASMINE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"merchant_external_id": "M-4471"}'

# Puis : combien de boutiques possède ce commerçant ?
curl "https://api.yasmine.akidly.com/v1/shops?merchant_external_id=M-4471" \
  -H "Authorization: Bearer $YASMINE_API_KEY"
```

### Lister, paginer

```bash
curl "https://api.yasmine.akidly.com/v1/shops?limit=50" \
  -H "Authorization: Bearer $YASMINE_API_KEY"
```

```json
{
  "data": [ { "external_id": "shop-4471", "name": "GlowArt Casa", "status": "active" } ],
  "next_cursor": "eyJpZCI6...",
  "has_more": true
}
```

Passez `next_cursor` en `?cursor=` pour la page suivante. Le curseur est
valable 24 h ; au-delà, refaites la première requête sans `cursor`.

Par défaut seules les boutiques actives sont renvoyées. `?status=archived`
ne montre que les archivées, `?status=all` ne filtre pas.

### Archiver

```bash
curl -X DELETE https://api.yasmine.akidly.com/v1/shops/shop-4471 \
  -H "Authorization: Bearer $YASMINE_API_KEY"
# 204 No Content
```

**La boutique n'est pas supprimée.** Elle disparaît de la liste par défaut
et n'accepte plus de modification, mais son historique d'appels reste
intact et consultable. Pour la retrouver : `?status=archived`.

### Les deux erreurs à gérer

| Code | Slug | Quand |
|---|---|---|
| 409 | `shop_external_id_already_exists` | `POST` avec un `external_id` déjà pris. Utilisez `PATCH` pour modifier |
| 404 | `shop_not_found` | Boutique inexistante, archivée, ou d'un autre compte — **corps identique dans les trois cas** |

Le 404 est volontairement indifférencié : une réponse distincte
permettrait de deviner quelles boutiques existent chez d'autres comptes.



## 0.bis Le catalogue — déclarer un produit, puis le commander par son nom

Même principe qu'une boutique, un niveau plus bas. Un produit déclaré ici
se commande ensuite en le citant : son nom, son prix et ses déclinaisons
n'ont plus à être répétés dans chaque demande d'appel.

### Un produit sans déclinaison

```bash
curl -X POST "$BASE/v1/shops/shop-4471/products" \
  -H "Authorization: Bearer $YK" \
  -H "Content-Type: application/json" \
  -d '{
    "external_id": "SERUM-VITC",
    "name": "Sérum vitamine C",
    "description": "Flacon 30 ml, concentration 15%",
    "unit_price": "320.00"
  }'
```

Vous envoyez un prix, vous relisez un prix. En interne le produit reçoit
une déclinaison unique qui le porte — vous n'avez pas à le savoir, mais
c'est pour cela que `variants` contient une entrée dans la réponse, avec
un `external_id` à `null` : vous ne l'avez pas nommée, et vous n'avez pas
à le faire.

### Un produit à plusieurs déclinaisons

Le prix vit sur la déclinaison : le XL peut coûter plus cher que le S.

```bash
curl -X POST "$BASE/v1/shops/shop-4471/products" \
  -H "Authorization: Bearer $YK" \
  -H "Content-Type: application/json" \
  -d '{
    "external_id": "TSHIRT-BLANC",
    "name": "T-shirt blanc en coton",
    "description": "Coton bio 180g, coupe droite",
    "variants": [
      { "external_id": "TSH-S",  "options": {"Taille": "S"},  "unit_price": "149.00" },
      { "external_id": "TSH-M",  "options": {"Taille": "M"},  "unit_price": "149.00" },
      { "external_id": "TSH-XL", "options": {"Taille": "XL"}, "unit_price": "169.00",
        "compare_at_price": "199.00", "stock_quantity": 0 }
    ]
  }'
```

**Dès qu'il y a plusieurs déclinaisons, chacune doit porter son
`external_id`** — sans quoi une commande ne pourrait pas désigner
laquelle. Le refus est un 422 qui nomme les entrées fautives.

`compare_at_price` est le prix barré, celui qui permet de dire « c'était
199, vous l'avez à 169 ». Il doit être **strictement supérieur** au prix :
l'inverse serait forcément une erreur de saisie.

`stock_quantity` est la quantité en stock de la déclinaison, si vous la
suivez. Absent : l'agent n'en dit rien. **`0` : épuisée** — l'agent ne
confirme pas un article qu'on ne peut pas livrer et propose une autre
déclinaison s'il en reste. Pour un produit sans choix, le champ se pose
au niveau du produit, à côté de `unit_price`.

Trois axes d'options au maximum (`{"Taille": "XL", "Couleur": "rouge"}`).

### Ce qui prime sur la boutique

Trois champs du produit l'emportent sur ceux de la boutique, quand ils
sont renseignés :

```bash
curl -X PATCH "$BASE/v1/products/SERUM-VITC" \
  -H "Authorization: Bearer $YK" \
  -H "Content-Type: application/json" \
  -d '{
    "max_discount": "20.00",
    "return_policy": "Produit d’hygiène : aucun retour une fois ouvert",
    "current_promotions": "Deuxième flacon à -50%"
  }'
```

| Champ | Règle |
|---|---|
| `return_policy` | **Remplace** celle de la boutique pour ce produit |
| `current_promotions` | **S'ajoute** à celle de la boutique — les deux sont vraies |
| `max_discount` | Sur une commande à plusieurs produits, c'est **la plus haute** qui vaut. Une valeur à `0` ne bloque rien : elle dit seulement que ce produit-là n'apporte pas de marge |
| `description` | Sert à répondre aux questions sur l'article, n'est jamais récitée d'office |

### Modifier les déclinaisons

⚠️ **`variants` remplace la liste entière.** Une déclinaison absente de la
nouvelle liste est supprimée du catalogue.

```bash
# Retirer la taille S : on renvoie la liste sans elle.
curl -X PATCH "$BASE/v1/products/TSHIRT-BLANC" \
  -H "Authorization: Bearer $YK" \
  -H "Content-Type: application/json" \
  -d '{
    "variants": [
      { "external_id": "TSH-M",  "options": {"Taille": "M"},  "unit_price": "149.00" },
      { "external_id": "TSH-XL", "options": {"Taille": "XL"}, "unit_price": "169.00" }
    ]
  }'
```

Pour n'en changer qu'une : relisez le produit, modifiez l'entrée, renvoyez
la liste complète. **Ne pas envoyer `variants` du tout les laisse
intactes** — renommer un produit n'efface pas ses déclinaisons.

Les commandes déjà passées ne bougent pas : elles ont retenu ce qui a été
dit au client au moment de l'appel.

### Lister et archiver

```bash
curl "$BASE/v1/products?limit=50&shop_external_id=shop-4471" \
  -H "Authorization: Bearer $YK"

curl -X DELETE "$BASE/v1/products/TSHIRT-BLANC" \
  -H "Authorization: Bearer $YK"
# 204 No Content
```

Comme pour les boutiques, `DELETE` **archive**. Le produit disparaît du
catalogue par défaut et ne peut plus être commandé ; les appels qui l'ont
mentionné restent intacts. Pour le retrouver : `?status=archived`.

### Les erreurs à gérer

| Code | Slug | Quand |
|---|---|---|
| 409 | `product_external_id_already_exists` | `external_id` déjà pris **dans cette boutique**. Le même identifiant reste libre dans une autre |
| 404 | `product_not_found` | Produit inexistant, archivé, ou d'une autre boutique — corps identique |
| 404 | `shop_not_found` | La boutique du chemin n'existe pas |
| 422 | `validation_error` | prix manquant (sur le produit sans choix, ou sur une déclinaison), `unit_price: null` sur un `PATCH`, `external_id` de déclinaison manquant ou en double, prix barré inférieur au prix, plus de trois axes d'options, `unit_price` sur un produit à plusieurs déclinaisons |


## 0.ter Commander depuis le catalogue — la demande d'appel allégée

Une fois la boutique et les produits déclarés, une demande d'appel se
réduit à qui appeler, quoi référencer, et combien.

```bash
curl -X POST "$BASE/v1/calls" \
  -H "Authorization: Bearer $YK" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{
    "customer": {
      "name": "Ahmed Bennani",
      "phone_number": "+212612345678"
    },
    "shop_external_id": "shop-4471",
    "order": {
      "external_id": "CMD-2026-8891",
      "amount": "487.00",
      "items": [
        { "product_external_id": "TSHIRT-BLANC", "variant_external_id": "TSH-XL", "quantity": 1 },
        { "product_external_id": "SERUM-VITC", "quantity": 1 }
      ]
    },
    "call_params": { "purpose": "confirmation" }
  }'
```

Ce qui est **déduit** et n'a plus à être transmis :

| Ce qui n'est plus envoyé | D'où cela vient |
|---|---|
| `shop_info` | La fiche déclarée par `POST /v1/shops` |
| `country` | **L'indicatif du numéro du client**, jamais le pays de la boutique — une boutique vend à plusieurs pays, et se tromper de pays c'est se tromper de dialecte |
| `currency` | Le pays de la boutique (`MA` → MAD, `DZ` → DZD, `TN` → TND) |
| `name`, `description`, `unit_price` de chaque ligne | La fiche produit et la déclinaison référencées |
| `max_discount` | Le plus haut parmi les produits commandés, sinon celui de la boutique |

Deux nuances valent d'être connues. Le pays se lit sur le numéro appelé :
`+212…` est marocain, et cela reste vrai même si la boutique est
tunisienne. Envoyez `country` explicitement pour un Marocain expatrié
joignable sur un `+33` que vous voulez faire appeler en darija — une valeur
fournie gagne toujours. La devise, elle, suit le pays de la boutique, parce
qu'un montant est ce que le commerçant encaisse.

`variant_external_id` est inutile sur un produit qui n'a qu'une
déclinaison — elle est retenue automatiquement. S'il en a plusieurs et
qu'aucune n'est désignée, la requête est refusée
(`order_item_variant_required`) en listant celles qui existent. Aucune
n'est choisie au hasard : l'appel annoncerait au client un article qu'il
n'a pas commandé.

**Ce que vous envoyez malgré tout l'emporte.** Un prix promotionnel valable
pour cette commande-là se transmet sur la ligne, et il prime sur le
catalogue :

```json
{ "product_external_id": "SERUM-VITC", "quantity": 2, "unit_price": "280.00" }
```

**Mélanger les deux formes est permis.** Une ligne sans référence, décrite
sur place avec `name`, reste valide — c'est la voie pour un article sur
mesure, des frais de port ou un cadeau :

```json
"items": [
  { "product_external_id": "TSHIRT-BLANC", "variant_external_id": "TSH-XL", "quantity": 1 },
  { "name": "Frais de livraison", "quantity": 1, "unit_price": "30.00" }
]
```

Un produit absent du catalogue ne doit jamais bloquer un appel.

## 0.ter Les commandes — déclarer, puis appeler par référence

Une commande se déclare comme un produit : sous sa boutique, avec **votre**
identifiant. Elle s'appelle ensuite par cette seule référence, autant de
fois qu'il faut — les appels s'empilent sur elle, il n'y a plus de doublon
de commande pour relancer un client.

### Déclarer

```bash
curl -X POST "$BASE/v1/shops/shop-4471/orders" \
  -H "Authorization: Bearer $YK" \
  -H "Content-Type: application/json" \
  -d '{
    "external_id": "CMD-2026-8891",
    "customer": { "name": "Ahmed Bennani", "phone_number": "+212612345678" },
    "amount": "487.00",
    "delivery_address": "12 rue des Orangers, Casablanca",
    "call_hours": "Mo-Fr 09:30-12:00,13:00-19:00; Sa 13:00-19:00",
    "items": [
      { "product_external_id": "TSHIRT-BLANC", "variant_external_id": "TSH-XL", "quantity": 1 },
      { "product_external_id": "SERUM-VITC", "quantity": 1 }
    ]
  }'
```

Les articles sont **figés à la déclaration** : ce que le catalogue dit à ce
moment-là — nom, prix, variantes — est copié dans la commande, et c'est
cette copie que chaque appel lira.

`call_hours` est la **plage d'appel** : les créneaux pendant lesquels ce
client peut être appelé, au format des horaires OpenStreetMap — jours en
anglais abrégés, heures sur 24 h. Ici du lundi au vendredi matin et
après-midi, le samedi après-midi, jamais le dimanche. Jamais avant 9 h ni
après 22 h : une plage qui en sort est refusée. Absente, c'est celle de la
boutique (`call_hours` sur `POST /v1/shops`) qui vaut, sinon tous les
jours de 9 h à 22 h. Un appel lancé hors plage est accepté et part à la
prochaine ouverture. Une ligne décrite sur place avec `name`
reste valide (frais de port, cadeau, article sur mesure). La réponse porte
`order_status` (`PENDING` tant qu'aucun appel n'a abouti), la liste `calls`
(vide pour l'instant) et `call_in_progress`.

### Appeler

```bash
curl -X POST "$BASE/v1/calls" \
  -H "Authorization: Bearer $YK" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{
    "shop_external_id": "shop-4471",
    "order_external_id": "CMD-2026-8891",
    "call_params": { "purpose": "confirmation" }
  }'
```

Trois champs. Le client, les articles et les montants viennent de la
commande ; `customer` ne se fournit pas — il est celui de la commande, et
se change par `PATCH` sur elle. Le nombre de tentatives précédentes n'est
plus à déclarer : le serveur compte les appels déjà passés.

Relancer le client plus tard : **la même requête**, avec une nouvelle
`Idempotency-Key`. Tant qu'un appel n'est pas terminé, un second est
refusé (`409 order_call_in_progress`).

### Relire, corriger, archiver

```bash
curl "$BASE/v1/orders/CMD-2026-8891" -H "Authorization: Bearer $YK"

curl -X PATCH "$BASE/v1/orders/CMD-2026-8891" \
  -H "Authorization: Bearer $YK" -H "Content-Type: application/json" \
  -d '{ "delivery_address": "14 rue des Orangers, Casablanca", "amount": "450.00" }'

curl -X DELETE "$BASE/v1/orders/CMD-2026-8891" -H "Authorization: Bearer $YK"
```

Le `PATCH` suit la règle des produits : absent = inchangé, `null` = effacé,
`""` = 422. `customer`, `amount` et `items` ne s'effacent pas. `items`
remplace la liste entière et refait la copie figée avec les prix du jour.
Modifier ou archiver **pendant un appel en cours** est refusé.

| Code | Slug | Quand |
|---|---|---|
| 409 | `order_external_id_already_exists` | `external_id` déjà pris **dans cette boutique** |
| 404 | `order_not_found` | Commande inexistante, archivée, ou d'une autre boutique — corps identique |
| 409 | `order_call_in_progress` | Un appel n'est pas terminé : modifier, archiver ou rappeler attendra |
| 422 | `validation_error` | ni `items` ni `items_text`, `customer`/`amount`/`items` à `null`, `external_id` différent de l'URL |
| 409 | `order_closed` | `POST /v1/calls` sur une commande fermée : la rouvrir d'abord |

### Fermer, rouvrir

Le statut d'appel dit ce que le client a répondu. Il ne dit pas si
**vous** en avez fini avec la commande — livrée, retournée, annulée de
votre côté. Comme chez Shopify, c'est un axe à part : `closed`.

```bash
# Le colis est arrivé : le dossier est clos.
curl -X POST "$BASE/v1/orders/CMD-2026-8891/close" \
  -H "Authorization: Bearer $YK" -H "Content-Type: application/json" \
  -d '{ "reason": "delivered" }'

# Les commandes fermées ne sont plus listées par défaut ; les retrouver :
curl "$BASE/v1/orders?lifecycle=closed" -H "Authorization: Bearer $YK"

# Vous vous êtes trompé de commande : la rouvrir.
curl -X POST "$BASE/v1/orders/CMD-2026-8891/reopen" -H "Authorization: Bearer $YK"
```

`reason` est obligatoire : `delivered`, `returned`, `cancelled` (annulée
par vous — le client qui refuse au téléphone, c'est `order_status`) ou
`other`. La fiche renvoie `closed`, `closed_at` et `closed_reason`. Une
commande fermée **ne peut plus être appelée** (`409 order_closed`) tant
qu'elle n'est pas rouverte. Fermer pendant un appel en cours est refusé.

## 1. Appel simple Maroc — commande à article unique (texte libre)

La majorité des commandes contiennent un seul article. Le moyen le plus simple de transmettre le détail est un résumé libre via `order.items_text` (max 500 caractères). Yasmine cite ce résumé tel quel auprès du client.

```bash
curl -X POST "$BASE/v1/calls" \
  -H "Authorization: Bearer $YK" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{
    "customer": {
      "name": "Ahmed Bennani",
      "phone_number": "+212612345678"
    },
    "shop_external_id": "boutique-casa-01",
    "shop_info": {
      "name": "GlowArt Casa"
    },
    "order": {
      "delivery_address": "12 Rue X, Casablanca",
      "items_text": "Crème hydratante visage 50ml",
      "amount": "249.00",
      "currency": "MAD"
    },
    "country": "MA",
    "call_params": {
      "purpose": "confirmation"
    }
  }'
```

Réponse (HTTP **201**) :

```json
{
  "id": "dcaebcdd-a81a-4386-a0df-85d9aaafe862",
  "status": "queued",
  "created_at": "2026-04-19T18:36:09.496546Z",
  "shop_id": "2671d5d2-efff-4e14-ba2a-3bfe80bd7840",
  "customer_phone_masked": "+212 6** *** *78",
  "purpose": "confirmation",
  "amount": "249.00",
  "currency": "MAD",
  "country": "MA",
  "metadata": null
}
```

L'agent parle en darija marocaine. Durée typique : 40-60 secondes.

Le sous-objet `customer` accepte aussi `gender`, `email`, `notes`, `language` — voir le schéma `CustomerInput` dans `docs/openapi.yaml` pour la liste complète des champs.

`items_text` est facultatif : un POST sans `items_text` ni `items` reste valide (Yasmine demandera alors au client de préciser ou s'appuiera uniquement sur `amount`).

---

## 1.bis Commande à plusieurs articles — liste structurée

Quand la commande contient plusieurs articles distincts à citer un par un, utilisez `order.items` (tableau, max 50 entrées). Chaque article expose `name` (requis), puis en optionnel : `quantity` (défaut 1), `variants` (paires nom/valeur, max 3 — ex. `{"name": "Taille", "value": "XXL"}`), `description` (texte libre ≤ 500, sert à répondre aux questions du client sur l'article) et `unit_price`.

```bash
curl -X POST "$BASE/v1/calls" \
  -H "Authorization: Bearer $YK" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{
    "customer": {
      "name": "Ahmed Bennani",
      "phone_number": "+212612345678"
    },
    "shop_external_id": "boutique-casa-01",
    "shop_info": {
      "name": "GlowArt Casa"
    },
    "order": {
      "external_id": "CMD-2026-00790",
      "delivery_address": "12 Rue X, Casablanca",
      "items": [
        { "name": "T-shirt blanc en coton", "quantity": 2, "variants": [{ "name": "Taille", "value": "XXL" }, { "name": "Couleur", "value": "blanc" }], "description": "T-shirt col rond 100% coton, coupe droite.", "unit_price": "120.00" },
        { "name": "Jean noir slim", "quantity": 1, "unit_price": "320.00" }
      ],
      "amount": "560.00",
      "currency": "MAD"
    },
    "country": "MA",
    "call_params": {
      "purpose": "confirmation"
    }
  }'
```

**Choix entre les deux formats** :

| Cas | Format recommandé |
|---|---|
| Commande à article unique | `items_text` (chaîne libre) |
| Plusieurs articles déjà découpés côté reseller (nom, variante, prix unitaire) | `items` (tableau structuré) |
| Plusieurs articles non découpés (ex. "t-shirt blanc et jean noir XXL") | `items_text` |
| Aucune des deux infos disponible | omettre les deux |

`items` et `items_text` peuvent cohabiter dans la même requête : si les deux sont fournis, la liste structurée prime au moment de l'appel.

---

## 2. Appel France avec fiche client enrichie

```bash
curl -X POST "$BASE/v1/calls" \
  -H "Authorization: Bearer $YK" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{
    "customer": {
      "name": "Claire Dupont",
      "phone_number": "+33612345678",
      "gender": "female",
      "email": "claire@example.com",
      "notes": "VIP, horaires bureau uniquement"
    },
    "shop_external_id": "boutique-paris-01",
    "shop_info": {
      "name": "GlowArt Paris",
      "sector": "beauté",
      "return_policy": "retour gratuit sous 14 jours"
    },
    "order": {
      "delivery_address": "25 Av des Champs-Élysées, 75008 Paris",
      "amount": "49.90",
      "currency": "EUR",
      "delivery_method": "express",
      "delivery_zone": "urban",
      "source": "web"
    },
    "country": "FR",
    "call_params": {
      "purpose": "retry",
      "tone": "friendly"
    }
  }'
```

La `country` bascule automatiquement le serveur vers le profil conversationnel français. Les champs facultatifs sont persistés côté serveur et réutilisés lors des appels suivants pour le même `(customer, boutique)` (COALESCE stickiness — un POST ultérieur sans ces champs n'écrase pas les valeurs existantes côté customer + boutique).

---

## 2.5. Forcer la langue d'un appel (override du défaut pays)

Le pays détermine une langue par défaut (`MA`/`DZ`/`TN` → `ar`, `FR` → `fr`). Si votre client préfère une autre langue parmi celles supportées, ajoutez le champ optionnel `language` à la racine du payload. Exemple : un client marocain qui préfère le français.

```bash
curl -X POST "$BASE/v1/calls" \
  -H "Authorization: Bearer $YK" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{
    "customer": {
      "name": "Sophie El Idrissi",
      "phone_number": "+212661111222"
    },
    "shop_external_id": "boutique-casa-01",
    "shop_info": {"name": "GlowArt Casa"},
    "order": {
      "delivery_address": "12 Rue X, Casablanca",
      "amount": "249.00",
      "currency": "MAD"
    },
    "country": "MA",
    "language": "fr",
    "call_params": {"purpose": "confirmation"}
  }'
```

Combinaisons supportées : `MA`/`DZ`/`TN` avec `ar` ou `fr`, `FR` uniquement avec `fr`. La combinaison `country=FR` + `language=ar` est rejetée en `422 language_not_supported_for_country`. Si vous omettez `language`, la langue locale du pays est appliquée — comportement strictement identique aux versions précédentes de l'API (rétrocompat garantie). La langue effective est ré-écho dans le champ `language` de la réponse.

---

## 3. Avec metadata reseller

Utilisez `metadata` pour transporter votre identifiant de commande interne, référence facture, ou tout contexte dont vous aurez besoin plus tard. Limite : **2 KB** sérialisé en JSON compact.

```bash
curl -X POST "$BASE/v1/calls" \
  -H "Authorization: Bearer $YK" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{
    "customer": {
      "name": "Ahmed Bennani",
      "phone_number": "+212612345678"
    },
    "shop_external_id": "boutique-casa-01",
    "shop_info": {"name": "GlowArt Casa"},
    "order": {
      "external_id": "CMD-2026-00789",
      "delivery_address": "12 Rue X, Casa",
      "amount": "180.00",
      "currency": "MAD"
    },
    "country": "MA",
    "call_params": {"purpose": "followup_delivery"},
    "metadata": {
      "invoice_ref": "INV-2026-1042",
      "source_crm": "shopify",
      "custom_tags": ["vip", "repeat_customer"]
    }
  }'
```

Cette `metadata` sera conservée côté serveur et vous sera renvoyée telle quelle dans la réponse et (à terme) dans les webhooks sortants.

---

## 3.2. Gérer mes clés API

Trois endpoints self-service pour créer, lister et révoquer vos clés API.

```bash
# Créer une nouvelle clé — secret_key visible UNE SEULE FOIS.
curl -X POST -H "Authorization: Bearer $YK" \
  -H "Content-Type: application/json" \
  -d '{"label":"ci-prod"}' \
  "$BASE/v1/me/api-keys"
# {
#   "id": "3f8e9d12-...",
#   "key_prefix": "yk_Xa7bC3dE5f",
#   "label": "ci-prod",
#   "created_at": "2026-04-21T12:00:00Z",
#   "secret_key": "yk_Xa7bC3dE5fG9hI1jK2lM3nO4pQ5rS6tU7vW8xY9zA"
# }

# Stockez immédiatement `secret_key` côté reseller — aucune autre route
# ne la ré-expose. Si vous la perdez : révocation + nouvelle création.

# Lister toutes les clés (sans secret, jamais).
curl -H "Authorization: Bearer $YK" "$BASE/v1/me/api-keys"
# { "data": [{ id, key_prefix, label, created_at, last_used_at,
#              last_used_ip, revoked_at, status }] }

# Révoquer une clé (soft delete, historique préservé avec status=revoked).
curl -X DELETE -H "Authorization: Bearer $YK" \
  "$BASE/v1/me/api-keys/3f8e9d12-4b5a-4c6d-8e7f-123456789abc"
# → 204 No Content
```

**Rate-limit création** : 5/min/reseller. Au-delà → 429. Empêche un script mal configuré de générer des milliers de clés par erreur.

**Anti-énumération** : un `DELETE` sur un UUID qui n'existe pas **ou** qui appartient à un autre reseller renvoie un 404 strictement identique — pas moyen de sonder l'existence des clés des autres.

**Auto-révocation safe** : si vous révoquez la clé que vous êtes en train d'utiliser, la requête courante termine normalement (204), la suivante retourne 401. Pas de besoin de re-authentifier manuellement.

---

## 3.2.1. Journal d'audit d'une clé API

Le serveur enregistre automatiquement trois types d'événements par clé : `created`, `revoked`, `rate_limited`. Utile pour détecter qu'une clé a été spamée (429 multiples) ou pour auditer qui a créé/révoqué quoi et quand.

```bash
# Lister les events d'une clé (paginé curseur, limit max 50).
KEY_ID="3f8e9d12-4b5a-4c6d-8e7f-123456789abc"
curl -H "Authorization: Bearer $YK" \
  "$BASE/v1/me/api-keys/$KEY_ID/events?limit=50"
# {
#   "data": [
#     { "id":"...", "event_type":"revoked", "ip":"1.2.3.4",
#       "user_agent":"curl/8.4.0", "metadata":null,
#       "created_at":"2026-04-21T14:30:00Z" },
#     { "id":"...", "event_type":"rate_limited", "ip":"1.2.3.4",
#       "user_agent":"curl/8.4.0",
#       "metadata":{"path":"/v1/calls","limit":"600 per 1 minute"},
#       "created_at":"2026-04-21T13:15:22Z" },
#     { "id":"...", "event_type":"created", "ip":"1.2.3.4",
#       "user_agent":"curl/8.4.0", "metadata":{"label":"ci-prod"},
#       "created_at":"2026-04-21T12:00:00Z" }
#   ],
#   "next_cursor": null,
#   "has_more": false
# }

# Page suivante avec le next_cursor de la réponse précédente.
curl -H "Authorization: Bearer $YK" \
  "$BASE/v1/me/api-keys/$KEY_ID/events?cursor=<next_cursor>"
```

**Dédup `rate_limited`** : pour éviter de remplir le journal si un script mal configuré se fait 429 en boucle, un seul event `rate_limited` est écrit par clé toutes les 60 s. Un burst de 1000 requêtes rate-limitées en 10 s → 1 event.

**Anti-énumération** : un GET sur un `key_id` qui n'existe pas **ou** qui appartient à un autre reseller → 404 `api_key_not_found` strictement identique au DELETE.

---

## 3.3. Suivre mon compte

Trois endpoints de lecture self-service.

```bash
# 1. Identité + profil société (anti-BOPLA : pas de balance ni status).
#    Les champs profil (owner_name, country, tax_id, business_type, …) sont
#    `null` tant que la fiche société n'a pas été renseignée côté ops.
#    Schéma complet : voir `MeOut` dans `docs/openapi.yaml`.
curl -H "Authorization: Bearer $YK" "$BASE/v1/me"

# 2. Solde.
curl -H "Authorization: Bearer $YK" "$BASE/v1/me/balance"
# {"balance_seconds":3600,"currency":null,"updated_at":"..."}

# 3. Historique ledger paginé (topup, debit_call, refund, adjustment).
curl -H "Authorization: Bearer $YK" "$BASE/v1/me/transactions?limit=10"
# Page suivante : passer next_cursor tel quel.
curl -H "Authorization: Bearer $YK" "$BASE/v1/me/transactions?cursor=eyJkaXIi..."

# 4. Consommation mensuelle (défaut = mois courant UTC).
curl -H "Authorization: Bearer $YK" "$BASE/v1/me/usage"
# Mois spécifique :
curl -H "Authorization: Bearer $YK" "$BASE/v1/me/usage?period=2026-04"
# {
#   "period":"2026-04",
#   "period_start":"2026-04-01T00:00:00+00:00",
#   "period_end":"2026-05-01T00:00:00+00:00",
#   "total_calls":152,
#   "total_seconds":2340,
#   "breakdown":{
#     "completed_calls":138, "completed_seconds":2280,
#     "failed_calls":10, "cancelled_calls":4
#   }
# }
```

Note sur la facturation : `total_seconds` inclut tous les calls du mois (y compris in-flight). Pour la facturation effective, utiliser `breakdown.completed_seconds` (calls en `call_status = 'ended'`).

Pas de 404 sur période sans appel — la réponse est 200 avec tous les compteurs à zéro.

---

## 3.4. Lister les appels avec pagination

`GET /v1/calls` renvoie une **enveloppe paginée** `{data, next_cursor, has_more}`. Tri `created_at DESC` avec tie-break stable `id DESC`.

```bash
# Page 1 — défaut 50 items, ou configurer avec ?limit= (max 200).
curl -H "Authorization: Bearer $YK" "$BASE/v1/calls?limit=10"
# {
#   "data": [{...}, {...}, ...],
#   "next_cursor": "eyJkaXIiOiJuZXh0IiwiaWQiOiIuLi4iLCJ0cyI6Ii4uLiJ9.XDg4Zjg...",
#   "has_more": true
# }

# Page suivante — passer next_cursor tel quel.
curl -H "Authorization: Bearer $YK" "$BASE/v1/calls?limit=10&cursor=eyJkaXIi..."
```

**Filtres combinables** :

```bash
# Appels du boutique `boutique-casa-01` terminés dans la dernière semaine.
curl -H "Authorization: Bearer $YK" \
  "$BASE/v1/calls?status=ended&shop_id=8f3a...&since=2026-04-14T00:00:00Z&limit=100"
```

**Rotation du curseur** : le curseur est signé HMAC-SHA256 avec une clé serveur. Si l'équipe Yasmine rotate cette clé, tous les curseurs émis avant deviennent invalides (`400 invalid_cursor`) — refaire la 1re requête sans `cursor`. Même comportement après 24 h (`400 cursor_expired`).

---

## 3.5. Récupérer un appel + consulter son solde

```bash
# Lecture d'un appel — scoped tenant, 404 anti-énum si autre reseller / inconnu / UUID invalide
curl -H "Authorization: Bearer $YK" \
  "$BASE/v1/calls/dcaebcdd-a81a-4386-a0df-85d9aaafe862" \
  | jq '{call_status, result, result_detail, customer_mood, flags, preferences, next_action, summary, recording_url, ended_at, call_duration_seconds, billable_duration_seconds, failure_reason, meta_error_code, meta_error_title}'
# {
#   "call_status": "ended",
#   "result": "confirmed",
#   "result_detail": "modified",
#   "customer_mood": "positive",
#   "flags": [],
#   "preferences": ["t-shirt couleur bleue", "livraison mardi 14h"],
#   "next_action": null,
#   "summary": "La cliente confirme la commande mais demande le t-shirt en bleu et la livraison mardi 14h.",
#   "recording_url": "/v1/calls/dcaebcdd-a81a-4386-a0df-85d9aaafe862/recording",
#   "ended_at": "2026-04-21T14:31:57Z",
#   "call_duration_seconds": 37,
#   "billable_duration_seconds": 37,
#   "failure_reason": null,
#   "meta_error_code": null,
#   "meta_error_title": null
# }

# Télécharger l'enregistrement audio (mix client + agent, WAV mono 16 kHz)
curl -H "Authorization: Bearer $YK" \
  -o "dcaebcdd.wav" \
  "$BASE/v1/calls/dcaebcdd-a81a-4386-a0df-85d9aaafe862/recording"
# → fichier WAV ~2 MB pour 60s. Rétention 30 jours.
# 410 Gone si l'audio a été purgé après rétention (slug `recording_gone`).

# Filtrer la liste : commandes confirmées d'une boutique
curl -H "Authorization: Bearer $YK" \
  "$BASE/v1/calls?status=ended&result=confirmed&shop_id=$MID&limit=50"

# Consulter le solde courant
curl -H "Authorization: Bearer $YK" "$BASE/v1/me/balance"
# {"balance_seconds":3540,"currency":null,"updated_at":"2026-04-21T14:32:11Z"}
```

`GET /v1/calls/{id}` est utile pour **réconcilier** après un webhook perdu : le crash mid-retry est un edge case accepté produit (`CHANGELOG.md` C6 Phase 2). Si vous voyez un appel `accepted` mais jamais `ended` côté votre webhook, polling cet endpoint quelques secondes plus tard donne **l'état terminal complet** : `call_status` (`ended`/`failed`/`cancelled`), classification métier (`result`/`result_detail`/`customer_mood`/`flags`/`preferences`/`next_action`/`summary`), `call_duration_seconds`, `billable_duration_seconds`, `failure_reason` et — pour `result=requires_action` avec `result_detail=failed` issu d'un échec opérateur — `meta_error_code` + `meta_error_title`.

Le 404 `call_not_found` ne distingue PAS un appel inexistant d'un appel appartenant à un autre reseller — anti-énumération volontaire.

---

## 4. Idempotency-Key — obligatoire, se protéger des retries réseau

Le header `Idempotency-Key` est **obligatoire** sur `POST /v1/calls` (sans lui : `400 missing_idempotency_key`). Format libre, 1-255 chars (UUID v4 recommandé). Scope par reseller, TTL 24 h.

```bash
# Générez la clé UNE fois pour votre retry loop (ou reutilisez votre order_id stable)
IDEMP=$(uuidgen)

# Tentative 1
curl -X POST "$BASE/v1/calls" \
  -H "Authorization: Bearer $YK" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $IDEMP" \
  -d '{ "customer": { "name": "X", "phone_number": "+212612345678" }, "shop_external_id": "x", "shop_info": {"name": "X Shop"}, "order": {"delivery_address": "Rue X, Casa", "amount": "100", "currency": "MAD"}, "country": "MA", "call_params": {"purpose": "confirmation"} }'

# Si timeout / erreur réseau, retry avec LA MÊME clé + LE MÊME body
curl -X POST "$BASE/v1/calls" \
  -H "Authorization: Bearer $YK" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $IDEMP" \
  -d '{ ... même payload ... }'
```

La 2e requête retourne **bit-for-bit la même réponse** que la 1ère, avec en plus le header `X-Idempotent-Replay: true`. Aucun nouvel appel n'est déclenché, aucun nouveau débit, aucun nouveau template Meta.

**Conflit** : si vous réutilisez la même clé avec un body différent, vous obtenez `409 idempotency_key_conflict` (le body original n'est jamais ré-exposé). Générez une nouvelle clé dès que la requête change.

**Choix de clé** :
- UUID v4 généré côté client (`uuidgen`, `crypto.randomUUID()`, `uuid.uuid4()`).
- Identifiant business stable par requête (ex. `order_id` + `purpose`) si vous voulez une dédup naturelle sur votre logique métier.
- Évitez les clés réutilisées sur plusieurs jours sans précaution : la fenêtre est de 24 h, après quoi la clé peut être réutilisée pour une nouvelle requête.

---

## 5. Parser un 422 côté client

Un payload invalide renvoie un `422` RFC 7807 avec `errors[]` listant chaque champ fautif :

```bash
# payload volontairement invalide — il sert a montrer la forme du 422
curl -X POST "$BASE/v1/calls" \
  -H "Authorization: Bearer $YK" \
  -H "Content-Type: application/json" \
  -d '{
    "customer": {
      "name": "",
      "phone_number": "+0000000000"
    },
    "shop_external_id": "ma boutique",
    "shop_info": {"name": ""},
    "order": {"delivery_address": "", "amount": "-10", "currency": "mad"},
    "country": "XX",
    "call_params": {"purpose": "invalid_enum"}
  }'
```

Réponse (HTTP **422**) :

```json
{
  "type": "https://docs.yasmine.akidly.com/errors/validation_error",
  "title": "Unprocessable Entity",
  "status": 422,
  "detail": "Un ou plusieurs champs du payload sont invalides.",
  "instance": "/v1/calls",
  "request_id": "a1b2c3d4",
  "errors": [
    { "loc": ["body", "customer", "phone_number"], "msg": "Value error, phone_number invalide — format attendu E.164 (ex: +212612345678)", "type": "value_error" },
    { "loc": ["body", "customer", "name"], "msg": "String should have at least 1 character", "type": "string_too_short" },
    { "loc": ["body", "shop_external_id"], "msg": "String should match pattern '^[a-zA-Z0-9_.-]+$'", "type": "string_pattern_mismatch" },
    { "loc": ["body", "shop_info", "name"], "msg": "String should have at least 1 character", "type": "string_too_short" },
    { "loc": ["body", "order", "delivery_address"], "msg": "String should have at least 1 character", "type": "string_too_short" },
    { "loc": ["body", "order", "amount"], "msg": "Decimal input should be greater than 0", "type": "greater_than" },
    { "loc": ["body", "order", "currency"], "msg": "Input should be 'MAD','EUR','USD','TND' or 'DZD'", "type": "literal_error" },
    { "loc": ["body", "country"], "msg": "Input should be 'MA', 'DZ', 'TN' or 'FR'", "type": "literal_error" },
    { "loc": ["body", "call_params", "purpose"], "msg": "Input should be one of the 5 enum values", "type": "literal_error" }
  ]
}
```

### Champs inconnus → 422 `extra_forbidden`

Tout champ qui ne fait pas partie du schéma `CallCreate` ou `CustomerInput` est rejeté. Un typo à la racine (par exemple `custommer` au lieu de `customer`) ou à l'intérieur de `customer` (par exemple `phonne_number`) ne passe pas silencieusement — le serveur renvoie **422** avec le chemin exact du champ fautif :

```json
{
  "type": "https://docs.yasmine.akidly.com/errors/validation_error",
  "title": "Unprocessable Entity",
  "status": 422,
  "instance": "/v1/calls",
  "request_id": "f1b14110-...",
  "detail": "Un ou plusieurs champs du payload sont invalides.",
  "errors": [
    { "loc": ["body", "customer", "phonne_number"], "msg": "Extra inputs are not permitted", "type": "extra_forbidden" }
  ]
}
```

Parser côté client : si `errors[].type === "extra_forbidden"`, c'est un champ en trop — vérifier l'orthographe ou supprimer du body. La liste des champs autorisés est dans la [spec OpenAPI](https://docs.yasmine.akidly.com/#tag/Calls).

Exemple de parsing JavaScript côté reseller :

```javascript
async function createCall(payload) {
  const res = await fetch(`${BASE}/v1/calls`, {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${YK}`,
      "Content-Type": "application/json",
      "Idempotency-Key": crypto.randomUUID(),
    },
    body: JSON.stringify(payload),
  });

  if (res.status === 422) {
    const problem = await res.json();
    // Mappez errors[].loc vers vos champs de formulaire UI
    const fieldErrors = {};
    for (const err of problem.errors || []) {
      const field = err.loc[err.loc.length - 1];
      fieldErrors[field] = err.msg;
    }
    throw new ValidationError(fieldErrors, problem.request_id);
  }

  if (!res.ok) {
    const problem = await res.json();
    // Routez sur problem.type (URI stable), pas sur status seul
    throw new ApiError(problem.type, problem.detail, problem.request_id);
  }

  return await res.json();
}
```

---

## 6. Gérer un 402 — solde insuffisant

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

**Retry aveugle : mauvaise idée.** Un 402 signifie que le solde reseller est vide — retenter immédiatement donnera le même 402 et ne crée pas l'appel.

Logique recommandée côté reseller :

```python
def create_call_with_balance_check(payload):
    res = requests.post(f"{BASE}/v1/calls", json=payload, headers={
        "Authorization": f"Bearer {YK}",
        "Idempotency-Key": str(uuid.uuid4()),
    })
    if res.status_code == 402:
        problem = res.json()
        # 1. Ne PAS retry.
        # 2. Alerter l'équipe ops (pour top-up).
        # 3. Mettre la commande en file de relance (side-queue).
        logger.error(
            "Solde Yasmine insuffisant, request_id=%s",
            problem.get("request_id"),
        )
        queue_for_retry_after_topup(payload)
        return None
    res.raise_for_status()
    return res.json()
```

Le self-service `GET /v1/me/balance` arrive en M4 — vous pourrez surveiller votre solde en amont et éviter les 402 en prod.

---

## 7. Annuler un appel en cours

`POST /v1/calls/{id}/cancel` coupe immédiatement un appel en cours, que le rail APPEL soit en `dialing`, `ringing` ou `connected`.

```bash
CALL_ID="dcaebcdd-a81a-4386-a0df-85d9aaafe862"

curl -X POST "$BASE/v1/calls/$CALL_ID/cancel" \
  -H "Authorization: Bearer $YK"
```

Réponse (HTTP **200**) :

```json
{
  "id": "dcaebcdd-a81a-4386-a0df-85d9aaafe862",
  "status": "cancelled",
  "cancelled_state": "ringing",
  "billed_seconds": 0,
  "cancelled_at": "2026-04-20T14:00:16.123456Z"
}
```

**Facturation** : `billed_seconds = 0` si annulé avant `connected`, sinon `ceil(now - accepted_at)` avec plancher 10 s. Un event webhook `call.cancelled` est émis sur cancel effectif (cf `docs/webhooks.md` §7).

**Idempotent** : rejouer le cancel sur un call déjà terminal (`ended`, `failed`, `cancelled`) retourne `200` avec le snapshot courant, **sans** re-déclencher d'event webhook ni de débit.

---

## 8. Header `X-API-Version`

Chaque réponse contient désormais le header `X-API-Version` — exemple :

```http
HTTP/1.1 201 Created
X-API-Version: v1
Content-Type: application/json
```

Si vous recevez une valeur inattendue (ex. `v2`), log + alerte — votre SDK a probablement besoin d'une mise à jour.
