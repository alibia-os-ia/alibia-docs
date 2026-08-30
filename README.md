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
  (pas à un revendeur en particulier)
- `ownerMemberId` renseigné = stock détenu par un `BusinessMember` précis
  (typiquement un `REVENDEUR`)
- Isolation multi-tenant stricte : tout accès filtre systématiquement par
  `businessId`, jamais par confiance dans l'ID envoyé par le client

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
  RETURNED
  DEFECTIVE
  RETRIEVED
}

model StockMovement {
  id                   String            @id @default(cuid())
  businessId           String
  stockId              String
  type                 StockMovementType
  quantity             Int
  note                 String?
  performedByMemberId  String
  relatedStockId       String?
  createdAt            DateTime          @default(now())

  @@index([businessId])
  @@index([stockId])
  @@map("stock_movements")
}
```

**Note sur `@@unique([productId, locationId, ownerMemberId])`** : PostgreSQL
traite `NULL` comme distinct de toute autre valeur dans une contrainte
unique classique. Un index unique **partiel** a donc été nécessaire pour
garantir qu'il n'existe qu'**une seule** ligne de stock boutique (
`ownerMemberId IS NULL`) par `(productId, locationId)` — sans quoi rien
n'empêchait la création de plusieurs lignes de stock boutique identiques.

## Fichiers créés / importants

| Fichier | Rôle |
| --- | --- |
| `src/modules/inventory/inventory.module.ts` | Regroupe Products, Locations, Stocks, Sales, Invoices |
| `src/modules/inventory/products/products.controller.ts` / `.service.ts` | CRUD produits |
| `src/modules/inventory/locations.controller.ts` / `.service.ts` | CRUD localisations (OWNER/ADMIN pour create/update) |
| `src/modules/inventory/stocks/stocks.controller.ts` / `.service.ts` | Création, lecture, ajustement, transfert, retrait, tarification du stock |
| `src/modules/inventory/constants/inventory-messages.ts` | i18n Inventory (toutes les clés `INVENTORY_*`) |
| `src/common/services/member-resolver.service.ts` (dans `CommonModule`) | `User.id` → `BusinessMember.id`, partagé Inventory/Finance |

## Règles métier — Stock

- `LIVREUR` n'a **aucun accès** à l'inventaire (ni lecture, ni écriture)
- `OWNER` / `ADMIN` / `STAFF` peuvent gérer tout le stock du business
- `REVENDEUR` ne peut modifier **que son propre stock** (`ownerMemberId ===
  callerMemberId`) ; à la création, il ne peut créer du stock qu'à son
  propre nom (`INVENTORY_STOCK_OWNER_FORBIDDEN` sinon)
- Cross-business → toujours **404** (jamais 403, pour ne pas révéler
  l'existence de la ressource)
- Action interdite dans le business (rôle non autorisé) → **403**
- **Toute** modification de quantité crée un `StockMovement` correspondant,
  dans la **même transaction** que la mutation du `Stock` — traçabilité
  garantie, jamais de mouvement orphelin ou de stock modifié sans trace
- Stock négatif **interdit** structurellement (vérifié avant chaque
  décrément, y compris sous concurrence via relecture dans la transaction)
- Trois concepts **distincts** à ne jamais confondre : `seller` (qui
  effectue la vente), `owner` (à qui appartient le stock vendu), `creator`
  (qui a créé la fiche produit)

## Actions de stock implémentées

| Action | Description |
| --- | --- |
| `POST /stocks` | Création (quantité initiale, tarification optionnelle) |
| `GET /stocks`, `GET /stocks/:id` | Lecture avec filtres combinables, lecture totale pour revendeur sur son propre stock |
| `POST /stocks/:id/adjust` | Ajustement signé (+/-), transaction atomique + `StockMovement ADJUSTED` |
| `POST /stocks/:id/transfer` | Transfert entre localisations — mouvements liés (`TRANSFERRED_OUT` + `TRANSFERRED_IN`), **jamais de fusion** entre stocks de propriétaires différents |
| `POST /stocks/:id/retrieve` | Retrait de stock (`RETRIEVED`) — factorisé avec `mark-defective` via une méthode commune `applyWithdrawal` |
| `POST /stocks/:id/mark-defective` | Marque une quantité comme défectueuse (`DEFECTIVE`) |
| `PATCH /stocks/:id/pricing` | Tarification (`supplierPrice`, `resellerPrice`, `publicPrice`, `minPrice`) — règle `minPrice <= resellerPrice` validée au niveau service |

## Tests (état après cette étape)

**178/178 tests E2E, 29 suites, 0 régression** (Phase 1 validée avant le
début de ce document de suivi détaillé — inclut Auth + Business + Users +
Inventory Phase 1 complet).

---

# Étape 11 — Module Sales : Pricing + Ventes (Phase 2A + 2B)

## Objectif

Permettre la vente de stock avec calcul automatique de commission,
snapshot immuable des prix au moment de la vente, et numérotation
séquentielle des ventes par business.

## Phase 2A — Pricing

Les prix vivent directement sur `Stock` (voir schéma Étape 10 :
`supplierPrice`, `resellerPrice`, `publicPrice`, `minPrice`), donc
naturellement propres à chaque triplet produit + localisation +
propriétaire. Configurés séparément de la création via
`PATCH /stocks/:id/pricing`, nullable au départ.

**Règles** :
- `minPrice <= resellerPrice`, validé au niveau service
- Vente impossible tant que `resellerPrice` ou `minPrice` manque sur le
  stock concerné
- Prix de vente effectif toujours `>= minPrice`
- Commission jamais négative (`Math.max(0, ...)`)

## Phase 2B — Sales

### Schéma Prisma

```prisma
model Sale {
  id              String   @id @default(cuid())
  businessId      String
  saleNumber      String   @unique
  sellerMemberId  String
  clientName      String?
  clientPhone     String?
  totalAmount     Int
  totalCommission Int
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt

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

### Fichiers créés / importants

| Fichier | Rôle |
| --- | --- |
| `src/modules/inventory/sales/sales.controller.ts` | `POST /sales`, `GET /sales`, `GET /sales/:id` |
| `src/modules/inventory/sales/sales.service.ts` | Logique de vente + visibilité |
| `src/modules/inventory/sales/sale-number.service.ts` | Génération atomique de `saleNumber` |
| `src/modules/inventory/sales/dto/create-sale.dto.ts`, `find-sales-query.dto.ts` | Validation |

### Numérotation séquentielle — pattern `upsert` atomique

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

`upsert` compile en `INSERT ... ON CONFLICT DO UPDATE` côté PostgreSQL —
atomique par nature, garantit l'unicité même sous créations concurrentes
sur le même business. **Doit toujours être appelé à l'intérieur de la même
transaction** que la création de la ressource numérotée (`Sale`, plus tard
`Invoice`), pour qu'un numéro généré ne soit jamais "perdu" si la création
échoue ensuite. Ce pattern a été **réutilisé tel quel** pour
`InvoiceNumberService` en Phase 2C (format `INV-YYYY-00001`).

### Règles métier — Vente

- Une vente peut contenir **plusieurs `SaleItem`**, potentiellement de
  **propriétaires différents** en une seule transaction
- **Vendeur ≠ Propriétaire** : un `REVENDEUR` peut vendre le stock d'un
  autre membre (le vendeur encaisse la vente, mais la commission et la
  traçabilité respectent le vrai propriétaire du stock)
- `SaleItem` **snapshotte** `unitPrice`, `resellerPrice`, `minPrice`,
  `commission` au moment de la vente — une modification ultérieure du
  pricing du `Stock` **ne modifie jamais l'historique** déjà créé
- Commission : `max(0, unitPrice - resellerPrice) * quantity`
- Une vente **débite immédiatement** le `Stock` et crée un
  `StockMovement` de type `SOLD`, le tout **atomique** : si une seule
  ligne ne peut pas être vendue (stock insuffisant, pricing manquant, prix
  sous le minimum), **toute la vente est rejetée**, aucune écriture
  partielle
- `LIVREUR` exclu de toute vente

### Visibilité `GET /sales`

| Rôle | Visibilité |
| --- | --- |
| OWNER / ADMIN / STAFF | Toutes les ventes du business |
| REVENDEUR | Ses propres ventes (vendeur) **OU** ventes effectuées par d'autres sur son propre stock (propriétaire d'au moins une ligne) |
| LIVREUR | Aucun accès |

