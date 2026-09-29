# Implementation Plan: Images dans les questions

**Branch**: `003-question-images` | **Date**: 2026-09-29 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/003-question-images/spec.md`

**Maquette** : aucune. Les écrans reprennent les composants et la direction visuelle de la 001
(brutalisme jaune et bleu).

## Summary

Permettre à l'auteur de placer jusqu'à 4 images, chacune avec sa description, au recto d'une
question. L'image devient la question et le verso reste du texte.

- Les images s'affichent en bloc au-dessus du texte du recto : lecture d'un sujet, mode cartes,
  séance de révision et éditeur. Elles s'agrandissent en plein écran, et leur description sert
  de repli.
- Un recto peut n'avoir que des images.
- Une image suit exactement la visibilité de son sujet et disparaît définitivement avec son
  retrait, sa question ou son sujet.

**Approche technique** :

- **Back** : la couche `back/functional/catalog` reçoit un modèle `QuestionImage`.
  - L'auteur envoie d'abord le fichier par une route multipart hors lomkit. Le back le
    réoriente, le réduit en trois variantes WebP sans métadonnées (`intervention/image`,
    Imagick) et le garde en attente.
  - L'image est ensuite rattachée dans le `mutate` de la question, qui porte la liste complète
    des images du recto par la relation `HasMany images`. Texte et images sont validés et
    enregistrés d'un seul coup.
  - Une route de diffusion contrôle chaque requête avec les règles de visibilité des questions
    et répond `404` à qui ne peut pas lire le sujet.
- **Web** : `functional/Catalog` fournit la galerie et la visionneuse, réutilisées par
  `functional/Learning`. `functional/Authoring` fournit le champ d'images, avec un envoi par
  `XMLHttpRequest` (progression, annulation) exposé par `technical/ApiClient`.

Les décisions et leurs alternatives sont dans [research.md](research.md).

## Affected Repos

Le back est le fournisseur et se fusionne en premier ; le web consomme ses endpoints et reste
ouvert tant que la merge request du back n'est pas fusionnée et déployée.

<!-- speckit-multirepo:begin -->
| Repo | Rôle | Ordre de fusion |
|---|---|---|
| back | API Laravel : envoi, traitement, rattachement, diffusion contrôlée et suppression des images | 1 |
| web | PWA Nuxt : champ d'images de l'éditeur, galerie et visionneuse en lecture, mode cartes et séance | 2 |
<!-- speckit-multirepo:end -->

## Technical Context

**Language/Version**: PHP 8.5 (Laravel 13) côté `back/` ; TypeScript 6 (Nuxt 4, Vue 3) côté `web/`

**Primary Dependencies**:

- `back/` :
  - déjà installés : `xefi/laravel-osdd`, `lomkit/laravel-rest-api` 2.23,
    `lomkit/laravel-access-control` 0.5, `stevebauman/purify` 6.3, `laravel/sanctum` ;
  - à ajouter : `intervention/image`, dernière version stable, avec le pilote Imagick
    (`php8.5-imagick` est déjà dans le runtime Sail). **L'ajout demande l'accord du
    développeur** (`back/AGENTS.md`).
- `web/` : aucune nouvelle dépendance. `v-dialog` de Vuetify sert de visionneuse, et
  `XMLHttpRequest` fait l'envoi avec progression.

**Storage**: PostgreSQL, une nouvelle table `question_images`. Fichiers WebP sur un disque
Laravel privé nommé par `QUESTION_IMAGES_DISK` (`local` par défaut, soit
`storage/app/private/question-images/`).

**Testing**: PHPUnit 12 par `./vendor/bin/sail artisan test` (`back/`), avec `Storage::fake` et
de vrais fichiers de test pour l'EXIF ; Vitest et `@nuxt/test-utils` avec happy-dom (`web/`) ;
Pint, ESLint et Prettier.

**Target Platform**: API Linux sous Docker ; navigateurs récents et PWA installée (Chrome,
Edge, Firefox, Safari, iOS et Android), qui affichent tous le WebP.

**Project Type**: application web, soit une API et une PWA, dans deux repos distincts

**Performance Goals**:

- une image de 5 Mo est traitée en moins de 2 secondes côté back (SC-002 : 10 secondes, envoi
  compris) ;
- une carte à 360 px charge une variante de 480 px, soit moins de 500 Ko par page (SC-003).

**Constraints**:

- visibilité identique à celle des questions, avec la même réponse `404` pour une image cachée
  ou inexistante (FR-016, principe VI) ;
- aucune métadonnée transmise et aucun fichier d'origine conservé (FR-014) ;
- aucune image dans le HTML mis en forme (FR-015) ;
- 360 px sans défilement horizontal ; cibles tactiles de 44 px ; description lue par les
  lecteurs d'écran (principe V) ;
- aucun cache partagé ni hors ligne des images (`private`, et `/api/*` en `NetworkOnly`).

**Scale/Scope**: 3 user stories et 19 exigences fonctionnelles. 4 surfaces web modifiées
(éditeur, lecture, mode cartes, séance) et 1 composant nouveau réutilisé partout (galerie et
visionneuse). Au plus 4 images par recto et 500 questions par sujet.

Aucune inconnue ne reste ouverte : toutes sont tranchées dans [research.md](research.md).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principe | Vérification | Avant conception | Après conception |
|---|---|---|---|
| I. La spec d'abord | Spec validée et poussée (`f8fb342`) ; chemins préfixés par `back/` ou `web/` | ✅ | ✅ |
| II. Couches OSDD | Tout le back dans `back/functional/catalog`, propriétaire des questions. Web : affichage dans `functional/Catalog` (réutilisé par `Learning`, qui dépend déjà du modèle `Question` de Catalog), saisie dans `functional/Authoring`, envoi générique dans `technical/ApiClient` | ✅ | ✅ `learning` n'est pas modifié côté back : la séance inclut `question.images` par la relation existante |
| III. Paquets imposés, API par contrat | Lecture et rattachement par lomkit (relation `HasMany`, `Control`, policy) ; web par modèles raom. Hors lomkit : l'envoi multipart et la diffusion du fichier, que lomkit ne sait pas porter, comme la désinscription de la 002. Back fusionné avant web | ✅ | ✅ `useUploadRequest` vise une route non lomkit, comme `useApiFetch` pour Fortify (research R5) |
| IV. Tests obligatoires | Un test par exigence ; la matrice de visibilité (5 états × 4 profils, plus l'image en attente) est couverte à 100 % | ✅ | ✅ Cas listés dans [quickstart.md](quickstart.md) et research R14 |
| V. Accessibilité, mobile d'abord | Description obligatoire et restituée ; repli textuel ; visionneuse fermée par Échap ; flèches de réordonnancement au clavier ; 360 px ; i18n dans chaque couche | ✅ | ✅ |
| VI. Sécurité, données personnelles | Contrôle du contenu réel ; limite de dimensions avant décodage ; réencodage sans métadonnées ; `404` neutre ; `private` ; `nosniff` ; Purify inchangé ; rattachement refusé pour l'image d'un autre compte | ✅ | ✅ |
| VII. Simplicité | Pas de couche `media` ni de médiathèque générique ; traitement synchrone sans état intermédiaire ; variantes fixes, sans URL renvoyée par l'API ; une seule dépendance ajoutée | ✅ | ✅ |

Aucun écart non justifié. La dépendance ajoutée est tracée dans Complexity Tracking.

## Project Structure

### Documentation (this feature)

```text
specs/003-question-images/
├── spec.md
├── plan.md              # ce fichier
├── research.md          # décisions techniques (Phase 0)
├── data-model.md        # table, états, visibilité, rattachement (Phase 1)
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
├── composer.json                         # + intervention/image
├── .env.example                          # + QUESTION_IMAGES_DISK=local
├── routes/console.php                    # + model:prune --model=QuestionImage toutes les heures
└── functional/
    └── catalog/
        ├── config/catalog.php            # images.* (disque, limites, largeurs, délai)
        ├── routes/api.php                # + POST question-images, GET question-images/{id}/{width}, Rest::resource question-images
        ├── lang/fr/                      # messages de validation et recto_empty
        ├── src/
        │   ├── Models/QuestionImage.php  # Prunable, HasControl ; Question::images()
        │   ├── Events/QuestionImageDeleted.php
        │   ├── Listeners/                # DeleteQuestionImages (QuestionDeleting), DeleteQuestionImageFiles
        │   ├── Images/                   # QuestionImageProcessor (orientation, variantes, métadonnées)
        │   ├── Http/Controllers/         # StoreQuestionImageController, ShowQuestionImageController
        │   ├── Http/Requests/            # StoreQuestionImageRequest
        │   ├── Rest/Resources/           # QuestionImageResource ; QuestionResource (+ relation, règles, suppression des absentes)
        │   ├── Rules/                    # VisibleTextLength étendue au recto avec images
        │   ├── Access/Controls/QuestionImageControl.php
        │   ├── Policies/QuestionImagePolicy.php
        │   └── Providers/CatalogServiceProvider.php
        ├── database/{migrations,factories}
        └── tests/
            ├── Feature/                  # QuestionImageUploadTest, QuestionImageAttachTest, QuestionImageVisibilityTest, QuestionImageDeletionTest, QuestionContentTest (img)
            └── fixtures/images/          # photo EXIF orientée, GIF animé, faux JPEG…

web/
├── technical/
│   └── ApiClient/
│       ├── app/composables/useUploadRequest.ts   # XHR : cookies, XSRF, progression, annulation
│       └── tests/
└── functional/
    ├── Catalog/
    │   ├── app/models/                   # QuestionImage ; Question (+ images)
    │   ├── app/components/               # QuestionImageGallery, QuestionImageViewer ; QuestionItem, faces du mode cartes
    │   ├── app/composables/              # useQuestionImageSources, useQuestionImageViewer ; include('images')
    │   ├── i18n/locales/fr.json
    │   └── tests/
    ├── Authoring/
    │   ├── app/components/               # QuestionImagesField, QuestionImageRow ; QuestionForm, QuestionCard
    │   ├── app/composables/              # useQuestionImages ; useQuestionDraft (payload, blocage, recto vide)
    │   ├── i18n/locales/fr.json
    │   └── tests/
    └── Learning/
        ├── app/components/SessionCard.vue      # galerie au-dessus du recto
        ├── app/composables/useReviewSession.ts # include('question.images')
        └── tests/
```

**Structure Decision**: les images n'ont pas de vie propre hors de leur question. Elles restent
dans la couche qui possède les questions (`catalog` au back, `Catalog` et `Authoring` au web),
sans nouvelle couche. L'envoi de fichier avec progression ne connaît pas le métier (une URL, un
fichier, des cookies) : il rejoint `technical/ApiClient`, qui détient déjà l'authentification
et le jeton XSRF.

## Complexity Tracking

> Dépendances ajoutées qui ne figurent pas dans les listes approuvées Xefi.

| Ajout | Pourquoi il est nécessaire | Alternative plus simple écartée parce que |
|---|---|---|
| `intervention/image` (pilote Imagick) | Orientation EXIF, redimensionnement, encodage WebP et retrait des métadonnées (FR-013, FR-014), avec une API testable | Imagick en direct : API bas niveau, plus de code et d'erreurs pour le même résultat. GD : décode une image de 8 000 px entièrement en mémoire. `spatie/laravel-medialibrary` : médiathèque polymorphe générique pour un seul usage |
| Deux routes hors lomkit (`POST` et `GET question-images`) | Un fichier multipart et un flux binaire ne passent pas par lomkit ni par raom | Base64 dans le `mutate` : +33 % de volume, sans progression ni annulation ; URL signées : une adresse copiée resterait lisible après un retrait |
