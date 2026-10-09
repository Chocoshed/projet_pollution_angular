# Pollution Tracker — Front-end Angular

Application web de **déclaration et de suivi des pollutions** réalisée dans le cadre d'un projet de cours Angular.
Les utilisateurs peuvent consulter les pollutions signalées, en déclarer de nouvelles, les modifier, les supprimer, les filtrer et les ajouter à leurs favoris.

Ce dépôt contient uniquement le **front-end**. Il communique avec une API REST séparée (back-end Node.js) déployée sur Render.

---

## Sommaire

- [Objectifs pédagogiques](#objectifs-pédagogiques)
- [Fonctionnalités](#fonctionnalités)
- [Stack technique](#stack-technique)
- [Architecture du projet](#architecture-du-projet)
- [Routes de l'application](#routes-de-lapplication)
- [Gestion d'état (NGXS)](#gestion-détat-ngxs)
- [Authentification](#authentification)
- [Environnements et API](#environnements-et-api)
- [Installation et lancement](#installation-et-lancement)
- [Déploiement](#déploiement)

---

## Objectifs pédagogiques

Le projet a été construit en plusieurs étapes (voir [consignes.md](consignes.md)) :

**Partie 1 — Formulaire et composants**
- Formulaire de déclaration avec **Reactive Forms**
- Validation côté client (champs requis, coordonnées numériques valides, date correcte)
- Communication entre composants via `@Input()` / `@Output()`
- Affichage d'un **récapitulatif** dynamique dans la même page, sans navigation

**Partie 2 — Service et backend**
- Service injectable utilisant `HttpClient`
- Opérations **CRUD** complètes sur les pollutions
- Page de détail d'une pollution
- Liste avec suppression et **filtres multicritères**
- Configuration de l'URL d'API selon l'environnement (dev / prod)

**Pour aller plus loin**
- Authentification JWT (inscription, connexion, déconnexion)
- Gestion d'état centralisée avec **NGXS**
- Système de favoris persistant
- Recherche en temps réel avec **RxJS**
- Gestion des utilisateurs (CRUD)
- Déploiement continu sur Render

---

## Fonctionnalités

### Pollutions
- **Liste** des pollutions sous forme de cartes
- **Recherche** textuelle en temps réel (debounce de 300 ms, appel à l'endpoint `/pollution/search`)
- **Filtres** combinables :
  - par type (Plastique, Chimique, Dépôt sauvage, Eau, Air, Autre)
  - par période (date de début / date de fin)
  - par statut : toutes, *mes pollutions*, *mes favoris*
- **Détail** d'une pollution (avec photo si une URL est fournie)
- **Déclaration** et **modification** via un même formulaire réactif, suivi d'un récapitulatif
- **Suppression** avec confirmation
- Seul l'auteur d'une pollution peut la modifier ou la supprimer

### Validation du formulaire
| Champ | Règles |
|---|---|
| Titre | requis, 3 caractères min. |
| Type | requis, liste déroulante |
| Description | requise, 10 caractères min. |
| Date d'observation | requise, date valide (validateur personnalisé) |
| Lieu | requis, 3 caractères min. |
| Latitude | requise, nombre entre -90 et 90 |
| Longitude | requise, nombre entre -180 et 180 |
| Photo (URL) | optionnelle, doit commencer par `http://` ou `https://` |

Les valeurs limites sont centralisées dans [pollution.constants.ts](src/app/core/constants/pollution.constants.ts).

### Favoris
- Ajout / retrait d'une pollution en favori depuis la liste
- Compteur de favoris affiché dans l'en-tête
- Page dédiée `/favoris`
- Persistance dans le `localStorage`

### Utilisateurs et authentification
- Inscription et connexion
- Affichage de l'utilisateur connecté dans l'en-tête, bouton de déconnexion
- Liste, détail, création, modification et suppression d'utilisateurs (routes protégées)

---

## Stack technique

| Outil | Version | Rôle |
|---|---|---|
| [Angular](https://angular.dev) | 20.3 | Framework front-end (composants *standalone*, nouvelle syntaxe de contrôle `@if` / `@for`) |
| [NGXS](https://www.ngxs.io) | 20 | Gestion d'état (+ plugins Logger et Redux DevTools) |
| RxJS | 7.8 | Programmation réactive (recherche, combinaison de flux) |
| TypeScript | 5.9 | Langage |
| Karma / Jasmine | — | Tests unitaires |
| Render | — | Hébergement du site statique |

---

## Architecture du projet

Le code suit une organisation **core / features / shared** :

```
src/
├── environments/              # URL de l'API selon l'environnement
│   ├── environment.ts         # dev  → http://localhost:3000/api
│   └── environment.prod.ts    # prod → https://tp-4-latest.onrender.com/api
└── app/
    ├── app.config.ts          # Providers : router, HttpClient + intercepteur, store NGXS
    ├── app.routes.ts          # Définition des routes
    ├── core/                  # Logique transverse, sans UI
    │   ├── constants/         # Types de pollution, règles de validation, config recherche
    │   ├── guards/            # authGuard : protège les routes privées
    │   ├── interceptors/      # authInterceptor : ajoute le JWT, gère les 401
    │   ├── models/            # Interfaces Pollution, Utilisateur, Auth
    │   ├── services/          # Appels HTTP : PollutionService, UtilisateurService, AuthService
    │   ├── store/             # States NGXS : auth, pollution
    │   └── state/             # State NGXS : favoris
    ├── features/              # Fonctionnalités métier
    │   ├── auth/              # Login, Register
    │   ├── pollution/
    │   │   ├── pages/         # Page conteneur (router-outlet)
    │   │   └── components/    # list, card, filters, form, recap, detail
    │   ├── favoris/           # Page des favoris
    │   └── utilisateur/       # list, detail, form (chargés en lazy loading)
    └── shared/
        └── components/header/ # Barre de navigation
```

### Composants principaux

| Composant | Rôle |
|---|---|
| `PollutionList` | Combine pollutions, recherche, filtres, favoris et utilisateur courant via `combineLatest` |
| `PollutionCardComponent` | Affichage d'une pollution dans la liste |
| `PollutionFiltersComponent` | Émet les filtres choisis vers la liste (`@Output`) |
| `PollutionForm` | Formulaire réactif de création / édition |
| `PollutionRecap` | Récapitulatif reçu via `@Input()` après soumission |
| `PollutionDetail` | Page de détail d'une pollution |
| `HeaderComponent` | Navigation, compteur de favoris, état de connexion |

---

## Routes de l'application

| Route | Composant | Protégée |
|---|---|:---:|
| `/` | redirection vers `/pollutions` | |
| `/login` | Connexion | |
| `/register` | Inscription | |
| `/pollutions` | Liste des pollutions | |
| `/pollutions/:id` | Détail | |
| `/pollutions/create` | Déclarer une pollution | ✅ |
| `/pollutions/:id/edit` | Modifier une pollution | ✅ |
| `/favoris` | Favoris (lazy loading) | ✅ |
| `/utilisateurs` | Liste / CRUD utilisateurs (lazy loading) | ✅ |

Les routes protégées utilisent `authGuard`, qui redirige vers `/login?returnUrl=...` si l'utilisateur n'est pas connecté.

---

## Gestion d'état (NGXS)

Trois *states* sont enregistrés dans [app.config.ts](src/app/app.config.ts) :

| State | Contenu | Actions principales |
|---|---|---|
| `AuthState` | utilisateur, token, `isAuthenticated`, loading, erreur | `Login`, `Register`, `Logout`, `InitAuth`, `GetCurrentUser` |
| `PollutionState` | liste des pollutions, pollution sélectionnée, loading, erreur | `LoadPollutions`, `LoadPollutionById`, `CreatePollution`, `UpdatePollution`, `DeletePollution` |
| `FavorisState` | identifiants des pollutions favorites | `LoadFavoris`, `AddFavoris`, `RemoveFavoris`, `ToggleFavoris`, `ClearFavoris` |

Chaque action asynchrone suit le schéma **Action → appel du service → `...Success` / `...Failure`**.
Au démarrage, le composant racine déclenche `InitAuth` (restauration de la session) et `LoadFavoris`.

Les plugins **Logger** et **Redux DevTools** permettent de suivre les actions dans la console et dans l'extension navigateur Redux DevTools.

---

## Authentification

1. À la connexion / inscription, l'API renvoie `{ user, token }`.
2. Le token JWT est stocké dans le `localStorage` et dans `AuthState`.
3. L'**intercepteur** [auth.interceptor.ts](src/app/core/interceptors/auth.interceptor.ts) ajoute l'en-tête `Authorization: Bearer <token>` à chaque requête.
4. En cas de réponse **401**, l'utilisateur est déconnecté et redirigé vers `/login`.
5. Au rechargement de la page, `InitAuth` revalide le token via `GET /utilisateur/me`.

---

## Environnements et API

L'URL de l'API est fournie par injection de dépendance (token `'API_URL'`) à partir du fichier d'environnement :

| Environnement | Fichier | URL |
|---|---|---|
| Développement (`ng serve`) | [environment.ts](src/environments/environment.ts) | `http://localhost:3000/api` |
| Production (`ng build`) | [environment.prod.ts](src/environments/environment.prod.ts) | `https://tp-4-latest.onrender.com/api` |

Endpoints utilisés :

| Méthode | Endpoint | Usage |
|---|---|---|
| GET | `/pollution` | Liste des pollutions |
| GET | `/pollution/:id` | Détail |
| GET | `/pollution/search?q=` | Recherche textuelle |
| POST | `/pollution` | Création |
| PUT | `/pollution/:id` | Modification |
| DELETE | `/pollution/:id` | Suppression |
| POST | `/utilisateur/register` | Inscription |
| POST | `/utilisateur/login` | Connexion |
| POST | `/utilisateur/logout` | Déconnexion |
| GET | `/utilisateur/me` | Utilisateur connecté |
| GET / POST / PUT / DELETE | `/utilisateur[/:id]` | CRUD utilisateurs |

---

## Installation et lancement

### Prérequis
- Node.js 20+ et npm
- Angular CLI (`npm install -g @angular/cli`) — optionnel, les scripts npm suffisent
- L'API back-end lancée sur `http://localhost:3000` pour le développement local

### Installation

```bash
npm install
```

### Serveur de développement

```bash
npm start
```

Puis ouvrir `http://localhost:4200/`. L'application se recharge automatiquement à chaque modification.

### Build de production

```bash
npm run build
```

Les fichiers générés sont placés dans `dist/tp3_2/browser`.

### Tests unitaires

```bash
npm test
```

---

## Déploiement

Le front-end est déployé sur **Render** en tant que site statique, configuré par [render.yaml](render.yaml) :

- commande de build : `npm install && npm run build`
- dossier publié : `./dist/tp3_2/browser`
- règle de réécriture `/* → /index.html` pour que le routage Angular fonctionne lors d'un accès direct à une URL

En production, l'application consomme l'API hébergée sur `https://tp-4-latest.onrender.com/api`.
