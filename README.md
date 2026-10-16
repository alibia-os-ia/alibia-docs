# ALIBIA — Suivi du backend

> Mis à jour le 16/10/2026, à partir du code de `main` (commit `c47054a`).
> API en production : `https://alibia-api-eu.onrender.com/api/v1`

Ce document a deux parties :

- **Partie A — Synthèse produit** : ce que fait ALIBIA, pour qui, combien ça
  coûte. C'est la base de la landing page.
- **Partie B — Journal technique** : décisions, schémas, règles métier,
  étape par étape (Étapes 10 à 17).

---

# PARTIE A — Synthèse produit (base de la landing page)

## En une phrase

ALIBIA est un logiciel de gestion commerciale en ligne (SaaS) pour les
commerçants : stock, ventes, factures, encaissements, dépenses, caisse,
revendeurs et tableaux de bord, avec une équipe aux rôles distincts.

## Pour qui

- Boutiques physiques et e-commerce qui vendent avec une équipe
- Commerçants qui travaillent avec des **revendeurs** (stock confié,
  commissions, reversements)
- Entrepreneurs qui gèrent **plusieurs commerces** depuis un seul compte

## Fonctionnalités réellement disponibles

Chaque ligne correspond à une route en production.

### Gestion de l'équipe
- Invitation de membres par email (lien valable **7 jours**)
- **5 rôles** : Propriétaire, Administrateur, Employé, Livreur, Revendeur
- Changement de rôle et désactivation d'un membre
- Un même compte peut être propriétaire d'un commerce et membre d'un autre
- **Plusieurs commerces** par compte, avec bascule instantanée entre eux

### Stock et produits
- Catalogue produits (nom, description, référence SKU unique par commerce)
- **Plusieurs emplacements** (boutique, entrepôt, dépôt…)
- Stock séparé par produit, emplacement **et propriétaire** : le stock
  confié à un revendeur reste distinct du stock de la boutique
- Ajustement, transfert entre emplacements, retrait, produits défectueux
- **Historique complet** : chaque mouvement de quantité est tracé (qui,
  quand, combien, pourquoi)
- Stock négatif impossible, même avec plusieurs utilisateurs simultanés
- **4 prix par stock** : prix fournisseur, prix revendeur, prix public,
  prix plancher

### Ventes et facturation
- Vente multi-produits en une opération, y compris avec du stock de
  propriétaires différents
- Contrôle du prix plancher (impossible de vendre en dessous)
- **Calcul automatique des commissions** des revendeurs
- Numérotation automatique : `SALE-2026-00001`, `INV-2026-00001`
- Facture officielle générée à la demande, une par vente, non modifiable
- Les prix de la vente sont figés : changer un tarif plus tard ne modifie
  jamais l'historique

### Encaissements et caisse
- Paiement total ou partiel d'une vente (espèces, Mobile Money, virement,
  autre)
- Protection contre le sur-paiement, même en cas de saisies simultanées
- **Caisse** : solde calculé en temps réel, entrées et sorties détaillées
- Suivi des **ventes impayées** (reste à encaisser)

### Dépenses
- 6 catégories : achat fournisseur, transport, livraison, publicité,
  loyer, autre
- Dépense « à payer » puis « payée » (la sortie de caisse est alors
  enregistrée automatiquement)
- Répartition des dépenses par catégorie

### Revendeurs
- Stock confié par revendeur
- Le revendeur ne voit et ne modifie que **son** stock et **ses** ventes
- Calcul du **montant dû** à chaque propriétaire de stock
  (ventes générées − commissions − déjà reversé)
- **Reversements** partiels ou totaux, enregistrés en sortie de caisse

### Tableaux de bord
- Synthèse financière sur une période : ventes, panier moyen,
  encaissements, dépenses, commissions
- Évolution **par jour, semaine, mois ou année** : ventes, encaissements,
  dépenses, flux de caisse
- Performances par vendeur (nombre de ventes, chiffre d'affaires, panier
  moyen)
- Commissions par vendeur et par propriétaire
- Encaissements par moyen de paiement

### Sécurité
- Vérification de l'email par code à 6 chiffres
- Double authentification (2FA) par email, activable
- Mot de passe oublié par code email
- Verrouillage du compte après **5 échecs** de connexion (15 minutes)
- Données strictement séparées entre commerces
- Connexion chiffrée (HTTPS), hébergement en Europe (Frankfurt)

## Offre et tarifs

| Formule | Prix | Durée |
| --- | --- | --- |
| **Essai gratuit** | 0 XAF | **30 jours**, sans paiement, dès la création du commerce |
| **Mensuel** | **10 000 XAF** | 1 mois |
| **Annuel** | **100 000 XAF** | 12 mois (2 mois offerts par rapport au mensuel) |

- L'abonnement est **par commerce**, pas par utilisateur : toute l'équipe
  est incluse, sans surcoût par membre.
