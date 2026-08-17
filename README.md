# Alibia API

Documentation officielle d'Alibia, SaaS de gestion de commerce.
Centralise guides, architecture, API et bonnes pratiques.
Aide développeurs et utilisateurs à comprendre et utiliser le système Alibia (web, mobile, API) avec une vision scalable et IA.

**Stack :** NestJS 11 · TypeScript strict · Prisma 6 · PostgreSQL · JWT/Passport · class-validator · Jest/Supertest

---

# Étape 01 — Initialisation du projet NestJS

## Objectif

Créer la base du projet alibia-api avec NestJS CLI de façon propre et professionnelle.

## Prérequis

| Outil      | Version  |
| ---------- | -------- |
| Node.js    | v24.15.0 |
| npm        | v11.15.0 |
| NestJS CLI | v11.0.23 |
| Git        | v2.54.0  |

## Commandes exécutées

```bash
npm install -g @nestjs/cli
nest new alibia-api
cd alibia-api
npm run start:dev
```

## Fichiers générés

| Fichier                 | Rôle                            |
| ----------------------- | ------------------------------- |
| `src/main.ts`           | Point d'entrée de l'application |
| `src/app.module.ts`     | Module racine — orchestre tout  |
| `src/app.controller.ts` | Reçoit les requêtes HTTP        |
| `src/app.service.ts`    | Contient la logique métier      |
| `tsconfig.json`         | Configuration TypeScript        |
| `eslint.config.mjs`     | Linting du code                 |
| `.prettierrc`           | Formatage du code               |

## Architecture de base

```
main.ts
└── AppModule
    ├── AppController (routes HTTP)
    └── AppService (logique)
```

## Décision d'architecture

Stratégie multi-tenant choisie : **Row-Level Tenancy**
Chaque table contiendra une colonne `businessId` qui filtre automatiquement les données par entreprise.

## Bonnes pratiques appliquées

- Projet initialisé via CLI officiel NestJS (pas from scratch)
- Package manager unifié : npm
- Watch mode activé en développement (`start:dev`)

## Erreurs fréquentes

- Lancer `nest new` dans le mauvais dossier → toujours vérifier son chemin avec `pwd` avant
- Utiliser Node < 18 → NestJS 11 requiert Node 18 minimum

---

# Étape 02 — Configuration TypeScript, ESLint et Prettier

## Objectif

Renforcer la configuration par défaut de NestJS pour un projet SaaS professionnel avec des règles strictes de qualité de code.

## Commandes exécutées

```bash
npm install --save-dev @types/node
npm run lint
npx tsc --noEmit
```

## Fichiers modifiés

### tsconfig.json

- `strict: true` → active toutes les vérifications TypeScript
- `noImplicitAny: true` → interdit les variables sans type
- `strictNullChecks: true` → interdit d'ignorer null/undefined
- `noUnusedLocals: true` → erreur si variable non utilisée
- `noUnusedParameters: true` → erreur si paramètre non utilisé
- `noImplicitReturns: true` → chaque fonction doit retourner
- `types: ["node"]` → types Node.js globaux (process, etc.)
- `include/exclude` → TypeScript ne compile que `src/`

### eslint.config.mjs

- Ignore les fichiers de test (`*.spec.ts`, `test/**`)
- Règles ajoutées :
  - `no-explicit-any` → error
  - `explicit-function-return-type` → warn
  - `no-unused-vars` → error
  - `no-floating-promises` → error
  - `no-console` → warn

---

# Étape 07 — Authentification (JWT + OTP + 2FA) + Sécurisation MVP

## Objectif

Mettre en place un système d'authentification complet et sécurisé :
inscription avec création de business, vérification d'email par OTP,
connexion avec option 2FA par email, gestion des tokens JWT
(access + refresh avec rotation), rate limiting, verrouillage de compte,
et validation stricte des entrées.

## Commandes exécutées

```bash
npm install @nestjs/jwt @nestjs/passport passport passport-jwt
npm install bcryptjs nodemailer
npm install --save-dev @types/passport-jwt @types/bcryptjs @types/nodemailer
npm install class-validator class-transformer
npm install @nestjs/throttler
```

## Fichiers créés / importants

| Fichier                                       | Rôle                                                        |
| --------------------------------------------- | ----------------------------------------------------------- |
| `src/modules/auth/auth.module.ts`             | Module Auth — JwtModule + PassportModule + Throttler        |
| `src/modules/auth/auth.controller.ts`         | Toutes les routes Auth                                      |
| `src/modules/auth/auth.service.ts`            | Logique métier complète                                     |
| `src/modules/auth/otp.service.ts`             | Génération / validation OTP (expiration + tentatives)       |
| `src/modules/auth/strategies/jwt.strategy.ts` | Stratégie Passport JWT                                      |
| `src/modules/auth/guards/jwt-auth.guard.ts`   | Guard de protection des routes                              |
| `src/modules/auth/dto/*.ts`                   | DTOs validés (register, login, verify-otp, refresh, reset…) |
| `src/modules/auth/constants/auth-messages.ts` | i18n FR/EN + `messageKey`                                   |
| `src/modules/mailer/*`                        | Envoi d'emails OTP                                          |
| `src/common/filters/all-exceptions.filter.ts` | Filtre global d'erreurs (Prisma assaini)                    |
| `src/common/validators/password.validator.ts` | Validateur de mot de passe fort                             |
| `src/tests/e2e/*.e2e-spec.ts`                 | 13 fichiers E2E isolés (rate-limit safe)                    |

## Architecture du flow d'authentification

