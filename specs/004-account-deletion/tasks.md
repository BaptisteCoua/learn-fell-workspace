---

description: "Tâches de la feature 004 — Suppression de compte (CINQ)"
---

# Tasks: Suppression de compte

**Input**: Documents de conception dans `specs/004-account-deletion/`

**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md),
[data-model.md](data-model.md), [contracts/api.md](contracts/api.md), [quickstart.md](quickstart.md)

**Tests**: obligatoires. La constitution (principe IV) exige un test automatisé par exigence
fonctionnelle, et l'invisibilité des sujets `withheld` doit être couverte à 100 % de ses cas, comme
celle des brouillons. Dans chaque story, les tests sont écrits en premier et doivent échouer avant
l'implémentation. La table des cas attendus est dans [quickstart.md](quickstart.md).

**Organization**: une phase par user story, dans l'ordre de la spec. Dans chaque story, le back
(fournisseur) passe avant le web (consommateur).

**Maquette de référence**: aucune. L'écran reprend les composants de la page Compte de la 001.

## Format: `[ID] [P?] [Story] Description`

- **[P]** : parallélisable (fichiers différents, aucune dépendance sur une tâche non terminée)
- **[Story]** : story concernée (US1 à US5, voir spec.md)
- Tous les chemins commencent par leur repo : `back/…` ou `web/…`

## Conventions de chemins

