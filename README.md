# alibia-docs

Documentation officielle d’Alibia, SaaS de gestion de commerce. Centralise guides, architecture, API et bonnes pratiques. Aide développeurs et utilisateurs à comprendre et utiliser le système Alibia (web, mobile, API) avec une vision scalable et IA.

# Étape 01 — Initialisation du projet NestJS

## Objectif

Créer la base du projet alibia-api avec NestJS CLI de façon
propre et professionnelle.

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

main.ts
└── AppModule
├── AppController (routes HTTP)
└── AppService (logique)

## Décision d'architecture

Stratégie multi-tenant choisie : **Row-Level Tenancy**
Chaque table contiendra une colonne `businessId` qui
filtre automatiquement les données par entreprise.

## Bonnes pratiques appliquées

- Projet initialisé via CLI officiel NestJS (pas from scratch)
- Package manager unifié : npm
- Watch mode activé en développement (start:dev)

## Erreurs fréquentes

- Lancer `nest new` dans le mauvais dossier → toujours
  vérifier son chemin avec `pwd` avant
- Utiliser Node < 18 → NestJS 11 requiert Node 18 minimum

# Étape 02 — Configuration TypeScript, ESLint et Prettier

## Objectif

Renforcer la configuration par défaut de NestJS pour un projet
SaaS professionnel avec des règles strictes de qualité de code.

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
- `include/exclude` → TypeScript ne compile que src/

### eslint.config.mjs

- Ignore les fichiers de test (\*.spec.ts, test/\*\*)
- Règles ajoutées :
  - no-explicit-any → error
  - explicit-function-return-type → warn
  - no-unused-vars → error
  - no-floating-promises → error
  - no-console → warn

### .prettierrc

- singleQuote: true
- trailingComma: all
- printWidth: 80
- endOfLine: lf (critique sur Windows)

### src/main.ts

- Type de retour explicite : Promise<void>
- bootstrap().catch() → erreur fatale visible
- eslint-disable-next-line pour console.error justifié

## Fichiers créés

- .gitattributes → force LF sur tous les fichiers Git

## Bonnes pratiques appliquées

- TypeScript strict dès le début = moins de bugs en production
- ESLint interdit any → le typage est obligatoire partout
- no-floating-promises → aucune Promise silencieuse
- Disable ESLint ciblé (ligne par ligne) jamais global

## Erreurs rencontrées et solutions

| Erreur              | Cause                | Solution                     |
| ------------------- | -------------------- | ---------------------------- |
| import.meta.dirname | Node < v21           | fileURLToPath + dirname      |
| baseUrl déprécié    | TS 7.0               | rootDir + chemins relatifs   |
| process introuvable | @types/node manquant | npm i --save-dev @types/node |
| test/ hors rootDir  | include trop large   | include: ["src/**/*"]        |

# Étape 03 — Variables d'environnement

## Objectif

Configurer un système de variables d'environnement sécurisé
avec validation au démarrage via Joi.

## Commandes exécutées

```bash
npm install @nestjs/config joi
```

## Fichiers créés

- `.env` → variables locales (jamais commité)
- `.env.example` → template commité sur Git
- `src/config/env.config.ts` → interface + validation Joi

## Fichiers modifiés

- `src/app.module.ts` → import ConfigModule.forRoot()

## Décisions importantes

- `isGlobal: true` → ConfigModule disponible partout
- Joi valide toutes les variables au démarrage
- App refuse de démarrer si variable requise manquante
- `.env` dans `.gitignore` → secrets jamais sur GitHub

## Erreurs rencontrées et solutions

| Erreur                               | Cause                                   | Solution                                 |
| ------------------------------------ | --------------------------------------- | ---------------------------------------- |
| Cannot read 'object'                 | import Joi from 'joi'                   | import \* as Joi from 'joi'              |
| Cannot find dist/main                | fichiers main.ts/app.module.ts inversés | remettre bon contenu dans chaque fichier |
| Found 0 errors mais pas de démarrage | ts-node sans NestJS bootstrap           | corriger main.ts                         |

## Bonnes pratiques

- Toujours avoir un .env.example à jour
- Valider chaque variable avec Joi (required/default)
- JWT_SECRET minimum 16 caractères en dev, 64 en prod
- Ne jamais hardcoder de valeurs de connexion dans le code

# Étape 04 — Docker + PostgreSQL + Prisma

## Objectif

Lancer PostgreSQL via Docker et connecter Prisma comme ORM.

## Commandes exécutées

```bash
docker compose up -d
npm install prisma@6 --save-dev
npm install @prisma/client@6
npx prisma init
npx prisma db push
npx prettier --write "src/**/*.ts"
```

## Fichiers créés

- `docker-compose.yml` → PostgreSQL 16 Alpine
- `prisma/schema.prisma` → Schéma de base de données
- `src/prisma/prisma.service.ts` → Client Prisma NestJS
- `src/prisma/prisma.module.ts` → Module global Prisma

## Fichiers modifiés

- `src/app.module.ts` → import PrismaModule

## Décisions d'architecture

- Prisma 6 (pas 7) → stable sur Windows
- @Global() sur PrismaModule → disponible partout
- OnModuleInit/OnModuleDestroy → connexion propre
- Volume Docker → données persistantes

## Erreurs rencontrées et solutions

| Erreur                       | Cause                      | Solution                          |
| ---------------------------- | -------------------------- | --------------------------------- |
| Prisma 7 schema-engine crash | Bug Windows Prisma 7       | Downgrade vers Prisma 6           |
| url non supporté             | Prisma 7 config différente | Prisma 6 supporte url dans schema |
| CRLF errors                  | Windows line endings       | npx prettier --write              |
| version obsolète             | Docker Compose v2          | Retirer la ligne version:         |

## Architecture finale

NestJS App
├── ConfigModule (isGlobal) → variables d'env validées
├── PrismaModule (isGlobal) → connexion PostgreSQL
└── PostgreSQL (Docker) → port 5432, données persistantes