```
POST /auth/register
  → hash password (bcrypt cost 12)
  → transaction : User + Business + BusinessMember(OWNER) + Subscription(TRIAL 14j)
  → OTP EMAIL_VERIFICATION + email

POST /auth/verify-email          (@HttpCode 200)
  → vérifie OTP → isVerified = true
  → accessToken + refreshToken (hashé en DB)

POST /auth/login                 (@HttpCode 200)
  → vérifie password + lockout
  → si !isVerified → nouvel OTP
  → si twoFactorEnabled → OTP TWO_FACTOR
  → sinon → tokens

POST /auth/verify-2fa            (@HttpCode 200)
  → vérifie OTP TWO_FACTOR → tokens

POST /auth/refresh               (@HttpCode 200)
  → rotation complète du refresh token
  → invalide l'ancien (détection de vol possible)

POST /auth/forgot-password       (@HttpCode 200)
  → anti-énumération (même message si email inconnu)
  → OTP PASSWORD_RESET

POST /auth/reset-password        (@HttpCode 200)
  → vérifie OTP → nouveau password + invalidation session

POST /auth/toggle-2fa            [JWT]
POST /auth/logout                [JWT] → refreshToken = null
GET  /auth/me                    [JWT]
```

## Décisions d'architecture & sécurité

- **Deux secrets JWT distincts** (`JWT_SECRET` / `JWT_REFRESH_SECRET`)
- **Refresh token hashé en DB** (bcrypt) + **rotation** à chaque usage
- **OTP** : 6 chiffres, 10 min, max 5 tentatives, un seul actif par type
- **Account lockout** en DB après 5 échecs (15 min) — survit aux redémarrages
- **Throttler** nommé `'default'` (cohérent avec les décorateurs `@Throttle`)
- **Rate limiting configurable** via `THROTTLE_TTL` / `THROTTLE_LIMIT` (env)
- **AllExceptionsFilter** global → jamais de fuite de détails Prisma
- **Helmet + CORS + ValidationPipe** (whitelist + forbidNonWhitelisted + transform)
- **i18n** FR/EN via header `Accept-Language` + `messageKey` stable
- **Swagger `/docs`** laissé accessible en MVP (décision prise)

## Bug critique découvert et corrigé

Le rate limiting **par route** était configuré depuis le début mais **totalement inactif**.
Cause : `ThrottlerModule.forRoot({ name: 'global' })` ne matchait pas les décorateurs `@Throttle({ default: {...} })`.
Seul le quota global (20 req/min toutes routes confondues) s'appliquait.
Corrigé en renommant le throttler en `'default'`.

## Schéma Prisma — champs de sécurité

```prisma
model User {
  // ...
  failedLoginAttempts Int       @default(0)
  lockedUntil         DateTime?
  refreshToken        String?
  twoFactorEnabled    Boolean   @default(false)
  isVerified          Boolean   @default(false)
}

model OtpCode {
  // ...
  attempts  Int       @default(0)
  usedAt    DateTime?
  expiresAt DateTime
}
```

## Tests (état final)

| Type               | Suites | Tests | Statut |
| ------------------ | ------ | ----- | ------ |
| Unitaires          | 2      | 22    | ✅     |
| E2E                | 13     | 40    | ✅     |
| `npm run lint`     | —      | —     | ✅     |
| `npm run build`    | —      | —     | ✅     |
| `npx tsc --noEmit` | —      | —     | ✅     |

### Couverture E2E Auth

- `register` (succès, validation, conflit email/phone/whatsapp)
- `login` + lockout + throttle
- `verify-email`
- `verify-2fa`
- `forgot-password` (anti-énumération)
- `reset-password`
- `refresh` (rotation)
- Routes protégées (`/me`, `/logout`)
- `/health`

Chaque fichier E2E crée sa **propre instance d'application** pour isoler le rate limiting.

## Environnement de test isolé

- `.env.test` → PostgreSQL dédiée (port 5433), secrets JWT de test, SMTP Ethereal
- Lancement des E2E : `npm run test:e2e`
- Aucune pollution de la base de développement

## Bonnes pratiques appliquées

- Jamais de mot de passe ou token en clair (ni en DB, ni dans les logs)
- `@HttpCode(200)` explicite sur toutes les routes POST qui retournent des tokens ou des messages (évite le 201 par défaut de Nest)
- Un seul niveau de responsabilité par service
- Access token court (15m) / Refresh token long (7d)
- Tests écrits en français pour cohérence avec l'équipe
- `messageKey` stable pour le frontend (i18n côté client possible)

## Erreurs rencontrées et solutions (extraits)

| Erreur                          | Cause                                  | Solution                                                    |
| ------------------------------- | -------------------------------------- | ----------------------------------------------------------- |
| Rate limiting par route inactif | Nom du throttler `global` ≠ `default`  | Renommé en `default`                                        |
| Routes POST renvoient 201       | Pas de `@HttpCode(200)`                | Ajouté sur login, verify-\*, refresh, forgot, reset, logout |
| `expiresIn` type incompatible   | `string` vs `StringValue` (ms)         | Cast `as StringValue`                                       |
| Validation absente              | `ValidationPipe` manquant              | Ajouté globalement (whitelist + forbidNonWhitelisted)       |
| `NODE_ENV=test` rejeté          | Joi trop strict                        | Ajout de `'test'` dans l'enum                               |
| Énumération d'emails            | `forgot-password` révélait l'existence | Message unique anti-énumération                             |

---

## Liste complète des routes Auth

**Préfixe global :** `/api/v1`
**Base URL exemple :** `http://localhost:3000/api/v1`

### `POST /auth/register`

- **Throttle :** 5 req / 60s
- **Status succès :** 201
- **Body :**