- Paiement **Orange Money** et **MTN Mobile Money** (Cameroun) via
  CinetPay. Le client choisit son moyen sur la page CinetPay ; la carte
  bancaire n'est disponible que si elle est activée sur le compte marchand
  (à confirmer avant de l'annoncer).
- Renouvellement anticipé : les jours restants d'une période payée ne sont
  jamais perdus, la nouvelle période s'ajoute à la fin.
- Les jours d'essai restants ne s'ajoutent pas à la première période payée.

## Ce qui n'existe PAS encore (à ne pas promettre sur la landing)

- Application mobile
- Facture en PDF (les données existent, le PDF n'est pas généré)
- Blocage automatique de l'accès à la fin de l'abonnement (l'état est
  calculé, mais pas encore appliqué aux modules)
- Mouvements de caisse manuels (types prévus en base, pas de route)
- Retours clients (`RETURNED` prévu en base, pas de route)
- Logo du commerce (champ prévu, pas d'envoi de fichier)
- Emails en anglais : l'API répond en français ou en anglais selon
  `Accept-Language`, mais les emails (OTP, invitations) sont en français
  uniquement

---

# PARTIE B — Journal technique

## Conventions communes

- Préfixe global : `/api/v1`. Auth par `Authorization: Bearer <JWT>`.
- Le `businessId` vient **toujours** du JWT, jamais du client.
- `JwtStrategy` relit le membership en base à **chaque** requête : une
  rétrogradation ou une révocation s'applique immédiatement.
- `BusinessContextGuard` (fail-closed) : toute route métier exige un
  business actif dans le token, sauf exemption explicite
  `@NoBusinessContext()`.
- Cross-business → **404** (jamais 403). Rôle insuffisant → **403**.
- Erreurs i18n `fr`/`en` via `Accept-Language`, avec un `messageKey` stable.
- Montants en entiers (XAF, sans décimales).

## Matrice des rôles (état réel du code)

| Domaine | OWNER | ADMIN | STAFF | REVENDEUR | LIVREUR |
| --- | --- | --- | --- | --- | --- |
| Produits (lecture/écriture) | ✅ | ✅ | ✅ | ✅ | ⚠️ voir note |
| Emplacements — lecture | ✅ | ✅ | ✅ | ✅ | ⚠️ voir note |
| Emplacements — création/modif | ✅ | ✅ | ❌ | ❌ | ❌ |
| Stock | ✅ | ✅ | ✅ | son stock | ❌ |
| Ventes, factures | ✅ | ✅ | ✅ | ses ventes + ventes de son stock | ❌ |
| Encaisser un paiement | ✅ | ✅ | ❌ | ❌ | ❌ |
| Lire les paiements | ✅ | ✅ | ❌ | visibilité filtrée | ❌ |
| Caisse, dépenses, synthèse, timeline, stats, commissions, impayés | ✅ | ✅ | ❌ | ❌ | ❌ |
| Reversements — créer | ✅ | ✅ | ❌ | ❌ | ❌ |
| Reversements — lire, solde dû | ✅ | ✅ | ❌ | les siens | ❌ |
| Inviter, désactiver un membre | ✅ | ✅ | ❌ | ❌ | ❌ |
| Changer un rôle | ✅ | ❌ | ❌ | ❌ | ❌ |
| Abonnement, paiement de l'abonnement | ✅ | ✅ | ❌ | ❌ | ❌ |

> ⚠️ **Écart constaté** : la doc précédente indiquait « LIVREUR n'a aucun
> accès à l'inventaire ». Dans le code, `ProductsController` n'a pas de
> `RolesGuard`, et la lecture des emplacements non plus : un LIVREUR peut
> donc lister, lire, créer et modifier des **produits**, et lire les
> emplacements. Le stock, les ventes et les factures l'excluent bien. À
> corriger si la règle « aucun accès » est toujours voulue.

Autres règles membres : on ne peut pas inviter quelqu'un comme OWNER ; un
ADMIN ne peut pas modifier un OWNER ; le dernier OWNER actif ne peut pas
être retiré.

---

# Étape 10 — Module Inventory : Produits, Localisations, Stock (Phase 1)

## Objectif

Poser les fondations de la gestion d'inventaire multi-tenant : produits,
localisations physiques, et stock séparé par produit + localisation +
propriétaire, avec traçabilité complète de tout mouvement de quantité.

## Décision d'architecture — stratégie de stock

Le stock n'est **jamais** un simple champ `quantity` sur `Product`. Il est
modélisé comme une entité à part entière (`Stock`), unique par triplet
`(productId, locationId, ownerMemberId)` :

- `ownerMemberId` **nullable** = stock appartenant à la boutique elle-même
- `ownerMemberId` renseigné = stock détenu par un `BusinessMember` précis
  (typiquement un `REVENDEUR`)
- Isolation multi-tenant stricte : tout accès filtre par `businessId`

## Schéma Prisma — modèles

```prisma
model Product {
  id                String   @id @default(cuid())
  businessId        String
  name              String
  description       String?
  sku               String?
  createdByMemberId String
  createdAt         DateTime @default(now())
  updatedAt         DateTime @updatedAt

  @@unique([businessId, sku])
  @@index([businessId])
  @@map("products")
}

model Location {
  id         String   @id @default(cuid())
  businessId String
  name       String
  createdAt  DateTime @default(now())
  updatedAt  DateTime @updatedAt

  @@unique([businessId, name])
  @@map("locations")
}

model Stock {
  id            String  @id @default(cuid())
  businessId    String
  productId     String
  locationId    String
  ownerMemberId String?
  quantity      Int     @default(0)

  // Pricing — optionnel à la création, géré via PATCH /stocks/:id/pricing
  supplierPrice Int?
  resellerPrice Int?
  publicPrice   Int?
  minPrice      Int?

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@unique([productId, locationId, ownerMemberId])
  @@index([businessId])
  @@map("stocks")
}

enum StockMovementType {
  ADDED
  ADJUSTED
  TRANSFERRED_OUT
  TRANSFERRED_IN
  SOLD
  RETURNED     // prévu, aucune route ne le produit encore
  DEFECTIVE
  RETRIEVED
}

model StockMovement {
  id                  String            @id @default(cuid())
  businessId          String
  stockId             String
  type                StockMovementType
  quantity            Int
  note                String?
  performedByMemberId String
  relatedStockId      String?
  createdAt           DateTime          @default(now())

  @@index([businessId])
  @@index([stockId])
  @@map("stock_movements")
}
```

**Note sur `@@unique([productId, locationId, ownerMemberId])`** :
PostgreSQL traite `NULL` comme distinct de toute autre valeur dans une
contrainte unique classique. Un index unique **partiel** (migration
`20260823080839_add_stock`) garantit qu'il n'existe qu'**une seule** ligne
de stock boutique (`ownerMemberId IS NULL`) par `(productId, locationId)`.

## Fichiers importants

| Fichier | Rôle |
| --- | --- |
| `src/modules/inventory/inventory.module.ts` | Regroupe Products, Locations, Stocks, Sales, Invoices |
| `src/modules/inventory/products/products.controller.ts` / `.service.ts` | CRUD produits |
| `src/modules/inventory/locations.controller.ts` / `.service.ts` | Localisations (OWNER/ADMIN pour create/update) |
| `src/modules/inventory/stocks/stocks.controller.ts` / `.service.ts` | Création, lecture, ajustement, transfert, retrait, tarification |
| `src/modules/inventory/constants/inventory-messages.ts` | i18n Inventory (`INVENTORY_*`) |
| `src/common/services/member-resolver.service.ts` (`CommonModule`) | `User.id` → `BusinessMember.id`, partagé Inventory/Finance/Subscription |

## Règles métier — Stock

- `OWNER` / `ADMIN` / `STAFF` gèrent tout le stock du business
- `REVENDEUR` ne modifie **que son propre stock**
  (`ownerMemberId === callerMemberId`) et ne crée du stock qu'à son nom
  (`INVENTORY_STOCK_OWNER_FORBIDDEN` sinon)
- `LIVREUR` exclu du stock (voir l'écart sur les produits ci-dessus)
- **Toute** modification de quantité crée un `StockMovement` dans la
  **même transaction** que la mutation du `Stock`
- Stock négatif **interdit** structurellement, y compris sous concurrence
- Trois concepts distincts : `seller` (qui vend), `owner` (à qui appartient
  le stock), `creator` (qui a créé la fiche produit)

## Routes

| Route | Description |
| --- | --- |
| `POST /products`, `GET /products`, `GET /products/:id`, `PATCH /products/:id` | Catalogue |
| `POST /locations`, `GET /locations`, `PATCH /locations/:id` | Emplacements |
| `POST /stocks` | Création (quantité initiale, tarification optionnelle) |
| `GET /stocks`, `GET /stocks/:id` | Lecture avec filtres combinables |
| `POST /stocks/:id/adjust` | Ajustement signé (+/-), `ADJUSTED` |
| `POST /stocks/:id/transfer` | Transfert (`TRANSFERRED_OUT` + `TRANSFERRED_IN`), jamais de fusion entre propriétaires |
| `POST /stocks/:id/retrieve` | Retrait (`RETRIEVED`), via `applyWithdrawal` commun |
| `POST /stocks/:id/mark-defective` | Défectueux (`DEFECTIVE`) |
| `PATCH /stocks/:id/pricing` | Tarification, règle `minPrice <= resellerPrice` |

---

# Étape 11 — Module Sales : Pricing + Ventes (Phase 2A + 2B)

## Objectif

Vendre du stock avec calcul automatique de commission, snapshot immuable
des prix, et numérotation séquentielle par business.

## Phase 2A — Pricing

Les prix vivent sur `Stock` (`supplierPrice`, `resellerPrice`,
`publicPrice`, `minPrice`), propres à chaque triplet.

- `minPrice <= resellerPrice`
- Vente impossible tant que `resellerPrice` ou `minPrice` manque
- Prix de vente effectif `>= minPrice`
- Commission jamais négative (`Math.max(0, ...)`)

## Phase 2B — Sales

```prisma
model Sale {
  id              String   @id @default(cuid())
  businessId      String
  saleNumber      String
  sellerMemberId  String
  clientName      String?
  clientPhone     String?
  totalAmount     Int
  totalCommission Int
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt

  @@unique([businessId, saleNumber])   // ← mis à jour, voir ci-dessous
  @@index([businessId])
  @@map("sales")
}

model SaleItem {
  id            String   @id @default(cuid())
  saleId        String
  stockId       String
  productId     String
  ownerMemberId String?
  quantity      Int
  unitPrice     Int
  resellerPrice Int
  minPrice      Int
  commission    Int
  createdAt     DateTime @default(now())

  @@index([saleId])
  @@index([stockId])
  @@map("sale_items")
}

model SaleCounter {
  id         String @id @default(cuid())
  businessId String
  year       Int
  lastNumber Int    @default(0)

  @@unique([businessId, year])
  @@map("sale_counters")
}
```

> **Correction (migration `20261005043715_sale_number_unique_per_business`)** :
> `saleNumber` n'est plus `@unique` globalement mais
> `@@unique([businessId, saleNumber])`. Avec un compteur par business, deux
> commerces produisent chacun leur `SALE-2026-00001` : l'ancienne contrainte
> globale aurait fait échouer la deuxième vente numéro 1 de l'année.

### Numérotation séquentielle — `upsert` atomique

```typescript
async generate(tx: Prisma.TransactionClient, businessId: string): Promise<string> {
  const year = new Date().getFullYear();
  const counter = await tx.saleCounter.upsert({
    where: { businessId_year: { businessId, year } },
    create: { businessId, year, lastNumber: 1 },
    update: { lastNumber: { increment: 1 } },
  });
  return `SALE-${year}-${String(counter.lastNumber).padStart(5, '0')}`;
}
```

`INSERT ... ON CONFLICT DO UPDATE` : atomique, unique sous concurrence.
**Toujours appelé dans la même transaction** que la création de la
ressource numérotée. Réutilisé tel quel par `InvoiceNumberService`.

### Règles métier — Vente

- Plusieurs `SaleItem`, éventuellement de **propriétaires différents**
- **Vendeur ≠ Propriétaire** : un `REVENDEUR` peut vendre le stock d'un
  autre membre
- `SaleItem` **snapshotte** les prix : modifier le pricing ensuite ne
  change jamais l'historique
- Commission : `max(0, unitPrice - resellerPrice) * quantity`
- Débit immédiat du stock + `StockMovement SOLD`, **atomique** : une seule
  ligne invalide rejette toute la vente
- `LIVREUR` exclu

### Routes

`POST /sales`, `GET /sales`, `GET /sales/:id`

### Visibilité `GET /sales`

| Rôle | Visibilité |
| --- | --- |
| OWNER / ADMIN / STAFF | Toutes les ventes du business |
| REVENDEUR | Ses ventes **OU** ventes contenant au moins une ligne de son stock |
| LIVREUR | Aucun accès |

Pattern `buildVisibilityFilter` (`OR: [{sellerMemberId}, {items: {some:
{ownerMemberId}}}]`), réutilisé pour les paiements et les factures.

---

# Étape 12 — Module Finance : Payment + Cash (Phase 3A)

## Objectif

Séparer **vente**, **paiement** et **caisse**. Une vente peut être non
payée, partiellement payée ou totalement payée.

## Schéma Prisma

```prisma
enum PaymentMethod { CASH  MOBILE_MONEY  BANK_TRANSFER  OTHER }

model Payment {
  id                 String        @id @default(cuid())
  businessId         String
  saleId             String
  amount             Int
  method             PaymentMethod @default(CASH)
  note               String?
  recordedByMemberId String
  createdAt          DateTime      @default(now())

  @@index([businessId])
  @@index([saleId])
  @@map("payments")
}

model CashAccount {
  id         String @id @default(cuid())
  businessId String @unique
  @@map("cash_accounts")
}

enum CashMovementType { SALE_PAYMENT  EXPENSE  SETTLEMENT  MANUAL_IN  MANUAL_OUT }
enum CashDirection    { IN  OUT }

model CashMovement {
  id                  String           @id @default(cuid())
  businessId          String
  cashAccountId       String
  type                CashMovementType
  direction           CashDirection
  amount              Int
  note                String?
  performedByMemberId String
  paymentId           String?          @unique
  createdAt           DateTime         @default(now())

  @@index([businessId])
  @@index([cashAccountId])
  @@map("cash_movements")
}
```

- `Payment` et `CashMovement` sont **immuables** (ni PATCH, ni DELETE)
- **Une caisse par business**, créée paresseusement (`upsert`) au premier
  mouvement
- **Solde jamais stocké** : `SUM(IN) - SUM(OUT)` à la demande
- `MANUAL_IN` / `MANUAL_OUT` : prévus, aucune route ne les crée encore

## Décision critique — sur-paiement concurrent

Verrou pessimiste `SELECT ... FOR UPDATE` sur la ligne `sales`, dans la
transaction, **avant** l'agrégation des paiements existants. Deux
paiements simultanés sur la même vente sont sérialisés ; deux ventes
différentes ne se bloquent jamais. Ce pattern est réutilisé pour le
paiement des dépenses, les reversements et les abonnements.

## Règles métier

- `amount` entier strictement positif, jamais au-delà de `totalAmount`
- Vente hors business → **404**
- `CASH` et `OTHER` → `CashMovement SALE_PAYMENT IN`
- `MOBILE_MONEY` et `BANK_TRANSFER` → **aucun** mouvement de caisse
- Un `Payment` génère au plus un `CashMovement` (`paymentId @unique`)

## Routes

| Route | Rôles |
| --- | --- |
| `POST /finance/payments` | OWNER, ADMIN |
| `GET /finance/payments` | OWNER, ADMIN, REVENDEUR (filtré) |
| `GET /finance/payments/by-method` | OWNER, ADMIN — **ajout** : encaissements agrégés par méthode sur une période |
| `GET /finance/cash-movements` | OWNER, ADMIN |
| `GET /finance/cash-balance` | OWNER, ADMIN |
| `GET /finance/cash-flow` | OWNER, ADMIN — **ajout** : entrées/sorties par type sur une période (`totalIn`, `totalOut`, `netFlow`, `byType`) |

## Incidents résolus

- **`finance.module.ts` dupliqué** : import obsolète dans `app.module.ts`,
  détecté par `tsc --noEmit`, aligné sur `@modules/finance/finance.module`.
- **Throttle `/auth/login` en E2E** : les tests signent les JWT
  directement via `JwtService.signAsync(...)`. Pattern de référence pour
  tout E2E multi-identités.
- **Fragilité SMTP en E2E** (non traitée) : mocker `MailerService` dans les
  tests qui n'ont pas besoin d'un envoi réel.

---

# Ajout — `POST /auth/switch-business`

`AuthService.login()` ne renvoie que le **premier** membership
(`orderBy: createdAt asc`). `POST /auth/switch-business` échange le token
courant contre un token scopé sur le business cible, sans re-login, via
`issueTokensForMembership()`.

- **Body** : `{ "businessId": "clx..." }`
- **200** : `AuthResponseDto` (nouveaux tokens + `user` + `business`)
- **404** `AUTH_NO_BUSINESS_ASSOCIATED` : non membre actif ou inexistant

Complément : `GET /business/mine` (exempté de contexte business) liste les
business de l'utilisateur pour alimenter le sélecteur.

---

# Étape 13 — Module Invoice (Phase 2C)

## Décisions

1. **Relation 1–1 avec `Sale`** (`saleId @unique`)
2. **Aucun montant dupliqué** : `Invoice` lit les montants de `Sale`, tant
   que `Sale` reste immuable
3. Traçabilité via `issuedByMemberId`
4. **Pas de statut**, pas de `note`, pas de `updatedAt` : émise et figée
5. **Création manuelle** (`POST /sales/:id/invoice`)
6. **PDF hors périmètre**

## Schéma Prisma

```prisma
model Invoice {
  id               String   @id @default(cuid())
  businessId       String
  saleId           String   @unique
  invoiceNumber    String
  issuedByMemberId String
  createdAt        DateTime @default(now())

  @@unique([businessId, invoiceNumber])   // ← unique PAR business
  @@index([businessId])
  @@map("invoices")
}

model InvoiceCounter {
  id         String @id @default(cuid())
  businessId String
  year       Int
  lastNumber Int    @default(0)

  @@unique([businessId, year])
  @@map("invoice_counters")
}
```

> **Correction** : `invoiceNumber` est unique **par business**
> (migrations `fix_invoice_number_unique_per_business` v1 et v2), pas
> globalement.

## Règles

- Vente hors business → **404** (`INVENTORY_SALE_NOT_FOUND`)
- REVENDEUR : même visibilité que `sales.findOne()`, sinon **404**
- Facture déjà existante → **409** (`INVOICE_ALREADY_EXISTS`)
- Numéro + facture dans **une seule transaction**, format `INV-YYYY-00001`

## Routes

`POST /sales/:id/invoice`, `GET /invoices`, `GET /invoices/:id` — OWNER,
ADMIN, STAFF, REVENDEUR (filtré).

Rattaché à `InventoryModule` : la facture découle de la vente.

## Reste à faire

- Génération PDF

---

# Étape 14 — Finance : Dépenses (Phase 3B)

## Schéma Prisma

```prisma
enum ExpenseCategory {
  SUPPLIER_PURCHASE
  TRANSPORT
  DELIVERY
  ADVERTISING
  RENT
  OTHER
}

enum ExpenseStatus { PENDING  PAID }

model Expense {
  id                 String          @id @default(cuid())
  businessId         String
  category           ExpenseCategory
  status             ExpenseStatus   @default(PENDING)
  amount             Int
  note               String?
  recordedByMemberId String
  cashMovementId     String?         @unique
  createdAt          DateTime        @default(now())
  paidAt             DateTime?

  @@index([businessId])
  @@index([status])
  @@map("expenses")
}
```

## Règles métier

- Une dépense naît **`PENDING`** : aucun impact sur la caisse
- `POST /finance/expenses/:id/pay` : verrou `FOR UPDATE` sur la dépense,
  puis `CashMovement EXPENSE OUT`, `status = PAID`, `paidAt = now()`, le
  tout atomique
- Déjà payée → **409** `EXPENSE_ALREADY_PAID` ; hors business → **404**

## Routes (OWNER, ADMIN)

| Route | Description |
| --- | --- |
| `POST /finance/expenses` | Créer (catégorie, montant, note) |
| `GET /finance/expenses` | Filtres `category`, `status`, `from`, `to` |
| `GET /finance/expenses/:id` | Détail |
| `POST /finance/expenses/:id/pay` | Payer (sortie de caisse) |
| `GET /finance/expenses/by-category` | Total, payé, à payer par catégorie sur une période (les 6 catégories sont toujours renvoyées) |

---

# Étape 15 — Finance : Reversements aux propriétaires (Phase 3D)

## Objectif

Payer ce qui est dû aux membres dont le stock a été vendu (revendeurs).

## Schéma Prisma

```prisma
model Settlement {
  id                  String   @id @default(cuid())
  businessId          String
  ownerMemberId       String
  amount              Int
  note                String?
  performedByMemberId String
  cashMovementId      String?  @unique
  createdAt           DateTime @default(now())

  @@index([businessId])
  @@index([ownerMemberId])
  @@map("settlements")
}
```

## Calcul du solde dû

```
totalGenerated = Σ (unitPrice × quantity − commission) des SaleItem du propriétaire
totalSettled   = Σ Settlement.amount du propriétaire
balanceDue     = totalGenerated − totalSettled
```

## Règles métier

- Le propriétaire doit être un membre **actif** du business, sinon 404
- Verrou `FOR UPDATE` sur la ligne `business_members` du propriétaire :
  deux reversements simultanés au même membre sont sérialisés
- Montant > `balanceDue` → **400** `SETTLEMENT_EXCEEDS_BALANCE`
- Reversement partiel autorisé
- Chaque reversement crée un `CashMovement SETTLEMENT OUT`
- REVENDEUR : ne voit que **ses** reversements et **son** solde (le filtre
  `ownerMemberId` envoyé est ignoré) ; le reversement d'un autre → 404

## Routes

| Route | Rôles |
| --- | --- |
| `POST /finance/settlements` | OWNER, ADMIN |
| `GET /finance/settlements/balance` | OWNER, ADMIN, REVENDEUR (le sien) |
| `GET /finance/settlements` | OWNER, ADMIN, REVENDEUR (les siens) |
| `GET /finance/settlements/:id` | OWNER, ADMIN, REVENDEUR (le sien) |

---

# Étape 16 — Finance : Analytique (Phase 3C)

Toutes ces routes sont **en lecture seule**, réservées à **OWNER et ADMIN**,
et prennent une période `from` / `to`.

| Route | Renvoie |
| --- | --- |
| `GET /finance/summary` | Ventes (nombre, total, panier moyen, commissions), encaissements, dépenses (total, payé, à payer) |
| `GET /finance/timeline` | Séries `sales`, `payments`, `expenses`, `cashFlow` ; paramètre `granularity` = `day` (défaut), `week`, `month`, `year` (via `date_trunc`) |
| `GET /finance/sales-stats` | Par vendeur : nombre de ventes, total, panier moyen |
| `GET /finance/commissions` | Total, par vendeur, par propriétaire de stock |
| `GET /finance/outstanding` | Ventes non soldées : `totalOutstanding`, `unpaidSalesCount`, détail par vente |
| `GET /finance/payments/by-method` | Encaissements par méthode |
| `GET /finance/cash-flow` | Entrées/sorties de caisse par type |
| `GET /finance/expenses/by-category` | Dépenses par catégorie |

Notes :

- Les périodes sans activité ne sont **pas** renvoyées dans la timeline :
  le frontend doit compléter les trous (0) pour ses graphiques.
- Dans `summary`, les dépenses payées sont comptées sur `paidAt`, les
  autres sur `createdAt`.
- `THROTTLE_LIMIT` en test relevé à 100 pour ces suites.

---

# Étape 17 — Abonnements SaaS + paiement CinetPay

## Modèle économique

- L'unité facturable est le **Business**, jamais le User
- Chaque Business naît avec un abonnement **TRIAL de 30 jours**
  (`TRIAL_PERIOD_DAYS`), qu'il soit créé à l'inscription ou via
  `POST /business` (`provisionBusinessWithOwner`)
- Plans payants (`subscription-plans.ts`, gelés à l'exécution) :

| Plan | Montant | Durée |
| --- | --- | --- |
| `MONTHLY` | 10 000 XAF | 1 mois |
| `ANNUAL` | 100 000 XAF | 12 mois |

Devise de facturation fixe `XAF`, indépendante de `Business.currency`.

## Schéma Prisma

```prisma
enum SubscriptionPlan   { TRIAL  MONTHLY  ANNUAL }
enum SubscriptionStatus { TRIAL  ACTIVE  EXPIRED  CANCELED }

model Subscription {
  id                 String             @id @default(cuid())
  plan               SubscriptionPlan   @default(TRIAL)
  status             SubscriptionStatus @default(TRIAL)
  trialEndsAt        DateTime?
  currentPeriodStart DateTime?
  currentPeriodEnd   DateTime?
  canceledAt         DateTime?
  businessId         String             @unique
  createdAt          DateTime           @default(now())
  updatedAt          DateTime           @updatedAt
  @@map("subscriptions")
}

enum SubscriptionPaymentStatus  { PENDING  SUCCEEDED  FAILED  CANCELED }
enum SubscriptionPaymentChannel { OM_CM  MTN_CM }

model SubscriptionPayment {
  id                    String                      @id @default(cuid())
  businessId            String
  subscriptionId        String
  plan                  SubscriptionPlan
  amount                Int
  currency              String
  merchantTransactionId String                      @unique   // clé d'idempotence
  providerTransactionId String?
  notifyTokenHash       String?                     // SHA-256, jamais le jeton en clair
  channel               SubscriptionPaymentChannel?
  status                SubscriptionPaymentStatus   @default(PENDING)
  periodStart           DateTime?
  periodEnd             DateTime?
  paidAt                DateTime?
  rawNotification       Json?                       // jamais exposé au frontend
  initiatedByMemberId   String
  createdAt             DateTime                    @default(now())
  @@index([businessId])
  @@index([subscriptionId])
  @@map("subscription_payments")
}
```

`SubscriptionPayment` est distinct de `Payment` (encaissement d'une vente).

## État d'accès (`subscription-access.ts`)

Fonction pure `getAccessState(subscription, now)`, calculée à la lecture
(aucun job ne réécrit les statuts) :

| Situation | `hasAccess` | `reason` |
| --- | --- | --- |
| TRIAL et `trialEndsAt > now` | ✅ | `TRIAL_RUNNING` |
| TRIAL échu | ❌ | `TRIAL_ENDED` |
| ACTIVE et `currentPeriodEnd > now` | ✅ | `PERIOD_RUNNING` |
| ACTIVE échu | ❌ | `PERIOD_ENDED` |
| EXPIRED ou CANCELED | ❌ | `STATUS_NO_ACCESS` |
| Aucun abonnement | ❌ | `NO_SUBSCRIPTION` |

Fail-closed : à l'instant exact de l'échéance, l'accès est déjà refusé.

> ⚠️ **Non appliqué aux modules** : aucun guard ne bloque encore les routes
> métier quand `hasAccess = false` (point P1-3 de l'audit). L'état est
> seulement exposé par `GET /subscription`.

## Activation d'une période (`activatePeriod`)

- Durée lue **exclusivement** dans `PLANS[plan]`
- Période payée encore en cours → **prolongée** depuis son échéance
- Sinon (trial, échue, expirée, annulée) → démarre à `now` ; les jours
  d'essai restants ne s'ajoutent pas
- Résultat : `ACTIVE`, `plan` payé, `canceledAt = null`, `trialEndsAt`
  conservé

## Flux de paiement

```
Frontend ──POST /subscription/checkout {plan}──► API
API : 1. crée SubscriptionPayment PENDING (montant/devise/ID côté serveur)
      2. POST CinetPay /v1/payment  (OAuth, IPv4, timeout 15 s, 3 retries 5xx/réseau)
      3. stocke providerTransactionId + SHA-256(notify_token)
API ──{paymentId, merchantTransactionId, paymentUrl}──► Frontend ──redirect──► CinetPay

CinetPay ──POST /subscription/payments/cinetpay/webhook──► API
API : 1. retrouve le paiement, compare notify_token à l'empreinte (401 sinon)
      2. relit le statut via GET CinetPay /v1/payment/{id}  (8 s max)
      3. finalisation atomique (FOR UPDATE) : montant + devise vérifiés,
         SUCCEEDED → activatePeriod ; échec définitif → FAILED
```

- Le contenu de la notification n'est **jamais** une preuve : seul le
  statut relu chez CinetPay fait foi
- Webhook public, `@SkipThrottle()`, aucun log du corps
- Codes : 200 (traité ou non payé), 400 (forme), 401 (jeton), 409
  (montant/devise différents → revue manuelle), 502 (réponse fournisseur
  inexploitable), 503 (CinetPay injoignable ou transaction annulée →
  CinetPay rejoue)
- Seuls SUCCESS (100) et FAILED (2010) sont **définitifs** ; tout code
  inconnu est traité comme non finalisable (fail-closed)
- Canaux reconnus : `OM_CM`, `MTN_CM` (alias `MOMO_CM` renvoyé en sandbox)

## Réconciliation

`SubscriptionPaymentReconciliationService` revérifie les paiements
`PENDING` créés il y a **5 min à 24 h** (50 par passage, séquentiel, arrêt
si CinetPay est indisponible), avec la même finalisation que le webhook.

> ⚠️ **Non déclenché** : aucun cron ni endpoint ne l'appelle encore (point
> P1-1). Un paiement dont le webhook se perd reste `PENDING`.

## Routes

| Route | Rôles | Description |
| --- | --- | --- |
| `GET /subscription` | OWNER, ADMIN | `{ subscription, access }` du business courant |
| `POST /subscription/checkout` | OWNER, ADMIN | Body `{ "plan": "MONTHLY" \| "ANNUAL" }` → `{ paymentId, merchantTransactionId, paymentUrl }` |
| `POST /subscription/payments/cinetpay/webhook` | Public (CinetPay) | Notification de paiement |

URL de retour après paiement (succès comme échec) :
`{FRONTEND_URL}/abonnement/paiement/retour`.

**Règle frontend** : la page de retour n'est jamais une preuve de paiement.
Elle interroge `GET /subscription` jusqu'à ce que l'accès change.

## Validation en production (sandbox)

16/10/2026 : checkout, paiement Orange Money (numéro de test `699990700`),
webhook reçu sur Render, abonnement passé `TRIAL` → `ACTIVE`/`MONTHLY` en
quelques secondes. Un paiement échoué a laissé l'abonnement en `TRIAL`.

---

# Étape 18 — Mise en production (Render)

| Élément | Valeur |
| --- | --- |
| API | Web Service `alibia-api-eu`, Frankfurt, Starter |
| Base | Postgres `alibia-db-eu`, PostgreSQL 16, Frankfurt, plan payant, 1 GB |
| Build | `npm ci --include=dev && npm run build` |
| Pre-deploy | `npx prisma migrate deploy` |
| Start | `npm run start:prod` |
| Health check | `/api/v1/health` |
| Auto-deploy | Off (déploiement manuel) |
| IP sortante | `74.220.51.140` (plage `74.220.51.0/24`), whitelistée chez CinetPay |

Correctifs appliqués pour la production (commit `d9dd785`) :

- `trust proxy` : vraie IP client derrière le proxy Render (rate limiting
  par client)
- Une seule configuration CORS (`FRONTEND_URLS` prioritaire sur
  `FRONTEND_URL`)
- `/health` exempté du rate limiting
- `enableShutdownHooks()` : arrêt propre au redéploiement
- Tests exclus du build de production
- `.env.example` nettoyé, artefacts retirés du suivi Git

Swagger : `/docs`, désactivé en production sauf `SWAGGER_ENABLED=true`
avec `SWAGGER_USER` + `SWAGGER_PASSWORD` (Basic Auth).

---

# Récapitulatif global

| Module | Statut |
| --- | --- |
| Auth (JWT, OTP email, 2FA, lockout, mot de passe oublié) | ✅ |
| Business (multi-business, invitations, rôles, membres) | ✅ |
| Users (profil, mot de passe) | ✅ |
| `POST /auth/switch-business` | ✅ |
| Inventory — Produits / Localisations / Stock | ✅ (écart LIVREUR sur produits) |
| Sales — Pricing + Ventes | ✅ |
| Invoice | ✅ (PDF à faire) |
| Finance — Payment + Cash | ✅ |
| Finance — Dépenses (3B) | ✅ |
| Finance — Analytique (3C) | ✅ |
| Finance — Reversements (3D) | ✅ |
| Abonnements + CinetPay | ✅ validé en sandbox production |
| Réconciliation automatique des paiements | ⏳ service prêt, non déclenché |
| Blocage d'accès si abonnement expiré | ⏳ à faire |
| CinetPay live | ⏳ NIF/NIU + validation CinetPay |
| Frontend / landing page | ⏳ à faire |

Tests : **619 tests unitaires** (20 suites) au vert ; **59 suites E2E**
(non relancées lors de cette mise à jour, Postgres de test requis).
