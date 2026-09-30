---

description: "Tâches de la feature 003 — Images dans les questions (CINQ)"
---

# Tasks: Images dans les questions

**Input**: Documents de conception dans `specs/003-question-images/`

**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md),
[data-model.md](data-model.md), [contracts/api.md](contracts/api.md), [quickstart.md](quickstart.md)

**Tests**: obligatoires.
- La constitution (principe IV) exige un test automatisé par exigence fonctionnelle.
- La matrice de visibilité (SC-004 : 5 états de sujet × 4 profils, plus l'image en attente)
  doit être couverte à 100 %.
- Dans chaque story, les tests sont écrits en premier et doivent échouer avant
  l'implémentation.

**Organization**: une phase par user story, dans l'ordre de la spec (les trois sont P1). Dans
chaque story, le back (fournisseur) passe avant le web (consommateur).

**Maquette de référence**: aucune. Les écrans reprennent les composants et la direction visuelle de
la 001 (brutalisme jaune et bleu).

## Format: `[ID] [P?] [Story] Description`

- **[P]** : parallélisable (fichiers différents, aucune dépendance sur une tâche non terminée)
- **[Story]** : story concernée (US1 à US3, voir spec.md)
- Tous les chemins commencent par leur repo : `back/…` ou `web/…`

## Conventions de chemins

- **Back** : tout se passe dans la couche existante `back/functional/catalog/` (espace de noms
  `Functional\Catalog\`), avec `src/`, `database/{migrations,factories}`, `routes/api.php`,
  `config/` et `tests/{Feature,Unit}`.
  - Les messages d'erreur métier sont dans `back/technical/osdd/lang/fr/errors.php`, et ceux de
    validation dans `back/technical/osdd/lang/fr/validation.php`.
  - Toutes les commandes passent par `./vendor/bin/sail`.
  - Lire `back/CLAUDE.md` et `back/AGENTS.md` avant la première modification.
- **Web** : couches existantes `web/functional/Catalog/`, `web/functional/Authoring/`,
  `web/functional/Learning/` et `web/technical/ApiClient/`.
  - Les tests sont dans `<couche>/tests/*.nuxt.spec.ts` et les textes dans
    `<couche>/i18n/locales/fr.json`. Une clé est la phrase anglaise en minuscules, et sa
    valeur le texte français.
  - Toutes les commandes passent par `pnpm`.
  - Lire `web/CLAUDE.md` avant la première modification.
- Les branches `003-question-images` de `back/` et `web/` sont créées par le hook
  `speckit.multirepo.branch` avant la première tâche.

---

## Phase 1: Setup (infrastructure partagée)

**Purpose**: installer la dépendance de traitement d'image, déclarer la configuration de la
couche et les fichiers de test.

- [X] T001 Installer `intervention/image` (dernière version stable, approuvée par le développeur le 2026-09-29) avec `./vendor/bin/sail composer require intervention/image` dans `back/composer.json`, puis ajouter la même contrainte au `require` de `back/functional/catalog/composer.json`. Vérifier avec `./vendor/bin/sail composer show intervention/image` la version installée et, avec `./vendor/bin/sail php -m`, que `imagick` et `exif` sont chargés. Reporter la version dans la ligne « Primary Dependencies » de `specs/003-question-images/plan.md`
- [X] T002 Créer `back/functional/catalog/config/catalog.php` avec `images.disk` = `env('QUESTION_IMAGES_DISK', 'local')`, `images.max_per_recto` = 4, `images.max_kilobytes` = 5120, `images.max_side` = 8000, `images.variant_widths` = `[480, 960, 1600]`, `images.webp_quality` = 80 et `images.pending_hours` = 24 ; la charger par `mergeConfigFrom` dans le `register()` de `back/functional/catalog/src/Providers/CatalogServiceProvider.php`, à côté de la surcharge de `purify`
- [X] T003 [P] Ajouter `QUESTION_IMAGES_DISK=local` dans `back/.env.example`
- [X] T004 [P] Ajouter les fichiers de test dans `back/functional/catalog/tests/fixtures/images/`, chacun de moins de 200 Ko, sauf `too-large.jpg` qui est généré au vol dans le test :
  - `rotated-with-gps.jpg` : JPEG avec balise EXIF `Orientation = 6`, coordonnées GPS, marque et modèle d'appareil, date de prise de vue ;
  - `transparent.png`, `photo.webp`, `animated.gif`, `animated.webp`, `vector.svg` ;
  - `text-renamed.jpg` : fichier texte renommé en `.jpg`.

---

## Phase 2: Foundational (prérequis bloquants)

**Purpose**: table et modèle, traitement des fichiers, routes d'envoi et de diffusion avec leur
contrôle d'accès, relation lomkit, et côté web : le modèle raom, l'envoi avec progression et la
galerie d'affichage de base.

**⚠️ CRITICAL**: aucune story ne commence avant la fin de cette phase.

### Back

- [X] T005 Créer la migration `back/functional/catalog/database/migrations/2026_09_29_160000_create_question_images_table.php` avec :
  - `question_id` FK `questions`, nullable, `restrictOnDelete`, indexée ;
  - `uploader_id` FK `users`, `restrictOnDelete`, indexée ;
  - `alt` varchar(250) nullable ;
  - `position` smallint nullable ;
  - `width` smallint et `height` smallint ;
  - `variant_widths` jsonb ;
  - timestamps ;
  - un index composé (`question_id`, `created_at`).

  Aucune unicité sur (`question_id`, `position`) et aucune cascade (data-model.md).
- [X] T006 [P] Créer l'événement `QuestionImageDeleted` dans `back/functional/catalog/src/Events/QuestionImageDeleted.php`, sur le modèle de `QuestionDeleting.php`
- [X] T007 Créer le modèle `QuestionImage` dans `back/functional/catalog/src/Models/QuestionImage.php` :
  - fillable `alt`, `position` ;
  - cast `variant_widths` en `array` ;
  - relations `question()` et `uploader()` (`Functional\Users\Models\User`) ;
  - `isPending()` (`question_id` nul) ;
  - `directory()`, qui renvoie `question-images/{id}` ;
  - `$dispatchesEvents = ['deleted' => QuestionImageDeleted::class]` ;
  - le trait `HasControl` ;
  - le trait `Prunable`, avec `prunable()` = `whereNull('question_id')->where('created_at', '<', now()->subHours(config('catalog.images.pending_hours')))` (FR-018).

  Ajouter `images()` (`hasMany`, `orderBy('position')`) dans `back/functional/catalog/src/Models/Question.php`
- [X] T008 [P] Créer `back/functional/catalog/database/factories/QuestionImageFactory.php`, avec les données de `faker()` (`xefi/faker-php-laravel`). États :
  - `pending()` : `question_id` nul, `alt` et `position` nuls ;
  - `attachedTo(Question $question, int $position)` : `alt` entre 1 et 250 caractères.

  L'état par défaut écrit de fausses variantes sur le disque de `catalog.images.disk` par `afterCreating`, pour les tests de diffusion
- [X] T009 Écrire `back/functional/catalog/tests/Unit/QuestionImageProcessorTest.php` à partir des fixtures T004. Il vérifie :
  - que `rotated-with-gps.jpg` ressort droit (largeur et hauteur inversées) ;
  - que chaque variante est un WebP sans bloc EXIF, XMP ni ICC de localisation ;
  - qu'une image de 700 px donne `variant_widths` `[480, 960]`, le fichier 960 faisant 700 px, sans agrandissement (data-model.md) ;
  - que `width` et `height` renvoyés sont ceux de la plus grande variante ;
  - que `animated.gif` et `animated.webp` lèvent l'exception de format.
- [X] T010 Créer `QuestionImageProcessor` dans `back/functional/catalog/src/Images/QuestionImageProcessor.php`, avec `intervention/image` et le pilote Imagick (research R3). La méthode `process(UploadedFile $file, QuestionImage $image): void` :
  - refuse un fichier à plusieurs images (animé) par `BusinessRuleException('image_invalid_format', 422)` ;
  - corrige l'orientation ;
  - pour chaque largeur de `catalog.images.variant_widths` inférieure ou égale à la largeur source, plus la largeur source si elle est plus petite que 1 600, encode un WebP de qualité `catalog.images.webp_quality`, sans métadonnées, dans `{directory}/{largeur}.webp` sur le disque `catalog.images.disk` ;
  - remplit `width`, `height` et `variant_widths`.

  Le fichier d'origine n'est jamais écrit (FR-013, FR-014). Faire passer T009
- [X] T011 [P] Ajouter dans `back/technical/osdd/lang/fr/errors.php` les codes :
  - `image_invalid_format` : « Choisissez une image au format JPEG, PNG ou WebP. » ;
  - `recto_empty` : « Ajoutez un texte ou une image au recto. » ;
  - `recto_image_limit` : « Un recto porte au plus 4 images. ».

  Ajouter dans `back/technical/osdd/lang/fr/validation.php` les messages propres aux attributs `file` (mimes, max, dimensions : « L'image ne doit pas dépasser 5 Mo. », « L'image ne doit pas dépasser 8 000 pixels de côté. ») et `alt` (required, max : « Décrivez cette image. », « La description ne doit pas dépasser 250 caractères. »), en suivant le format existant du fichier
- [X] T012 Créer `StoreQuestionImageRequest` dans `back/functional/catalog/src/Http/Requests/StoreQuestionImageRequest.php`. Règles : `file` `required|file|mimes:jpg,jpeg,png,webp|max:5120|dimensions:max_width=8000,max_height=8000`, les valeurs étant lues dans `config('catalog.images.*')` (research R4)
- [X] T013 Créer `StoreQuestionImageController` dans `back/functional/catalog/src/Http/Controllers/StoreQuestionImageController.php`, sur le modèle de `back/functional/reminders/src/Http/Controllers/UnsubscribeController.php`. Dans une transaction, il crée une `QuestionImage` en attente avec `uploader_id` = l'utilisateur courant, appelle `QuestionImageProcessor::process`, et répond `201 { data: { id, width, height, variant_widths } }` (contracts/api.md §1). En cas d'échec du traitement, la ligne est annulée et le dossier effacé
- [X] T014 Créer `QuestionImageAccess` dans `back/functional/catalog/src/Support/QuestionImageAccess.php`, avec `canView(?User $user, QuestionImage $image): bool`. Les règles sont celles du tableau « Règles de visibilité » de data-model.md :
  - image en attente : seulement si `$user` est son `uploader` ;
  - image rattachée : si la question est lisible par `$user`, c'est-à-dire si `Question::query()` restreint comme dans `QuestionResource::searchQuery()` (visiteur : sujet `Published` ; connecté : `controlled()`) contient `question_id`.

  Aucune règle n'est dupliquée : la requête de `searchQuery()` est extraite dans une méthode statique de `QuestionResource` ou de `QuestionControl`, réutilisée par les deux
- [X] T015 Créer `ShowQuestionImageController` dans `back/functional/catalog/src/Http/Controllers/ShowQuestionImageController.php`, pour `GET question-images/{id}/{width}`, avec `width` ∈ `catalog.images.variant_widths` :
  - `404` si l'image est absente, si `QuestionImageAccess::canView` est faux ou si la largeur est hors liste, toujours avec le même corps (FR-016) ;
  - sinon, le flux du fichier de la largeur demandée, ou de la plus grande disponible si elle n'a pas été produite ;
  - en-têtes `Content-Type: image/webp` et `X-Content-Type-Options: nosniff` ;
  - `Cache-Control: private, max-age=3600` si le sujet est `Published`, sinon `private, no-store`.
- [X] T016 Déclarer dans `back/functional/catalog/routes/api.php`, à côté des `Rest::resource` existants :
  - `Route::post('question-images', StoreQuestionImageController::class)`, avec `auth:sanctum`, `verified` et `throttle:30,1` ;
  - `Route::get('question-images/{id}/{width}', ShowQuestionImageController::class)`, avec `whereNumber` sur les deux paramètres et le middleware de session Sanctum, mais sans `auth` (les visiteurs voient les images publiées).

  Vérifier avec `./vendor/bin/sail artisan route:list --path=question-images` qu'aucune route ne masque les routes lomkit `question-images/search`
- [X] T017 Créer `QuestionImageControl` dans `back/functional/catalog/src/Access/Controls/QuestionImageControl.php`. Il reprend les périmètres de `QuestionControl.php` à travers `question` (`subjects.moderate` : tout ; lecture : sujets publiés ou dont on est l'auteur), et exclut toujours les images en attente (`whereNotNull('question_id')`)
- [X] T018 Créer `QuestionImagePolicy` dans `back/functional/catalog/src/Policies/QuestionImagePolicy.php`, qui étend `PubliclyReadablePolicy`. Elle refuse `create`, `update` et `delete` directs. `attach(User $user, Question $question, QuestionImage $image)` est vrai si l'image est en attente avec `uploader_id` = `$user->id`, ou si `question_id` = `$question->id` (research R6). L'enregistrer dans `CatalogServiceProvider`
- [X] T019 Créer `QuestionImageResource` dans `back/functional/catalog/src/Rest/Resources/QuestionImageResource.php` :
  - champs `id`, `question_id`, `alt`, `position`, `width`, `height`, `variant_widths`, sans `uploader_id` ;
  - `updateRules` : `alt` `required|string|max:250` après rognage, `position` `required|integer|min:0|max:3`.

  Créer `back/functional/catalog/src/Rest/Controllers/QuestionImagesController.php` sur le modèle de `QuestionsController.php`, et le déclarer dans `routes/api.php` pour la seule lecture. Toute écriture directe répond `403` (contracts/api.md §3)
- [X] T020 Déclarer la relation `HasMany::make('images', QuestionImageResource::class)` dans `back/functional/catalog/src/Rest/Resources/QuestionResource.php`, sans encore changer ses règles. Vérifier par un test de `QuestionContentTest.php` que `include: [{relation: 'images'}]` renvoie les images triées par `position`

### Web

- [X] T021 [P] Créer le modèle raom `QuestionImage` dans `web/functional/Catalog/app/models/QuestionImage.ts` : `@Resource('question-images')`, champs `id`, `question_id`, `alt`, `position`, `width`, `height`, `variant_widths`. Ajouter `images = HasMany(() => QuestionImage, 'images')` dans `web/functional/Catalog/app/models/Question.ts`, et couvrir la déclaration dans `web/functional/Catalog/tests/models.nuxt.spec.ts`
- [X] T022 [P] Créer `useUploadRequest` dans `web/technical/ApiClient/app/composables/useUploadRequest.ts`, avec son test `web/technical/ApiClient/tests/uploadRequest.nuxt.spec.ts` (`XMLHttpRequest` doublé). `upload(path, file)` renvoie `{ progress: Ref<number>, promise, abort() }` :
  - `XMLHttpRequest` vers `apiBaseUrl + path`, en `FormData` avec le champ `file` ;
  - `withCredentials = true` et en-tête `X-XSRF-TOKEN` lu par `web/technical/ApiClient/app/utils/xsrfToken.ts` ;
  - appel préalable à `/sanctum/csrf-cookie` si le cookie manque ;
  - progression tirée de `upload.onprogress` ;
  - rejet avec les erreurs `422` au format de `useApiError`, et rejet distinct en cas d'annulation (research R5).
- [X] T023 [P] Créer `useQuestionImageSources` dans `web/functional/Catalog/app/composables/useQuestionImageSources.ts`, avec son test. `(image)` renvoie `{ src, srcset, width, height }` :
  - `src` : `${apiBaseUrl}/question-images/{id}/480` ;
  - `srcset` : uniquement les largeurs de `variant_widths` suivies de `w` ;
  - `width`, `height` : les valeurs de l'image.
- [X] T024 Créer `QuestionImageGallery` dans `web/functional/Catalog/app/components/QuestionImageGallery.vue`, avec le test `web/functional/Catalog/tests/QuestionImageGallery.nuxt.spec.ts`. Le composant :
  - prend `images` et `eager?: boolean`, et n'affiche rien pour une liste vide ;
  - affiche les images triées par `position` : 1 image en pleine largeur, 2 à 4 en grille de deux colonnes, sans défilement horizontal à 360 px ;
  - pose sur chaque `<img>` `src`, `srcset`, `sizes` (`(max-width: 599px) 100vw, 600px` pour une image, `50vw` et `300px` en grille), `width`, `height`, `alt`, `crossorigin="use-credentials"`, et `loading="lazy"` sauf si `eager`.

  Styles en BEM et SCSS scopé, avec les variables `--cinq-*` (research R11)

**Checkpoint**: l'envoi, la diffusion contrôlée et la galerie existent. Les stories peuvent
commencer.

---

## Phase 3: User Story 1 - Ajouter des images au recto d'une question (Priority: P1) 🎯 MVP

**Goal**: l'auteur, ou un administrateur, ajoute jusqu'à 4 images décrites au recto, les
réordonne, les remplace, les retire, et peut enregistrer une carte « image seule ».

**Independent Test**: dans un brouillon, ajouter 2 images décrites au recto, inverser leur
ordre, enregistrer, recharger la page et retrouver les 2 images dans le nouvel ordre avec leur
description.

### Tests for User Story 1 ⚠️

- [X] T025 [P] [US1] Écrire `back/functional/catalog/tests/Feature/QuestionImageUploadTest.php` (`Storage::fake`). Il couvre :
  - `201` et variantes écrites pour un JPEG, un PNG et un WebP ;
  - `422` pour 7 Mo (`UploadedFile::fake()->create(..., 7168, 'image/jpeg')`), pour 9 000 px de côté, pour `vector.svg`, `animated.gif`, `animated.webp` et `text-renamed.jpg`, avec les messages de contracts/api.md §1 (FR-002) ;
  - `401` pour un visiteur ;
  - `403` pour une adresse non confirmée ;
  - `429` au 31e envoi dans la minute.
- [X] T026 [P] [US1] Écrire `back/functional/catalog/tests/Feature/QuestionImageAttachTest.php`, avec les helpers de `back/functional/catalog/tests/Concerns/WritesCatalog.php` (ajouter un helper `mutateQuestionWithImages`). Il couvre :
  - création et mise à jour d'une question avec 1 à 4 images, `alt` et `position` enregistrés (FR-001, FR-005) ;
  - 5 images : `recto_image_limit` (FR-003) ;
  - `alt` absent, vide ou de 251 caractères : `422` (FR-004) ;
  - positions en double ou hors de 0 à 3 : `422` ;
  - recto vide avec une image : accepté ; recto vide sans image : `recto_empty` (FR-008) ;
  - réordonnancement, puis image absente de la liste : supprimée, fichier compris (FR-005, FR-017) ;
  - `mutate` sans `relations.images` : images conservées, et un recto vide reste valide s'il a déjà des images ;
  - image en attente d'un autre compte, ou image d'une autre question : `403`, rien n'est enregistré ;
  - un administrateur (`subjects.moderate`) ajoute, remplace et retire une image sur le sujet d'un autre ;
  - l'auteur d'un sujet retiré est bloqué par `RetiredSubjectLock`.
- [X] T027 [P] [US1] Ajouter à `back/functional/learning/tests/Feature/ContentChangesTest.php` un test : une carte en boîte 3, due dans 4 jours, garde sa boîte et son échéance quand l'auteur ajoute, remplace, réordonne puis retire une image de sa question (FR-019)
- [X] T028 [P] [US1] Écrire `web/functional/Authoring/tests/QuestionImagesField.nuxt.spec.ts`, avec `useUploadRequest` doublé. Il couvre :
  - refus avant envoi d'un GIF, d'un SVG, d'un fichier de plus de 5 Mo et d'un cinquième fichier, avec le message des formats ou des limites (FR-002, FR-003) ;
  - `accept="image/jpeg,image/png,image/webp"` sans attribut `capture` (FR-006) ;
  - barre de progression et bouton « Annuler » pendant l'envoi (FR-007) ;
  - erreur d'envoi affichée sans perte du reste ;
  - champ de description avec compteur de 250 ;
  - flèches monter et descendre, avec aria-label « move image {n} up/down » ;
  - « Remplacer » et « Retirer » ;
  - bouton d'ajout désactivé quand `isOffline` est vrai.
- [X] T029 [P] [US1] Étendre `web/functional/Authoring/tests/SubjectEditor.nuxt.spec.ts`, avec `stubAuthoringApi` et un `aQuestionImage` à ajouter dans `web/functional/Authoring/tests/support/authoringApi.ts`. Il vérifie que le `mutate` de `questions` porte `relations.images` complet (`operation: 'update'`, `key`, `attributes.alt`, `attributes.position`), que l'enregistrement est bloqué pendant un envoi ou tant qu'une description manque, qu'un recto vide avec image est enregistrable, que les images sont chargées par `include('images')` et visibles dans `QuestionCard`, et que le message de `recto_empty` s'affiche

### Implementation for User Story 1

- [X] T030 [US1] Modifier `back/functional/catalog/src/Rules/VisibleTextLength.php`, ou créer à côté `RectoContent.php` si la règle est partagée avec le verso. Le recto est valide avec 1 à 5 000 caractères visibles, ou avec 0 caractère visible si la question aura au moins une image : `relations.images` du payload s'il est présent, sinon les images déjà rattachées (R7)
- [X] T031 [US1] Dans `back/functional/catalog/src/Rest/Resources/QuestionResource.php` :
  - appliquer la règle T030 au recto en création et en mise à jour, le verso restant inchangé ;
  - ajouter les règles de `relations.images` : `array|max:4` (`recto_image_limit`), positions distinctes de 0 à 3 ;
  - dans `mutating()`, garder `RetiredSubjectLock` et la limite de 500 ;
  - après l'application des relations, dans la même transaction, supprimer comme modèles les images de la question absentes de `relations.images`, quand ce champ est présent.

  Faire passer T026
- [X] T032 [US1] Créer le listener `DeleteQuestionImageFiles` dans `back/functional/catalog/src/Listeners/DeleteQuestionImageFiles.php`, pour `QuestionImageDeleted` : `DB::afterCommit(fn () => Storage::disk(config('catalog.images.disk'))->deleteDirectory($image->directory()))`. L'enregistrer dans `CatalogServiceProvider`
- [X] T033 [US1] Faire passer T025 et T027 : ajuster `StoreQuestionImageController` et les messages si besoin, sans toucher à `back/functional/learning/`. Puis lancer `./vendor/bin/sail artisan test functional/catalog functional/learning` et `./vendor/bin/sail bin pint --dirty`
- [X] T034 [US1] Créer `useQuestionImages` dans `web/functional/Authoring/app/composables/useQuestionImages.ts`. Il gère une liste d'éléments `{ image: QuestionImage | null, file?: File, previewUrl, progress, status: 'uploading' | 'ready' | 'failed', alt }` et offre :
  - `add(files)`, avec contrôle du type, de la taille et du nombre avant l'envoi (limites reprises de contracts/api.md) ;
  - `upload` par `useUploadRequest('/question-images', file)` ;
  - `cancel(index)`, `retry(index)`, `replace(index, file)`, `remove(index)`, `move(index, step)` ;
  - `isUploading`, `hasMissingAlt`, `hasImages` ;
  - `toRelationPayload()`, qui assigne `alt` et `position` sur les modèles `QuestionImage` et les place dans `question.images`.

  Les URL d'aperçu locales sont libérées par `URL.revokeObjectURL`
- [X] T035 [US1] Créer `QuestionImageRow` dans `web/functional/Authoring/app/components/QuestionImageRow.vue`. Chaque image affiche : aperçu, `v-textarea` de description (compteur de 250, erreur « describe this image. »), flèches de monter et descendre, « Remplacer », « Retirer », barre de progression et « Annuler » pendant l'envoi, message d'échec et « Réessayer ». Les cibles tactiles font 44 px
- [X] T036 [US1] Créer `QuestionImagesField` dans `web/functional/Authoring/app/components/QuestionImagesField.vue`. Il porte le bouton « Ajouter une image », un `<input type="file" accept="image/jpeg,image/png,image/webp">` caché sans `capture`, désactivé à 4 images ou quand `useConnectionStatus().isOffline` est vrai, ainsi que la liste des `QuestionImageRow` et le message d'aide sur les formats et les limites. Faire passer T028
- [X] T037 [US1] Intégrer le champ :
  - dans `web/functional/Authoring/app/components/QuestionForm.vue` : placer `QuestionImagesField` au-dessus de l'éditeur du recto ;
  - dans `web/functional/Authoring/app/composables/useQuestionDraft.ts` : initialiser les images depuis la question éditée, considérer la question comme non vide dès qu'elle a un texte au recto ou une image, bloquer `save()` si `isUploading` ou `hasMissingAlt`, envoyer toujours la liste complète des images, et afficher le message de `recto_empty` ;
  - dans `web/functional/Authoring/app/composables/useSubjectEditor.ts` : charger les questions avec `include('images')` ;
  - dans `web/functional/Authoring/app/components/QuestionCard.vue` : afficher `QuestionImageGallery`.

  Faire passer T029
- [X] T038 [P] [US1] Ajouter les textes de l'éditeur dans `web/functional/Authoring/i18n/locales/fr.json`, au vouvoiement : « Ajouter une image », « Description de l'image », « Décrivez cette image. », « Remplacer », « Retirer », « Annuler l'envoi », « Réessayer », « Monter l'image {n} », « Descendre l'image {n} », « Formats acceptés : JPEG, PNG ou WebP, 5 Mo au plus, 4 images par recto. », « L'envoi de l'image a échoué. », « Ajoutez un texte ou une image au recto. »

**Checkpoint**: US1 est complète. Un auteur crée une carte « image seule » et la retrouve dans
l'éditeur.

---

## Phase 4: User Story 2 - Voir les images en lisant et en révisant (Priority: P1)

**Goal**: les images s'affichent au-dessus du recto en lecture, en mode cartes et en séance ;
elles s'agrandissent en plein écran, et leur description est lue ou affichée en repli.

**Independent Test**: un visiteur voit les 2 images d'une question d'un sujet publié. Un
apprenant les voit au recto en séance, en agrandit une, puis affiche le verso et répond.

### Tests for User Story 2 ⚠️

- [X] T039 [P] [US2] Étendre `web/functional/Catalog/tests/QuestionImageGallery.nuxt.spec.ts`. Il vérifie :
  - qu'un clic ou la touche Entrée sur une image ouvre la visionneuse (`role="dialog"`, image de 1 600 px), et que la touche Échap ou « Fermer » la referme en rendant le focus à l'image (FR-011) ;
  - qu'un événement `error` sur une image la remplace par un cadre qui affiche sa description (FR-012) ;
  - que `loading` vaut `eager` quand la prop le demande.
- [X] T040 [P] [US2] Étendre `web/functional/Catalog/tests/SubjectPage.nuxt.spec.ts`, avec `web/functional/Catalog/tests/support/catalogApi.ts`. Il vérifie :
  - que les questions sont demandées avec `include` `images` ;
  - que `QuestionItem` affiche les images au-dessus du texte du recto, dans l'ordre ;
  - qu'une question sans image est rendue comme avant ;
  - qu'en mode cartes (`/sujets/[id]/cartes`), les images sont sur la face recto (FR-009, FR-010).
- [X] T041 [P] [US2] Étendre `web/functional/Learning/tests/ReviewSession.nuxt.spec.ts`, avec `web/functional/Learning/tests/support/learningApi.ts`. Il vérifie :
  - que `card-progress` est demandé avec `include` `question.images` ;
  - que `SessionCard` affiche les images en `eager` avant que le verso soit affiché ;
  - que les boutons « Je savais » et « Je ne savais pas » restent atteignables ;
  - qu'une image en échec affiche sa description et que la séance continue.
- [X] T042 [P] [US2] Ajouter à `back/functional/learning/tests/Feature/` (fichier de la séance, par exemple `ReviewSessionTest.php`) un test : `card-progress/search` avec `include: [{relation: 'question.images'}]` renvoie les images d'une carte de sujet publié, triées

### Implementation for User Story 2

- [X] T043 [US2] Créer `useQuestionImageViewer` dans `web/functional/Catalog/app/composables/useQuestionImageViewer.ts`, avec `open(image, trigger)`, `close()` et le retour du focus sur l'élément déclencheur, puis `QuestionImageViewer` dans `web/functional/Catalog/app/components/QuestionImageViewer.vue` : un `v-dialog` `fullscreen` qui montre la variante de 1 600 px (ou la plus grande), sa description en légende et un bouton « Fermer » de 44 px. Le dialogue se ferme par Échap (comportement par défaut de `v-dialog`), par le bouton ou par un toucher sur l'image, et coupe l'animation si `prefers-reduced-motion`
- [X] T044 [US2] Dans `web/functional/Catalog/app/components/QuestionImageGallery.vue` :
  - rendre chaque image activable (`button` de 44 px au moins, aria-label « agrandir l'image : {alt} »), qui ouvre la visionneuse ;
  - gérer l'événement `error` : un cadre qui affiche la description, sans casser la grille ;
  - brancher la prop `eager`.

  Faire passer T039
- [X] T045 [US2] Afficher les images :
  - dans `web/functional/Catalog/app/components/QuestionItem.vue` : `QuestionImageGallery` au-dessus de `RichTextView` du recto, sans rendre le texte quand `recto_html` est vide ;
  - dans `web/functional/Catalog/app/composables/useSubjectDetails.ts` : `include('images')` ;
  - dans `web/functional/Catalog/app/pages/sujets/[id]/cartes.vue` et `web/functional/Catalog/app/composables/useCardDeck.ts` : la même galerie sur la face recto et dans le rappel du recto au verso, avec `include('images')`.

  Faire passer T040
- [X] T046 [US2] Afficher les images en séance :
  - dans `web/functional/Learning/app/components/SessionCard.vue` : `QuestionImageGallery` `eager` au-dessus de `RichTextView` du recto ;
  - dans `web/functional/Learning/app/composables/useReviewSession.ts` : `include('question.images')`.

  Faire passer T041 et T042
- [X] T047 [P] [US2] Ajouter les textes dans `web/functional/Catalog/i18n/locales/fr.json` : « Agrandir l'image : {alt} », « Fermer », « Image non chargée : {alt} »

**Checkpoint**: US2 est complète. Les images sont visibles partout où le recto s'affiche.

---

## Phase 5: User Story 3 - Des images qui suivent la visibilité de leur sujet (Priority: P1)

**Goal**: une image n'est visible que de qui peut lire son sujet, disparaît avec son retrait, sa
question ou son sujet, et une image en attente expire après 24 heures.

**Independent Test**: l'adresse d'une image de brouillon donne `404` au visiteur et à un autre
inscrit, s'affiche une fois le sujet publié, puis donne de nouveau `404` après son retrait.

### Tests for User Story 3 ⚠️

- [X] T048 [P] [US3] Écrire `back/functional/catalog/tests/Feature/QuestionImageVisibilityTest.php`, sur le modèle du data provider `viewers()` de `SubjectVisibilityTest.php`. Il couvre la matrice complète de SC-004 :
  - 5 états (brouillon, publié, dépublié, retiré, supprimé) × 4 profils (visiteur, autre inscrit, auteur, administrateur) sur `GET /question-images/{id}/960` : `200` ou `404` selon le tableau de data-model.md ;
  - une image en attente : `200` pour son seul envoyeur ;
  - retiré puis rétabli et republié : de nouveau `200` ;
  - un corps de `404` identique à celui d'un id inexistant ;
  - `Cache-Control` `private, max-age=3600` pour un sujet publié et `private, no-store` sinon ;
  - `nosniff` ;
  - une largeur hors liste en `404` ;
  - une petite image servie en sa plus grande variante ;
  - `question-images/search` et l'`include` qui ne renvoient jamais une image en attente ni celle d'un sujet caché.
- [X] T049 [P] [US3] Écrire `back/functional/catalog/tests/Feature/QuestionImageDeletionTest.php`. Il couvre :
  - la suppression d'une question, qui supprime ses images, lignes et dossiers ;
  - la suppression d'un sujet, qui les supprime aussi (FR-017) ;
  - `model:prune` : une image en attente depuis plus de 24 heures supprimée avec son dossier, une image de 23 heures et une image rattachée conservées (FR-018, avec `$this->travel`) ;
  - une transaction annulée qui n'efface aucun fichier.
- [X] T050 [P] [US3] Ajouter à `back/functional/catalog/tests/Feature/QuestionContentTest.php` un test : un recto et un verso contenant `<img src="https://exemple.test/x.png">` sont enregistrés sans balise `<img>` (FR-015)

### Implementation for User Story 3

- [X] T051 [US3] Créer le listener `DeleteQuestionImages` dans `back/functional/catalog/src/Listeners/DeleteQuestionImages.php`, pour `QuestionDeleting` : `$question->images()->get()->each->delete()`, sur le modèle de `DeleteSubjectQuestions.php`. L'enregistrer dans `CatalogServiceProvider`
- [X] T052 [US3] Planifier l'élagage horaire dans `back/routes/console.php` : `Schedule::command('model:prune', ['--model' => [QuestionImage::class]])->hourly()`, à côté de l'élagage quotidien existant (research R9)
- [X] T053 [US3] Faire passer T048, T049 et T050 : corriger `QuestionImageAccess`, `ShowQuestionImageController` ou `QuestionImageControl` sans dupliquer les règles de visibilité. Puis lancer `./vendor/bin/sail artisan test functional/catalog` et `./vendor/bin/sail bin pint --dirty`
- [X] T054 [US3] Mettre à jour `back/CLAUDE.md` (sections Run et Stack) : images de questions stockées sur `QUESTION_IMAGES_DISK`, élagage horaire des images en attente par `schedule:work`, extension `imagick` requise par `intervention/image`

**Checkpoint**: les trois stories sont complètes et testées.

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: vérifications transverses, performance et validation de bout en bout.

- [X] T055 [P] Mesurer dans un test de `back/functional/catalog/tests/Feature/QuestionImageUploadTest.php` qu'une image de 5 Mo et 4 000 px est traitée en moins de 2 secondes dans Sail. Consigner le temps mesuré dans `specs/003-question-images/quickstart.md`
- [ ] T056 [P] Vérifier à 360, 768 et 1 440 px de large, dans le build de production de `web/`, l'éditeur avec 4 images, la lecture d'un sujet, le mode cartes et la séance : ni défilement horizontal ni libellé tronqué, focus visible, cibles de 44 px. Vérifier aussi que les images de `/api/question-images/*` ne sont pas servies par le service worker (`NetworkOnly`, `web/technical/Pwa/nuxt.config.ts` inchangé)
- [ ] T057 [P] Mesurer à 360 px, dans les outils du navigateur, qu'une carte à 4 images charge moins de 500 Ko (SC-003). Consigner le résultat dans `specs/003-question-images/quickstart.md`
- [X] T058 Lancer toutes les vérifications :
  - `./vendor/bin/sail artisan test` et `./vendor/bin/sail bin pint --dirty` dans `back/` ;
  - `pnpm test`, `pnpm lint` et `pnpm exec prettier --check .` dans `web/`.
- [ ] T059 Dérouler tous les scénarios manuels de `specs/003-question-images/quickstart.md` (US1, US2, US3, Leitner) et consigner les résultats dans ce fichier

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)** : aucune dépendance. T001 précède T010.
- **Foundational (Phase 2)** : dépend du Setup et bloque toutes les stories.
  - Au back : T005 → T007 → T010 → T013 → T016, et T014 → T015 → T016.
  - Au web : T021 → T023 → T024.
- **US1 (Phase 3)**, **US2 (Phase 4)**, **US3 (Phase 5)** : dépendent de la phase 2. US2 et US3 n'ont
  pas besoin de US1 pour leurs tests : les factories créent des images rattachées.
- **Polish (Phase 6)** : dépend des trois stories.

### Within Each Story

- Tests d'abord, en échec.
- Back avant web : le back est le fournisseur (constitution, principe III).
- Règles et ressource (T030, T031) avant le web qui les consomme (T034 à T037).

### Merge Order

La merge request de `back/` est fusionnée et déployée avant celle de `web/`, qui reste ouverte
jusque-là. Les deux sont liées par le numéro 003.

### Parallel Opportunities

- Phase 1 : T003 et T004 en parallèle de T002.
- Phase 2 : T006, T008, T011 en parallèle au back ; T021, T022, T023 en parallèle au web, et le web
  en parallèle du back.
- Dans chaque story, tous les tests marqués [P].
- US2 et US3 peuvent avancer en parallèle de US1 une fois la phase 2 finie.

---

## Parallel Example: User Story 1

```bash
# Tests de la story, en même temps :
Task: "QuestionImageUploadTest dans back/functional/catalog/tests/Feature/QuestionImageUploadTest.php"
Task: "QuestionImageAttachTest dans back/functional/catalog/tests/Feature/QuestionImageAttachTest.php"
Task: "Leitner inchangé dans back/functional/learning/tests/Feature/ContentChangesTest.php"
Task: "QuestionImagesField dans web/functional/Authoring/tests/QuestionImagesField.nuxt.spec.ts"
Task: "SubjectEditor dans web/functional/Authoring/tests/SubjectEditor.nuxt.spec.ts"
```

---

## Implementation Strategy

### MVP First (User Story 1)

1. Phase 1, puis phase 2 (qui inclut déjà le contrôle d'accès de la diffusion).
2. US1 : un auteur crée des cartes à images. **Valider** le scénario indépendant de US1.
3. US2 : les lecteurs et les apprenants voient les images.
4. US3 : matrice de visibilité complète, suppressions et élagage.

Les trois stories sont P1 : la feature n'est livrable qu'avec les trois. La diffusion contrôlée
est posée dès la phase 2, pour qu'aucune étape intermédiaire n'expose une image de brouillon.

---

## Notes

- [P] = fichiers différents, aucune dépendance en cours.
- Cocher chaque tâche terminée dans ce fichier, et le commiter dans le workspace à la fin de
  chaque phase.
- Messages de commit à l'impératif, en anglais, sans mention d'IA, dans chaque repo
  (`git -C back`, `git -C web`).
- Claude ne fusionne jamais une branche : le développeur s'en charge.

---

## Phase 7: Convergence

- [X] T060 Cocher dans ce fichier les tâches T001 à T055 dont le code est en place (évaluation du 2026-09-30 : toutes sauf T028, T033 et T053, qui attendent T061 et T063), puis commiter dans le workspace ce fichier et les modifications en cours de `plan.md`, `data-model.md` et `quickstart.md` per plan: workflow (partial)
- [X] T061 Démarrer Sail dans `back/`, vérifier `composer show intervention/image` et `php -m` (`imagick`, `exif`), puis lancer `./vendor/bin/sail artisan test functional/catalog functional/learning` et `./vendor/bin/sail bin pint --dirty` ; corriger tout échec, puis cocher T033 et T053 per Constitution IV (partial)
- [X] T062 Dans `web/functional/Authoring/app/composables/useQuestionDraft.ts`, quand le recto n'a ni texte ni image, afficher « Ajoutez un texte ou une image au recto. » (clé `add a text or an image to the recto.`) au lieu du message générique, et le couvrir dans `web/functional/Authoring/tests/SubjectEditor.nuxt.spec.ts` sans appel à l'API per FR-008, US1/AC10 (partial)
- [X] T063 Corriger `web/functional/Authoring/tests/QuestionImagesField.nuxt.spec.ts` (l. 145 et 150) pour le compteur rendu par Vuetify (`0 / 250`, `13 / 250`), puis relancer `pnpm test` jusqu'au vert per Constitution IV, T028 (partial)
- [X] T064 Vérifier que `useQuestionImages` affiche le message d'un `422` au format métier `{ code: 'image_invalid_format', message }` (WebP animé, fichier illisible) comme celui de `errors.file`, avec un cas de test dans `web/functional/Authoring/tests/QuestionImagesField.nuxt.spec.ts` ; ou aligner `StoreQuestionImageController` sur `errors.file`, comme le dit contracts/api.md §1 per FR-002, contracts §1 (contradicts)
- [X] T065 Refuser un `alt` qui contient des balises HTML dans `back/functional/catalog/src/Rest/Resources/QuestionImageResource.php` et dans les règles de `relations.images` de `QuestionResource`, avec un cas `422` dans `back/functional/catalog/tests/Feature/QuestionImageAttachTest.php` per FR-004, research R7 (partial)
- [X] T066 Imposer des positions continues de 0 à n-1 dans `back/functional/catalog/src/Rules/DistinctImagePosition.php` (ou une règle voisine), avec des cas `[0,3]` et `-1` en `422` dans `QuestionImageAttachTest.php` per data-model: rattachement (partial)
- [X] T067 Lancer `pnpm exec prettier --write` sur les 6 fichiers en échec, dont `QuestionImagesField.nuxt.spec.ts`, `SubjectEditor.nuxt.spec.ts`, `QuestionImageViewer.vue`, `useQuestionImageSources.ts`, `cartes.vue` et `uploadRequest.nuxt.spec.ts`, puis `pnpm exec prettier --check .` per T058 (partial)
- [X] T068 Revoir les plafonds mémoire d'Imagick dans `back/functional/catalog/src/Images/QuestionImageProcessor.php`, appliqués à tout le processus : les limiter au traitement ou les déclarer dans `config/catalog.php`, et justifier dans le message de commit la modification de `back/tests/TestCase.php` (`SearchRequest`) per plan: Constitution VII (unrequested)
- [X] T069 [P] Déplacer dans des composables les `computed` écrits directement dans `<script setup>` de `web/functional/Authoring/app/components/QuestionImageRow.vue`, `web/functional/Catalog/app/components/QuestionImageViewer.vue`, `QuestionCard.vue`, `QuestionItem.vue` et `SessionCard.vue` per web/CLAUDE.md: composable-first (contradicts)
- [X] T070 [P] Compléter les tests : ouverture de la visionneuse à la touche Entrée dans `web/functional/Catalog/tests/QuestionImageGallery.nuxt.spec.ts` (T039), et fichier d'au moins 5 Mo dans le test de performance de `back/functional/catalog/tests/Feature/QuestionImageUploadTest.php` (T055) per FR-011, SC-002 (partial)
- [X] T071 Commiter le travail web dans `web/` sur `003-question-images`, par étape, puis pousser les branches `003-question-images` de `back/` et `web/` avec leur propre amont (aujourd'hui `back/` suit `origin/main` avec 6 commits d'avance) per CLAUDE.md: multi-repo landing (partial)