```json
{
  "firstName": "Jean",
  "lastName": "Dupont",
  "email": "jean@example.com",
  "password": "Password123!",
  "phone": "+237699000001",
  "businessName": "Ma Boutique",
  "countryName": "Cameroun",
  "countryCode": "CM",
  "countryFlag": "🇨🇲",
  "city": "Douala",
  "whatsapp": "+237699000002",
  "currency": "XAF",
  "currencySymbol": "FCFA"
}
```

- **Réponse 201 :**

```json
{
  "message": "Compte créé avec succès. Vérifiez votre email pour activer votre compte.",
  "messageKey": "AUTH_REGISTER_SUCCESS"
}
```

### `POST /auth/verify-email`

- **HttpCode :** 200
- **Throttle :** 5 req / 60s
- **Body :**

```json
{
  "email": "jean@example.com",
  "code": "847291"
}
```

- **Réponse 200 :** `AuthResponseDto` (accessToken + refreshToken + user + business)

### `POST /auth/login`

- **HttpCode :** 200
- **Throttle :** 5 req / 60s
- **Body :**

```json
{
  "email": "jean@example.com",
  "password": "Password123!"
}
```

- **Réponses possibles :**
  - 200 + tokens (si tout OK)
  - 200 + `{ messageKey: "AUTH_ACCOUNT_NOT_VERIFIED_CODE_SENT" }`
  - 200 + `{ messageKey: "AUTH_TWO_FACTOR_CODE_SENT" }`
  - 401 `AUTH_INVALID_CREDENTIALS` / `AUTH_ACCOUNT_LOCKED`

### `POST /auth/verify-2fa`

- **HttpCode :** 200
- **Throttle :** 5 req / 60s
- **Body :** identique à `verify-email`
- **Réponse 200 :** `AuthResponseDto`

### `POST /auth/refresh`

- **HttpCode :** 200
- **Throttle :** 10 req / 60s
- **Body :**

```json
{
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

- **Réponse 200 :** `AuthResponseDto` (rotation)

### `POST /auth/forgot-password`

- **HttpCode :** 200
- **Throttle :** 3 req / 5 min
- **Body :**

```json
{
  "email": "jean@example.com"
}
```

- **Réponse 200 (toujours la même – anti-énumération) :**

```json
{
  "message": "Si un compte existe avec cet email, un code de réinitialisation a été envoyé.",
  "messageKey": "AUTH_PASSWORD_RESET_EMAIL_SENT"
}
```

### `POST /auth/reset-password`

- **HttpCode :** 200
- **Throttle :** 5 req / 60s
- **Body :**

```json
{
  "email": "jean@example.com",
  "code": "847291",
  "newPassword": "NouveauPassword123!"
}
```

- **Réponse 200 :**

```json
{
  "message": "Mot de passe réinitialisé avec succès",
  "messageKey": "AUTH_PASSWORD_RESET_SUCCESS"
}
```

### `POST /auth/toggle-2fa` (protégé)

- **Guard :** JWT
- **Body :**

```json
{
  "password": "Password123!",
  "enabled": true
}
```

### `POST /auth/logout` (protégé)

- **HttpCode :** 200
- **Guard :** JWT
- **Réponse :** 200 (vide)

### `GET /auth/me` (protégé)

- **Guard :** JWT
- **Réponse :** profil utilisateur + businesses

### `GET /health`

- **Réponse :** statut de l'API

---

## Format Postman (prêt à l'emploi)

### Variables d'environnement recommandées

| Variable        | Valeur initiale                | Description                 |
| --------------- | ------------------------------ | --------------------------- |
| `BASE_URL`      | `http://localhost:3000/api/v1` | Préfixe API                 |
| `ACCESS_TOKEN`  | _(vide)_                       | Rempli après login / verify |
| `REFRESH_TOKEN` | _(vide)_                       | Rempli après login / verify |
| `EMAIL`         | `jean@example.com`             | Email de test               |
| `PASSWORD`      | `Password123!`                 | Mot de passe de test        |

### Collection minimale (ordre logique)

1. **Register** → `POST {{BASE_URL}}/auth/register`
2. **Verify Email** → `POST {{BASE_URL}}/auth/verify-email`
   → dans le script de test Postman :
   ```js
   if (pm.response.code === 200) {
     const json = pm.response.json();
     pm.environment.set("ACCESS_TOKEN", json.accessToken);
     pm.environment.set("REFRESH_TOKEN", json.refreshToken);
   }
   ```
3. **Login** → `POST {{BASE_URL}}/auth/login` (même script de récupération des tokens)
4. **Refresh** → `POST {{BASE_URL}}/auth/refresh`
   Body : `{ "refreshToken": "{{REFRESH_TOKEN}}" }`
5. **Me** → `GET {{BASE_URL}}/auth/me`
   Header : `Authorization: Bearer {{ACCESS_TOKEN}}`
6. **Logout** → `POST {{BASE_URL}}/auth/logout`
   Header : `Authorization: Bearer {{ACCESS_TOKEN}}`
7. **Forgot Password** / **Reset Password** / **Verify 2FA** / **Toggle 2FA**

---

## État actuel de la mission « Sécurisation MVP »

| Point                                                             | Statut                            |
| ----------------------------------------------------------------- | --------------------------------- |
| Audit initial                                                     | ✅                                |
| Mots de passe & données sensibles                                 | ✅                                |
| JWT (access + refresh + rotation)                                 | ✅                                |
| Sécurité HTTP (Helmet, CORS, ValidationPipe, AllExceptionsFilter) | ✅                                |
| Rate limiting (corrigé + testé)                                   | ✅                                |
| Account lockout                                                   | ✅                                |
| E2E minimum + endpoints manquants                                 | ✅ (40 tests)                     |
| Protection `/docs`                                                | ✅ (laissé ouvert – décision MVP) |
| Build / Lint / Tests finaux                                       | ✅                                |
| Rapport final                                                     | ✅                                |