Ce pattern de visibilité (`buildVisibilityFilter`, basé sur `OR:
[{sellerMemberId}, {items: {some: {ownerMemberId}}}]`) a servi de modèle
direct pour la visibilité de `GET /finance/payments` (Étape 12) et
`GET /invoices` (Étape 13).

---

# Étape 12 — Module Finance : Payment + CashAccount + CashMovement (Phase 3A)

## Objectif

Séparer strictement **vente** (`Sale`), **paiement** (`Payment`) et
**caisse** (`CashAccount`/`CashMovement`). Créer une vente ne signifie
**jamais** automatiquement que l'argent est encaissé — une vente peut être
non payée, partiellement payée, ou totalement payée, les paiements étant
enregistrés séparément.

## Schéma Prisma

```prisma
enum PaymentMethod {
  CASH
  MOBILE_MONEY
  BANK_TRANSFER
  OTHER
}

model Payment {
  id                  String        @id @default(cuid())
  businessId          String
  saleId              String
  amount              Int
  method              PaymentMethod @default(CASH)
  note                String?
  recordedByMemberId  String
  createdAt           DateTime      @default(now())

  @@index([businessId])
  @@index([saleId])
  @@map("payments")
}

model CashAccount {
  id         String @id @default(cuid())
  businessId String @unique

  @@map("cash_accounts")
}

enum CashMovementType {
  SALE_PAYMENT
  EXPENSE
  SETTLEMENT
  MANUAL_IN
  MANUAL_OUT
}

enum CashDirection {
  IN
  OUT
}

model CashMovement {
  id                   String           @id @default(cuid())
  businessId           String
  cashAccountId        String
  type                 CashMovementType
  direction            CashDirection
  amount               Int
  note                 String?
  performedByMemberId  String
  paymentId            String?          @unique
  createdAt            DateTime         @default(now())

  @@index([businessId])
  @@index([cashAccountId])
  @@map("cash_movements")
}
```