- **Back** : couches existantes `back/functional/{users,catalog,learning,moderation,reminders}/`,
  chacune avec `src/` (espace de noms `Functional\<Couche>\`), `database/migrations`, `routes/`,
  `lang/fr/` et `tests/{Feature,Unit}`. Générer avec les commandes `osdd:*` et
  `--layer=functional/<couche>` (`back/CLAUDE.md`). Toutes les commandes via `./vendor/bin/sail`.
  Les écouteurs sont déclarés dans le service provider de leur couche, comme
  `back/functional/reminders/src/Providers/RemindersServiceProvider.php:50-52`.
- **Web** : couches existantes `web/functional/{Account,Catalog,Home,Authoring,Moderation}/`.
  Commandes via `pnpm`. Composants : logique dans un composable (`nuxt:vue-component-structure`),
  traductions dans `<couche>/i18n/locales/fr.json`, clés en anglais minuscule (`nuxt:i18n-conventions`).
- Les branches `004-account-deletion` de `back/` et `web/` sont créées par le hook
  `speckit.multirepo.branch` avant la première tâche.

---

## Phase 1: Setup

**Purpose**: aucune couche ni dépendance nouvelle (plan, « Primary Dependencies »). Seule la
constante du délai est posée ici, car toutes les stories la lisent.

- [X] T001 Ajouter la constante `public const DELETION_GRACE_DAYS = 30;` et la méthode `eraseOn(): CarbonImmutable` (`deletion_requested_at` + 30 jours, dans le fuseau `timezone` du compte) à `back/functional/users/src/Models/User.php` ; aucune option de configuration (data-model, principe VII)

---

## Phase 2: Foundational (prérequis bloquants)

**Purpose**: colonnes, clés nullables, statut `withheld`, événements, nom public, et types web
tolérant un auteur nul. Aucune story ne peut commencer avant la fin de cette phase.

### Back

- [X] T002 Migration `osdd:migration add_deletion_request_to_users_table --layer=functional/users` dans `back/functional/users/database/migrations/` : `deletion_requested_at` « `timestamp` nullable, indexé » et `keeps_published_subjects` « `boolean` nullable » ; casts `datetime` et `boolean` dans `back/functional/users/src/Models/User.php`
- [X] T003 [P] Migration dans `back/functional/catalog/database/migrations/` : `subjects.author_id` et `question_images.uploader_id` deviennent nullables, la contrainte `restrictOnDelete` reste (data-model)
- [X] T004 [P] Migration dans `back/functional/moderation/database/migrations/` : `reports.reporter_id` et `moderation_decisions.admin_id` deviennent nullables, la contrainte `restrictOnDelete` reste ; relations `reporter()` et `admin()` typées nullables dans `back/functional/moderation/src/Models/Report.php` et `ModerationDecision.php`
- [X] T005 [P] Ajouter le cas `Withheld = 'withheld'` à `back/functional/catalog/src/Enums/SubjectStatus.php` ; `isPublic()` reste vrai seulement pour `Published` ; ajouter un test unitaire qui vérifie `isPublic()` pour les quatre cas dans `back/functional/catalog/tests/Unit/SubjectStatusTest.php`
- [X] T006 [P] Créer les événements `AccountDeletionRequested(User $user)` et `AccountDeletionCancelled(User $user)` dans `back/functional/users/src/Events/` (research R2)
- [X] T007 Ajouter à `back/functional/users/src/Models/User.php` la méthode `isPendingDeletion(): bool` et l'accesseur `public_name` (« égal à `display_name`, ou nul quand `deletion_requested_at` est renseigné ») ; faire exposer `display_name` depuis `public_name` dans `back/functional/users/src/Rest/Resources/PublicUserResource.php` (research R4)
- [X] T008 Test `back/functional/users/tests/Feature/PublicUserNameTest.php` : `author.display_name` d'un sujet, `reporter.display_name` d'un signalement et `admin.display_name` d'une décision valent le nom pour un compte actif et `null` pour un compte en suppression (contrat, « Ressources lomkit existantes ») ; `author` vaut `null` quand `author_id` est nul

### Web

- [X] T009 [P] Rendre l'auteur et son nom nullables dans les types : `author` et `display_name` dans `web/functional/Catalog/app/models/Subject.ts`, `reporter` et `admin` dans `web/functional/Moderation/app/models/` ; ajouter `'withheld'` à `SubjectStatus` (`web/functional/Catalog/app/models/Subject.ts:11`)
- [X] T010 [P] Créer `web/functional/Catalog/app/utils/authorName.ts` : `authorName(author, fallback)` renvoie `author?.display_name ?? fallback` ; clés `"deleted author": "Auteur supprimé"` dans `web/functional/Catalog/i18n/locales/fr.json` et `"deleted account": "Compte supprimé"` dans `web/functional/Moderation/i18n/locales/fr.json`

**Checkpoint**: `./vendor/bin/sail artisan migrate` passe, `./vendor/bin/sail artisan test` et `pnpm test` sont verts, rien ne change pour un compte actif.

---

## Phase 3: User Story 1 - Demander la suppression de son compte (Priority: P1) 🎯 MVP

**Goal**: depuis la page Compte, confirmer avec son mot de passe ; être déconnecté partout, ne
plus recevoir de rappel, recevoir l'email avec la date.

**Independent Test**: un inscrit sans sujet publié demande la suppression ; ses deux navigateurs
sont déconnectés, l'email arrive dans Mailpit avec la date, aucun rappel ne part le soir même.

### Tests (écrits d'abord, en échec)

- [X] T011 [P] [US1] Test `back/functional/users/tests/Feature/AccountDeletionRequestTest.php` : `GET api/account/deletion` renvoie `can_request: true` et `erase_on` à J+30 dans le fuseau du compte (FR-002) ; `POST` avec un mot de passe faux renvoie 422 `errors.password` sans rien modifier (FR-003) ; `POST` valide renseigne `deletion_requested_at`, laisse `keeps_published_subjects` nul sans sujet publié, renvoie `{ erase_on }` (FR-013), supprime toutes les lignes `sessions` du compte et la requête suivante reçoit 401 (FR-006) ; `AccountDeletionRequested` est déclenché ; 401 sans session
- [X] T012 [P] [US1] Test `back/functional/users/tests/Feature/AccountDeletionNotificationTest.php` : `AccountDeletionRequestedNotification` est envoyée au compte, en file, avec la date d'effacement en toutes lettres et le nom affiché ; un échec d'envoi n'annule pas la demande (FR-012, edge case)
- [X] T013 [P] [US1] Test `back/functional/reminders/tests/Feature/RemindersOfPendingDeletionTest.php` : avec cartes dues et email et push activés, aucun rappel ne part pour un compte en suppression ; réglages et appareils restent en base (FR-007)
- [X] T014 [P] [US1] Test `back/functional/catalog/tests/Feature/HiddenSubjectsOfPendingDeletionTest.php` : dès la demande, brouillons, sujets dépubliés et sujets `retired` de l'auteur sont introuvables pour un visiteur et un autre inscrit en liste, en recherche et à l'adresse directe, et l'auteur ne peut plus les ouvrir puisque sa session est fermée (FR-008) ; ils restent visibles pour `subjects.moderate` comme aujourd'hui ; le nom de l'auteur n'apparaît dans aucune réponse (FR-011). Aucun code n'est attendu : seul l'auteur voit ces sujets, et il est déconnecté
- [X] T015 [P] [US1] Test web `web/functional/Account/tests/AccountDeletionPage.nuxt.spec.ts` : la page Compte mène à « Supprimer mon compte » (FR-001) ; l'écran liste ce qui sera effacé et conservé et la date (FR-002) ; mot de passe faux : message sous le champ ; succès : store de session vidé et arrivée sur la page de confirmation avec la date (FR-013) ; textes en français, vouvoiement (FR-025)

### Implémentation back

- [X] T016 [US1] Action `back/functional/users/src/Actions/RequestAccountDeletion.php` : vérifier le mot de passe (`Hash::check`, erreur de validation `password` sinon) ; enregistrer `deletion_requested_at = now()` et `keeps_published_subjects` reçu (nul si non fourni) ; supprimer les sessions du compte (`DB::table('sessions')->where('user_id', …)->delete()`, comme `back/functional/users/src/Actions/ResetUserPassword.php:28`) ; déclencher `AccountDeletionRequested` ; envoyer la notification ; le tout dans une transaction, la notification après le commit
- [X] T017 [US1] Notification `back/functional/users/src/Notifications/AccountDeletionRequestedNotification.php` (canal `mail`, `ShouldQueue`) et textes dans `back/functional/users/lang/fr/notifications.php` : objet « Votre compte CINQ sera supprimé le {date} », corps et bouton « Me reconnecter » vers `/connexion` selon le contrat (« Email »), sans lien d'annulation
- [X] T018 [US1] Contrôleur `back/functional/users/src/Http/Controllers/AccountDeletionController.php` (`show` et `store`) et routes `GET` et `POST account/deletion` (middleware `auth:sanctum`) dans `back/functional/users/routes/account.php` ; `store` invalide la session courante et régénère le jeton CSRF avant de répondre `{ erase_on }` ; `show` renvoie `can_request`, `blocked_reason` (nul pour l'instant, rempli en US5) et `erase_on`
- [X] T019 [US1] Exclure les comptes en suppression de l'éligibilité dans `back/functional/reminders/src/Domain/ReminderEligibility.php` (et de la sélection de `back/functional/reminders/src/Support/ReminderScheduler.php` si elle lit les comptes directement) ; ne modifier ni `next_reminder_at` ni les réglages (data-model, « Rappels »)

### Implémentation web

- [X] T020 [US1] Ajouter `getAccountDeletion()` (`GET /account/deletion`) et `requestAccountDeletion(password, keepPublishedSubjects?)` (`POST`) à `web/functional/Account/app/composables/useAuth.ts`, par `useApiFetch` comme les autres routes de compte (research R11)
- [X] T021 [US1] Composable `web/functional/Account/app/composables/useAccountDeletion.ts` : chargement de l'état, mot de passe, envoi, erreurs par `useApiError`, `sessionStore.clear()` puis `navigateTo('/compte-supprime?le=<erase_on>')` après succès
- [X] T022 [US1] Page `web/functional/Account/app/pages/supprimer-mon-compte.vue` (middleware `auth`) : ce qui sera effacé, ce qui sera conservé, date, champ mot de passe (`AccountField`), bouton « Supprimer mon compte » ; 360 px, cibles de 44 px, focus visible
- [X] T023 [US1] Page `web/functional/Account/app/pages/compte-supprime.vue` (sans middleware) : confirmation, date lue dans `le`, « Vous avez changé d'avis ? Reconnectez-vous avant cette date. » et lien vers `/connexion`
- [X] T024 [US1] Entrée « Supprimer mon compte » vers `/supprimer-mon-compte` dans `web/functional/Account/app/composables/useAccountMenu.ts` (rendue par `web/functional/Account/app/components/AccountMenuPanel.vue`) ; toutes les clés de l'écran dans `web/functional/Account/i18n/locales/fr.json`

**Checkpoint**: US1 se démontre seule : demande, déconnexion, email, rappels coupés.

---

## Phase 4: User Story 2 - Décider du sort de ses sujets publiés (Priority: P1)

**Goal**: choisir « Laisser mes sujets publiés, sans mon nom » ou « Tout effacer », avec le nombre
de sujets et d'apprenants sous les yeux.

**Independent Test**: une autrice avec 2 sujets publiés et 1 brouillon choisit « Laisser » ; ses
sujets restent apprenables sous « Auteur supprimé ». Une autre choisit « Tout effacer » ; ses
sujets disparaissent pour tous sauf la modération.

### Tests (écrits d'abord, en échec)

- [X] T025 [P] [US2] Test `back/functional/learning/tests/Feature/AuthoredSubjectsSummaryTest.php` : `GET api/learning/authored-subjects-summary` compte les sujets `published` de l'auteur et les apprenants distincts autres que lui ; 0 et 0 sans sujet ; 401 sans session (FR-004)
- [X] T026 [P] [US2] Ajouter à `back/functional/users/tests/Feature/AccountDeletionRequestTest.php` : avec un sujet publié et sans `keep_published_subjects`, 422 `subject_choice_required` ; le choix est enregistré dans `keeps_published_subjects` ; sans sujet publié, aucun choix n'est exigé et un choix envoyé reste sans effet (FR-004)
- [X] T027 [P] [US2] Test `back/functional/catalog/tests/Feature/WithheldSubjectsTest.php`, à couvrir à 100 % comme les brouillons : avec « Tout effacer », les sujets `published` passent `withheld`, `published_at` inchangé ; introuvables pour visiteur et inscrit en liste, recherche, adresse directe, questions (FR-009) ; visibles pour `subjects.moderate` ; non modifiables ; retirables par la modération (data-model, transitions)
- [X] T028 [P] [US2] Test `back/functional/learning/tests/Feature/WithheldSubjectsInReviewsTest.php` : les cartes d'un sujet `withheld` ne sont plus proposées en séance ni comptées dans `due_today_count`, et ne déclenchent aucun rappel (`back/functional/learning/src/Queries/DueCardsQuery.php:42`) ; la progression des apprenants reste en base (FR-009)
- [X] T029 [P] [US2] Test `back/functional/catalog/tests/Feature/KeptSubjectsOfPendingDeletionTest.php` : avec « Laisser », les sujets restent `published`, apprenables, questions et images servies, `author.display_name` nul (FR-010)
- [X] T030 [P] [US2] Tests web : `web/functional/Account/tests/AccountDeletionPage.nuxt.spec.ts` (choix affiché seulement si `published_subjects_count > 0`, avec les deux nombres ; bouton refusé sans choix) ; `web/functional/Catalog/tests/SubjectCard.nuxt.spec.ts` et `SubjectPage.nuxt.spec.ts` (« Auteur supprimé » quand l'auteur ou son nom est nul) ; un test dans `web/functional/Moderation/tests/` pour « Compte supprimé » et le statut « Retenu » (FR-010, FR-011)

### Implémentation back

- [X] T031 [US2] Contrôleur `back/functional/learning/src/Http/Controllers/AuthoredSubjectsSummaryController.php` et route `GET learning/authored-subjects-summary` (`auth:sanctum`) dans `back/functional/learning/routes/api.php`, au format du contrat
- [X] T032 [US2] Écouteur `back/functional/catalog/src/Listeners/WithholdSubjectsOfLeavingAuthor.php` sur `AccountDeletionRequested`, déclaré dans `back/functional/catalog/src/Providers/CatalogServiceProvider.php` : si l'auteur a un sujet `published` et que `keeps_published_subjects` est nul, lever `BusinessRuleException('subject_choice_required')`, ce qui annule la transaction de `RequestAccountDeletion` (T016) ; texte dans `back/technical/osdd/lang/fr/errors.php`. La couche `users` n'a ainsi pas à connaître les sujets (principe II)
- [X] T033 [US2] Dans le même écouteur, si `keeps_published_subjects === false` : passer chaque sujet `published` de l'auteur à `withheld` sans toucher à `published_at`. Un choix envoyé par un compte sans sujet publié est enregistré mais sans effet
- [X] T034 [US2] Ajouter `Withheld` aux statuts verrouillés de `back/functional/catalog/src/Support/RetiredSubjectLock.php`, pour qu'un sujet retenu ne soit modifiable que par la modération, comme un sujet retiré ; vérifier dans T027 que `back/functional/catalog/src/Access/Controls/SubjectControl.php:29-32` laisse `subjects.moderate` le voir

### Implémentation web

- [X] T035 [US2] Ajouter à `web/functional/Account/app/composables/useAccountDeletion.ts` la lecture du résumé (`GET /learning/authored-subjects-summary` par `useApiFetch`) et le choix obligatoire ; afficher dans `web/functional/Account/app/pages/supprimer-mon-compte.vue` les deux options en boutons radio, avec les nombres
- [X] T036 [US2] Remplacer chaque lecture directe du nom de l'auteur par `authorName(subject.author, t('deleted author'))` : `web/functional/Catalog/app/components/SubjectCard.vue:12`, `web/functional/Catalog/app/pages/sujets/[id]/index.vue:27`, `web/functional/Home/app/composables/useLandingPage.ts:37-38` (initiales comprises), `web/functional/Authoring/app/pages/sujets/[id]/modifier.vue:47`, `web/functional/Moderation/app/pages/admin/historique.vue:57`, `web/functional/Moderation/app/components/ReportGroupCard.vue:57`
- [X] T037 [US2] « Compte supprimé » pour le signalant et l'administrateur : `web/functional/Moderation/app/components/ReportGroupCard.vue:37`, `web/functional/Moderation/app/pages/admin/historique.vue:47` ; statut « Retenu (suppression de compte en cours) » dans la file de modération (`web/functional/Moderation/app/components/SubjectModerationPanel.vue`) et dans `web/functional/Authoring/app/components/SubjectStatusBadge.vue`

**Checkpoint**: US1 et US2 se démontrent ensemble ; les sujets suivent le choix dès la demande.

---

## Phase 5: User Story 3 - Revenir sur sa décision (Priority: P1)

**Goal**: se reconnecter pendant le délai annule tout et rétablit tout.

**Independent Test**: après « Tout effacer », se reconnecter au jour 10 ; message d'annulation ;
sujets de retour pour les apprenants avec leur progression ; rappels repris.

### Tests (écrits d'abord, en échec)

- [X] T038 [P] [US3] Test `back/functional/users/tests/Feature/AccountDeletionCancellationTest.php` : connexion réussie d'un compte en suppression → colonnes remises à nul, `AccountDeletionCancelled` déclenché, réponse `{ deletion_cancelled: true }` ; compte actif : clé absente ; mot de passe faux et verrouillage : même réponse qu'un compte actif (FR-014, FR-024) ; après `reset-password`, la connexion suivante annule de la même façon ; droits d'administration intacts (FR-015)
- [X] T039 [P] [US3] Test `back/functional/catalog/tests/Feature/RestoreWithheldSubjectsTest.php` : après annulation, les sujets `withheld` repassent `published` avec le même `published_at`, signés du nom ; les sujets « Laisser » retrouvent le nom ; un sujet retiré pendant le délai reste `retired` (FR-015)
- [X] T040 [P] [US3] Ajouter à `back/functional/reminders/tests/Feature/RemindersOfPendingDeletionTest.php` : après annulation, le rappel repart au créneau normal avec les mêmes réglages et appareils (FR-015, US3 scénario 5)
- [X] T041 [P] [US3] Test web `web/functional/Account/tests/LoginPage.nuxt.spec.ts` : réponse `{ deletion_cancelled: true }` → message « Votre demande de suppression est annulée. » visible sur la page d'arrivée ; absent sinon

### Implémentation back

- [X] T042 [US3] Annuler dans `back/functional/users/src/Actions/AuthenticateUser.php` après authentification réussie (après les contrôles `invalid_credentials` et `email_not_verified`) : remettre à nul les deux colonnes, déclencher `AccountDeletionCancelled`, marquer la requête pour la réponse
- [X] T043 [US3] `LoginResponse` Fortify dans `back/functional/users/src/Http/Responses/LoginResponse.php`, liée dans `back/functional/users/src/Providers/FortifyServiceProvider.php` : `{ deletion_cancelled: true }` seulement après une annulation, sinon la réponse actuelle (contrat, `POST /api/login`)
- [X] T044 [US3] Écouteur `back/functional/catalog/src/Listeners/RestoreWithheldSubjects.php` sur `AccountDeletionCancelled` : `withheld` → `published` pour les sujets de l'auteur, `published_at` inchangé

### Implémentation web

- [X] T045 [US3] `login()` de `web/functional/Account/app/composables/useAuth.ts` renvoie `deletionCancelled` ; `web/functional/Account/app/composables/useLoginForm.ts` le transmet à la page d'arrivée (paramètre de requête ou état de session) ; message dans `AccountNotice` et clé dans `web/functional/Account/i18n/locales/fr.json`

**Checkpoint**: demande puis annulation rendent un compte identique à l'état d'avant.

---

## Phase 6: User Story 4 - L'effacement définitif (Priority: P1)

**Goal**: au bout de 30 jours, effacer tout ce qui identifie la personne, en une fois.

**Independent Test**: demande avec « Laisser », horloge à J+31, `model:prune` ; plus aucune
donnée du compte ; sujets apprenables sous « Auteur supprimé » ; réinscription possible.

### Tests (écrits d'abord, en échec)

- [X] T046 [P] [US4] Test `back/functional/users/tests/Feature/EraseAccountsTest.php` : `model:prune` efface un compte à J+30 et pas à J+29 (FR-016) ; un écouteur qui lève une exception laisse le compte et toutes ses données intacts, et l'élagage suivant le reprend (FR-022) ; sessions, jetons de réinitialisation, rôles et permissions supprimés ; réinscription avec la même adresse comme une adresse neuve, ancienne connexion en `invalid_credentials` (FR-023, FR-024) ; l'élagage des comptes non confirmés reste inchangé (`back/functional/users/tests/Feature/PruneUnverifiedUsersTest.php`)
- [X] T047 [P] [US4] Test `back/functional/catalog/tests/Feature/EraseSubjectsOfUserTest.php` : « Tout effacer » → sujets, questions, images et fichiers supprimés, progression des autres apprenants supprimée, signalements du sujet supprimés, décisions conservées avec `subject_title` (FR-018, FR-021) ; « Laisser » → sujets `published` avec `author_id` nul, images avec `uploader_id` nul, progression d'autrui intacte, autres sujets supprimés, images jamais rattachées supprimées (FR-019)
- [X] T048 [P] [US4] Test `back/functional/learning/tests/Feature/EraseLearningsOfUserTest.php` : apprentissages, progression et réponses du compte supprimés, y compris sur ses propres sujets conservés (FR-017)
- [X] T049 [P] [US4] Test `back/functional/moderation/tests/Feature/AnonymizeModerationOfUserTest.php` : `reporter_id` et `admin_id` mis à nul, signalements et décisions conservés et visibles dans la file et l'historique avec `reporter` et `admin` nuls (FR-020)
- [X] T050 [P] [US4] Ajouter à `back/functional/reminders/tests/Feature/ReminderSettingLifecycleTest.php` : l'effacement par élagage supprime réglages, appareils et journal (FR-017), par l'écouteur existant `DeleteRemindersOfUser`

### Implémentation back

- [X] T051 [US4] Élargir `prunable()` de `back/functional/users/src/Models/User.php:32-37` aux comptes dont `deletion_requested_at` est antérieur à maintenant moins `DELETION_GRACE_DAYS` jours ; redéfinir `prune()` pour exécuter `pruning()` et `delete()` dans `DB::transaction` ; une exception est rapportée (`report()`) et n'interrompt pas l'élagage des autres comptes (research R7)
- [X] T052 [US4] Écouteur `back/functional/users/src/Listeners/DeleteSessionsOfUser.php` sur `eloquent.deleting: User` : sessions et `password_reset_tokens` de l'adresse ; vérifier que `HasRoles` détache rôles et permissions
- [X] T053 [US4] Écouteur `back/functional/catalog/src/Listeners/EraseSubjectsOfUser.php` sur `eloquent.deleting: User`, selon research R8 : « Tout effacer » → `delete()` de chaque sujet (chaîne existante `SubjectDeleting`) ; « Laisser » → `author_id` à nul sur les sujets `published` et `uploader_id` à nul sur leurs images, `delete()` des autres sujets ; images non rattachées du compte supprimées ; jamais de suppression de masse sur le builder, pour que les événements se déclenchent
- [X] T054 [P] [US4] Écouteur `back/functional/learning/src/Listeners/DeleteLearningsOfUser.php` sur `eloquent.deleting: User` : `delete()` de chaque apprentissage du compte (chaîne `LearningDeleting` → `DeleteCardsOfLearning`) ; déclaré dans `back/functional/learning/src/Providers/LearningServiceProvider.php`
- [X] T055 [P] [US4] Écouteur `back/functional/moderation/src/Listeners/AnonymizeModerationOfUser.php` sur `eloquent.deleting: User` : `reporter_id` et `admin_id` à nul ; déclaré dans `back/functional/moderation/src/Providers/ModerationServiceProvider.php`

**Checkpoint**: l'effacement est complet, atomique et repris en cas d'échec.

---

## Phase 7: User Story 5 - Protéger la dernière administration (Priority: P2)

**Goal**: le dernier compte qui détient une permission d'administration ne peut pas se supprimer.

**Independent Test**: un seul compte avec `users:grant-admin` voit l'écran expliquer le refus ;
avec deux, l'un peut se supprimer, et l'autre devient le dernier.

- [X] T056 [P] [US5] Test `back/functional/users/tests/Feature/LastAdministratorTest.php` : seul détenteur de `categories.manage` (ou de toute autre permission d'administration) → `GET` renvoie `can_request: false`, `blocked_reason: "last_admin"`, et `POST` 422 `last_admin` ; deux détenteurs → l'un est accepté, puis l'autre est refusé tant que la demande du premier est en cours ; après annulation du premier, l'autre peut de nouveau demander (FR-005, US5)
- [X] T057 [P] [US5] Test web dans `web/functional/Account/tests/AccountDeletionPage.nuxt.spec.ts` : `can_request: false` → message explicatif, ni champ mot de passe ni bouton de confirmation
- [X] T058 [US5] `back/functional/users/src/Support/LastAdministratorGuard.php` : pour chaque permission parmi `categories.manage`, `subjects.moderate`, `reports.review`, `moderation.history.view` détenue par le compte, vérifier qu'un autre compte sans demande en cours la détient (permissions spatie, jamais le rôle) ; l'appeler dans `AccountDeletionController::show` et dans `RequestAccountDeletion` (`BusinessRuleException('last_admin')`, texte dans `back/technical/osdd/lang/fr/errors.php`)
- [X] T059 [US5] Afficher le refus dans `web/functional/Account/app/pages/supprimer-mon-compte.vue` à partir de `blocked_reason` ; clé dans `web/functional/Account/i18n/locales/fr.json`

---

## Phase 8: Polish & vérifications transverses

- [X] T060 [P] Test `back/functional/users/tests/Feature/AccountDeletionPrivacyTest.php` : inscription avec l'adresse d'un compte en suppression → même réponse qu'une adresse prise ; lien de désinscription d'un compte en suppression ou effacé → page de lien non valide de la 002 ; mot de passe oublié → réponse neutre existante (FR-024, SC-006)
- [X] T061 [P] Ajouter l'élagage des comptes à `back/CLAUDE.md` (section Run : `model:prune` efface aussi les comptes dont la suppression date de plus de 30 jours) et la commande de simulation de [quickstart.md](quickstart.md)
- [X] T062 Lancer `./vendor/bin/sail bin pint --dirty --format agent` et `./vendor/bin/sail artisan test --compact` dans `back/` ; `pnpm lint`, `pnpm exec prettier --check .` et `pnpm test` dans `web/`
- [X] T063 Dérouler les six parcours manuels de [quickstart.md](quickstart.md) à 360 px et noter le résultat dans ce fichier

---

### Résultats des parcours manuels (T063, 2026-10-01, 360 × 780)

Compte seedé n° 14 (2 sujets publiés, 2 apprenants), back et web en local :

1. **Écran** : choix proposé avec « 2 sujets publiés, appris par 2 personnes », date « 31 octobre 2026 » dans le panneau, aucun défilement horizontal à 360 px. ✅
2. **« Tout effacer »** : page « Compte fermé » avec la date ; sujets 24 et 25 en `withheld`, sujet retiré 5 inchangé ; email reçu dans Mailpit (objet, nom, date, bouton « Me reconnecter »). ✅
3. **Annulation** : reconnexion → toast « Votre demande de suppression est annulée. Bon retour sur CINQ. » ; sujets republiés avec leur `published_at` d'origine, signés du nom. ✅
4. **Sessions** : la première demande laissait une ligne `sessions` rattachée au compte, réécrite en fin de requête par la garde `sanctum`. Corrigé (`Auth::forgetUser()` dans `AccountDeletionController::store`), vérifié en relançant la demande : plus aucune session. Les tests automatisés ne reproduisent pas ce cas (session et garde du client de test), la vérification est manuelle. ✅
5. **Effacement réel** et **dernier admin** : non déroulés sur la base de dev, pour ne pas détruire de données seedées ; couverts par `EraseAccountsTest`, `EraseSubjectsOfUserTest`, `EraseLearningsOfUserTest`, `AnonymizeModerationOfUserTest` et `LastAdministratorTest`. Le compte 14 a été rétabli par reconnexion.

### Écarts entre les tâches et l'implémentation

- T007 : lomkit ne renomme pas un champ ; l'accesseur porte donc sur `display_name` lui-même (nul pendant le délai) et `ownDisplayName()` sert les emails du compte (research R4 mis à jour).
- T008, T026 : les tests qui touchent sujets ou signalements vivent dans `catalog` (`AuthorNameTest`, `WithheldSubjectsTest`) et `moderation` (`ModerationNamesTest`), pas dans `users`, pour respecter le sens des dépendances.
- T023, T024 : pages à plat `supprimer-mon-compte.vue` et `compte-supprime.vue`, car `pages/compte.vue` existe déjà et un dossier `pages/compte/` en ferait une route parente.
- T030 : « Auteur supprimé » et « Compte supprimé » testés dans `SubjectCard.nuxt.spec.ts` et `ReportGroupCard.nuxt.spec.ts` ; la page sujet utilise le même `authorName`.
- T034 : un sujet retenu a son propre message, `subject_withheld`, plutôt que `subject_retired`.
- T045 : l'annulation s'annonce par un toast après la connexion.
- T047 : le test de l'effacement est réparti par couche (sujets et images dans `catalog`, progression des apprenants dans `learning`, signalements dans `moderation`).
- Ajout hors tâches : `UnsubscribeController` refuse le lien d'un compte en cours de suppression (edge case de la spec), testé dans `UnsubscribeTest`.

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)** : aucune dépendance.
- **Foundational (Phase 2)** : dépend du Setup ; bloque toutes les stories.
- **US1 (Phase 3)** : dépend de la Phase 2. C'est le MVP.
- **US2 (Phase 4)** : dépend de US1 (l'action et l'écran de demande).
- **US3 (Phase 5)** : dépend de US1 ; T044 dépend de T033 (statut `withheld`).
- **US4 (Phase 6)** : dépend de US1 et US2 (le choix guide l'effacement).
- **US5 (Phase 7)** : dépend de US1 seulement ; peut se faire en parallèle de US2 à US4.
- **Polish (Phase 8)** : après toutes les stories.

### Back avant web

Dans chaque story, les tâches back précèdent les tâches web, et la merge request web reste ouverte
tant que celle du back n'est pas fusionnée et déployée (constitution, flux de travail).

### Parallel Opportunities

- Phase 2 : T003, T004, T005, T006 en parallèle (couches différentes), puis T009 et T010 côté web.
- Dans chaque story, tous les tests marqués [P] s'écrivent en parallèle.
- US4 : T054 et T055 en parallèle (couches `learning` et `moderation`), après T051.
- US5 en parallèle de US2 à US4, une fois US1 terminée.

### Parallel Example: User Story 4

```text
T046 EraseAccountsTest (users)
T047 EraseSubjectsOfUserTest (catalog)
T048 EraseLearningsOfUserTest (learning)
T049 AnonymizeModerationOfUserTest (moderation)
T050 ReminderSettingLifecycleTest (reminders)
```

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Phases 1 et 2.
2. Phase 3 (US1) : la demande, la déconnexion, l'email et l'arrêt des rappels.
3. Stop : valider US1 seule, sans sujet publié.

### Incremental Delivery

1. US1, puis US2 (choix des sujets), puis US3 (annulation), puis US4 (effacement), puis US5.
2. La feature n'est livrable au public qu'avec US4 : sans effacement définitif, l'obligation RGPD
   n'est pas tenue.

---

## Phase 9: Convergence

- [X] T064 Verrouiller en écriture un sujet dont `author_id` est nul, y compris pour `subjects.moderate`, dans `back/functional/catalog/src/Support/RetiredSubjectLock.php` (nouveau code `subject_authorless`, texte dans `back/technical/osdd/lang/fr/errors.php`), en laissant la décision de retrait de `back/functional/moderation` passer ; le tester dans `back/functional/catalog/tests/Feature/KeptSubjectsOfPendingDeletionTest.php` per spec Assumptions « Contenu laissé à la communauté » (contradicts)
- [X] T065 Afficher « Retenu (suppression de compte en cours) » pour un sujet `withheld` dans la file de modération, `web/functional/Moderation/app/components/ReportGroupCard.vue`, et l'attendre dans `web/functional/Moderation/tests/ReportGroupCard.nuxt.spec.ts` per contracts/api.md « Ressources lomkit existantes » (partial)
