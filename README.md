# Alibia API

Documentation officielle d’Alibia, SaaS de gestion de commerce.  
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
| `src/modules/mailer/*`                        | Envoi d’emails OTP                                          |
| `src/common/filters/all-exceptions.filter.ts` | Filtre global d’erreurs (Prisma assaini)                    |
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
  → invalide l’ancien (détection de vol possible)

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
Seul le quota global (20 req/min toutes routes confondues) s’appliquait.  
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

Chaque fichier E2E crée sa **propre instance d’application** pour isoler le rate limiting.

## Environnement de test isolé

- `.env.test` → PostgreSQL dédiée (port 5433), secrets JWT de test, SMTP Ethereal
- Lancement des E2E : `npm run test:e2e`
- Aucune pollution de la base de développement

## Bonnes pratiques appliquées

- Jamais de mot de passe ou token en clair (ni en DB, ni dans les logs)
- `@HttpCode(200)` explicite sur toutes les routes POST qui retournent des tokens ou des messages (évite le 201 par défaut de Nest)
- Un seul niveau de responsabilité par service
- Access token court (15m) / Refresh token long (7d)
- Tests écrits en français pour cohérence avec l’équipe
- `messageKey` stable pour le frontend (i18n côté client possible)

## Erreurs rencontrées et solutions (extraits)

| Erreur                          | Cause                                  | Solution                                                    |
| ------------------------------- | -------------------------------------- | ----------------------------------------------------------- |
| Rate limiting par route inactif | Nom du throttler `global` ≠ `default`  | Renommé en `default`                                        |
| Routes POST renvoient 201       | Pas de `@HttpCode(200)`                | Ajouté sur login, verify-\*, refresh, forgot, reset, logout |
| `expiresIn` type incompatible   | `string` vs `StringValue` (ms)         | Cast `as StringValue`                                       |
| Validation absente              | `ValidationPipe` manquant              | Ajouté globalement (whitelist + forbidNonWhitelisted)       |
| `NODE_ENV=test` rejeté          | Joi trop strict                        | Ajout de `'test'` dans l’enum                               |
| Énumération d’emails            | `forgot-password` révélait l’existence | Message unique anti-énumération                             |

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

- **Réponse :** statut de l’API

---

## Format Postman (prêt à l’emploi)

### Variables d’environnement recommandées

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

**L’API Auth est prête pour un MVP sécurisé.**

```

```