**L'API Auth est prête pour un MVP sécurisé.**

---

# Étape 08 — Module Business : gestion des membres (rôles, invitations, listing)

## Objectif

Construire le système de gestion d'équipe multi-tenant sur `feature/business-module` :
lecture/modification du business courant, système d'invitation de membres par
token à usage unique (email), et listing unifié des membres actifs/inactifs/invités
avec filtre et recherche.

## Commandes exécutées

```bash
npx prisma migrate dev --name add_invitations
npx prisma migrate dev --name add_business_member_is_active
```

## Fichiers créés / importants

| Fichier                                                 | Rôle                                                                   |
| ------------------------------------------------------- | ---------------------------------------------------------------------- |
| `src/modules/business/business.controller.ts`           | Toutes les routes Business                                             |
| `src/modules/business/business.service.ts`              | Logique métier (get/update business, invite, accept, members)          |
| `src/modules/business/business.module.ts`               | Importe `MailerModule` + `AuthModule`                                  |
| `src/modules/business/types.ts`                         | `MemberListStatus`, `UnifiedMemberEntry`                               |
| `src/modules/business/dto/update-business.dto.ts`       | Validation PATCH /me                                                   |
| `src/modules/business/dto/invite-member.dto.ts`         | `email` + `role`                                                       |
| `src/modules/business/dto/accept-invitation.dto.ts`     | `token` uniquement (flow en 2 temps)                                   |
| `src/modules/business/constants/business-messages.ts`   | i18n Business (`UPDATE_SUCCESS`, `NOT_FOUND`, `INVALID_STATUS_FILTER`) |
| `src/modules/business/constants/invitation-messages.ts` | i18n Invitation (7 clés)                                               |
| `src/common/utils/invitation-token.util.ts`             | Génération + hash SHA-256 du token d'invitation                        |
| `src/modules/mailer/mailer.service.ts`                  | + `sendInvitationEmail` / `buildInvitationTemplate`                    |
| `src/modules/auth/auth.service.ts`                      | + `issueTokensForMembership` (méthode publique)                        |
| `src/tests/e2e/business-update-me.e2e-spec.ts`          | 4 tests                                                                |
| `src/tests/e2e/business-invite-member.e2e-spec.ts`      | 6 tests                                                                |
| `src/tests/e2e/business-accept-invitation.e2e-spec.ts`  | 7 tests                                                                |
| `src/tests/e2e/business-get-members.e2e-spec.ts`        | 9 tests                                                                |

## Schéma Prisma — ajouts

```prisma
enum InvitationStatus {
  PENDING
  ACCEPTED
  EXPIRED
  REVOKED
}

model Invitation {
  id         String           @id @default(cuid())
  email      String
  role       Role             @default(STAFF)
  tokenHash  String           @unique
  status     InvitationStatus @default(PENDING)
  expiresAt  DateTime
  acceptedAt DateTime?
  createdAt  DateTime         @default(now())

  invitedById String
  invitedBy   User     @relation(fields: [invitedById], references: [id])

  businessId String
  business   Business @relation(fields: [businessId], references: [id], onDelete: Cascade)

  @@index([businessId, email])
  @@map("invitations")
}

model BusinessMember {
  // ...
  isActive Boolean @default(true)
}
```

## Architecture du flow d'invitation (décision : token, en deux temps)

Trois approches ont été évaluées (attacher un compte existant / exiger un compte
préexistant / vrai système de token) — **le système de token à usage unique** a
été retenu pour sa sécurité et sa conformité aux standards du marché (façon
Slack/Notion).

```
POST /business/members/invite         [JWT, OWNER/ADMIN]  (@HttpCode 200)
  → rejette role: OWNER (409 INVITATION_OWNER_ROLE_NOT_ALLOWED)
  → rejette si email déjà membre (409 INVITATION_ALREADY_MEMBER)
  → invalide toute invitation PENDING existante pour cet email (→ REVOKED)
  → génère token (32 bytes) + tokenHash (SHA-256, stocké seul en DB)
  → crée Invitation (expiresAt = +7 jours)
  → envoi email (template HTML, lien FRONTEND_URL/invitation/accepter?token=...)
  → envoi email non-bloquant (try/catch + log, comme pour l'OTP register)

POST /business/members/accept-invite   [JWT]  (@HttpCode 200)
  → l'utilisateur a déjà fait /auth/register + vérifié son email (flow en 2 temps)
  → hash le token reçu, cherche l'Invitation correspondante
  → vérifie status (404 si inexistante, 409 si ACCEPTED/REVOKED/EXPIRED)
  → vérifie que l'email du JWT correspond à celui de l'invitation
    (sinon 404 générique — ne révèle jamais qu'un token existe pour un autre compte)
  → transaction : crée BusinessMember + marque Invitation ACCEPTED
  → régénère un accessToken + refreshToken reflétant le nouveau businessId/role
    (via AuthService.issueTokensForMembership, réutilise generateTokens +
    updateRefreshToken déjà existants)
```

## Décisions d'architecture & sécurité

- **Token en clair envoyé par email, seul le hash SHA-256 est stocké en DB**
  (`tokenHash`, unique) — jamais de token réversible en base
- **Pourquoi SHA-256 et pas bcrypt ici** : le token a 32 bytes d'entropie
  (256 bits), un bruteforce est déjà impossible ; bcrypt ralentirait sans
  bénéfice de sécurité additionnel (bcrypt sert à ralentir le bruteforce de
  mots de passe _faibles_, pas de tokens aléatoires)
