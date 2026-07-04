# Étape 07 — Authentification (JWT + OTP + 2FA)

## Objectif

Mettre en place un système d'authentification complet et sécurisé :
inscription avec création de business, vérification d'email par OTP,
connexion avec option 2FA par email, et gestion des tokens JWT
(access + refresh).

## Commandes exécutées

```bash
npm install @nestjs/jwt @nestjs/passport passport passport-jwt
npm install bcryptjs nodemailer
npm install --save-dev @types/passport-jwt @types/bcryptjs @types/nodemailer
npm install class-validator class-transformer
```

## Fichiers créés

| Fichier                                       | Rôle                                                                            |
| --------------------------------------------- | ------------------------------------------------------------------------------- |
| `src/modules/auth/auth.module.ts`             | Module Auth — configure JwtModule + PassportModule                              |
| `src/modules/auth/auth.controller.ts`         | Routes HTTP (`register`, `login`, `verify-email`, `verify-2fa`, `me`, `logout`) |
| `src/modules/auth/auth.service.ts`            | Logique métier : register, login, vérification OTP, génération tokens           |
| `src/modules/auth/otp.service.ts`             | Génération et vérification des codes OTP à 6 chiffres                           |
| `src/modules/auth/strategies/jwt.strategy.ts` | Stratégie Passport — extrait et valide le JWT du header `Authorization`         |
| `src/modules/auth/guards/jwt-auth.guard.ts`   | Guard réutilisable pour protéger les routes (`@UseGuards(JwtAuthGuard)`)        |
| `src/modules/auth/dto/register.dto.ts`        | DTO d'inscription (user + business), validé avec `class-validator`              |
| `src/modules/auth/dto/login.dto.ts`           | DTO de connexion (email + password)                                             |
| `src/modules/auth/dto/verify-otp.dto.ts`      | DTO de vérification OTP (email + code à 6 chiffres)                             |
| `src/modules/auth/dto/auth-response.dto.ts`   | DTO de réponse (tokens + user + business)                                       |
| `src/modules/auth/auth.service.spec.ts`       | Tests unitaires Jest (17 tests, mocks Prisma/Jwt/Mailer/Otp)                    |
| `src/modules/mailer/mailer.module.ts`         | Module d'envoi d'email                                                          |
| `src/modules/mailer/mailer.service.ts`        | Service Nodemailer — envoi des emails OTP (template HTML)                       |

## Fichiers modifiés

- `src/app.module.ts` → import `AuthModule`
- `src/main.ts` → ajout de `app.useGlobalPipes(new ValidationPipe(...))`
- `src/config/env.config.ts` → ajout des variables JWT (access + refresh) et MAIL, ajout de `'test'` dans l'enum `NODE_ENV`
- `package.json` → dépendances JWT/Passport/Mailer/validation

## Architecture du flow d'authentification

```
POST /auth/register
  → hash password (bcrypt, 12 rounds)
  → transaction Prisma : User + Business + BusinessMember (OWNER) + Subscription (TRIAL 14j)
  → génère OTP (EMAIL_VERIFICATION) + envoie email

POST /auth/verify-email
  → vérifie OTP → user.isVerified = true
  → génère accessToken + refreshToken

POST /auth/login
  → vérifie password (bcrypt.compare)
  → si !isVerified → renvoie un nouvel OTP
  → si twoFactorEnabled → envoie OTP (TWO_FACTOR), pas de tokens tout de suite
  → sinon → génère accessToken + refreshToken directement

POST /auth/verify-2fa
  → vérifie OTP (TWO_FACTOR) → génère les tokens

GET /auth/me         [protégé JwtAuthGuard]
POST /auth/logout    [protégé JwtAuthGuard] → refreshToken = null en DB
```

## Décisions d'architecture