**Payment est immuable** : pas de `updatedAt`, pas de `PATCH`, pas de
`DELETE`. **CashMovement est immuable** de la même façon. **Une seule
caisse par Business** (`CashAccount.businessId @unique`), créée
**paresseusement** via `upsert` au premier mouvement financier qui
l'exige — jamais à l'inscription (`AuthService.register()` volontairement
non modifié pour ça).

**Solde de caisse jamais stocké** — toujours recalculé à la demande :
`SUM(direction=IN) - SUM(direction=OUT)`.

## Fichiers créés / importants

| Fichier | Rôle |
| --- | --- |
| `src/modules/finance/finance.module.ts` | Regroupe Payments + Cash |
| `src/modules/finance/payments/payments.controller.ts` / `.service.ts` | `POST /finance/payments`, `GET /finance/payments` |
| `src/modules/finance/cash/cash.controller.ts` / `.service.ts` | `GET /finance/cash-movements`, `GET /finance/cash-balance` |
| `src/modules/finance/constants/finance-messages.ts` | i18n Finance |
| `src/modules/finance/payments/dto/create-payment.dto.ts`, `find-payments-query.dto.ts` | Validation |
| `src/modules/finance/cash/dto/find-cash-movements-query.dto.ts` | Validation filtre cash |

## Décision critique — race condition sur le contrôle de sur-paiement

**Problème identifié** : le contrôle initial (`aggregate` des paiements
existants, comparaison avec `Sale.totalAmount`, **hors transaction**) est
vulnérable à une race condition : deux requêtes HTTP concurrentes sur la
même vente peuvent toutes deux lire le même `alreadyPaid`, passer le
contrôle, et ensemble dépasser le total autorisé.

**Solution retenue** : verrou pessimiste `SELECT ... FOR UPDATE` sur la
ligne `Sale`, **à l'intérieur de la transaction Prisma**, avant
l'agrégation des paiements existants.

