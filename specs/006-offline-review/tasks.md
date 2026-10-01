---

description: "Tâches de la feature 006 — Révision hors ligne (CINQ)"
---

# Tasks: Révision hors ligne

**Input**: Documents de conception dans `specs/006-offline-review/`

**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md),
[data-model.md](data-model.md), [contracts/api.md](contracts/api.md), [quickstart.md](quickstart.md)

**Tests**: obligatoires (constitution, principe IV). Dans chaque story, les tests sont écrits en
premier et doivent échouer avant l'implémentation. La table des cas est dans
[quickstart.md](quickstart.md).

**Organization**: une phase par user story ; dans chaque phase, le back avant le web.

## Format: `[ID] [P?] [Story] Description`

- **[P]** : parallélisable (fichiers différents, aucune dépendance sur une tâche non terminée)
- **[Story]** : story concernée (US1 à US4, voir spec.md)
- Tous les chemins commencent par leur repo : `back/…` ou `web/…`

## Conventions de chemins

- **Back** : couche `back/functional/learning/` (`src/`, espace de noms `Functional\Learning\`,
  `database/migrations/`, `tests/{Unit,Feature}`, helpers de test dans
  `tests/Concerns/ReviewsCards.php`) ; `back/functional/users/` pour `/user` ;
  `back/functional/reminders/tests/` pour l'espacement. Commandes via `./vendor/bin/sail`.
- **Web** : couche `web/functional/Learning/` (`app/{pages,composables,components,utils,offline,plugins,models}`,
  `i18n/locales/fr.json`, `tests/`, support d'API dans `tests/support/learningApi.ts`) ;
  `web/technical/ApiClient/`, `web/technical/Pwa/`, `web/functional/Account/`. Commandes via `pnpm`.
  Le service worker n'existe qu'en build de production (`web/CLAUDE.md`).
- Clés de traduction web : phrases anglaises en minuscules, valeurs françaises vouvoyées.
- Les branches `006-offline-review` de `back/` et `web/` sont créées par le hook
  `speckit.multirepo.branch` avant la première tâche.

---

## Phase 1: Setup

- [X] T001 Dans `web/`, ajouter `idb` en dépendance directe et `fake-indexeddb` en dépendance de développement, en version stable la plus récente (`pnpm add idb` puis `pnpm add -D fake-indexeddb`, attendues en 8.x et 6.x, research R7) ; vérifier que `web/package.json` et `web/pnpm-lock.yaml` sont seuls modifiés

---

## Phase 2: Foundational (prérequis bloquants)

**Purpose**: le schéma des réponses, le fuseau du compte, les règles Leitner côté web, l'état
« injoignable » de la session et la base IndexedDB. Rien de visible ne change encore.

### Back

- [X] T002 Créer une migration dans `back/functional/learning/database/migrations/` qui ajoute à `review_answers` : `answer_id` « uuid, unique, non nul », `due_on` « date, non nul », `status` « string(16), non nul », et un index `(card_progress_id, answered_at)`. Les lignes existantes reçoivent un UUID chacune, `due_on` = date de `answered_at` dans le fuseau du compte (`users.timezone`), `status` = `applied`, puis les trois colonnes passent non nulles ; **aucune valeur par défaut en base** (data-model)
- [X] T003 [P] Créer l'enum `back/functional/learning/src/Enums/AnswerStatus.php` (string, cas `Applied = 'applied'` et `Discarded = 'discarded'`) ; ajouter `answer_id`, `due_on`, `status` au `#[Fillable]` de `back/functional/learning/src/Models/ReviewAnswer.php`, avec les casts `due_on` → `date:Y-m-d` et `status` → `AnswerStatus` ; mettre à jour la factory de `ReviewAnswer` si elle existe (via `faker()` de `xefi/faker-php-laravel`)
- [X] T004 [P] Test dans `back/functional/users/tests/Feature/LoginTest.php` : `GET /api/user` renvoie `timezone` (le fuseau du compte) ; puis l'ajouter à la réponse de `back/functional/users/src/Http/Controllers/CurrentUserController.php` (contrat, research R6)
- [X] T005 Rendre la borne de `back/functional/learning/src/Queries/DueCardsQuery.php` paramétrable : par défaut « aujourd'hui dans le fuseau du compte » (comportement actuel, utilisé par `due`, `LearningResource` et les rappels), ou « aujourd'hui + N jours » ; aucun appelant existant ne change ; `./vendor/bin/sail artisan test --compact functional/learning functional/reminders` reste vert

### Web

- [X] T006 [P] Tests `web/functional/Learning/tests/leitner.spec.ts` : la table des cas Leitner de [data-model.md](data-model.md) au complet (boîtes 1 à 5 × sait / ne sait pas → boîte d'arrivée et jour + 1, 2, 4, 8, 16 jours ; boîte 5 sait reste en 5 à + 16 ; ne sait pas → boîte 1 à + 1) ; `localDay(instant, tz)` renvoie la date `YYYY-MM-DD` dans le fuseau du compte, pas celui de l'appareil (ex. `2026-10-05T23:30:00Z` → `2026-10-06` en `Europe/Paris`, `2026-10-05` en `America/Montreal`) ; `addDays` traverse un changement d'heure et une fin de mois (SC-005, FR-007)
- [X] T007 Dans `web/functional/Learning/app/utils/leitner.ts`, ajouter `arrivalBox(box, known)`, `nextReviewOn(arrivalBox, day)` (à partir de `BOX_INTERVAL_DAYS` existant) et `localDay(instant, timezone)` par `Intl.DateTimeFormat('en-CA', { timeZone })`, en dates `YYYY-MM-DD` sans heure (date-fns pour l'arithmétique) ; T006 passe au vert
- [X] T008 [P] Tests `web/technical/ApiClient/tests/session.nuxt.spec.ts` : `fetchUser()` sur erreur réseau (aucune réponse) → `isUnreachable = true`, `user = null` ; sur `401` → `isUnreachable = false`, `user = null` ; une page dont `meta.availableOffline` est vrai passe le middleware `auth` quand `isUnreachable`, une autre page redirige vers `/connexion?redirect=` ; `timezone` est lu depuis `/user`
- [X] T009 Dans `web/technical/ApiClient/app/stores/useSessionStore.ts`, ajouter `timezone: string` à `ISessionUser` et l'état `isUnreachable` (distinguer une `FetchError` sans `response` d'un `401`) ; dans `web/technical/ApiClient/app/middleware/auth.ts`, laisser passer quand `isUnreachable` et `to.meta.availableOffline` (research R8) ; T008 passe au vert
- [X] T010 [P] Tests `web/functional/Learning/tests/offlineDatabase.spec.ts` (avec `fake-indexeddb/auto`) : ouverture de la base `cinq-offline-review` avec les magasins `pack` (clé `current`) et `answers` (clé `answer_id`, index `answered_at`) ; écriture puis relecture après fermeture et réouverture ; `discard()` supprime la base ; si `indexedDB.open` échoue, le module bascule sur un stockage en mémoire et signale `isPersistent = false`
- [X] T011 Créer `web/functional/Learning/app/offline/database.ts` (ouverture par `idb`, schéma de [data-model.md](data-model.md), repli en mémoire, `navigator.storage.persist()` demandé au premier enregistrement), `web/functional/Learning/app/offline/pack.ts` (lire, remplacer d'un bloc dans une transaction, appliquer une réponse locale à une carte et aux comptes par boîte de son `learning` : `box_{from}` −1, `box_{to}` +1) et `web/functional/Learning/app/offline/answers.ts` (ajouter, lister par `answered_at`, retirer, compter) ; types `IOfflinePack`, `IOfflineCard`, `IPendingAnswer` selon data-model ; T010 passe au vert

**Checkpoint**: `./vendor/bin/sail artisan test --compact` et `pnpm test` verts ; aucun comportement visible changé.

---

## Phase 3: User Story 1 - Réviser sans réseau (Priority: P1) 🎯 MVP

**Goal**: hors ligne, « Mes révisions » et la séance fonctionnent depuis le paquet ; chaque
réponse est calculée localement et gardée sur l'appareil.

**Independent Test**: 12 cartes dues sur 2 sujets, appareil hors ligne : « Mes révisions » montre
les 2 sujets et leurs 12 cartes ; une séance se lance et se termine par un bilan, sans erreur de
réseau.

### Tests (écrits d'abord, en échec)

- [X] T012 [P] [US1] Test `back/functional/learning/tests/Feature/UpcomingCardsTest.php` : l'instruction `upcoming` renvoie les cartes du compte dont `next_review_on` ≤ aujourd'hui + 7 jours dans son fuseau (J et J+7 incluses, J+8 exclue ; un compte en `Asia/Tokyo` et un en `America/Montreal` au même instant), triées par `next_review_on`, `subject_id`, `position` ; sujets `Draft`, `Retired`, `Withheld` exclus ; cartes d'un autre compte exclues ; `question.images` porte `alt` (FR-001)
- [X] T013 [P] [US1] Ajouter dans `web/functional/Learning/tests/support/learningApi.ts` un handler `card-progress/search` qui distingue l'instruction `upcoming` de `due`, une fabrique `aPack(...)` qui remplit la base (`fake-indexeddb`), et un moyen de simuler la perte de réseau (le `fetch` lève une `FetchError` sans réponse, `navigator.onLine = false`, événement `offline`)
- [X] T014 [P] [US1] Tests `web/functional/Learning/tests/OfflineRevisionsPage.nuxt.spec.ts` : hors ligne avec un paquet, `/revisions` affiche les sujets, leurs cartes du jour, la répartition dans les boîtes et « Hors ligne — cartes à jour du {date} » (US1-1, FR-005) ; une carte d'échéance J+3 du paquet compte parmi les cartes du jour quand l'horloge avance de 3 jours (US1-6, FR-007) ; sans paquet, la page indique que la révision hors ligne sera possible après une première ouverture en ligne (edge case) ; base indisponible : message « La révision hors ligne n'est pas disponible sur cet appareil », la révision en ligne reste possible (FR-019)
- [X] T015 [P] [US1] Tests `web/functional/Learning/tests/OfflineReviewSession.nuxt.spec.ts` : hors ligne, la séance contient les cartes du jour des sujets choisis, des plus en retard aux plus récentes (US1-2, FR-006) ; verso avant réponse ; après « Je savais » sur une carte en boîte 2, « boîte 2 → boîte 3 » et « dans 4 jours » s'affichent, calculés localement ; la carte n'est pas reposée (US1-3) ; la réponse est dans le magasin `answers` avec `answer_id`, `known`, `answered_at`, `due_on` = `next_review_on` de la carte, **avant tout appel réseau** (FR-008) ; le bilan s'affiche après la dernière carte, avec les comptes par boîte du paquet ajustés (US1-4) ; une carte avec image affiche sa description et « Image non disponible hors ligne », sans `<img>` (US1-5, FR-002) ; en ligne, une coupure en pleine séance ne change rien à l'affichage (edge case)

### Implémentation — back

- [X] T016 [US1] Créer `back/functional/learning/src/Rest/Instructions/UpcomingCardsInstruction.php` (sans champ, horizon en constante `HORIZON_DAYS = 7`, s'appuie sur `DueCardsQuery` avec la borne + 7 jours de T005, même tri que `DueCardsInstruction`) et l'enregistrer dans `back/functional/learning/src/Rest/Resources/CardProgressResource.php` sous le nom `upcoming` ; T012 passe au vert

### Implémentation — web

- [X] T017 [US1] Créer `web/functional/Learning/app/composables/useOfflineReview.ts` (état partagé par onglet) avec `refreshPack()` : lit toutes les pages de `Learning.query().include('subject').limit(50)` et de `CardProgress.query().instruction('upcoming', []).include('question').include('question.images').include('subject').limit(100)`, puis remplace le paquet (`user_id`, `timezone` de la session, `updated_at`, `learnings`, `cards` avec `image_alts[]`) ; **ne fait rien si le magasin `answers` n'est pas vide** (FR-017) ; expose aussi `pack`, `isPersistent`, `isOffline` (via `useConnectionStatus` de `web/technical/Pwa`, ou `useSessionStore().isUnreachable`)
- [X] T018 [US1] Créer `web/functional/Learning/app/plugins/offline-review.client.ts` : à l'ouverture, si une session est connue et que le réseau est là, `refreshPack()` (FR-003, ouverture en ligne)
- [X] T019 [US1] Dans `web/functional/Learning/app/composables/useRevisions.ts`, hors ligne, construire la liste depuis le paquet : sujets de `pack.learnings`, cartes du jour = `next_review_on` ≤ `localDay(maintenant, pack.timezone)`, comptes par boîte du paquet ; `startSession` inchangé (navigation côté client vers `/revisions/seance?sujets=…`)
- [X] T020 [US1] Dans `web/functional/Learning/app/composables/useReviewSession.ts`, remplacer l'envoi direct et la relecture de la carte (`CardProgress.actions('answer', …)` puis `CardProgress.query().where('id', …)`) par : calcul local (`arrivalBox`, `nextReviewOn`, jour de `localDay` dans le fuseau du compte), écriture de la réponse (`answer_id` = `crypto.randomUUID()`, `answered_at` ISO avec décalage, `due_on`), mise à jour de la carte et des comptes dans le paquet, puis résultat `{ known, fromBox, toBox, nextReviewOn }` (research R1) ; hors ligne, les cartes viennent du paquet, triées par `next_review_on`, `subject_id`, `position` ; le bilan lit les comptes par boîte du paquet (repli sur `learnings/search` sans paquet) ; plus de `saveFailed` sur une coupure
- [X] T021 [P] [US1] Créer `web/functional/Learning/app/components/OfflineImageNotice.vue` (une ligne par description, mention « Image non disponible hors ligne ») et l'utiliser dans `web/functional/Learning/app/components/SessionCard.vue` à la place de `QuestionImageGallery` pour une carte venue du paquet (FR-002, research R11)
- [X] T022 [P] [US1] Créer `web/functional/Learning/app/components/OfflineStatus.vue` (`role="status"`) : « Hors ligne — cartes à jour du {date} » ; « La révision hors ligne n'est pas disponible sur cet appareil » si `isPersistent` est faux (FR-019) ; « La révision hors ligne sera possible après une première ouverture en ligne » sans paquet ; l'afficher dans `web/functional/Learning/app/pages/revisions/index.vue` et `web/functional/Learning/app/pages/revisions/seance.vue`
- [X] T023 [US1] Ajouter `meta.availableOffline: true` dans `definePageMeta` de `web/functional/Learning/app/pages/revisions/index.vue` et `web/functional/Learning/app/pages/revisions/seance.vue` ; dans `web/functional/Learning/nuxt.config.ts`, `routeRules` `'/revisions': { ssr: false, prerender: true }` et `'/revisions/**': { ssr: false }` (research R8)
- [X] T024 [US1] Dans `web/technical/Pwa/nuxt.config.ts`, ajouter `revisions/index.html` aux `globPatterns` et, **avant** la règle de navigation générale, une règle `request.mode === 'navigate' && url.pathname.startsWith('/revisions')` en `NetworkOnly` avec `precacheFallback: { fallbackURL: '/revisions' }` ; vérifier sur un build de production (`pnpm build`) que `.output/public/revisions/index.html` existe et que, hors ligne, `/revisions` et `/revisions/seance` s'ouvrent ; à défaut, créer une page coquille prérendue dédiée et y faire pointer le repli (research R8, point à vérifier)
- [X] T025 [P] [US1] Dans `web/technical/Pwa/app/pages/hors-ligne.vue`, ajouter la mention « La révision hors ligne sera possible après une première ouverture en ligne de CINQ sur cet appareil » et un lien vers « Mes révisions » ; clé dans `web/technical/Pwa/i18n/locales/fr.json` (edge case « jamais ouvert en ligne »)
- [X] T026 [US1] Ajouter les clés de US1 dans `web/functional/Learning/i18n/locales/fr.json` (« offline — cards up to date as of {date} », « image not available offline », « offline review is not available on this device. », « offline review will be possible after opening cinq online once. »), en vouvoyant ; T014 et T015 passent au vert

**Checkpoint**: US1 se démontre seule, réponses gardées sur l'appareil sans encore partir.

---

## Phase 4: User Story 2 - Retrouver ses réponses sur le serveur (Priority: P1)

**Goal**: la file part d'elle-même au retour du réseau ; le serveur applique chaque réponse à
l'heure où elle a été donnée, dans l'ordre, une fois et une seule.

**Independent Test**: lundi, hors ligne, 5 réponses ; mercredi, retour du réseau : en moins d'une
minute, les 5 réponses sont sur le serveur et chaque carte revient à la date calculée depuis lundi.

### Tests (écrits d'abord, en échec)

- [X] T027 [P] [US2] Tests `back/functional/learning/tests/Feature/DatedAnswersTest.php` : boîte 2, « sait » avec `answered_at` lundi et reçue mercredi → boîte 3, `next_review_on` = lundi + 4 jours, `answered_at` gardé, `status = applied` (US2-3, FR-011, SC-003) ; deux réponses de la même carte (échéances successives) reçues dans le désordre → même état final et mêmes lignes que dans l'ordre (FR-012) ; même `answer_id` envoyé deux fois → une seule ligne, carte modifiée une fois, `200` les deux fois (FR-016, SC-002) ; `answered_at` dans le futur → compté à l'heure de réception (FR-015) ; `answered_at` avant l'échéance (horloge en retard) → compté à l'heure de réception si la carte est due pour ce `due_on`, sinon `discarded` (FR-015) ; sans `answer_id`, `answered_at` ni `due_on`, l'action fonctionne comme pour le web actuel (contrat) ; champs mal formés → `422`
- [X] T028 [P] [US2] Test unitaire `back/functional/learning/tests/Unit/CardAnswerReplayTest.php` : les règles de rejeu de [data-model.md](data-model.md) pas à pas (retour à l'état d'avant la plus ancienne réponse postérieure, rejeu dans l'ordre de `answered_at`, `applied` seulement si `S.next_review_on` = `due_on` et ≤ jour de la réponse), et la table des cas Leitner dans le rejeu (SC-005)
- [X] T029 [P] [US2] Test `back/functional/reminders/tests/Feature/ReminderEligibilityTest.php` : une réponse envoyée mercredi avec `answered_at` lundi fait partir l'espacement de lundi (edge case rappels)
- [X] T030 [P] [US2] Tests `web/functional/Learning/tests/AnswerSync.nuxt.spec.ts` : à l'événement `online`, les réponses partent dans l'ordre de `answered_at`, une requête à la fois, avec `answer_id`, `card_progress_id`, `known`, `answered_at`, `due_on` (FR-010, US2-1) ; une réponse ne quitte la base qu'après un `2xx` ; coupure au 3ᵉ envoi → les 2 premières sont retirées, la reprise envoie les 3 suivantes, aucune deux fois (FR-016, US2-5, SC-002) ; `401` → arrêt, file gardée ; `422` → réponse retirée ; à l'ouverture en ligne, `useRevisions()` attend la fin de l'envoi avant d'afficher (US2-2) ; après un envoi qui vide la file, le paquet est remis à jour (FR-017) ; le compteur « {n} réponses à envoyer » s'affiche tant qu'il en reste, puis disparaît (FR-009) ; une réponse donnée en ligne part aussitôt

### Implémentation — back

- [X] T031 [US2] Créer `back/functional/learning/src/Domain/CardAnswerReplay.php` : reçoit la carte verrouillée, le fuseau et la réponse entrante ; applique les étapes 3 à 6 du rejeu de [data-model.md](data-model.md) (futur ramené à maintenant, retour à l'état d'avant la plus ancienne réponse `applied` postérieure, rejeu ordonné, horloge en retard rejouée à maintenant, `from_box`/`to_box`/`status` mis à jour, `to_box` = `from_box` pour une réponse `discarded`, état final et `last_answered_at` écrits sur la carte) en s'appuyant sur `LeitnerSchedule` ; T028 passe au vert
- [X] T032 [US2] Dans `back/functional/learning/src/Rest/Actions/AnswerCard.php`, ajouter les champs facultatifs `answer_id` (« uuid »), `answered_at` (« date ISO 8601 »), `due_on` (« date `Y-m-d` »), avec leurs valeurs par défaut (UUID généré, maintenant, échéance courante de la carte) ; dans la transaction : `answer_id` connu → rien ; sinon `lockForUpdate` puis `CardAnswerReplay` ; mettre à jour le helper `answer()` de `back/functional/learning/tests/Concerns/ReviewsCards.php` pour accepter ces champs ; T027 et T029 passent au vert

### Implémentation — web

- [X] T033 [US2] Dans `web/functional/Learning/app/composables/useOfflineReview.ts`, ajouter `pendingCount` et `flush()` : un seul envoi en cours par onglet (promesse partagée), réponses dans l'ordre de `answered_at` par `CardProgress.actions('answer', [...])` (modèle raom, jamais de `fetch` direct), retrait après `2xx`, arrêt sur erreur réseau ou `401`, retrait sur `422`, puis `refreshPack()` si la file est vide (research R2, R10)
- [X] T034 [US2] Déclencher `flush()` : dans `web/functional/Learning/app/plugins/offline-review.client.ts` à l'ouverture (avant `refreshPack()`) et à l'événement `online` ; dans `web/functional/Learning/app/composables/useReviewSession.ts` juste après l'écriture de chaque réponse, si le réseau est là ; dans `web/functional/Learning/app/composables/useRevisions.ts`, attendre `flush()` avant le chargement en ligne (FR-010)
- [X] T035 [P] [US2] Créer `web/functional/Learning/app/components/PendingAnswers.vue` (`role="status"`, masqué à 0) : « {n} réponses à envoyer », avec pluriel (« 1 réponse à envoyer ») ; l'afficher dans `web/functional/Learning/app/pages/revisions/index.vue` et `web/functional/Learning/app/pages/revisions/seance.vue` ; clés dans `web/functional/Learning/i18n/locales/fr.json` (FR-009) ; T030 passe au vert

**Checkpoint**: US1 et US2 ensemble forment le parcours complet hors ligne → en ligne.

---

## Phase 5: User Story 3 - Une carte révisée sur deux appareils (Priority: P2)

**Goal**: seule la première réponse d'une échéance compte ; une réponse écartée ou sur une carte
disparue ne produit aucune erreur.

**Independent Test**: carte due mardi, « Je ne savais pas » à 8 h hors ligne sur le téléphone,
« Je savais » à 12 h en ligne sur l'ordinateur ; à 18 h, retour du réseau du téléphone : carte en
boîte 1, la réponse de 12 h annulée.

### Tests (écrits d'abord, en échec)

- [X] T036 [P] [US3] Tests `back/functional/learning/tests/Feature/FirstAnswerWinsTest.php` : le scénario de US3 (réponse de 12 h « sait » appliquée, puis réponse de 8 h « ne sait pas » reçue à 18 h) → carte en boîte 1, `next_review_on` = mercredi, réponse de 12 h `discarded` avec `to_box` = `from_box`, réponse de 8 h `applied` (US3-1, FR-013) ; deux réponses pour la même échéance reçues dans l'ordre → la seconde `discarded`, `200` ; réponse en double arrivée après l'échéance suivante (même `due_on` que la première) → `discarded`, pas appliquée à l'échéance suivante (research R3)
- [X] T037 [P] [US3] Tests `back/functional/learning/tests/Feature/UnavailableCardAnswersTest.php` : réponse sur une carte supprimée (question supprimée, sujet arrêté), sur un sujet `Draft`, `Retired`, `Withheld` (sujet retenu), et sur la carte d'un autre compte → `200`, aucune ligne dans `review_answers`, carte inchangée, même corps de réponse dans tous ces cas (FR-014, US3-3, principe VI)
- [X] T038 [P] [US3] Réécrire, dans `back/functional/learning/tests/Feature/ReviewSessionTest.php`, le test de la ligne 128 (seconde réponse refusée en `409 card_not_due`) : la seconde réponse donne `200` et est `discarded`, la carte garde l'effet de la première ; réécrire le test « nobody answers someone else's card » (ligne 142) : `200` sans effet au lieu de `404` ; vérifier `back/functional/learning/tests/Feature/WithheldSubjectsInReviewsTest.php`
- [X] T039 [P] [US3] Tests `web/functional/Learning/tests/AnswerSync.nuxt.spec.ts` : une réponse écartée ou ignorée par le serveur (`200`) ne produit aucun message d'erreur, et le paquet est remis à jour depuis le serveur après l'envoi : la carte y prend la boîte renvoyée par `upcoming` (US3-2)

### Implémentation

- [X] T040 [US3] Dans `back/functional/learning/src/Rest/Actions/AnswerCard.php`, remplacer `findOrFail` et le `BusinessRuleException('card_not_due', 409)` par : carte absente, d'un autre compte (`user_id`) ou de sujet non public (`! $card->subject->status->isPublic()`) → retour sans effet ni ligne ; retirer la traduction de `card_not_due` si elle n'est plus utilisée (dans `back/technical/osdd/lang/fr/errors.php` ou le fichier de langue de la couche) ; T036 à T038 passent au vert
- [X] T041 [US3] Dans `web/functional/Learning/app/composables/useReviewSession.ts` et `web/functional/Learning/app/composables/useOfflineReview.ts`, retirer toute gestion du code `card_not_due` et tout message d'erreur lié à l'envoi d'une réponse ; T039 passe au vert

**Checkpoint**: les quatre cas de FR-013 à FR-015 sont couverts côté back, sans erreur côté web.

---

## Phase 6: User Story 4 - Garder les données hors ligne à jour et à soi (Priority: P2)

**Goal**: le paquet se met à jour de lui-même ; il appartient au compte connecté et part avec lui.

**Independent Test**: apprendre un sujet en ligne, passer hors ligne : ses cartes sont là ; se
déconnecter : plus aucune carte ni réponse du compte sur l'appareil.

### Tests (écrits d'abord, en échec)

- [X] T042 [P] [US4] Tests `web/functional/Learning/tests/PackRefresh.nuxt.spec.ts` : le paquet est remis à jour à l'ouverture en ligne, en fin de séance, après avoir appris un sujet et après avoir arrêté d'en apprendre un (US4-1, FR-003) ; un compte connecté différent de `pack.user_id` → base vidée avant tout affichage (US4-4, FR-004) ; le même compte qui se reconnecte après une session expirée → file gardée puis envoyée (edge case session) ; paquet mis à jour il y a plus de 7 jours → seules ses cartes sont révisables, message « Reconnectez-vous au réseau pour réviser les cartes suivantes » (US4-5)
- [X] T043 [P] [US4] Tests `web/functional/Account/tests/Logout.nuxt.spec.ts` : déconnexion sans réponse en attente → base vidée, déconnexion comme avant (US4-3, SC-004) ; en ligne avec réponses en attente → envoi tenté d'abord, puis déconnexion sans avertissement si la file est vide ; hors ligne avec 2 réponses en attente → `ConfirmDialog` « 2 réponses ne sont pas encore envoyées », « Attendre le réseau » ferme le dialogue sans se déconnecter, « Me déconnecter quand même » vide la base puis déconnecte (US4-2, FR-018) ; demande de suppression de compte → base vidée (edge case 004)

### Implémentation

- [X] T044 [US4] Dans `web/functional/Learning/app/composables/useOfflineReview.ts`, ajouter `discard()` (supprime la base et remet l'état à zéro) ; dans `web/functional/Learning/app/plugins/offline-review.client.ts`, surveiller `useSessionStore().user` : identifiant différent de `pack.user_id` → `discard()` avant tout affichage ; même identifiant → `flush()` (research R9)
- [X] T045 [US4] Appeler `refreshPack()` en fin de séance (`finish()` de `web/functional/Learning/app/composables/useReviewSession.ts`, après `flush()`), après avoir appris un sujet (`web/functional/Learning/app/components/LearnSubjectPanel.vue` ou son composable) et après avoir arrêté d'en apprendre un (`useRevisions.ts`, après `learning.delete()`) (FR-003)
- [X] T046 [US4] Dans `web/functional/Learning/app/composables/useRevisions.ts` et `web/functional/Learning/app/components/OfflineStatus.vue`, si `localDay(maintenant)` dépasse `pack.updated_at` + 7 jours, afficher « Reconnectez-vous au réseau pour réviser les cartes suivantes » (US4-5)
- [X] T047 [US4] Dans `web/functional/Account/app/composables/useAccountMenu.ts`, `logOut()` : `flush()` si le réseau est là ; s'il reste des réponses, ouvrir `ConfirmDialog` (`web/technical/Theme/app/components/ConfirmDialog.vue`) avec « {n} réponses ne sont pas encore envoyées » et les choix « Attendre le réseau » / « Me déconnecter quand même » ; puis `discard()` et la déconnexion existante ; dans `web/functional/Account/app/composables/useAuth.ts`, appeler `discard()` dans `requestAccountDeletion()` après `sessionStore.clear()` ; branchement du dialogue dans `AccountMenu.vue` / `AccountMenuPanel.vue` (FR-004, FR-018, research R9)
- [X] T048 [US4] Ajouter les clés de US4 dans `web/functional/Account/i18n/locales/fr.json` (« {count} answers have not been sent yet », « wait for the network », « log out anyway ») et `web/functional/Learning/i18n/locales/fr.json` (« reconnect to the network to review the next cards. »), avec pluriel et vouvoiement ; T042 et T043 passent au vert

**Checkpoint**: les quatre stories fonctionnent ensemble.

---

## Phase 7: Polish & vérifications transverses

- [X] T049 [P] Dans `web/technical/Pwa/app/components/OfflineBanner.vue`, compléter le texte : la révision reste possible et les réponses partiront au retour du réseau ; les autres enregistrements restent suspendus (FR-020) ; clé dans `web/technical/Pwa/i18n/locales/fr.json` ; test existant du bandeau mis à jour
- [X] T050 [P] Vérifier par un test dans `web/functional/Learning/tests/OfflineRevisionsPage.nuxt.spec.ts` que, hors ligne, « Apprendre » et « Arrêter d'apprendre » restent indisponibles (FR-020), et que les cibles du compteur et de l'indication font au moins 44 px quand elles sont actives (principe V)
- [X] T051 [P] Dans `specs/001-learning-content/contracts/api.md`, à la section `card-progress`, signaler que l'action `answer` est élargie et que `409 card_not_due` est retiré par la feature 006, avec un lien vers `specs/006-offline-review/contracts/api.md`
- [X] T052 Lancer `./vendor/bin/sail bin pint --dirty --format agent` et `./vendor/bin/sail artisan test --compact` dans `back/` ; `pnpm lint`, `pnpm exec prettier --check .` et `pnpm test` dans `web/`
- [X] T053 Dérouler les sept parcours manuels de [quickstart.md](quickstart.md) à 360 px sur le build de production, et noter le résultat dans ce fichier

---

### Résultats des parcours manuels (T053, 2026-10-01, 360 × 780)

Le port 3000, seul accepté par Sanctum, était occupé par le serveur `nuxt dev` du développeur : le
parcours s'est fait sur ce serveur, qui sert le code de la branche, avec le back en Sail et un compte
seedé. La perte de réseau a été simulée dans la page (`navigator.onLine` à faux, événements
`offline` puis `online`), l'API restant joignable.

1. **En ligne** : « Mes révisions » affiche 21 cartes ; la base `cinq-offline-review` contient le
   paquet (21 cartes de l'instruction `upcoming`, fuseau `Europe/Paris` lu sur `/api/user`). ✅
2. **Hors ligne, séance** lancée par navigation côté client : « Hors ligne — cartes à jour du
   1 octobre 2026 à 15:04 », 21 cartes depuis l'appareil. ✅
3. **Réponse hors ligne** : « Boîte 1 → boîte 2 », « Revient : dans 2 jours », « 1 réponse à
   envoyer » ; la réponse est dans IndexedDB avec son `answer_id`, `answered_at` et `due_on`. ✅
4. **Retour du réseau** : la file se vide en moins de 3 s, le compteur disparaît ; en base, la réponse
   est `applied`, boîte 2, échéance le 3 octobre, identique à l'appareil (SC-003). ✅
5. **Déconnexion avec une réponse en attente**, l'API joignable : la réponse part d'abord (en base,
   `applied`, boîte 1, échéance le 2 octobre), puis la déconnexion efface la base IndexedDB
   (`indexedDB.databases()` vide, SC-004). ✅

Non déroulés dans un navigateur, faute de pouvoir lancer le build de production sur le port 3000
sans arrêter le serveur du développeur : l'ouverture de `/revisions` réellement hors ligne par le
service worker, la réouverture de l'onglet hors ligne, l'avertissement de déconnexion quand l'API est
injoignable, et la navigation privée. Ils sont couverts par les tests automatisés
(`OfflineRevisionsPage`, `OfflineReviewSession`, `Logout`, `offlineDatabase`) et, pour le service
worker, par l'inspection du build : `revisions` est précaché et sa règle de navigation, avec repli
sur `/revisions`, passe avant celle de `/hors-ligne`. À refaire sur le build de production avant
l'ouverture au public.

### Écarts entre les tâches et l'implémentation

- **Rejeu (T031)** : les réponses postérieures `discarded` sont rejouées aussi, pas seulement les
  `applied`. Sans cela, deux réponses reçues dans le désordre laisseraient la plus récente écartée
  (FR-012). Corrigé dans [data-model.md](data-model.md), étape 4.
- **Back** : une factory `ReviewAnswerFactory` est créée, car des tests de rappels inséraient des
  réponses à la main. L'ordre de séance est partagé par `DueCardsQuery::inReviewOrder()`. Le
  remplissage des colonnes existantes est écrit pour PostgreSQL.
- **Paquet (T011, T017)** : les cartes gardent la forme de l'API (`question.images` réduites à `alt`
  et `position`) plutôt que `image_alts[]`, pour que `SessionCard` affiche les deux de la même façon.
  Le magasin `answers` n'a pas d'index `answered_at` : un horodatage avec décalage ne se trie pas
  comme une chaîne, donc la file est triée par l'instant qu'il désigne. Chaque réponse porte
  `user_id`. [data-model.md](data-model.md) est à jour.
- **Synchronisation (T018, T044)** : au lieu de surveiller la session, la synchronisation part à
  l'ouverture (`app:mounted`), à la connexion et à la confirmation d'email (`useAuth`), et au retour
  du réseau. Une surveillance permanente déclenchait des appels dans les tests de toutes les autres
  couches.
- **`flush()`** accepte `{ refreshPack: false }` : envoyer chaque réponse en ligne ne recharge pas le
  paquet à chaque fois ; il est rechargé à l'ouverture, en fin de séance, et après apprendre ou
  arrêter.
- **Hors ligne, « Arrêter d'apprendre » est masqué** dans « Mes révisions » (FR-020). Le compteur
  et l'indication hors ligne ne sont pas des cibles tactiles, donc la règle des 44 px de T050 ne
  s'applique pas.
- **Typage** : le web n'a pas de vérificateur de types configuré (`nuxi typecheck` demande
  `vue-tsc`) ; seuls ESLint et Prettier ont tourné.

## Dependencies & Execution Order

- **Phase 1** avant tout le web.
- **Phase 2** bloque toutes les stories : T002 → T003 ; T005 avant T016 ; T007, T009 et T011 avant le web de US1.
- **US1** (MVP) dépend de la Phase 2. Côté back, T016 seul ; côté web, T017 → T018 → T019/T020 → T021 à T026.
- **US2** dépend de la Phase 2 et de T020 (la file existe). T031 → T032 ; T033 → T034 → T035.
- **US3** dépend de T032 (même action) côté back, et de T033 côté web.
- **US4** dépend de T017 et T033 ; T044 avant T047.
- **Polish** après les quatre stories.
- **Fusion** : le back (T002 à T005, T016, T031, T032, T040) est fusionné et poussé sur `main` avant le web ; ses champs facultatifs gardent le web en place fonctionnel entre les deux.

### Parallel Opportunities

- Phase 2 : T003, T004 en parallèle après T002 ; T006, T008, T010 en parallèle entre eux et avec le back.
- US1 : T012 à T015 en parallèle ; T021, T022, T025 en parallèle une fois T017 fait.
- US2 : T027 à T030 en parallèle ; T035 en parallèle de T034.
- US3 : T036 à T039 en parallèle.
- US4 : T042 et T043 en parallèle.
- Polish : T049 à T051 en parallèle.

## Implementation Strategy

1. Phase 1 et 2 : schéma, fuseau, règles Leitner côté web, base IndexedDB.
2. US1 : réviser hors ligne, réponses gardées. C'est le MVP démontrable, mais il ne doit pas être
   livré seul : sans US2, les réponses ne partent pas.
3. US2 : l'envoi et le rejeu daté. US1 + US2 forment la première livraison utile.
4. US3, puis US4 : première réponse gagnante, puis mise à jour et effacement du paquet.
5. Finitions, puis fusion sur `main`, back puis web.