- **Deux secrets JWT distincts** (`JWT_SECRET` / `JWT_REFRESH_SECRET`) → un refresh token volé ne permet pas de forger un access token, et inversement.
- **OTP à 6 chiffres, expiration 10 minutes** → stocké en DB (table `OtpCode`), un seul code actif à la fois par type (les anciens sont invalidés à la création d'un nouveau).
- **refreshToken hashé en DB** (bcrypt) avant stockage → même si la DB fuite, les refresh tokens ne sont pas directement exploitables.
- **JwtStrategy revérifie l'utilisateur en DB** (`isActive`) à chaque requête protégée → un compte désactivé perd l'accès immédiatement, même avec un access token encore valide.
- **`login()` retourne un type union** (`AuthResponseDto | { message: string }`) → même code 200 pour "connecté" et "OTP envoyé", le front distingue via la présence de `accessToken`.
- **class-validator + ValidationPipe global** (`whitelist`, `forbidNonWhitelisted`, `transform`) → tout payload invalide ou avec des champs inconnus est rejeté en 400 avant d'atteindre le service.

## Erreurs rencontrées et solutions

| Erreur                                                                          | Cause                                                                                           | Solution                                                                                                            |
| ------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| `Impossible d'assigner le type 'string \| undefined' à 'number \| StringValue'` | `expiresIn` de `@nestjs/jwt` attend le type `StringValue` (lib `ms`), pas un `string` classique | Cast explicite `as StringValue` (import `type { StringValue } from 'ms'`)                                           |
| Injection de `PrismaService` qui échoue silencieusement                         | `AuthModule` n'importait pas `PrismaModule`                                                     | Confirmé que `PrismaModule` est `@Global()` → pas besoin de le réimporter partout                                   |
| DTO acceptent n'importe quelle donnée                                           | Décorateurs Swagger (`@ApiProperty`) confondus avec validation                                  | Ajout des décorateurs `class-validator` (`@IsEmail`, `@IsString`, `@MinLength`, etc.) + `ValidationPipe` global     |
| `"NODE_ENV" must be one of [development, staging, production]`                  | Schéma Joi n'autorisait pas `test`                                                              | Ajout de `'test'` dans `.valid(...)` et dans le type `EnvironmentVariables`                                         |
| Erreur 500 au lieu de 400 sur payload invalide                                  | `ValidationPipe` absent de `main.ts`                                                            | Ajout de `app.useGlobalPipes(new ValidationPipe({ whitelist: true, forbidNonWhitelisted: true, transform: true }))` |

## Environnement de test isolé

- `.env.test` → base PostgreSQL dédiée (`alibia_test_db`, port différent du dev), secrets JWT de test (≥16 caractères), SMTP factice via **Ethereal** (emails jamais réellement livrés, consultables sur ethereal.email).
- Lancement : `npm run start:dev -- --env-file=.env.test`
- Permet de dérouler le flow complet (register → verify-email → login → me → logout) sans polluer la base de développement.

## Tests

- **Jest unitaires** (`auth.service.spec.ts`, 17 tests) : couvrent chaque branche de `register`, `verifyEmail`, `login` (succès, compte inactif, mauvais password, email non vérifié, 2FA activé), `verifyTwoFactor`, `logout`, `getMe`. Tout Prisma/Jwt/Mailer/Otp est mocké — aucune dépendance externe réelle.
- **Postman** (`Alibia-Auth.postman_collection.json`) : flow end-to-end complet avec capture automatique des tokens via scripts `pm.test` + cas d'erreur (email dupliqué → 409, mauvais password → 401, DTO invalide → 400).

## Bonnes pratiques appliquées

- Jamais de mot de passe en clair, ni en DB ni dans les logs
- Un seul quote/scope de responsabilité par service (`AuthService` orchestre, `OtpService` et `MailerService` sont des briques dédiées)
- Tokens courts pour l'access (`15m`), plus longs pour le refresh (`7d`) — équilibre sécurité/UX
- Tests unitaires écrits en français comme le reste de la codebase, pour cohérence avec l'équipe