```typescript
return this.prisma.$transaction(async (tx) => {
  const lockedSales = await tx.$queryRaw<LockedSaleRow[]>`
    SELECT id, "businessId", "totalAmount"
    FROM sales
    WHERE id = ${dto.saleId}
    FOR UPDATE
  `;
  const sale = lockedSales[0];
  // ... vérifications ...

  const existingPayments = await tx.payment.aggregate({
    where: { saleId: dto.saleId },
    _sum: { amount: true },
  });
  const alreadyPaid = existingPayments._sum.amount ?? 0;

  if (alreadyPaid + dto.amount > sale.totalAmount) {
    throw new BadRequestException(/* ... */);
  }
  // ... création du Payment + CashMovement ...
});
```

`SELECT ... FOR UPDATE` verrouille la ligne `sales` correspondante jusqu'à
la fin de la transaction. Une deuxième requête concurrente sur la **même**
vente attend que la première commit avant de lire `totalAmount` — elle
voit donc forcément l'`alreadyPaid` à jour, incluant le paiement de la
première. Deux ventes différentes ne se bloquent jamais entre elles
(verrous sur des lignes distinctes).

## Règles métier

- `saleId` obligatoire, `amount` strictement positif (`@IsInt`,
  `@IsPositive` sur le DTO)
- `amount` ne peut jamais faire dépasser `Sale.totalAmount` (contrôle
  concurrency-safe ci-dessus)
- La vente doit appartenir au business courant → sinon **404**
- `CASH` et `OTHER` → créent un `CashMovement` `SALE_PAYMENT` `IN`
- `MOBILE_MONEY` et `BANK_TRANSFER` → **aucun** `CashMovement` (n'impactent
  pas la caisse physique)
- `CashMovement.paymentId` unique — un `Payment` ne peut jamais générer
  plus d'un `CashMovement`

## Permissions Finance

| Endpoint | Rôles autorisés |
| --- | --- |
| `POST /finance/payments` | OWNER, ADMIN uniquement |
| `GET /finance/payments` | OWNER, ADMIN, REVENDEUR (visibilité filtrée, mêmes règles que `GET /sales`) |
| `GET /finance/cash-movements` | OWNER, ADMIN uniquement |
| `GET /finance/cash-balance` | OWNER, ADMIN uniquement |
| STAFF | Aucun accès Finance |
| LIVREUR | Aucun accès Finance |

## Incident résolu — fichier `finance.module.ts` dupliqué

Une erreur de manipulation a laissé deux copies du module (
`src/modules/finance/payments/finance.module.ts` et
`src/modules/finance/finance.module.ts`), l'ancienne référençant un import
`app.module.ts` obsolète (`@modules/finance/payments/finance.module` au
lieu de `@modules/finance/finance.module`). `tsc --noEmit` a immédiatement
révélé l'incohérence (`Cannot find module`) après suppression du doublon —
corrigé en alignant l'import sur le bon chemin.

## Incident résolu — throttle `/auth/login` dans les tests E2E

`/auth/login` est limité à **5 requêtes / 60s** (`@Throttle({ default: {
ttl: 60000, limit: 5 } })`). Les premiers jets des fichiers E2E Finance
enchaînaient jusqu'à 6 logins réels dans un seul `beforeAll`, provoquant
des `429 Too Many Requests` en cascade sur tous les tests du fichier.

**Solution adoptée** : signer les JWT de test **directement** via
`JwtService.signAsync(...)` (récupéré depuis le `TestingModule`), sans
jamais passer par l'endpoint `/auth/login` réel. `JwtStrategy.validate()`
ne vérifie que `user.isActive` en base, jamais le mot de passe — donc
parfaitement légitime pour des tests qui exercent la logique métier
Finance, pas le flow d'authentification (déjà couvert par ses propres
suites `auth-login*`). Ce pattern (`signToken` local) est désormais la
référence pour tout futur fichier E2E nécessitant plusieurs identités.

## Tests (état après cette étape)

| Fichier | Tests | Statut |
| --- | --- | --- |
| `finance-payments-create.e2e-spec.ts` | 13 | ✅ |
| `finance-payments-cash-read.e2e-spec.ts` | 14 | ✅ |

**27 nouveaux tests E2E confirmés**, suite complète re-vérifiée à 0
régression imputable (deux échecs transitoires rencontrés en cours de
route sur `auth-register` et `business-invite-member` — tous deux
confirmés être des envois SMTP réels via Gmail dépassant occasionnellement
le timeout de 20s en usage intensif, reproductibles indépendamment de tout
changement de code, systématiquement verts en isolation).