- **Flow d'acceptation en deux temps** (register classique existant, puis
  `accept-invite` léger avec juste le token) plutôt qu'un temps unique — évite
  de dupliquer la logique de création de compte + vérification email déjà
  testée dans `AuthService`
- **`role: OWNER` interdit à l'invitation** — un OWNER ne peut être créé que
  via promotion explicite (`PATCH /members/:id/role`, à venir), jamais
  directement par invitation
- **Ré-invitation d'un email déjà en attente** : l'ancienne invitation
  `PENDING` est automatiquement marquée `REVOKED`, une nouvelle est créée
  (pas de blocage 409)
- **Multi-business autorisé** : un `User` peut appartenir à plusieurs
  `Business` (`BusinessMember` en `@@unique([userId, businessId])`, pas de
  contrainte unique sur `userId` seul) — accepter une invitation alors qu'on
  est déjà membre d'un autre business est un cas valide
- **JWT régénéré après acceptation** : sans ça, l'utilisateur resterait
  bloqué avec l'ancien `businessId`/`role` (ou aucun) jusqu'à une reconnexion
  manuelle — le nouveau token bascule automatiquement sur le business
  nouvellement rejoint (pas de vrai "switch business" pour l'instant)
- **`BusinessMember.isActive`** (et non un statut sur `User`) : un membre peut
  être désactivé _dans ce business précis_ sans affecter son compte global ou
  son appartenance à d'autres businesses. Champ en lecture seule pour
  l'instant (pas encore d'endpoint pour le modifier)
- **Listing unifié `GET /business/members`** : un seul tableau combinant
  `BusinessMember` (statut `ACTIVE`/`INACTIVE` selon `isActive`) et
  `Invitation` `PENDING` non expirées (statut `PENDING`), plutôt que deux
  listes séparées — plus simple à afficher côté frontend
- **Filtre `?status=` strict** : une valeur invalide renvoie une `400`
  explicite (`BUSINESS_INVALID_STATUS_FILTER`) plutôt que d'être ignorée
  silencieusement
