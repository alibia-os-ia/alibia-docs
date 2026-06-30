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