**Piste d'amélioration identifiée, non traitée** : mocker `MailerService`
dans les tests E2E qui n'ont pas besoin de vérifier l'envoi réel, pour
éliminer cette fragilité structurelle liée à la latence SMTP.

---

# Ajout — `POST /auth/switch-business`

## Contexte

Le module Business notait déjà (Étape 08) l'absence d'un vrai "switch
business" comme limitation connue reportée. Le besoin s'est concrétisé en
pratique : `AuthService.login()` ne retourne toujours que le **premier**
membership créé (`orderBy: createdAt asc, take: 1`), sans aucun moyen pour
un utilisateur multi-business de choisir vers lequel se connecter après un
login classique.

## Décision

Ajout d'un endpoint dédié, réutilisant `issueTokensForMembership()` déjà
existant (initialement écrit pour `acceptInvitation()`) :

```typescript
async switchBusiness(
  userId: string,
  email: string,
  targetBusinessId: string,
  lang: SupportedLang = 'fr',
): Promise<AuthResponseDto> {
  const membership = await this.prisma.businessMember.findUnique({
    where: { userId_businessId: { userId, businessId: targetBusinessId } },
    include: { business: true },
  });

  if (!membership || !membership.isActive) {
    throw new NotFoundException(
      buildMessage(AuthMessageKey.NO_BUSINESS_ASSOCIATED, lang),
    );
  }
  // ... émission des tokens scopés sur targetBusinessId ...
}
```

**Pas de re-login nécessaire** : l'utilisateur reste authentifié avec son
token courant, échange simplement contre un nouveau token scopé sur le
business cible — pattern standard des SaaS multi-tenant (Slack, Notion).

## Route

### `POST /auth/switch-business` (protégé)

- **Guard :** JWT
- **Body :**

```json
{ "businessId": "clx1234..." }
```

- **Réponse 200 :** `AuthResponseDto` (nouveaux `accessToken`/
  `refreshToken` + `user` + `business` cible)
- **404** `AUTH_NO_BUSINESS_ASSOCIATED` — cible inexistante, ou
  l'utilisateur n'en est pas membre actif

## Tests

`auth-switch-business.e2e-spec.ts` — **5/5 ✅** (accès sans token, bascule
réussie avec vérification que le nouveau token est bien scopé via un appel
réel à `GET /business/me`, refus sur business non-membre, refus sur
`businessId` inexistant, refus sur body vide).

---

# Étape 13 — Module Invoice (Phase 2C)

## Objectif

