# Implementation Plan: Comptes, contenu et révision Leitner

**Branch**: `001-learning-content` | **Date**: 2026-09-24 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/001-learning-content/spec.md`

**Maquette**: [CINQ — Design 001](https://claude.ai/artifact/DxQK5ap9dAK6UYGuZURLsP)

## Summary

Livrer CINQ de bout en bout :

- des comptes par email et mot de passe, avec confirmation obligatoire de l'adresse ;
- un catalogue public de sujets et de questions mises en forme, écrits par les inscrits, rangés en
  catégories et en tags, et trouvables par recherche ;
- la modération des sujets par les administrateurs ;
- la révision des questions par la méthode Leitner (5 boîtes, 1, 2, 4, 8 et 16 jours) ;
- une PWA installable, utilisable dès 360 px.

**Approche technique** : le back Laravel expose une API `lomkit/laravel-rest-api`, découpée en
quatre couches OSDD (users, catalog, moderation, learning) ; l'authentification est une session
Sanctum SPA gérée par Fortify. Le web Nuxt consomme cette API uniquement par des modèles
`laravel-raom-nuxt`, et reprend en composants Vuetify thémés la direction visuelle de la maquette.
Les décisions et leurs alternatives sont dans [research.md](research.md).

## Affected Repos

Le back est le fournisseur et se fusionne en premier ; le web consomme ses endpoints et reste
ouvert tant que la merge request du back n'est pas fusionnée et déployée.

<!-- speckit-multirepo:begin -->
| Repo | Rôle | Ordre de fusion |
|---|---|---|
| back | API Laravel : comptes, catalogue, modération, révision | 1 |
| web | PWA Nuxt : tous les écrans de la maquette | 2 |
<!-- speckit-multirepo:end -->

## Technical Context

**Language/Version**: PHP 8.5 (Laravel 13) côté `back/` ; TypeScript 6 (Nuxt 4, Vue 3) côté `web/`

**Primary Dependencies**:

- `back/` :
  - déjà installés : `xefi/laravel-osdd`, `lomkit/laravel-rest-api`,
    `lomkit/laravel-access-control`, `spatie/laravel-permission`, `xefi/faker-php-laravel`,
    `laravel/boost` ;
  - à ajouter : `laravel/sanctum`, `laravel/fortify`, `stevebauman/purify`.
- `web/` :
  - déjà installés : `laravel-raom-nuxt`, `vuetify`, `@nuxtjs/i18n`, `@vite-pwa/nuxt`, `pinia`,
    `nuxt-osdd` ;
  - à ajouter : `@tiptap/vue-3`, `@tiptap/starter-kit`, `@tiptap/extension-link`, `date-fns`.

**Storage**: PostgreSQL 18, avec les extensions `pg_trgm` pour la recherche ; Redis pour le cache
et la file d'attente ; sessions en base (pilote `database`)

**Testing**: PHPUnit 12 par `./vendor/bin/sail artisan test` (`back/`), Vitest et
`@nuxt/test-utils` avec happy-dom (`web/`) ; Pint, ESLint et Prettier

**Target Platform**: API Linux sous Docker (Sail en développement) ; navigateurs récents sur
ordinateur et mobile, PWA installable sur Android et iOS

**Project Type**: application web, soit une API et une PWA, dans deux repos distincts

**Performance Goals**: recherche et ouverture d'un sujet en moins de 2 s pour 95 % des requêtes
avec 10 000 sujets et 500 000 questions (SC-005) ; aucune attente perceptible entre deux cartes
d'une séance (SC-009)

**Constraints**:

- pas de lecture hors ligne en 001 (FR-039) ;
- 360 px de large minimum, sans défilement horizontal (SC-008) ;
- accessibilité et typographie fluide (FR-052, principe V) ;
- aucun texte utilisateur codé en dur ;
- assainissement du contenu à l'enregistrement (principe VI).

**Scale/Scope**: 10 000 sujets, 500 000 questions ; 6 user stories, 52 exigences fonctionnelles ;
43 écrans de maquette (ordinateur, mobile, PWA)

Aucune inconnue ne reste ouverte : toutes sont tranchées dans [research.md](research.md).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principe | Vérification | Avant conception | Après conception |
|---|---|---|---|
| I. La spec d'abord | Spec validée et poussée ; chemins préfixés par `back/` ou `web/` ; les écarts relevés en conception remontent dans la spec | ✅ | ✅ |
| II. Couches OSDD | Back : `osdd:start` puis couches `functional/users`, `catalog`, `moderation`, `learning` ; web : couches `technical/*` et `functional/*` | ✅ | ✅ |
| III. Paquets imposés, API par contrat | lomkit pour tout le CRUD et les actions ; `Control` par modèle ; permissions, pas de noms de rôles ; web uniquement par modèles raom ; back fusionné avant web | ✅ | ✅ Fortify et Sanctum restent les seuls contrôleurs hors lomkit (authentification, pas du CRUD) |
| IV. Tests obligatoires | Un test par exigence ; `LeitnerSchedule` et périmètres d'accès couverts à 100 % | ✅ | ✅ Cas listés dans [quickstart.md](quickstart.md) |
| V. Accessibilité, mobile d'abord | Thème Vuetify et règles globales de la maquette ; i18n par couche | ✅ | ✅ |
| VI. Sécurité, données personnelles | Purify à l'enregistrement ; réponses identiques pour les emails inconnus ; 404 hors périmètre ; suppression de compte prévue avant l'ouverture au public | ✅ | ✅ |
| VII. Simplicité | Pas de moteur de recherche externe ; pas de couche technique à usage unique ; nouvelles dépendances justifiées ci-dessous | ✅ | ✅ |

Aucun écart non justifié. Les dépendances hors listes Xefi sont tracées dans Complexity Tracking.

## Project Structure

### Documentation (this feature)

```text
specs/001-learning-content/
├── spec.md
├── plan.md              # ce fichier
├── research.md          # décisions techniques (Phase 0)
├── data-model.md        # tables, statuts, règle Leitner (Phase 1)
├── quickstart.md        # guide de validation (Phase 1)
├── contracts/
│   └── api.md           # contrat back ↔ web (Phase 1)
├── checklists/
│   └── requirements.md
└── tasks.md             # Phase 2, créé par /speckit-tasks
```

### Source Code (repository root)

```text
back/
├── compose.yaml                       # + service mailpit
├── routes/api.php                     # Rest::resource(...) de chaque couche, si non monté par la couche
├── technical/
│   └── osdd/                          # couche technique créée par osdd:start
└── functional/
    ├── users/                         # comptes : User, Fortify, Sanctum, permissions, users:grant-admin
    │   ├── src/{Models,Actions,Console,Providers}
    │   ├── database/{migrations,factories,seeders}
    │   └── tests/{Feature,Unit}
    ├── catalog/                       # catégories, tags, sujets, questions, recherche, assainissement
    │   ├── src/{Models,Rest/Resources,Rest/Actions,Rest/Instructions,Controls,Enums,Listeners,Providers}
    │   ├── database/{migrations,factories,seeders}
    │   └── tests/{Feature,Unit}
    ├── moderation/                    # signalements, décisions
    │   ├── src/{Models,Rest/Resources,Rest/Actions,Controls,Enums,Providers}
    │   ├── database/{migrations,factories,seeders}
    │   └── tests/{Feature,Unit}
    └── learning/                      # apprentissages, progression, réponses, LeitnerSchedule
        ├── src/{Models,Domain,Rest/Resources,Rest/Actions,Rest/Instructions,Controls,Listeners,Jobs,Providers}
        ├── database/{migrations,factories,seeders}
        └── tests/{Feature,Unit}

web/
├── nuxt.config.ts                     # osdd : couches techniques et fonctionnelles déclarées
├── technical/
│   ├── ApiClient/                     # plugin $laravelRaom.fetch (cookies, CSRF, SSR), store de session
│   ├── Theme/                         # thème Vuetify CINQ, polices, règles CSS globales, app.vue
│   ├── Pwa/                           # manifeste, service worker, invitation d'installation, hors ligne, mise à jour
│   ├── RichText/                      # éditeur TipTap restreint et rendu du HTML assaini
│   ├── Internationalization/          # existant
│   ├── State/                         # existant
│   └── Vuetify/                       # existant
└── functional/
    ├── Home/                          # accueil (landing)
    ├── Account/                       # connexion, inscription, confirmation, mot de passe, menu du compte
    ├── Catalog/                       # catalogue, recherche, sujet, mode cartes, contenu introuvable, signalement
    ├── Authoring/                     # mes sujets, nouveau sujet, éditeur
    ├── Moderation/                    # file de modération, catégories, historique
    └── Learning/                      # mes révisions, séance, bilan
```

Chaque couche web contient `app/{pages,components,composables}`, `models/` (modèles raom de ses
ressources), `i18n/locales/fr.json` et `tests/`.

**Structure Decision**: deux repos indépendants, déjà déclarés dans `repos.yml`. Le back suit
`xefi/laravel-osdd` (couches dans `back/functional/` et `back/technical/`, et non dans
`back/layers/` comme l'écrit à tort `back/CLAUDE.md`, que la première tâche corrige). Le web suit
`nuxt-osdd`, où chaque domaine de la maquette devient une couche fonctionnelle.

## Renommage en CINQ

Le produit s'appelle désormais **CINQ** (anciennement « Learn Fell »). Les dépôts GitHub gardent
leur nom (`learn-fell-api`, `learn-fell-web`, `learn-fell-workspace`). Dans le code, le
renommage fait partie de la 001, sur la branche de feature :

- `back/` : `APP_NAME=CINQ` dans `.env.example`, expéditeur des emails, `back/CLAUDE.md` ;
- `web/` : `name` et `short_name` du manifeste PWA (`web/technical/Pwa/nuxt.config.ts`), textes
  de `web/functional/Home/i18n/locales/fr.json` et du test associé, `web/CLAUDE.md`.

## Règles d'implémentation issues de la maquette

Ces règles viennent de la relecture de la maquette et s'appliquent à tous les écrans du web :

1. **Typographie fluide** : tailles en `rem`, avec `clamp()` sur mobile ; aucun texte sous 12 px ;
   le réglage de taille de texte du téléphone est respecté (FR-052).
2. **Dimensions** : `box-sizing: border-box` sur tous les liens, boutons et champs. Un lien et un
   bouton de même hauteur déclarée font la même hauteur.
3. **Onglets** : chaque entrée de menu réserve l'espace du soulignement de l'état actif. Seule sa
   couleur change ; le texte ne bouge pas.
4. **Boutons côte à côte** : dans une carte, ils se partagent sa largeur (`flex: 1 1 0`) et
   s'alignent sur les bords du contenu au-dessus.
5. **Césure** : les titres passent à la ligne avec `hyphens: auto` (`lang="fr"`) plutôt que de
   déborder.
6. **Direction visuelle** : couleurs, polices, bordures de 4 px et ombres portées de la maquette,
   pas le design system Xefi. Le thème Vuetify désactive les angles arrondis et l'élévation.

## Complexity Tracking

> Dépendances ajoutées qui ne figurent pas dans les listes approuvées Xefi.

| Ajout | Pourquoi il est nécessaire | Alternative plus simple écartée parce que |
|---|---|---|
| `laravel/sanctum`, `laravel/fortify` | Session SPA sécurisée et flux de compte complets (FR-001 à FR-006) | Des contrôleurs écrits à la main réimplémenteraient la vérification d'email, le verrouillage et la réinitialisation ; `tymon/jwt-auth` obligerait à stocker un jeton dans la PWA |
| `stevebauman/purify` | Assainir le HTML des questions à l'enregistrement (FR-015, principe VI) | `strip_tags` ne filtre pas les attributs (`onclick`, `javascript:`) ; un filtre maison est risqué |
| `@tiptap/vue-3` et extensions | Éditeur recto et verso limité au gras, à l'italique, aux listes, au code et aux liens (maquette de l'éditeur) | Vuetify n'a pas d'éditeur riche ; un `contenteditable` maison est coûteux en sécurité et en accessibilité |
| `date-fns` | Dates relatives (« revient dans 4 jours », « il y a 2 jours ») | Liste Xefi approuvée : aucun écart, noté pour mémoire |
