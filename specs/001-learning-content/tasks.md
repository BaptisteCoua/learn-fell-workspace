---

description: "Tâches de la feature 001 — Comptes, contenu et révision Leitner (CINQ)"
---

# Tasks: Comptes, contenu et révision Leitner

**Input**: Documents de conception dans `specs/001-learning-content/`

**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md),
[data-model.md](data-model.md), [contracts/api.md](contracts/api.md), [quickstart.md](quickstart.md)

**Tests**: obligatoires. La constitution (principe IV) exige un test automatisé par exigence
fonctionnelle, et une couverture à 100 % des règles Leitner et des accès aux contenus non publiés.
Dans chaque story, les tests sont écrits en premier et doivent échouer avant l'implémentation.

**Organization**: une phase par user story, dans l'ordre des priorités de la spec. Dans chaque
story, le back (fournisseur) passe avant le web (consommateur).

**Maquette de référence**: [CINQ — Design 001](https://claude.ai/artifact/DxQK5ap9dAK6UYGuZURLsP).
Chaque tâche web reprend l'écran du même nom sur le canevas.

## Format: `[ID] [P?] [Story] Description`

- **[P]** : parallélisable (fichiers différents, aucune dépendance sur une tâche non terminée)
- **[Story]** : story concernée (US1 à US6, voir spec.md)
- Tous les chemins commencent par leur repo : `back/…` ou `web/…`

## Conventions de chemins

- **Back** : couches OSDD dans `back/functional/<couche>/` et `back/technical/<couche>/`, chacune
  avec `src/` (espace de noms `Functional\<Couche>\`), `database/{migrations,factories,seeders}`,
  `routes/api.php`, `lang/fr/`, `config/` et `tests/{Feature,Unit}`. Commandes via `./vendor/bin/sail`.
- **Web** : couches `nuxt-osdd` dans `web/technical/<Couche>/` et `web/functional/<Couche>/`, chacune
  avec `nuxt.config.ts`, `app/{pages,components,composables,models,stores,middleware}`,
  `i18n/locales/fr.json` et `tests/`. Commandes via `pnpm`.

---

## Phase 1: Setup (infrastructure partagée)

**Purpose**: initialiser l'architecture OSDD du back, les dépendances et l'environnement, et
renommer le produit en CINQ dans le code.

- [X] T001 Initialiser l'architecture OSDD avec `./vendor/bin/sail artisan osdd:start` dans `back/` : vérifier que `back/app/`, `back/database/` et `back/config/` sont remplacés par `back/functional/users/` et `back/technical/osdd/`, que `back/composer.json` déclare le dépôt de chemins des couches, et que `./vendor/bin/sail artisan test` passe toujours
- [X] T002 Créer les couches fonctionnelles avec `./vendor/bin/sail artisan osdd:layer` : `back/functional/catalog/`, `back/functional/moderation/`, `back/functional/learning/` (chacune avec `composer.json` de type `layer`, un `LayerServiceProvider` et un `routes/api.php` monté par `withRouting(api:)`)
- [X] T003 Installer `laravel/sanctum`, `laravel/fortify` et `stevebauman/purify` dans `back/composer.json` avec `./vendor/bin/sail composer require`, puis publier leurs migrations et configurations dans `back/functional/users/` (Sanctum, Fortify) et `back/functional/catalog/` (Purify)
- [X] T004 [P] Ajouter le service `mailpit` (ports 1025 et 8025) dans `back/compose.yaml`
- [X] T005 [P] Corriger `back/.env.example` : `APP_NAME=CINQ`, `APP_LOCALE=fr`, `DB_CONNECTION=pgsql` avec les variables PostgreSQL de Sail, `SESSION_DRIVER=database`, `SESSION_DOMAIN=localhost`, `SANCTUM_STATEFUL_DOMAINS=localhost:3000`, `FRONTEND_URL=http://localhost:3000`, `MAIL_MAILER=smtp`, `MAIL_HOST=mailpit`, `MAIL_PORT=1025`, `MAIL_FROM_NAME=CINQ`
- [X] T006 [P] Corriger `back/CLAUDE.md` : produit « CINQ », couches dans `back/functional/` et `back/technical/` (et non `layers/`), commandes `osdd:*`, test par `./vendor/bin/sail artisan test`
- [X] T007 [P] Ajouter `@tiptap/vue-3`, `@tiptap/pm`, `@tiptap/starter-kit`, `@tiptap/extension-link` et `date-fns` avec `pnpm add` dans `web/package.json`
- [X] T008 [P] Renommer le produit en CINQ côté web : `name` et `short_name` à `CINQ`, `theme_color` `#FACC15` et `background_color` `#FFFBEB` dans `web/technical/Pwa/nuxt.config.ts` ; titre « Bienvenue sur CINQ » dans `web/functional/Home/i18n/locales/fr.json` et `web/functional/Home/tests/HomePage.nuxt.spec.ts` ; produit « CINQ » dans `web/CLAUDE.md`
- [X] T009 Déclarer les couches dans `web/nuxt.config.ts` : `technical` ajoute `RichText`, `functional` ajoute `Account`, `Catalog`, `Authoring`, `Moderation`, `Learning` ; activer `experimental: { decorators: true }` (requis par `laravel-raom-nuxt`) ; créer le `nuxt.config.ts` vide et `i18n/locales/fr.json` de chaque nouvelle couche sous `web/technical/RichText/` et `web/functional/<Couche>/`

---

## Phase 2: Foundational (prérequis bloquants)

**Purpose**: comptes (modèle), permissions, session SPA, enveloppe d'erreur, catalogue de base,
thème, client API et session côté web.

**⚠️ CRITICAL**: aucune story ne commence avant la fin de cette phase.

### Back

- [X] T010 Compléter le modèle `User` dans `back/functional/users/src/Models/User.php` et une migration `back/functional/users/database/migrations/2026_09_24_000100_add_profile_columns_to_users_table.php` : `display_name` varchar(60) obligatoire, « 2 à 60 caractères, public » ; `timezone` varchar(64), défaut `Europe/Paris` ; email unique insensible à la casse ; `MustVerifyEmail`, `HasRoles`, `Prunable` (« sans `email_verified_at` et créé il y a plus de 7 jours »)
- [X] T011 Publier les migrations de `spatie/laravel-permission` dans `back/functional/users/database/migrations/` et écrire la migration de données de référence `back/functional/users/database/migrations/2026_09_24_000200_seed_roles_and_permissions.php` : rôles `member` et `admin` ; permissions `categories.manage`, `subjects.moderate`, `reports.review`, `moderation.history.view`, toutes données au rôle `admin` (`updateOrInsert`, idempotente)
- [X] T012 Configurer Sanctum en mode SPA et CORS avec identifiants dans `back/functional/users/config/sanctum.php` et `back/technical/osdd/config/cors.php` (`supports_credentials: true`, origine `FRONTEND_URL`) ; ajouter le middleware `statefulApi()` et `SESSION_DRIVER=database` avec la migration `sessions`
- [X] T013 Configurer Fortify en mode headless dans `back/functional/users/config/fortify.php` (`views: false`, fonctionnalités `registration`, `resetPasswords`, `emailVerification`) et l'enregistrer dans `back/functional/users/src/Providers/FortifyServiceProvider.php`
- [X] T014 Créer l'exception métier `BusinessRuleException` (code et message traduit) et son rendu JSON `{ "code", "message" }` dans `back/technical/osdd/src/Exceptions/BusinessRuleException.php`, rendue par le gestionnaire d'exceptions de `back/bootstrap/app.php`
- [X] T015 [P] Ajouter les traductions françaises de base (validation, auth, passwords, pagination) dans `back/technical/osdd/lang/fr/` et régler la locale par défaut à `fr`
- [X] T016 [P] Créer la commande `users:grant-admin {email}` dans `back/functional/users/src/Console/GrantAdminCommand.php`, qui donne le rôle `admin` à un compte existant, avec son test `back/functional/users/tests/Feature/GrantAdminCommandTest.php`
- [X] T017 [P] Planifier `model:prune` chaque jour dans `back/routes/console.php`
- [X] T018 Créer les migrations du catalogue dans `back/functional/catalog/database/migrations/` : extension `pg_trgm` ; `categories` (`name` varchar(60) obligatoire, `name_normalized` varchar(60) unique, `position` integer unique) ; `tags` (`name` varchar(30) unique) ; `subjects` (`author_id`, `category_id`, `title` varchar(120) « 3 à 120 caractères », `description` text « 2 000 caractères au plus », `status` varchar(16) `draft`/`published`/`retired`, `published_at`, `retired_reason`, `retired_at`, `search_document` text avec index GIN `gin_trgm_ops`) ; `subject_tag` (clé composée) ; `questions` (`subject_id`, `recto_html`, `verso_html`, `position` unique par sujet). Toutes les clés étrangères en `RESTRICT`, aucune cascade
- [X] T019 [P] Créer l'enum `SubjectStatus` (`draft`, `published`, `retired`) dans `back/functional/catalog/src/Enums/SubjectStatus.php`
- [X] T020 Créer les modèles `Category`, `Tag`, `Subject`, `Question` avec leurs relations et casts dans `back/functional/catalog/src/Models/`
- [X] T021 [P] Créer `TextNormalizer` (minuscules, sans accents, espaces réduits) dans `back/functional/catalog/src/Support/TextNormalizer.php` et son test `back/functional/catalog/tests/Unit/TextNormalizerTest.php`
- [X] T022 [P] Créer les factories `CategoryFactory`, `TagFactory`, `SubjectFactory` (états `draft`, `published`, `retired`) et `QuestionFactory` avec `faker()` de `xefi/faker-php-laravel` dans `back/functional/catalog/database/factories/`
- [X] T023 Créer `CatalogSeeder` dans `back/functional/catalog/database/seeders/CatalogSeeder.php` : 6 catégories (Langues, Histoire, Informatique, Sciences, Géographie, Musique), une vingtaine de sujets de statuts variés répartis entre plusieurs auteurs, avec 5 à 30 questions chacun, via les factories uniquement

### Web

- [X] T024 Créer le thème CINQ dans `web/technical/Theme/nuxt.config.ts` : thème Vuetify `cinq` (jaune `#FACC15`, bleu `#3B82F6`, crème `#FFFBEB`, encre `#0A0A0A`, erreur `#B91C1C`), valeurs par défaut `rounded: 0` et `elevation: 0` ; polices Archivo, Space Grotesk et JetBrains Mono chargées depuis Google Fonts dans `app.head`
- [X] T025 Créer les règles globales de la maquette dans `web/technical/Theme/app/assets/styles/cinq.scss` : `box-sizing: border-box` sur `a`, `button`, `input`, `select`, `textarea` ; tailles en `rem` avec `clamp()` sous 600 px et plancher de 12 px ; `hyphens: auto` et `text-wrap: balance` sur les titres ; soulignement réservé (`border-bottom: 4px solid transparent`) sur tous les onglets ; focus visible bleu 4 px ; `prefers-reduced-motion` ; classes utilitaires de bordure 4 px et d'ombre portée
- [X] T026 Créer la mise en page commune dans `web/technical/Theme/app/layouts/default.vue`, avec `web/technical/Theme/app/components/AppHeader.vue` (en-tête visiteur, inscrit, admin selon la session) et `web/technical/Theme/app/components/AppTabBar.vue` (barre du bas mobile : Catalogue, Réviser, Sujets, Créer, Compte) ; libellés dans `web/technical/Theme/i18n/locales/fr.json`
- [X] T027 Créer le plugin `web/technical/ApiClient/app/plugins/laravel-raom.ts` : `$fetch.create` avec `baseURL` depuis `runtimeConfig.public.apiBaseUrl`, `credentials: 'include'`, en-tête `X-XSRF-TOKEN` lu dans le cookie `XSRF-TOKEN`, appel préalable à `/sanctum/csrf-cookie`, transmission de l'en-tête `cookie` pendant le rendu serveur ; fourni sous `laravelRaom.fetch`
- [X] T028 Créer `web/technical/ApiClient/app/composables/useApiError.ts`, qui transforme les réponses `{ code, message }` et les erreurs de validation 422 en messages affichables par champ
- [X] T029 Créer le store de session `web/technical/ApiClient/app/stores/session.ts` (utilisateur, permissions, `fetchUser` via `GET /user`, `can(permission)`) et les middlewares `web/technical/ApiClient/app/middleware/auth.ts`, `guest.ts` et `permission.ts`
- [X] T030 [P] Écrire les tests du store et des middlewares dans `web/technical/ApiClient/tests/session.nuxt.spec.ts`

**Checkpoint**: fondations prêtes, les stories peuvent commencer.

---

## Phase 3: User Story 1 — Consulter le catalogue sans compte (Priority: P1) 🎯 MVP

**Goal**: un visiteur parcourt les catégories, cherche un sujet et lit ses questions et réponses,
sans jamais voir un brouillon ni un sujet retiré.

**Independent Test**: avec le contenu de `CatalogSeeder`, un visiteur trouve un sujet par catégorie
et par recherche (« irreguliers » trouve « Verbes irréguliers anglais »), en lit toutes les
questions, et reçoit « contenu introuvable » sur l'adresse directe d'un brouillon.

### Tests for User Story 1 ⚠️

- [X] T031 [P] [US1] Écrire `back/functional/catalog/tests/Feature/PublicCatalogTest.php` : liste par catégorie limitée aux sujets publiés, triée par `published_at` décroissant, paginée par 20 ; champs listés (titre, description, catégorie, tags, nom affiché de l'auteur, nombre de questions) ; email de l'auteur jamais exposé (FR-022, FR-024, FR-026)
- [X] T032 [P] [US1] Écrire `back/functional/catalog/tests/Feature/SubjectVisibilityTest.php` : pour un brouillon et un sujet retiré, en visiteur, en inscrit, en auteur et en admin, vérifier la liste, la recherche et l'accès direct (404, jamais 403) ; couverture de tous les cas (FR-023, SC-006)
- [X] T033 [P] [US1] Écrire `back/functional/catalog/tests/Feature/SubjectSearchTest.php` : insensible à la casse et aux accents, sur titre, description et tags, fragments de mots, refus sous 2 caractères (FR-025)

### Implementation for User Story 1

- [X] T034 [US1] Créer les contrôles d'accès `SubjectControl` (périmètres : public `status = published`, auteur ses sujets, `subjects.moderate` tous), `QuestionControl` (selon le sujet) et `CategoryControl` (lecture pour tous) dans `back/functional/catalog/src/Controls/`, et attacher `HasControl` aux modèles
- [X] T035 [US1] Créer les ressources lomkit `CategoryResource`, `TagResource`, `SubjectResource` et `QuestionResource` (champs et relations de [contracts/api.md](contracts/api.md), auteur réduit à `id` et `display_name`, comptage `questions_count`, limites de pagination `[20]`) et leurs contrôleurs dans `back/functional/catalog/src/Rest/`, enregistrés dans `back/functional/catalog/routes/api.php` avec lecture ouverte aux visiteurs
- [X] T036 [US1] Tenir `subjects.search_document` à jour (titre, description et tags normalisés par `TextNormalizer`) à chaque enregistrement d'un sujet et à chaque synchronisation de ses tags, par le listener `back/functional/catalog/src/Listeners/RefreshSubjectSearchDocument.php`
- [X] T037 [US1] Créer l'instruction lomkit `search` (champ `q`, normalisé, `ILIKE` sur `search_document`, 422 sous 2 caractères) dans `back/functional/catalog/src/Rest/Instructions/SearchInstruction.php`
- [X] T038 [P] [US1] Créer les modèles raom `Category`, `Tag`, `Subject` et `Question` dans `web/functional/Catalog/app/models/`
- [X] T039 [P] [US1] Créer le rendu du HTML assaini `web/technical/RichText/app/components/RichTextView.vue` (styles du gras, des listes, du code et des liens de la maquette)
- [X] T040 [US1] Créer l'accueil `web/functional/Home/app/pages/index.vue` d'après les écrans « Accueil — landing » et « Mobile · Accueil » : hero et recherche, 3 sujets publiés les plus récents, la méthode en 5 boîtes, les auteurs de ces sujets, l'appel à l'inscription
- [X] T041 [US1] Créer le catalogue par catégorie `web/functional/Catalog/app/pages/categories/[id].vue`, avec `web/functional/Catalog/app/components/SubjectCard.vue` et `CategoryChips.vue`, d'après « Catalogue par catégorie »
- [X] T042 [US1] Créer la recherche `web/functional/Catalog/app/pages/recherche.vue` (états résultats, aucun résultat avec catégories, moins de 2 caractères) d'après « Recherche »
- [X] T043 [US1] Créer la page d'un sujet `web/functional/Catalog/app/pages/sujets/[id]/index.vue` (en-tête, tags, questions avec « Afficher la réponse » et « Afficher toutes les réponses », invitation à se connecter pour signaler) d'après « Sujet — vues, apprendre, signaler », vue visiteur
- [X] T044 [US1] Créer le mode cartes `web/functional/Catalog/app/pages/sujets/[id]/cartes.vue` (retourner, précédente, suivante, progression) d'après « Sujet — mode cartes »
- [X] T045 [US1] Créer la page d'erreur `web/technical/Theme/app/error.vue` (« Contenu introuvable » pour les 404) d'après « Contenu introuvable (404) »
- [X] T046 [P] [US1] Ajouter les textes de la couche dans `web/functional/Catalog/i18n/locales/fr.json` et `web/functional/Home/i18n/locales/fr.json`
- [X] T047 [P] [US1] Écrire `web/functional/Catalog/tests/SubjectCard.nuxt.spec.ts`, `SearchPage.nuxt.spec.ts` et `SubjectPage.nuxt.spec.ts` (affichage, états de la recherche, révélation des réponses)

**Checkpoint**: le catalogue public fonctionne seul, avec le contenu du seeder.

---

## Phase 4: User Story 2 — Créer un compte et se connecter (Priority: P1)

**Goal**: inscription avec confirmation obligatoire de l'email, connexion, déconnexion, mot de
passe oublié.

**Independent Test**: créer un compte, voir la connexion refusée avant confirmation, confirmer
depuis Mailpit, se déconnecter, se reconnecter, réinitialiser le mot de passe.

### Tests for User Story 2 ⚠️

- [ ] T048 [P] [US2] Écrire `back/functional/users/tests/Feature/RegistrationTest.php` : compte inactif sans session, email de confirmation, mots de passe différents refusés, 8 caractères minimum, email déjà pris (actif et inactif) (FR-001, FR-002, scénarios 1, 2 et 6)
- [ ] T049 [P] [US2] Écrire `back/functional/users/tests/Feature/EmailVerificationTest.php` : lien valable 24 h et à usage unique, lien expiré (`link_expired`), renvoi qui invalide le précédent, connexion ouverte après confirmation (scénarios 3 et 5)
- [ ] T050 [P] [US2] Écrire `back/functional/users/tests/Feature/LoginTest.php` : message générique `invalid_credentials`, `email_not_verified` seulement après un mot de passe correct, verrou `locked` de 15 min après 5 échecs sur un même compte, session de 30 jours, déconnexion (FR-003, FR-004, FR-006, scénarios 4, 7, 8 et 10)
- [ ] T051 [P] [US2] Écrire `back/functional/users/tests/Feature/PasswordResetTest.php` : même réponse que l'email existe ou non, lien de 60 min à usage unique, confirmation du nouveau mot de passe, fermeture des autres sessions (FR-005, FR-006, scénario 9)
- [ ] T052 [P] [US2] Écrire `back/functional/users/tests/Feature/PruneUnverifiedUsersTest.php` : un compte non confirmé de plus de 7 jours est supprimé, pas un compte confirmé ni un compte récent

### Implementation for User Story 2

- [ ] T053 [US2] Créer l'action Fortify `back/functional/users/src/Actions/CreateNewUser.php` (`display_name` 2 à 60 caractères, email unique, mot de passe de 8 caractères minimum et `confirmed`, `timezone` IANA) et une réponse d'inscription 201 sans ouverture de session
- [ ] T054 [US2] Personnaliser l'authentification dans `back/functional/users/src/Providers/FortifyServiceProvider.php` : `authenticateUsing` qui lève `email_not_verified` pour un compte inactif au mot de passe correct, limiteur `login` à 5 tentatives par email sur 15 minutes (réponse `locked` avec `retry_after`), mise à jour du fuseau horaire à la connexion, « se souvenir » par défaut pendant 30 jours
- [ ] T055 [US2] Régler la vérification d'email dans `back/functional/users/config/auth.php` (`verification.expire: 1440`, `passwords.users.expire: 60`) et faire ouvrir la session par la route de vérification, avec `link_expired` si le lien est expiré ou déjà utilisé
- [ ] T056 [US2] Créer les notifications françaises `back/functional/users/src/Notifications/VerifyEmailNotification.php` et `ResetPasswordNotification.php`, signées CINQ, dont les liens pointent vers les pages du web (`FRONTEND_URL`), avec leurs textes dans `back/functional/users/lang/fr/`
- [ ] T057 [US2] Créer l'action Fortify `back/functional/users/src/Actions/ResetUserPassword.php`, qui confirme le nouveau mot de passe et supprime les autres sessions de l'utilisateur dans `sessions`
- [ ] T058 [US2] Créer `GET /user` (`id`, `display_name`, `email`, `permissions[]`) dans `back/functional/users/src/Http/Controllers/CurrentUserController.php`, routé dans `back/functional/users/routes/api.php` sous `auth:sanctum`
- [ ] T059 [US2] Créer `web/functional/Account/app/composables/useAuth.ts` (inscription, connexion, déconnexion, renvoi du lien, mot de passe oublié, réinitialisation, par le `fetch` de `laravelRaom` ; fuseau pris dans `Intl`)
- [ ] T060 [P] [US2] Créer `web/functional/Account/app/components/GoogleSoonButton.vue` (« Continuer avec Google », désactivé, étiquette « Bientôt »)
- [ ] T061 [US2] Créer la connexion `web/functional/Account/app/pages/connexion.vue` (états erreur, compte bloqué, compte non confirmé avec renvoi du lien, lien renvoyé) d'après « Connexion »
- [ ] T062 [US2] Créer l'inscription `web/functional/Account/app/pages/inscription/index.vue` (mots de passe différents, email pris, email en attente) d'après « Inscription — étape 1 »
- [ ] T063 [US2] Créer la confirmation `web/functional/Account/app/pages/inscription/confirmation.vue` (envoyé, renvoyé, confirmé, expiré ; traite le lien de vérification) d'après « Confirmation de l'email — étape 2 »
- [ ] T064 [US2] Créer `web/functional/Account/app/pages/mot-de-passe-oublie.vue` et `web/functional/Account/app/pages/reinitialiser-mot-de-passe.vue` d'après « Mot de passe oublié » et « Nouveau mot de passe »
- [ ] T065 [US2] Créer le menu du compte `web/functional/Account/app/components/AccountMenu.vue` (ordinateur) et la page `web/functional/Account/app/pages/compte.vue` (mobile), avec la section administration selon les permissions et la déconnexion, d'après « Menu du compte et déconnexion »
- [ ] T066 [P] [US2] Ajouter les textes dans `web/functional/Account/i18n/locales/fr.json`
- [ ] T067 [P] [US2] Écrire `web/functional/Account/tests/LoginPage.nuxt.spec.ts`, `RegisterPage.nuxt.spec.ts` et `ConfirmationPage.nuxt.spec.ts`

**Checkpoint**: les comptes fonctionnent seuls ; les stories suivantes peuvent s'appuyer sur la connexion.

---

## Phase 5: User Story 3 — Créer et publier un sujet (Priority: P1)

**Goal**: un inscrit crée un brouillon, y écrit des questions mises en forme, les réordonne, puis
publie et dépublie son sujet.

**Independent Test**: créer un brouillon de 3 questions invisible aux visiteurs, le publier et le
voir en visiteur, puis le dépublier.

### Tests for User Story 3 ⚠️

- [ ] T068 [P] [US3] Écrire `back/functional/catalog/tests/Feature/SubjectAuthoringTest.php` : création en brouillon, titre de 3 à 120 caractères, description de 2 000 au plus, catégorie obligatoire, tags normalisés (10 au plus, 30 caractères chacun, doublons ignorés), modification visible dès l'enregistrement, suppression définitive avec ses questions, « Mes sujets » avec statut et motif de retrait (FR-011 à FR-014, FR-017, FR-019, FR-020)
- [ ] T069 [P] [US3] Écrire `back/functional/catalog/tests/Feature/SubjectPublicationTest.php` : publication refusée sans question (`subject_has_no_question`) ou pour un sujet retiré (`subject_retired`), dépublication, dernière question d'un sujet publié non supprimable (`last_question_of_published_subject`) (FR-016 à FR-018, FR-033)
- [ ] T070 [P] [US3] Écrire `back/functional/catalog/tests/Feature/QuestionContentTest.php` : recto et verso de 1 à 5 000 caractères de texte visible, 500 questions au plus, script, `onclick` et lien `javascript:` retirés, mise en forme autorisée conservée, réordonnancement (FR-014, FR-015)

### Implementation for User Story 3

- [ ] T071 [US3] Configurer la liste blanche HTML dans `back/functional/catalog/config/purify.php` (`p, br, strong, em, ul, ol, li, code, pre, a[href]`, liens `http(s)` et `mailto`, `rel="noopener nofollow ugc"`) et créer le cast `back/functional/catalog/src/Casts/SanitizedHtml.php` appliqué à `recto_html` et `verso_html`
- [ ] T072 [US3] Ajouter les règles de création et de modification à `SubjectResource` et `QuestionResource` dans `back/functional/catalog/src/Rest/Resources/` (compte à l'email confirmé, limites de taille, tags par nom, 500 questions, modification interdite à l'auteur d'un sujet retiré) et le périmètre d'écriture à `SubjectControl` et `QuestionControl`
- [ ] T073 [US3] Créer les actions lomkit `publish` et `unpublish` dans `back/functional/catalog/src/Rest/Actions/PublishSubject.php` et `UnpublishSubject.php`, avec les erreurs `subject_has_no_question` et `subject_retired`
- [ ] T074 [US3] Créer l'action `reorder` des questions dans `back/functional/catalog/src/Rest/Actions/ReorderQuestions.php`, et le refus de supprimer la dernière question d'un sujet publié dans `QuestionResource`
- [ ] T075 [US3] Déclencher les événements `SubjectDeleting` et `QuestionDeleted` dans `back/functional/catalog/src/Events/`, et supprimer les questions d'un sujet supprimé par le listener `back/functional/catalog/src/Listeners/DeleteSubjectQuestions.php`
- [ ] T076 [P] [US3] Créer l'éditeur `web/technical/RichText/app/components/RichTextEditor.vue` (TipTap : gras, italique, listes, code en ligne et en bloc, lien ; barre d'outils avec libellés accessibles)
- [ ] T077 [P] [US3] Créer `web/functional/Authoring/app/components/TagInput.vue` (10 tags au plus, suggestions de `Tag` pendant la saisie)
- [ ] T078 [US3] Créer « Mes sujets » `web/functional/Authoring/app/pages/mes-sujets.vue` (filtres par statut avec leurs nombres, publier, dépublier, supprimer avec confirmation, publication refusée sans question, motif des sujets retirés, liste vide) d'après « Mes sujets — filtres, publier, supprimer »
- [ ] T079 [US3] Créer « Nouveau sujet » `web/functional/Authoring/app/pages/sujets/nouveau.vue` (erreurs par champ, récapitulatif d'erreurs) d'après « Nouveau sujet »
- [ ] T080 [US3] Créer l'éditeur de sujet `web/functional/Authoring/app/pages/sujets/[id]/modifier.vue` (champs du sujet, questions avec monter, descendre, modifier, supprimer, ajout, publication refusée, dernière question, échec d'enregistrement avec saisie conservée et « Réessayer », bandeau et lecture seule d'un sujet retiré) d'après « Éditeur — questions, publication, retrait »
- [ ] T081 [US3] Ajouter l'entrée auteur sur la page d'un sujet dans `web/functional/Catalog/app/pages/sujets/[id]/index.vue` (« Modifier », « Vous êtes l'auteur de ce sujet »)
- [ ] T082 [P] [US3] Ajouter les textes dans `web/functional/Authoring/i18n/locales/fr.json` et `web/technical/RichText/i18n/locales/fr.json`
- [ ] T083 [P] [US3] Écrire `web/functional/Authoring/tests/MySubjectsPage.nuxt.spec.ts`, `SubjectEditor.nuxt.spec.ts` et `web/technical/RichText/tests/RichTextEditor.nuxt.spec.ts`

**Checkpoint**: la création et la publication fonctionnent ; le catalogue se remplit par les auteurs.

---

## Phase 6: User Story 6 — Apprendre un sujet avec la méthode Leitner (Priority: P1)

**Goal**: apprendre un sujet, réviser un ou plusieurs sujets en séance avec « Je savais » ou « Je
ne savais pas », voir le bilan.

**Independent Test**: apprendre un sujet de 5 questions, répondre 3 « je savais » et 2 « je ne
savais pas », vérifier 3 cartes en boîte 2 dues dans 2 jours et 2 en boîte 1 dues demain.

### Tests for User Story 6 ⚠️

- [ ] T084 [P] [US6] Écrire `back/functional/learning/tests/Unit/LeitnerScheduleTest.php` : les 10 transitions du tableau de [data-model.md](data-model.md) (boîtes 1 à 5, « je savais » et « je ne savais pas »), dates calculées dans plusieurs fuseaux (SC-010)
- [ ] T085 [P] [US6] Écrire `back/functional/learning/tests/Feature/LearnSubjectTest.php` : apprendre crée une progression en boîte 1 due aujourd'hui par question, `already_learning`, sujet non publié refusé, arrêter d'apprendre supprime tout (FR-041, FR-050)
- [ ] T086 [P] [US6] Écrire `back/functional/learning/tests/Feature/ReviewSessionTest.php` : instruction `due` limitée aux sujets choisis et publiés, tri du plus en retard au plus récent, réponse qui applique la règle et renvoie `from_box`, `to_box`, `next_review_on`, deuxième réponse refusée (`card_not_due`), réponses enregistrées une par une, comptages de `learnings` (FR-042 à FR-049)
- [ ] T087 [P] [US6] Écrire `back/functional/learning/tests/Feature/ContentChangesTest.php` : question ajoutée en boîte 1 pour chaque apprenant, question supprimée retirée, sujet dépublié en pause puis repris, sujet supprimé avec sa progression (FR-051)

### Implementation for User Story 6

- [ ] T088 [US6] Créer les migrations dans `back/functional/learning/database/migrations/` : `learnings` (unique `user_id`, `subject_id`) ; `card_progress` (`box` smallint « 1 à 5 », `next_review_on` date, `last_answered_at`, unique `user_id`, `question_id`, index `user_id`, `subject_id`, `next_review_on`) ; `review_answers` (`known` boolean, `from_box` et `to_box` smallint « 1 à 5 », `answered_at`) ; aucune cascade
- [ ] T089 [US6] Créer les modèles `Learning`, `CardProgress`, `ReviewAnswer` dans `back/functional/learning/src/Models/` et leurs factories dans `back/functional/learning/database/factories/`
- [ ] T090 [US6] Créer la classe pure `back/functional/learning/src/Domain/LeitnerSchedule.php` : boîte d'arrivée (« je savais » boîte + 1 plafonnée à 5, « je ne savais pas » boîte 1) et prochaine date (1, 2, 4, 8, 16 jours selon la boîte d'arrivée) dans le fuseau de l'utilisateur
- [ ] T091 [US6] Créer `LearningControl` et `CardProgressControl` (l'utilisateur voit ses seules données) dans `back/functional/learning/src/Controls/`, puis les ressources `LearningResource` (création par `subject_id`, suppression, comptages `due_today_count`, `box_1_count` à `box_5_count`, `next_review_on`) et `CardProgressResource` dans `back/functional/learning/src/Rest/Resources/`, routées dans `back/functional/learning/routes/api.php`
- [ ] T092 [US6] Créer les progressions d'un nouvel apprentissage (boîte 1, dues aujourd'hui) et les supprimer avec lui par les listeners `back/functional/learning/src/Listeners/CreateCardsForLearning.php` et `DeleteCardsOfLearning.php`
- [ ] T093 [US6] Créer l'instruction `due` (`subject_ids[]`, aujourd'hui dans le fuseau de l'utilisateur, sujets publiés, tri par `next_review_on` puis position de la question) dans `back/functional/learning/src/Rest/Instructions/DueCardsInstruction.php`
- [ ] T094 [US6] Créer l'action `answer` (`card_progress_id`, `known`) dans `back/functional/learning/src/Rest/Actions/AnswerCard.php` : `card_not_due` si la carte n'est pas due, sinon application de `LeitnerSchedule`, enregistrement de `ReviewAnswer`, réponse `from_box`, `to_box`, `next_review_on`
- [ ] T095 [US6] Répercuter les changements de contenu par les listeners de `back/functional/learning/src/Listeners/` : question créée, par le job `back/functional/learning/src/Jobs/AddQuestionToLearners.php` ; question supprimée ; sujet supprimé (écoute `QuestionDeleted` et `SubjectDeleting` de la couche catalog)
- [ ] T096 [P] [US6] Créer les modèles raom `Learning` et `CardProgress` dans `web/functional/Learning/app/models/`
- [ ] T097 [P] [US6] Créer `web/functional/Learning/app/components/BoxBar.vue` (5 boîtes, nombre de cartes, intervalle, libellé accessible)
- [ ] T098 [US6] Créer `web/functional/Learning/app/components/LearnSubjectPanel.vue` (« Apprendre ce sujet », progression, « Réviser ce sujet », « Arrêter d'apprendre » avec confirmation, invitation pour un visiteur ; boutons qui partagent la largeur sur mobile) et l'intégrer à `web/functional/Catalog/app/pages/sujets/[id]/index.vue`
- [ ] T099 [US6] Créer « Mes révisions » `web/functional/Learning/app/pages/revisions/index.vue` (sélection de plusieurs sujets, « Réviser la sélection · N cartes », tout ou aucun, rien à réviser, aucun sujet appris, arrêter d'apprendre) d'après « Mes révisions — sélection des sujets »
- [ ] T100 [US6] Créer `web/functional/Learning/app/composables/useReviewSession.ts` (chargement des cartes dues, réponse, suivi des résultats, bilan) et la séance `web/functional/Learning/app/pages/revisions/seance.vue` (recto, « Afficher la réponse », réponses possibles seulement après le verso et désactivées après le premier clic, retour sur la boîte et la date, quitter avec confirmation, erreur d'enregistrement, bilan) d'après « Séance — auto-évaluation et bilan »
- [ ] T101 [P] [US6] Ajouter les textes dans `web/functional/Learning/i18n/locales/fr.json`
- [ ] T102 [P] [US6] Écrire `web/functional/Learning/tests/ReviewSession.nuxt.spec.ts` (réponses bloquées avant le verso, un seul clic pris en compte, bilan) et `RevisionsPage.nuxt.spec.ts`

**Checkpoint**: le cœur du produit, apprendre et retenir, fonctionne.

---

## Phase 7: User Story 4 — Gérer les catégories (Priority: P2)

**Goal**: un administrateur crée, renomme, réordonne et supprime les catégories.

**Independent Test**: créer une catégorie, la renommer, la voir proposée aux auteurs, être bloqué en
voulant supprimer une catégorie qui contient des sujets.

### Tests for User Story 4 ⚠️

- [ ] T103 [P] [US4] Écrire `back/functional/catalog/tests/Feature/CategoryManagementTest.php` : écriture réservée à `categories.manage`, nom en double refusé sans tenir compte de la casse ni des accents (`category_name_taken`), suppression d'une catégorie non vide refusée avec le nombre de sujets (`category_not_empty`), réordonnancement (FR-008 à FR-010)

### Implementation for User Story 4

- [ ] T104 [US4] Ajouter à `CategoryResource` et `CategoryControl` l'écriture pour `categories.manage`, le calcul de `name_normalized`, les erreurs `category_name_taken` et `category_not_empty`, et l'action `reorder` dans `back/functional/catalog/src/Rest/Actions/ReorderCategories.php`
- [ ] T105 [US4] Créer la page `web/functional/Moderation/app/pages/admin/categories.vue` (création, renommage, réordonnancement, suppression avec confirmation ou refus, sous le middleware `permission`) d'après « Catégories »
- [ ] T106 [P] [US4] Ajouter les textes dans `web/functional/Moderation/i18n/locales/fr.json` et écrire `web/functional/Moderation/tests/CategoriesPage.nuxt.spec.ts`

**Checkpoint**: les catégories sont gérées depuis l'interface.

---

## Phase 8: User Story 5 — Signaler un sujet et modérer (Priority: P2)

**Goal**: un inscrit signale un sujet ; un administrateur ignore les signalements ou retire le
sujet, le rétablit, et consulte l'historique.

**Independent Test**: signaler un sujet, le retirer avec un motif en admin, voir le statut et le
motif côté auteur, l'impossibilité de le republier et l'entrée dans l'historique.

### Tests for User Story 5 ⚠️

- [ ] T107 [P] [US5] Écrire `back/functional/moderation/tests/Feature/ReportTest.php` : signalement d'un sujet publié par un inscrit confirmé qui n'en est pas l'auteur, motifs de la liste fermée, commentaire de 500 caractères au plus, second signalement en attente refusé (`report_already_pending`), sujet toujours visible (FR-027 à FR-029)
- [ ] T108 [P] [US5] Écrire `back/functional/moderation/tests/Feature/ModerationDecisionTest.php` : file réservée à `reports.review`, regroupée par sujet et triée du plus ancien au plus récent ; ignorer clôt les signalements ; retirer exige un motif (`reason_required`), passe le sujet en `retired` et clôt ses signalements ; rétablir le repasse en `draft` ; historique complet et en lecture seule (FR-030 à FR-034)

### Implementation for User Story 5

- [ ] T109 [US5] Créer les migrations `reports` (`reason` varchar(24) `inappropriate`/`incorrect`/`spam`/`copyright`/`other`, `comment` varchar(500) null, `status` `pending`/`closed`, index unique partiel `subject_id`, `reporter_id` `WHERE status = 'pending'`) et `moderation_decisions` (`decision` `ignored`/`retired`/`restored`, `subject_title` varchar(120), `reason` obligatoire pour `retired`) dans `back/functional/moderation/database/migrations/`
- [ ] T110 [US5] Créer les enums `ReportReason`, `ReportStatus`, `DecisionType` dans `back/functional/moderation/src/Enums/`, les modèles `Report` et `ModerationDecision` dans `back/functional/moderation/src/Models/` et leurs factories
- [ ] T111 [US5] Créer `ReportControl` et `ReportResource` (création par un inscrit, lecture pour `reports.review`, `report_already_pending`) dans `back/functional/moderation/src/`, routés dans `back/functional/moderation/routes/api.php`
- [ ] T112 [US5] Créer `ModerationDecisionResource` (création pour `subjects.moderate` : `ignored` clôt les signalements, `retired` exige un motif, retire le sujet et clôt ses signalements, `restored` repasse le sujet en brouillon ; lecture pour `moderation.history.view`) et l'application de la décision dans `back/functional/moderation/src/Actions/ApplyModerationDecision.php`
- [ ] T113 [US5] Clore les signalements en attente d'un sujet supprimé par le listener `back/functional/moderation/src/Listeners/CloseReportsOfDeletedSubject.php` (écoute `SubjectDeleting`)
- [ ] T114 [P] [US5] Créer les modèles raom `Report` et `ModerationDecision` dans `web/functional/Moderation/app/models/`
- [ ] T115 [US5] Créer `web/functional/Moderation/app/components/ReportSubjectDialog.vue` (5 motifs, commentaire, confirmation, « déjà signalé ») et l'intégrer à la page d'un sujet `web/functional/Catalog/app/pages/sujets/[id]/index.vue`, feuille du bas sur mobile
- [ ] T116 [US5] Créer la file `web/functional/Moderation/app/pages/admin/moderation.vue` (signalements regroupés, commentaires, ignorer, retirer avec motif obligatoire, file vide) d'après « File de modération »
- [ ] T117 [US5] Créer l'historique `web/functional/Moderation/app/pages/admin/historique.vue` (filtres par décision, sujets retirés, rétablir avec confirmation) d'après « Historique et rétablissement »
- [ ] T118 [US5] Ajouter la vue admin à la page d'un sujet et à l'éditeur (« Retirer le sujet » avec motif, « Rétablir », bandeau administrateur) dans `web/functional/Catalog/app/pages/sujets/[id]/index.vue` et `web/functional/Authoring/app/pages/sujets/[id]/modifier.vue`
- [ ] T119 [P] [US5] Écrire `web/functional/Moderation/tests/ModerationQueue.nuxt.spec.ts` et `ReportSubjectDialog.nuxt.spec.ts`

**Checkpoint**: toutes les stories fonctionnent.

---

## Phase 9: Polish & PWA (transverse)

**Purpose**: comportement PWA (FR-037 à FR-040), accessibilité et validation finale.

- [ ] T120 Passer `registerType` à `prompt` et configurer Workbox (`NetworkOnly` pour l'API, page de repli hors ligne) dans `web/technical/Pwa/nuxt.config.ts`
- [ ] T121 [P] Générer les icônes de la maquette (« 5 » souligné de bleu sur jaune) en 192, 512 et maskable 512 dans `web/technical/Pwa/public/icons/`, et les déclarer dans le manifeste
- [ ] T122 [P] Créer `web/technical/Pwa/app/components/InstallPrompt.vue` (`beforeinstallprompt`, instructions iPhone, « Plus tard » mémorisé) et l'entrée « Installer l'application » du menu du compte d'après « Installation »
- [ ] T123 [P] Créer `web/technical/Pwa/app/components/UpdatePrompt.vue` (« Mettre à jour » ou « Plus tard ») et `OfflineBanner.vue` (bandeau, actions d'enregistrement suspendues, retour en ligne), et la page `web/technical/Pwa/app/pages/hors-ligne.vue` d'après « Hors ligne, retour en ligne, mise à jour »
- [ ] T124 Vérifier chaque écran à 360 px de large (aucun défilement horizontal), le contraste, le focus, les cibles de 44 px et l'agrandissement du texte du téléphone, et corriger dans les couches concernées, en particulier `web/technical/Theme/app/assets/styles/cinq.scss` (SC-008, FR-052)
- [ ] T125 Lancer `./vendor/bin/sail artisan test` et `./vendor/bin/sail bin pint --test` dans `back/`, puis `pnpm test`, `pnpm lint` et `pnpm exec prettier --check .` dans `web/` ; tout doit être vert
- [ ] T126 Dérouler les 7 scénarios de [quickstart.md](quickstart.md) de bout en bout et consigner tout écart dans `specs/001-learning-content/quickstart.md`

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)** : aucune dépendance ; T001 avant tout le reste du back.
- **Foundational (Phase 2)** : dépend de la phase 1 ; bloque toutes les stories.
- **US1 (Phase 3)** : dépend de la phase 2 ; fonctionne avec le contenu du seeder, sans compte.
- **US2 (Phase 4)** : dépend de la phase 2 ; indépendante de US1.
- **US3 (Phase 5)** : dépend de US2 (connexion) ; enrichit les pages de US1.
- **US6 (Phase 6)** : dépend de US2 (connexion) ; utilise des sujets publiés (seeder ou US3).
- **US4 (Phase 7)** : dépend de US2 et de la permission `categories.manage`.
- **US5 (Phase 8)** : dépend de US2 ; enrichit la page sujet (US1) et l'éditeur (US3).
- **Polish (Phase 9)** : après les stories retenues.

### Within Each User Story

- Tests back d'abord, en échec, puis migrations, modèles, contrôles, ressources et actions.
- Le web d'une story commence quand ses endpoints sont implémentés et testés côté back.
- Les tests web ferment la story.

### Ordre de fusion

Deux merge requests liées par le numéro 001 : `back` en premier, `web` ensuite. La merge request web
reste ouverte tant que celle du back n'est pas fusionnée et déployée.

### Parallel Opportunities

- Phase 1 : T004 à T008 en parallèle, après T001 à T003.
- Phase 2 : T015 à T017, T019, T021, T022 en parallèle côté back ; côté web, T030 en parallèle une fois T029 fait ; tout le web de la phase 2 en parallèle du back.
- Dans chaque story : toutes les tâches de tests `[P]` ensemble ; les modèles raom et les composants `[P]` pendant l'implémentation back.
- Entre stories : une fois US2 terminée, US3, US6, US4 et US5 peuvent avancer en parallèle côté back (couches différentes).

---

## Parallel Example: User Story 6

```bash
# Tests back, ensemble :
T084 back/functional/learning/tests/Unit/LeitnerScheduleTest.php
T085 back/functional/learning/tests/Feature/LearnSubjectTest.php
T086 back/functional/learning/tests/Feature/ReviewSessionTest.php
T087 back/functional/learning/tests/Feature/ContentChangesTest.php

# Pendant l'implémentation back, côté web :
T096 web/functional/Learning/app/models/
T097 web/functional/Learning/app/components/BoxBar.vue
```

---

## Implementation Strategy

### MVP First (User Story 1)

1. Phases 1 et 2.
2. Phase 3 (US1) : le catalogue public, rempli par le seeder.
3. **Stop et validation** : un visiteur consulte et cherche des sujets.

### Incremental Delivery

1. US1 : la vitrine publique.
2. US2 : les comptes.
3. US3 : les auteurs remplissent le catalogue.
4. US6 : la révision Leitner, cœur du produit. C'est la première version présentable au public, sous réserve de la suppression de compte (RGPD, feature dédiée).
5. US4 et US5 : l'administration et la modération.
6. Phase 9 : la PWA complète et la validation finale.

Chaque incrément passe ses tests avant le suivant.