Générer un document de facturation officiel et immuable pour une vente,
strictement dissocié du paiement (une facture peut exister sans qu'aucun
paiement n'ait encore été enregistré).

## Décisions de conception validées

1. **Relation 1–1 stricte avec `Sale`** — `saleId @unique`, imposée au
   niveau base de données. Une vente ne peut jamais avoir plus d'une
   facture.
2. **Aucun montant dupliqué sur `Invoice`** — tant que `Sale` reste
   immuable après création (aucune mutation observée dans
   `sales.service.ts`), `Invoice` référence simplement `Sale` via relation
   Prisma pour accéder à `totalAmount`/`totalCommission`. **Réserve
   explicitement actée** : si `Sale` devient un jour modifiable, cette
   décision devra être revue (snapshot des montants à envisager alors).
3. **Traçabilité** via `issuedByMemberId`, cohérent avec
   `Payment.recordedByMemberId` / `StockMovement.performedByMemberId`.
4. **Pas de statut** (`DRAFT`/`CANCELED`...) — une facture créée est
   immédiatement considérée comme émise et reste figée. Pas de `note`, pas
   de `updatedAt`.
5. **Création manuelle**, jamais automatique à la création de la `Sale` —
   séparation volontaire entre vente et facturation
   (`POST /sales/:id/invoice`, à la demande).
6. **PDF hors périmètre** pour cette phase — uniquement les données
   structurées en base.

## Schéma Prisma

```prisma
model Invoice {
  id                String @id @default(cuid())
  businessId        String
  saleId            String @unique
  invoiceNumber     String @unique
  issuedByMemberId  String
  createdAt         DateTime @default(now())

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

Relations inverses ajoutées sur `Business` (`invoices`,
`invoiceCounters`), `Sale` (`invoice Invoice?`), `BusinessMember`
(`invoicesIssued`).

## Fichiers créés / importants

| Fichier | Rôle |
| --- | --- |
| `src/modules/inventory/invoices/invoices.controller.ts` | `POST /sales/:id/invoice`, `GET /invoices`, `GET /invoices/:id` |
| `src/modules/inventory/invoices/invoices.service.ts` | Logique de création + visibilité |
| `src/modules/inventory/invoices/invoice-number.service.ts` | Génération atomique, calquée sur `SaleNumberService` |
| `src/modules/inventory/invoices/constants/invoice-messages.ts` | i18n Invoice dédié |
| `src/modules/inventory/invoices/dto/find-invoices-query.dto.ts` | Validation |

## Numérotation

Même mécanique `upsert` atomique que `SaleNumberService` (voir Étape 11),
format `INV-YYYY-00001`, compteur indépendant par `businessId` + année via
`InvoiceCounter`.

## Règles métier

- La vente doit exister et appartenir au business courant → sinon **404**
  (`INVENTORY_SALE_NOT_FOUND`, réutilisation de la clé existante)
- Pour un `REVENDEUR` : la vente doit lui être visible selon la **même
  règle** que `sales.findOne()` (vendeur OU propriétaire d'une ligne) →
  sinon **404** (jamais 403, cohérent avec le pattern déjà en place)
- Facture déjà existante pour cette vente → **409**
  (`INVOICE_ALREADY_EXISTS`), avec la contrainte `saleId @unique` comme
  filet de sécurité ultime en base
- Génération du numéro + création de l'`Invoice` dans une **seule
  transaction**
- `issuedByMemberId` = le membre authentifié qui déclenche la création

## Permissions

Mêmes règles de visibilité que `Sale` : OWNER / ADMIN / STAFF / REVENDEUR
(selon les règles de visibilité des ventes), LIVREUR exclu — aussi bien
pour la création (`POST /sales/:id/invoice`) que pour la consultation
(`GET /invoices`, `GET /invoices/:id`).

## Emplacement dans l'architecture

Rattaché à `InventoryModule` (et non `FinanceModule`) : `Invoice` découle
directement de `Sale` (document commercial dérivé de la vente), tandis que
Finance reste dédié à l'argent réellement encaissé (`Payment`/`Cash`).

## Tests

`invoices.e2e-spec.ts` écrit (20 scénarios prévus : création par rôle,
refus LIVREUR, vente inexistante/cross-business, visibilité REVENDEUR,
numérotation séquentielle et compteurs indépendants par business, lecture
liste/détail, isolation cross-business).

**⚠️ Statut d'exécution non confirmé dans cette session de travail** — le
fichier a été écrit et `tsc --noEmit` validé, mais le run final
(`npx jest --config src/tests/jest-e2e.json --runInBand invoices.e2e-spec`)
n'a pas encore été exécuté/rapporté au moment de la rédaction de cette
documentation. À confirmer avant de considérer la Phase 2C comme close.

## Reste à faire sur Invoice

- Confirmer le run complet des 20 tests E2E
- Génération PDF (explicitement reportée, hors périmètre Phase 2C)
- Commit final une fois les tests confirmés verts

---

# Récapitulatif global — état du backend au terme de cette session

| Module | Statut |
| --- | --- |
| Auth (JWT, OTP, 2FA, lockout) | ✅ MVP terminé |
| Business (invitations, rôles, membres) | ✅ MVP terminé |
| Users (profil, mot de passe) | ✅ MVP terminé |
| `POST /auth/switch-business` | ✅ terminé et testé (5/5) |
| Inventory — Produits/Localisations/Stock (Phase 1) | ✅ terminé (178/178 avant cette session) |
| Sales — Pricing + Ventes (Phase 2A/2B) | ✅ terminé |
| Invoice (Phase 2C) | ⏳ implémenté, tests écrits, run final à confirmer |
| Finance — Payment + Cash (Phase 3A) | ✅ terminé et testé (27/27) |
| Finance — Expense / Settlement (Phase 3B) | ⏳ non commencé (explicitement mis en attente) |