- **Recherche `?search=`** : `firstName`/`lastName`/`email` pour les membres,
  `email` pour les invitations pending ; insensible à la casse
  (`mode: 'insensitive'`, natif PostgreSQL) ; accents non normalisés (pas
  d'extension `unaccent`)
- **Tri par `createdAt` décroissant** (les plus récents en premier)

## Matrice de permissions Business (finale)

| Action                         | Rôles autorisés      | Restrictions supplémentaires                                                                                                        |
| ------------------------------ | -------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| Lire business / lister membres | Tous rôles connectés | —                                                                                                                                   |
| Modifier business              | OWNER, ADMIN         | —                                                                                                                                   |
| Inviter un membre              | OWNER, ADMIN         | Cible ne peut pas être `OWNER`                                                                                                      |
| Changer le rôle d'un membre    | OWNER uniquement     | Ne peut pas modifier son propre rôle. Toute cible autorisée, y compris promotion en `OWNER`                                         |
| Retirer (désactiver) un membre | OWNER, ADMIN         | Auto-retrait interdit. ADMIN ne peut pas retirer un OWNER. Le dernier OWNER actif ne peut jamais être retiré (protection défensive) |

## Incident résolu — confusion de port sur les migrations

**Symptôme** : `npx prisma migrate dev` a appliqué une migration sur
`alibia_test_db` (port 5433) au lieu de `alibia_db` (port 5432).

**Cause** : `$env:DATABASE_URL` settée manuellement dans une session
PowerShell (pour cibler la DB de test) reste active pour toute la session
tant qu'elle n'est pas effacée — elle prend le pas sur `.env` (dotenv
n'écrase jamais une variable déjà présente dans le process).

**Correction** : `Remove-Item Env:\DATABASE_URL` puis `npx prisma migrate
status` pour confirmer la bonne base avant toute nouvelle commande.

**Règle adoptée** : toujours effacer `$env:DATABASE_URL` juste après une
commande ciblant `.env.test`, ou utiliser une nouvelle fenêtre de terminal.

## Bug ESLint résolu — `Logger` non déclaré

`this.logger.error(...)` utilisé dans `inviteMember()` sans avoir importé
`Logger` de `@nestjs/common` ni déclaré `private readonly logger = new
Logger(BusinessService.name)` → ESLint `no-unsafe-call` /
`no-unsafe-member-access` (type non résolu). Corrigé en ajoutant les deux.

## Décision — protection défensive inatteignable via HTTP

Pour `DELETE /business/members/:id`, la règle « le dernier OWNER actif ne
peut jamais être retiré » s'est révélée **structurellement inatteignable**
via un flux HTTP normal une fois les autres règles appliquées (auto-retrait
interdit, ADMIN ne peut pas viser un OWNER) : le seul appelant théorique
capable de retirer le dernier OWNER serait ce même OWNER, déjà bloqué par
la règle d'auto-retrait en amont.

**Décision** : garder `LAST_OWNER_CANNOT_BE_REMOVED` comme **garde-fou
défensif** dans le service (protège contre une incohérence de données
introduite autrement — migration, correction manuelle en DB, futur bug),
sans modifier les règles précédentes. Testée directement au niveau
`BusinessService` (bypass de l'API) plutôt que via une requête HTTP,
puisqu'aucun chemin HTTP légitime ne peut l'atteindre.

## Incident de test — désactivations en cascade entre tests

Un premier jet du test « dernier OWNER » utilisait un compte `tempOwner`
créé/supprimé à la volée pour contourner l'blocage d'auto-retrait, laissant
par erreur `ownerMemberId` et `secondOwnerMemberId` déjà inactifs pour les
tests suivants du même fichier. Résultat : les tests suivants tombaient
silencieusement dans la branche **idempotente** (`200` sans jamais exercer
la vraie règle testée), au lieu d'échouer franchement — révélé seulement au
moment d'écrire le test final (`rejects` reçu comme `resolved`).
**Corrigé** en supprimant ce test intermédiaire et en le remplaçant par un
appel direct à `BusinessService.deactivateMember()` (voir ci-dessus).

## Tests (état après cette étape)

| Fichier                                     | Tests | Statut |
| ------------------------------------------- | ----- | ------ |
| `business-update-me.e2e-spec.ts` (existant) | 4     | ✅     |
| `business-invite-member.e2e-spec.ts`        | 6     | ✅     |
| `business-accept-invitation.e2e-spec.ts`    | 7     | ✅     |
| `business-get-members.e2e-spec.ts`          | 9     | ✅     |
| `business-update-member-role.e2e-spec.ts`   | 7     | ✅     |
| `business-deactivate-member.e2e-spec.ts`    | 10    | ✅     |

**Total suite E2E complète : 19 suites, 85 tests, tous ✅**
(40 Auth + 4 business-update-me + 6 invite + 7 accept + 9 get-members +
7 update-role + 10 deactivate-member)

Unitaires inchangés : 2 suites, 22 tests ✅

## Environnement

- `FRONTEND_URL` ajoutée à `env.config.ts` (validation Joi, `.required()`),
  et renseignée dans `.env` / `.env.example` / `.env.test`
  (`http://localhost:3000` en dev)
- Utilisée pour construire le lien d'invitation :
  `${FRONTEND_URL}/invitation/accepter?token=...`

## Liste complète des routes Business (mise à jour)

**Préfixe global :** `/api/v1`

### `GET /business/me` (protégé)

- **Guard :** JWT
- **Réponse 200 :** `BusinessWithDetails` (business + `members[]` + `subscription`)

### `PATCH /business/me` (OWNER, ADMIN)

- **HttpCode :** 200
- **Body :** `name`, `address`, `city`, `whatsapp`, `currency`, `currencySymbol`
  (le `slug` reste volontairement immutable)
- **Réponse 200 :** `BusinessWithDetails`
- **400** si propriété inattendue dans le body

### `POST /business/members/invite` (OWNER, ADMIN)

- **HttpCode :** 200
- **Body :**

```json
{
  "email": "nouveau.membre@example.com",
  "role": "STAFF"
}
```

- **Réponse 200 :**

```json
{
  "message": "Invitation envoyée avec succès.",
  "messageKey": "INVITATION_SENT"
}
```

- **409** `INVITATION_OWNER_ROLE_NOT_ALLOWED` / `INVITATION_ALREADY_MEMBER`
- **403** `COMMON_FORBIDDEN_ROLE` (si appelant STAFF/LIVREUR/REVENDEUR)

### `POST /business/members/accept-invite` (protégé, tous rôles)

- **HttpCode :** 200
- **Prérequis :** l'invité a déjà fait `/auth/register` + `/auth/verify-email` +
  `/auth/login` avec l'email invité, et présente son JWT
- **Body :**

```json
{
  "token": "a1b2c3d4e5f6..."
}
```

- **Réponse 200 :**

```json
{
  "message": "Invitation acceptée, bienvenue dans le business.",
  "messageKey": "INVITATION_ACCEPT_SUCCESS",
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

- **404** `INVITATION_NOT_FOUND` (token inexistant, ou email JWT ≠ email invitation)
- **409** `INVITATION_EXPIRED` / `INVITATION_ALREADY_ACCEPTED` / `INVITATION_REVOKED`

### `GET /business/members` (protégé, tous rôles)

- **Query params optionnels :**
  - `status` : liste séparée par virgules parmi `ACTIVE`, `INACTIVE`, `PENDING`
    (défaut : les trois). Valeur invalide → `400 BUSINESS_INVALID_STATUS_FILTER`
  - `search` : recherche libre sur `firstName`/`lastName`/`email` (membres) ou
    `email` (invitations pending), insensible à la casse
- **Exemple :** `GET /business/members?status=ACTIVE,PENDING&search=amadou`
- **Réponse 200 :** tableau `UnifiedMemberEntry[]`, trié par `createdAt` décroissant

```json
[
  {
    "type": "member",
    "status": "ACTIVE",
    "id": "clx...",
    "userId": "clx...",
    "email": "amadou@example.com",
    "firstName": "Amadou",
    "lastName": "Diallo",
    "role": "STAFF",
    "createdAt": "2026-08-15T12:00:00.000Z"
  },
  {
    "type": "invitation",
    "status": "PENDING",
    "id": "clx...",
    "userId": null,
    "email": "invite@example.com",
    "firstName": null,
    "lastName": null,
    "role": "STAFF",
    "createdAt": "2026-08-15T11:00:00.000Z"
  }
]
```

### `PATCH /business/members/:id/role` (OWNER uniquement)

- **HttpCode :** 200
- **`:id`** = `BusinessMember.id` (jamais un `userId` ni un id d'invitation)
- **Body :**

```json
{
  "role": "ADMIN"
}
```

- **Réponse 200 :**

```json
{
  "message": "Rôle du membre mis à jour avec succès.",
  "messageKey": "BUSINESS_ROLE_UPDATE_SUCCESS"
}
```

- **403** `COMMON_FORBIDDEN_ROLE` (appelant non-OWNER)
- **404** `BUSINESS_MEMBER_NOT_FOUND` (id inexistant, ou membre d'un autre business)
- **409** `BUSINESS_CANNOT_MODIFY_OWN_ROLE` (un OWNER ne peut jamais modifier son propre rôle via cette route)
- Toute valeur de rôle cible est acceptée, **y compris `OWNER`** (promotion libre par un OWNER)

### `DELETE /business/members/:id` (OWNER, ADMIN)

- **HttpCode :** 200 — désactivation (`isActive = false`), **jamais de suppression physique** (conserve l'historique, réactivable plus tard)
- **`:id`** = `BusinessMember.id`
- **Réponse 200 :**

```json
{
  "message": "Membre désactivé avec succès.",
  "messageKey": "BUSINESS_MEMBER_DEACTIVATED_SUCCESS"
}
```

- **403** `COMMON_FORBIDDEN_ROLE` (appelant STAFF/LIVREUR/REVENDEUR)
- **404** `BUSINESS_MEMBER_NOT_FOUND` (id inexistant, ou membre d'un autre business)
- **409** `BUSINESS_CANNOT_REMOVE_SELF` (auto-retrait interdit sur cette route — prévoir `POST /business/leave` plus tard pour le départ volontaire)
- **409** `BUSINESS_ADMIN_CANNOT_REMOVE_OWNER` (un ADMIN ne peut retirer que des rôles inférieurs à OWNER)
- **409** `BUSINESS_LAST_OWNER_CANNOT_BE_REMOVED` (protection défensive, inatteignable en usage normal — voir section Incidents)
- **Idempotent** : désactiver un membre déjà inactif renvoie `200` sans erreur

---

## Décision — scope minimum du module Business pour le MVP

Analyse effectuée avant de passer au module suivant : les 6 endpoints Business
existants (`GET/PATCH /me`, `invite`, `accept-invite`, `GET /members`,
`PATCH .../role`, `DELETE .../:id`) sont jugés **suffisants pour un MVP** —
ils couvrent le cycle de vie complet d'une équipe (créer, inviter, consulter,
promouvoir, retirer). Décision : **ne rien retirer** (déjà codé et testé,
coût de maintenance négligeable), et reporter à une v2 réelle (déclenchée par
un besoin utilisateur constaté, pas anticipé) :

- Gestion de la `Subscription` (changement de plan, facturation)
- Upload du logo business (`Business.logo`)
- Vrai "switch business" pour un utilisateur multi-tenant
- `POST /business/leave` (départ volontaire d'un membre)
- Historique/journalisation des actions d'équipe

## Reste à faire sur Business

1. ~~`POST /business/members/invite`~~ ✅
2. ~~`GET /business/members`~~ ✅
3. ~~`PATCH /business/members/:id/role`~~ ✅
4. ~~`DELETE /business/members/:id`~~ ✅

**Module Business considéré comme terminé pour le MVP.**

## État actuel de la mission « Module Business »

| Point                                                     | Statut        |
| --------------------------------------------------------- | ------------- |
| GET/PATCH `/business/me`                                  | ✅            |
| Système de permissions par rôle (`RolesGuard`)            | ✅            |
| Invitation par token (email + acceptation en 2 temps)     | ✅            |
| Listing unifié membres + invitations (filtre + recherche) | ✅            |
| Changement de rôle d'un membre                            | ✅            |
| Retrait (désactivation) d'un membre (règle dernier OWNER) | ✅            |
| Build / Lint / Tests finaux (85/85 confirmé)              | ✅            |
| Module Users                                              | ⏳ à démarrer |

---

# Étape 09 — Module Users (MVP)

## Objectif

Couvrir la gestion du propre compte utilisateur, en complément de ce qui
existe déjà dans Auth (`GET /auth/me`, `POST /auth/toggle-2fa`,
`POST /auth/logout`).

## Décision — scope minimum MVP

| Endpoint                   | Statut | Justification                                                                                                                             |
| -------------------------- | ------ | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `PATCH /users/me`          | ✅     | Modifier `firstName`/`lastName`/`phone` — action basique attendue dès le lancement                                                        |
| `PATCH /users/me/password` | ✅     | Changer son mot de passe en étant connecté (avec l'ancien en confirmation), distinct du flow `forgot/reset` qui suppose une perte d'accès |

**Explicitement reporté** (pas de besoin utilisateur constaté à date) :

- `GET /users/me` — déjà couvert par `GET /auth/me`, pas de duplication
- Upload d'avatar — nécessite une vraie infra de stockage (S3/Cloudinary),
  le champ `avatar` existe déjà en DB mais reste inutilisé pour l'instant
- Changement d'**email** — volontairement **immutable pour le MVP**
  (nécessiterait sa propre vérification OTP, comme à l'inscription) → v2
- Désactivation/suppression de son propre compte — cas rare, et risqué à
  bâcler (cascade si l'utilisateur est l'unique OWNER d'un business)
- CRUD admin sur les comptes d'autres utilisateurs — déjà couvert côté
  business par `PATCH .../role` et `DELETE .../:id`

## Fichiers créés / importants

| Fichier                                           | Rôle                                                                                      |
| ------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| `src/modules/users/users.module.ts`               | Importe `AuthModule` (délégation de `changePassword`)                                     |
| `src/modules/users/users.controller.ts`           | `PATCH /me`, `PATCH /me/password`                                                         |
| `src/modules/users/users.service.ts`              | `updateMe()` (logique propre) + `changePassword()` (délègue à `AuthService`)              |
| `src/modules/users/dto/update-user.dto.ts`        | `firstName?`, `lastName?`, `phone?` — tous optionnels                                     |
| `src/modules/users/constants/user-messages.ts`    | i18n Users (`UPDATE_SUCCESS`, `PHONE_ALREADY_USED`, `NO_FIELDS_TO_UPDATE`, `NOT_FOUND`)   |
| `src/modules/auth/dto/change-password.dto.ts`     | `currentPassword`, `newPassword` (`@IsStrongPassword()`)                                  |
| `src/modules/auth/auth.service.ts`                | + `changePassword()` (logique réelle : vérif password, hash, invalidation `refreshToken`) |
| `src/modules/auth/constants/auth-messages.ts`     | + `PASSWORD_CHANGE_SUCCESS`                                                               |
| `src/tests/e2e/users-update-me.e2e-spec.ts`       | 6 tests                                                                                   |
| `src/tests/e2e/users-change-password.e2e-spec.ts` | 4 tests                                                                                   |

## Architecture

```
PATCH /users/me                 [JWT]
  → rejette body vide (400 USER_NO_FIELDS_TO_UPDATE)
  → vérifie unicité du phone si fourni (409 USER_PHONE_ALREADY_USED)
  → met à jour firstName/lastName/phone (email non modifiable)

PATCH /users/me/password        [JWT]
  → UsersController → UsersService.changePassword() → délègue à
    AuthService.changePassword() (logique réelle)
  → vérifie currentPassword (401 AUTH_INVALID_PASSWORD si incorrect)
  → hash newPassword (bcrypt cost 12, cohérent avec register/reset)
  → invalide refreshToken (= null) → force une reconnexion
  → 200 AUTH_PASSWORD_CHANGE_SUCCESS
```

## Décisions d'architecture

- **Email immutable pour le MVP** — cohérent avec le `slug` du Business,
  changement d'email reporté en v2 (nécessiterait sa propre vérification OTP)
- **Body vide rejeté en `400`** (`BadRequestException`), pas en `409` —
  écart corrigé en cours de route (`ConflictException` utilisé par réflexe
  au départ, corrigé après relecture de la décision initiale)
- **Vérification d'unicité du `phone`** faite manuellement dans le service
  (`findUnique` puis comparaison d'id) plutôt que de laisser Prisma lever une
  erreur de contrainte `@unique` brute — permet un message i18n propre au
  lieu d'une 500 générique
- **Logique de changement de mot de passe centralisée dans `AuthService`**,
  pas dupliquée dans `UsersService` — `UsersService.changePassword()` se
  contente de déléguer (`return this.authService.changePassword(...)`),
  cohérent avec le pattern déjà utilisé pour
  `BusinessService.acceptInvitation()` → `AuthService.issueTokensForMembership()`
- **`UsersModule` importe `AuthModule`** pour permettre l'injection
  d'`AuthService` dans `UsersService`
- **Changement de mot de passe invalide le `refreshToken`** (comme
  `resetPassword()`), forçant une reconnexion — traite un changement
  volontaire de mot de passe comme un signal de sécurité, cohérent avec le
  reste du projet

## Incident résolu — dépendance manquante non détectée par `tsc`/`lint`

**Symptôme** : `Nest can't resolve dependencies of the UsersService
(PrismaService, ?). Please make sure that the argument AuthService at index
[1] is available in the UsersModule module` — uniquement au lancement des
tests E2E, alors que `tsc --noEmit` et `npm run lint` étaient restés
silencieux à chaque étape précédente.

**Cause** : l'injection de dépendances de NestJS est résolue **à
l'exécution** (runtime), pas à la compilation TypeScript. Le constructeur de
`UsersService` déclarait bien `private readonly authService: AuthService`
(donc valide aux yeux de `tsc`), mais `UsersModule` n'avait jamais reçu
`imports: [AuthModule]` — l'erreur ne pouvait donc se révéler qu'au moment
où Nest instancie réellement le graphe de dépendances, ce qui n'arrive qu'au
démarrage de l'app (donc dans les tests E2E, jamais dans `tsc`/`lint`/tests
unitaires isolés).

**Leçon retenue** : une modification de module (`imports: [...]`) doit être
vérifiée par un vrai démarrage d'application (E2E) en plus de la
compilation — `tsc --noEmit` seul ne suffit pas à garantir qu'un module
Nest est correctement câblé.

## Tests (état après cette étape)

| Fichier                             | Tests | Statut |
| ----------------------------------- | ----- | ------ |
| `users-update-me.e2e-spec.ts`       | 6     | ✅     |
| `users-change-password.e2e-spec.ts` | 4     | ✅     |

**Total suite E2E complète : 22 suites, 95 tests, tous ✅**
(85 après le module Business + 6 update-me + 4 change-password)

Unitaires inchangés : 2 suites, 22 tests ✅

## Liste complète des routes Users

**Préfixe global :** `/api/v1`

### `PATCH /users/me` (protégé)

- **Guard :** JWT
- **Body (tous champs optionnels, au moins un requis) :**

```json
{
  "firstName": "Jean",
  "lastName": "Dupont",
  "phone": "+237612345678"
}
```

- **Réponse 200 :**

```json
{
  "message": "Profil mis à jour avec succès.",
  "messageKey": "USER_UPDATE_SUCCESS"
}
```

- **400** `USER_NO_FIELDS_TO_UPDATE` (body vide)
- **409** `USER_PHONE_ALREADY_USED` (phone déjà pris par un autre compte)

### `PATCH /users/me/password` (protégé)

- **Guard :** JWT
- **Body :**

```json
{
  "currentPassword": "AncienPassword123!",
  "newPassword": "NouveauPassword456!"
}
```

- **Réponse 200 :**

```json
{
  "message": "Mot de passe modifié avec succès.",
  "messageKey": "AUTH_PASSWORD_CHANGE_SUCCESS"
}
```

- **400** validation `@IsStrongPassword()` sur `newPassword`
- **401** `AUTH_INVALID_PASSWORD` (currentPassword incorrect)
- **Effet de bord** : invalide le `refreshToken` en DB → force une reconnexion

## Statut

**Module Users MVP terminé.** Prochaine étape : module **Inventaire**
(première feature métier au-dessus des fondations Auth/Business/Users).
