---

description: "Tâches de la feature 002 — Rappels de révision (CINQ)"
---

# Tasks: Rappels de révision

**Input**: Documents de conception dans `specs/002-review-reminders/`

**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md),
[data-model.md](data-model.md), [contracts/api.md](contracts/api.md), [quickstart.md](quickstart.md)

**Tests**: obligatoires. La constitution (principe IV) exige un test automatisé par exigence
fonctionnelle ; le plan demande une couverture à 100 % de `NextReminderSlot`, `ReminderSpacing` et
`ReminderEligibility`. Dans chaque story, les tests sont écrits en premier et doivent échouer avant
l'implémentation.

**Organization**: une phase par user story, dans l'ordre des priorités de la spec. Dans chaque
story, le back (fournisseur) passe avant le web (consommateur).

**Maquette de référence**: aucune. Les écrans reprennent les composants et la direction visuelle de
la 001 (brutalisme jaune et bleu).

## Format: `[ID] [P?] [Story] Description`

- **[P]** : parallélisable (fichiers différents, aucune dépendance sur une tâche non terminée)
- **[Story]** : story concernée (US1 à US4, voir spec.md)
- Tous les chemins commencent par leur repo : `back/…` ou `web/…`

## Conventions de chemins

- **Back** : nouvelle couche OSDD `back/functional/reminders/`, avec `src/` (espace de noms
  `Functional\Reminders\`), `database/{migrations,factories,seeders}`, `routes/api.php`, `lang/fr/`,
  `config/` et `tests/{Feature,Unit}`. Elle dépend de `functional/users` et `functional/learning`,
  jamais l'inverse. Commandes via `./vendor/bin/sail`.
- **Web** : nouvelle couche `web/functional/Reminders/`, avec `nuxt.config.ts`,
  `app/{pages,components,composables,models,utils}`, `i18n/locales/fr.json` et `tests/`. Le script
  du service worker reste dans `web/technical/Pwa/`. Commandes via `pnpm`.
- Les branches `002-review-reminders` de `back/` et `web/` sont créées par le hook
  `speckit.multirepo.branch` avant la première tâche.

---

## Phase 1: Setup (infrastructure partagée)

**Purpose**: créer les deux couches, installer la dépendance Web Push et déclarer les clés VAPID.

- [X] T001 Créer la couche `back/functional/reminders/` avec `./vendor/bin/sail artisan osdd:layer functional/reminders --target-path=/var/www/html/functional --generators=service-provider --generators=test --generators=routes`, puis `./vendor/bin/sail artisan osdd:phpunit` ; déclarer dans `back/functional/reminders/composer.json` la dépendance aux couches `functional/users` et `functional/learning`
- [X] T002 Installer `laravel-notification-channels/webpush` ^13.0 avec `./vendor/bin/sail composer require` dans `back/composer.json` ; vérifier avec `./vendor/bin/sail composer show laravel-notification-channels/webpush` que la version installée exige `illuminate/*` ^13.13 et `minishlink/web-push` ^11
- [X] T003 Publier la configuration du paquet dans `back/functional/reminders/config/webpush.php` : `model` à `Functional\Reminders\Models\PushSubscription`, clés lues dans `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`, `VAPID_SUBJECT` ; appliquer la surcharge dans le `register()` de `back/functional/reminders/src/Providers/RemindersServiceProvider.php` (convention du `back/CLAUDE.md`)
- [X] T004 [P] Ajouter `VAPID_PUBLIC_KEY=`, `VAPID_PRIVATE_KEY=` et `VAPID_SUBJECT=mailto:contact@cinq.test` dans `back/.env.example`
- [X] T005 [P] Déclarer la couche `Reminders` dans la liste `functional` de `web/nuxt.config.ts` ; créer `web/functional/Reminders/nuxt.config.ts` avec `runtimeConfig.public.vapidPublicKey: ''` et `i18n.locales: [{ code: 'fr', file: 'fr.json' }]`, et `web/functional/Reminders/i18n/locales/fr.json` vide
- [X] T006 [P] Ajouter `NUXT_PUBLIC_VAPID_PUBLIC_KEY=` dans `web/.env.example` (même valeur que `VAPID_PUBLIC_KEY` du back, research R2)

---

## Phase 2: Foundational (prérequis bloquants)

**Purpose**: tables, modèles, création des réglages pour chaque compte, requête des cartes dues
partagée avec `learning`, calcul du prochain créneau, et modèles raom côté web.

**⚠️ CRITICAL**: aucune story ne commence avant la fin de cette phase.

### Back

- [X] T007 Créer la migration `back/functional/reminders/database/migrations/2026_09_29_100000_create_reminder_settings_table.php` : `user_id` FK `users` « unique », `RESTRICT` ; `email_enabled` boolean « défaut `false` » ; `send_time` time « défaut `19:00` » ; `activated_at` timestamp null ; `proposal_seen_at` timestamp null ; `email_disabled_reason` varchar(20) null ; `email_bounce_count` smallint « défaut 0 » ; `unsubscribe_version` integer « défaut 0 » ; `next_reminder_at` timestamp null « indexé » ; timestamps
- [X] T008 Publier la migration du paquet dans `back/functional/reminders/database/migrations/2026_09_29_100100_create_push_subscriptions_table.php` et l'adapter : `subscribable` morph, `endpoint` varchar(1024) « unique » (`PushSubscription::ENDPOINT_MAX_LENGTH` du paquet), `public_key`, `auth_token`, `content_encoding` varchar null, plus `device_label` varchar(60) obligatoire et `last_delivered_at` timestamp null ; timestamps
- [X] T009 Créer la migration `back/functional/reminders/database/migrations/2026_09_29_100200_create_reminder_sends_table.php` : `user_id` FK `users` `RESTRICT`, `local_date` date, `cards_count` integer « supérieur ou égal à 1 », `subject_ids` jsonb, `channels` jsonb, `sent_at` timestamp ; index unique `(user_id, local_date)` (FR-007). Aucune clé en cascade
- [X] T010 [P] Créer l'enum `EmailDisabledReason` (`Unsubscribed = 'unsubscribed'`, `Bounced = 'bounced'`) dans `back/functional/reminders/src/Enums/EmailDisabledReason.php`
- [X] T011 Créer le modèle `ReminderSetting` dans `back/functional/reminders/src/Models/ReminderSetting.php` : `Notifiable` et `HasPushSubscriptions` (research R3), relation `user()`, casts (`send_time` au format `H:i`, `email_disabled_reason` en `EmailDisabledReason`, dates), `routeNotificationForMail()` qui renvoie l'adresse de l'utilisateur, `hasActiveChannel()` (email activé ou au moins un appareil)
- [X] T012 [P] Créer le modèle `PushSubscription` qui étend celui du paquet dans `back/functional/reminders/src/Models/PushSubscription.php`, avec `device_label` et `last_delivered_at` ; `public_key` et `auth_token` dans `$hidden`
- [X] T013 [P] Créer le modèle `ReminderSend` dans `back/functional/reminders/src/Models/ReminderSend.php` : casts `local_date` en date, `subject_ids` et `channels` en tableau ; `Prunable` au-delà de 90 jours (research R13)
- [X] T014 [P] Créer les factories `ReminderSettingFactory` (états `emailEnabled`, `unsubscribed`, `bounced`), `PushSubscriptionFactory` et `ReminderSendFactory` avec `faker()` de `xefi/faker-php-laravel` dans `back/functional/reminders/database/factories/`
- [X] T015 Écrire le test `back/functional/reminders/tests/Feature/ReminderSettingLifecycleTest.php` : un compte inscrit a une ligne `reminder_settings` à `email_enabled = false`, sans appareil et `next_reminder_at` nul (FR-001) ; la migration de rattrapage crée la ligne des comptes existants ; supprimer un compte supprime ses réglages, ses appareils et son journal
- [X] T016 Créer l'écouteur `CreateReminderSetting` de la création de `User` dans `back/functional/reminders/src/Listeners/CreateReminderSetting.php`, et la migration de données `back/functional/reminders/database/migrations/2026_09_29_100300_create_reminder_settings_for_existing_users.php` (idempotente) ; enregistrer l'écouteur dans `RemindersServiceProvider`
- [X] T017 Créer l'écouteur `DeleteRemindersOfUser` de la suppression de `User` (`deleting`) dans `back/functional/reminders/src/Listeners/DeleteRemindersOfUser.php`, qui supprime appareils, journal puis réglages sans cascade en base (research R13)
- [X] T018 Extraire la règle des cartes dues de `back/functional/learning/src/Rest/Instructions/DueCardsInstruction.php` dans une requête réutilisable `back/functional/learning/src/Queries/DueCardsQuery.php` (`forUser(User $user, ?array $subjectIds = null): Builder`, `next_review_on <= aujourd'hui` dans le fuseau du compte, sujet `published`) ; l'instruction l'utilise sans changer son comportement ; `back/functional/learning/tests/Feature/ReviewSessionTest.php` doit rester vert (research R5, SC-006)
- [X] T019 [P] Écrire le test unitaire `back/functional/reminders/tests/Unit/NextReminderSlotTest.php` (100 % des cas) : heure à venir aujourd'hui ; heure passée ; rappel déjà consigné aujourd'hui ; 02:30 à `Europe/Paris` le dernier dimanche de mars (avancé à 03:30) ; passage à l'heure d'hiver (première occurrence) ; fuseau changé vers l'est et vers l'ouest sans deux rappels la même date locale ; résultat en UTC
- [X] T020 Créer la classe pure `NextReminderSlot` dans `back/functional/reminders/src/Domain/NextReminderSlot.php` (entrées : `send_time`, fuseau, instant présent, dernière `local_date` consignée ; data-model.md), jusqu'au vert de T019
- [X] T021 Créer `ReminderScheduler::refresh(ReminderSetting)` dans `back/functional/reminders/src/Support/ReminderScheduler.php` : `next_reminder_at` = `NextReminderSlot` si un canal est actif, sinon nul ; et son test `back/functional/reminders/tests/Feature/ReminderSchedulerTest.php`
- [X] T022 Créer l'écouteur `RefreshReminderAfterTimezoneChange` de la mise à jour de `User` (seulement si `timezone` a changé) dans `back/functional/reminders/src/Listeners/RefreshReminderAfterTimezoneChange.php`, avec son test `back/functional/reminders/tests/Feature/TimezoneChangeTest.php` (fuseau mis à jour à la connexion → créneau recalculé dès le jour suivant)

### Web

- [X] T023 [P] Créer les modèles raom `ReminderSetting` (ressource `reminder-settings` : `id`, `email_enabled`, `send_time`, `activated_at`, `proposal_seen_at`, `email_disabled_reason`, `next_reminder_at`, `devices_count`) et `PushSubscription` (ressource `push-subscriptions` : `id`, `endpoint`, `device_label`, `last_delivered_at`, `created_at`) dans `web/functional/Reminders/app/models/`
- [X] T024 [P] Créer le support de test `web/functional/Reminders/tests/support/remindersApi.ts`, qui remplace `$laravelRaom.fetch` comme `web/functional/Catalog/tests/support/catalogApi.ts`, et simule `Notification`, `PushManager` et `navigator.serviceWorker`

**Checkpoint**: chaque compte a ses réglages, désactivés ; le prochain créneau se calcule ; la
requête des cartes dues est partagée.

---

## Phase 3: User Story 1 - Activer et régler ses rappels (Priority: P1) 🎯 MVP

**Goal**: la proposition après le premier « Apprendre ce sujet » et la section « Rappels » du
Compte : canaux, heure, appareils.

**Independent Test**: un inscrit apprend son premier sujet, voit la proposition, active l'email à
19 h, puis retrouve ce réglage dans sa page Compte, le passe à 8 h et ajoute les notifications sur
son appareil.

### Tests for User Story 1 ⚠️

> Écrits en premier, ils doivent échouer avant l'implémentation.

- [X] T025 [P] [US1] Écrire `back/functional/reminders/tests/Feature/ReminderSettingsResourceTest.php` : `search` ne renvoie que la ligne du compte connecté ; celle d'un autre répond 404 ; adresse non confirmée refusée (`verified`) ; `mutate update` de `email_enabled` et `send_time` ; `send_time` hors « `06:00` à `23:30`, minutes à `00` ou `30` » → 422 `invalid_send_time` (FR-003) ; création et suppression → 403 ; `proposal_seen_at` rempli s'il était nul ; `activated_at` rempli au premier canal actif ; réactiver l'email remet `email_disabled_reason` à nul et `email_bounce_count` à 0 et incrémente `unsubscribe_version` ; `next_reminder_at` recalculé (19 h → 8 h, et activation à 20 h avec 19 h → lendemain 19 h) ; `devices_count` renvoyé
- [X] T026 [P] [US1] Écrire `back/functional/reminders/tests/Feature/PushSubscriptionsResourceTest.php` : `register-device` avec `endpoint` « URL `https`, 1024 caractères au plus », `public_key`, `auth_token`, `content_encoding` « `aes128gcm` ou `aesgcm` », `device_label` « 60 caractères au plus » → 200 ; donnée invalide → 422 `invalid_push_subscription` ; un endpoint connu d'un autre compte est rattaché au compte connecté (research R11) ; `public_key` et `auth_token` jamais renvoyés ; liste des appareils du compte, les plus récents d'abord ; appareil d'un autre compte → 404 ; `DELETE` coupe un seul appareil (FR-006) ; le dernier appareil supprimé met `next_reminder_at` à nul ; `mutate` refusé
- [X] T027 [P] [US1] Écrire `back/functional/reminders/tests/Feature/DismissProposalTest.php` : l'action `dismiss-proposal` remplit `proposal_seen_at`, n'active aucun canal et laisse `next_reminder_at` nul (FR-002)
- [X] T028 [P] [US1] Écrire `web/functional/Reminders/tests/ReminderProposal.nuxt.spec.ts` : après le premier apprentissage, le dialogue s'ouvre avec les deux cases décochées et 19:00 ; « Plus tard » appelle `dismiss-proposal` et le dialogue ne revient plus ; « Activer » avec l'email coché enregistre et confirme ; aucun dialogue si `proposal_seen_at` est rempli ou si un canal est actif (FR-002)
- [X] T029 [P] [US1] Écrire `web/functional/Reminders/tests/AccountReminders.nuxt.spec.ts` : select de 06:00 à 23:30 par 30 minutes ; activer l'email ; activer les notifications : autorisation accordée → `register-device` appelé et appareil listé ; refusée → message d'aide et rien d'enregistré (FR-004) ; iPhone hors PWA installée → message et lien d'installation (FR-005) ; appareil courant reconnu par son `endpoint` ; deux appareils listés, chacun coupé séparément (FR-006)
- [X] T030 [P] [US1] Écrire `web/functional/Reminders/tests/deviceLabel.spec.ts` : « Chrome sur Android », « Safari sur iPhone », « Firefox sur Windows », repli sur l'agent utilisateur, 60 caractères au plus

### Implementation for User Story 1

- [X] T031 [US1] Créer `ReminderSettingControl` et `PushSubscriptionControl` (périmètre du compte connecté, data-model.md « Accès ») dans `back/functional/reminders/src/Access/Controls/`, et les policies `ReminderSettingPolicy` (lecture et modification par le propriétaire, ni création ni suppression) et `PushSubscriptionPolicy` (lecture et suppression par le propriétaire) dans `back/functional/reminders/src/Policies/`
- [X] T032 [US1] Créer `ReminderSettingResource` dans `back/functional/reminders/src/Rest/Resources/ReminderSettingResource.php` : champs de contracts/api.md §1, `devices_count`, règles de `send_time`, et les effets de la modification (proposition vue, `activated_at`, réactivation de l'email, `ReminderScheduler::refresh`) ; son contrôleur dans `back/functional/reminders/src/Rest/Controllers/`
- [X] T033 [US1] Créer l'action standalone `DismissProposal` dans `back/functional/reminders/src/Rest/Actions/DismissProposal.php`
- [X] T034 [US1] Créer `PushSubscriptionResource` (lecture, suppression qui appelle `ReminderScheduler::refresh`) dans `back/functional/reminders/src/Rest/Resources/PushSubscriptionResource.php` et l'action standalone `RegisterDevice` dans `back/functional/reminders/src/Rest/Actions/RegisterDevice.php` (`updatePushSubscription`, `activated_at`, `proposal_seen_at`, `refresh`)
- [X] T035 [US1] Déclarer `Rest::resource('reminder-settings', …)` et `Rest::resource('push-subscriptions', …)` avec les middlewares `auth:sanctum` et `verified` dans `back/functional/reminders/routes/api.php`
- [X] T036 [P] [US1] Ajouter les messages `invalid_send_time` et `invalid_push_subscription` de contracts/api.md §6 dans `back/technical/osdd/lang/fr/errors.php`, que lit `BusinessRuleException`
- [X] T037 [P] [US1] Créer l'utilitaire `deviceLabel()` dans `web/functional/Reminders/app/utils/deviceLabel.ts` (`navigator.userAgentData` quand il existe, sinon l'agent utilisateur ; research R11)
- [X] T038 [US1] Créer `useReminderSettings` dans `web/functional/Reminders/app/composables/useReminderSettings.ts` : lecture de la ligne du compte, modification de `email_enabled` et `send_time`, liste des appareils, créneaux de 06:00 à 23:30
- [X] T039 [US1] Créer `usePushDevice` dans `web/functional/Reminders/app/composables/usePushDevice.ts` : support (`serviceWorker`, `PushManager`, `Notification`) ; iOS ou iPadOS hors PWA installée via `usePwaInstall` de `web/technical/Pwa/` ; `Notification.requestPermission()` ; `pushManager.subscribe({ userVisibleOnly: true, applicationServerKey })` avec `runtimeConfig.public.vapidPublicKey` ; action `register-device` ; appareil courant reconnu par l'`endpoint` ; coupure (`subscription.unsubscribe()` pour l'appareil courant, puis suppression de la ligne)
- [X] T040 [US1] Créer `useReminderProposal` dans `web/functional/Reminders/app/composables/useReminderProposal.ts` : `offer()` n'ouvre le dialogue que si `proposal_seen_at` est nul et qu'aucun canal n'est actif ; « Activer » enregistre les canaux cochés et l'heure ; « Plus tard » appelle `dismiss-proposal`
- [X] T041 [P] [US1] Créer `ReminderDeviceRow.vue` (nom de l'appareil, « Cet appareil », date d'enregistrement, bouton « Couper ») dans `web/functional/Reminders/app/components/`
- [X] T042 [US1] Créer `AccountReminders.vue` dans `web/functional/Reminders/app/components/` : interrupteur email, select de l'heure, liste des `ReminderDeviceRow`, bouton « Activer sur cet appareil » ou message d'installation iOS ou d'autorisation refusée (contracts/api.md §3)
- [X] T043 [US1] Créer `ReminderProposalDialog.vue` dans `web/functional/Reminders/app/components/` : cases « Notifications sur cet appareil » et « Email » décochées, heure à 19:00, boutons « Activer » et « Plus tard »
- [X] T044 [US1] Brancher la proposition : `web/functional/Learning/app/composables/useLearningPanel.ts` appelle `useReminderProposal().offer()` après un `learn()` réussi, et `web/functional/Learning/app/components/LearnSubjectPanel.vue` affiche `ReminderProposalDialog`
- [X] T045 [US1] Ajouter la section « Rappels » (`AccountReminders`) dans `web/functional/Account/app/pages/compte.vue`
- [X] T046 [P] [US1] Ajouter les textes de la proposition, de la section et des messages d'aide (vouvoiement, clés en anglais minuscules) dans `web/functional/Reminders/i18n/locales/fr.json`

**Checkpoint**: la proposition et la section « Rappels » fonctionnent ; rien ne part encore.

---

## Phase 4: User Story 2 - Recevoir le rappel du jour (Priority: P1)

**Goal**: à l'heure choisie, un rappel sur chaque canal actif, s'il y a des cartes dues dans des
sujets publiés, avec un lien qui ouvre la séance.

**Independent Test**: un apprenant avec 12 cartes dues et l'email activé à 19 h reçoit à 19 h un
email qui annonce 12 cartes ; le lien ouvre une séance de 12 cartes ; le lendemain, sans carte due,
il ne reçoit rien.

### Tests for User Story 2 ⚠️

- [X] T047 [P] [US2] Écrire `back/functional/reminders/tests/Feature/ReminderEligibilityTest.php` (100 % des cas de FR-008 et FR-019) : adresse non confirmée ; aucun canal actif ; aucun apprentissage ; aucune carte due ; cartes dues seulement dans des sujets dépubliés ou retirés → `null` ; sinon `{cards_count, subject_ids}` égal au résultat de `DueCardsQuery` (SC-006)
- [X] T048 [P] [US2] Écrire `back/functional/reminders/tests/Feature/DispatchDueRemindersCommandTest.php` : seules les lignes à `next_reminder_at <= now()` avec un canal actif sont mises en file ; `next_reminder_at` avancé au créneau suivant ; option `--now="2026-09-26 19:00"` ; aucune tâche pour un compte sans canal
- [X] T049 [P] [US2] Écrire `back/functional/reminders/tests/Feature/SendReviewReminderTest.php` : 12 cartes sur 2 sujets → email et push avec 12 et les 2 sujets ; une ligne `reminder_sends` écrite avant l'envoi avec les canaux servis ; deux tâches pour le même compte et le même jour → un seul rappel ; heure changée pour plus tard le même jour → pas de second rappel (SC-003) ; cartes toutes révisées avant l'heure → rien ; compte qui n'apprend plus rien → rien ; nombre compté au moment de l'envoi
- [X] T050 [P] [US2] Écrire `back/functional/reminders/tests/Feature/ReviewReminderNotificationTest.php` : objet « 1 carte à réviser aujourd'hui » et « 12 cartes à réviser aujourd'hui » ; « Bonjour {nom affiché}, » ; bouton « Réviser maintenant » vers `{FRONTEND_URL}/revisions/seance?sujets=1,2` ; push : titre « CINQ », texte, icône, badge, tag `review-reminder`, `data.url`, TTL 14 400 et urgence `normal` ; aucune donnée personnelle hors nom affiché et nombre de cartes (FR-011)
- [X] T051 [P] [US2] Écrire `web/technical/Pwa/tests/swPush.spec.ts` : l'événement `push` affiche titre, texte, icône, badge et tag reçus ; `notificationclick` met au premier plan un onglet de CINQ et le dirige vers `data.url`, sinon ouvre un nouvel onglet
- [X] T052 [P] [US2] Écrire `web/functional/Learning/tests/ReminderLink.nuxt.spec.ts` : sans session, `/revisions/seance?sujets=1,2` mène à `/connexion?redirect=…`, puis à la séance après connexion (FR-010)

### Implementation for User Story 2

- [X] T053 [US2] Créer `ReminderEligibility` dans `back/functional/reminders/src/Domain/ReminderEligibility.php` : conditions 1 à 4 de research R5, sur `DueCardsQuery` ; renvoie `{cards_count, subject_ids}` ou `null`
- [X] T054 [US2] Créer `ReviewReminderNotification` (canaux `mail` et `WebPushChannel`) dans `back/functional/reminders/src/Notifications/ReviewReminderNotification.php` : `toMail()` (contracts/api.md §4) et `toWebPush()` (contracts/api.md §5)
- [X] T055 [P] [US2] Ajouter les textes de l'email et de la notification, avec leur pluriel, dans `back/functional/reminders/lang/fr/notifications.php`
- [X] T056 [US2] Créer la tâche `SendReviewReminder` (`ShouldBeUnique` par compte, 3 essais) dans `back/functional/reminders/src/Jobs/SendReviewReminder.php` : vérifie l'éligibilité, écrit `reminder_sends` dans une transaction (un doublon échoue sur l'index unique et n'envoie rien), envoie chaque canal dans son propre appel, consigne les canaux servis
- [X] T057 [US2] Créer la commande `reminders:dispatch {--now=}` dans `back/functional/reminders/src/Console/DispatchDueRemindersCommand.php` : lit l'index `next_reminder_at`, met `SendReviewReminder` en file, avance le créneau par `ReminderScheduler`
- [X] T058 [US2] Planifier `reminders:dispatch` chaque minute avec `withoutOverlapping()` et `onOneServer()` dans `back/routes/console.php`
- [X] T059 [US2] Créer `web/technical/Pwa/public/sw-push.js` (écouteurs `push` et `notificationclick`, research R10) et ajouter `workbox.importScripts: ['sw-push.js']` dans `web/technical/Pwa/nuxt.config.ts`

**Checkpoint**: le rappel du jour part par email et par push ; US1 et US2 forment le MVP.

---

## Phase 5: User Story 3 - Des rappels qui s'espacent (Priority: P2)

**Goal**: chaque jour jusqu'au 7e jour sans révision, un jour sur deux jusqu'au 21e, puis chaque
semaine ; une réponse remet le rythme quotidien.

**Independent Test**: un apprenant qui ne révise plus reçoit un rappel les 7 premiers jours, puis un
jour sur deux jusqu'au 21e, puis un par semaine ; il répond à une carte et reçoit de nouveau un
rappel le lendemain.

### Tests for User Story 3 ⚠️

- [X] T060 [P] [US3] Écrire le test unitaire `back/functional/reminders/tests/Unit/ReminderSpacingTest.php` : chaque jour d'inactivité du 1er au 60e (fournisseur de données), avec l'écart minimal « 0 à 7 → 1 jour, 8 à 21 → 2 jours, 22 et plus → 7 jours » ; aucun rappel encore envoyé → autorisé ; reprise quotidienne après une réponse (SC-005)
- [X] T061 [P] [US3] Écrire `back/functional/reminders/tests/Feature/ReminderSpacingEligibilityTest.php` : dernière réponse il y a 3 jours → rappel ; 10 jours et rappel la veille → rien ; 30 jours et dernier rappel il y a moins de 7 jours → rien ; jamais de réponse → compté depuis `activated_at` ; une réponse → rappel le lendemain (FR-012, FR-013)

### Implementation for User Story 3

- [X] T062 [US3] Créer la classe pure `ReminderSpacing` dans `back/functional/reminders/src/Domain/ReminderSpacing.php`, avec les paliers en constantes (research R6)
- [X] T063 [US3] Ajouter la condition 5 dans `back/functional/reminders/src/Domain/ReminderEligibility.php` : date locale de la dernière `review_answers.answered_at` du compte, ou de `activated_at`, et dernière `local_date` de `reminder_sends`

**Checkpoint**: les rappels s'espacent et reprennent après une réponse.

---

## Phase 6: User Story 4 - Couper les rappels (Priority: P2)

**Goal**: désinscription en un clic depuis l'email, sans connexion ; retrait automatique des
appareils révoqués et de l'email refusé.

**Independent Test**: depuis un email de rappel, « Ne plus recevoir ces rappels » affiche une
confirmation sans connexion ; le lendemain, plus aucun email, alors que les notifications
continuent.

### Tests for User Story 4 ⚠️

- [X] T064 [P] [US4] Écrire `back/functional/reminders/tests/Feature/UnsubscribeTest.php` : lien valide → 204 et `email_disabled_reason = unsubscribed`, idempotent ; signature altérée, compte inconnu ou `v` périmé → la même réponse 403 `invalid_link`, sans rien modifier (FR-016) ; appel `POST` sans cookie avec `List-Unsubscribe=One-Click` accepté (FR-015) ; limite de 6 appels par minute ; les notifications restent actives
- [X] T065 [P] [US4] Écrire `back/functional/reminders/tests/Unit/UnsubscribeLinkTest.php` : URL web `{FRONTEND_URL}/rappels/desinscription?id=…&v=…&signature=…` et URL d'API pour `List-Unsubscribe`, sans expiration, `v` égal à `unsubscribe_version`
- [X] T066 [P] [US4] Compléter `back/functional/reminders/tests/Feature/ReviewReminderNotificationTest.php` : en-têtes `List-Unsubscribe` et `List-Unsubscribe-Post: List-Unsubscribe=One-Click`, pied « Vous recevez cet email parce que vous avez activé les rappels de révision. » et lien « Ne plus recevoir ces rappels » (FR-014)
- [X] T067 [P] [US4] Écrire `back/functional/reminders/tests/Feature/DeliveryFailuresTest.php` avec un client HTTP simulé : push 201 → `last_delivered_at` mis à jour ; push 410 ou 404 → appareil supprimé et email tout de même envoyé (FR-017) ; 3 refus SMTP 5xx consécutifs → email désactivé `bounced` (FR-018) ; un envoi accepté remet `email_bounce_count` à 0
- [X] T068 [P] [US4] Écrire `web/functional/Reminders/tests/UnsubscribePage.nuxt.spec.ts` : lien valide → « Vous ne recevrez plus les rappels par email » et lien vers `/compte` ; 403 → « Ce lien n'est pas valide » ; page accessible sans session
- [X] T069 [P] [US4] Compléter `web/functional/Reminders/tests/AccountReminders.nuxt.spec.ts` : email coupé par le lien affiché désactivé et réactivable (US4 scénario 5) ; mention d'un email désactivé suite à des refus (FR-018)

### Implementation for User Story 4

- [X] T070 [US4] Créer `UnsubscribeLink` dans `back/functional/reminders/src/Support/UnsubscribeLink.php` (`URL::signedRoute` relatif réécrit vers le web, comme `back/functional/users/src/Support/EmailVerificationLink.php` ; research R7)
- [X] T071 [US4] Créer `UnsubscribeController` dans `back/functional/reminders/src/Http/Controllers/UnsubscribeController.php` : signature vérifiée avant toute lecture en base, `v` comparé à `unsubscribe_version`, 204 ou 403 `invalid_link` ; déclarer `POST reminders/unsubscribe/{id}` avec `throttle:6,1` dans `back/functional/reminders/routes/api.php` ; ajouter `invalid_link` dans `back/technical/osdd/lang/fr/errors.php`
- [X] T072 [US4] Ajouter les en-têtes `List-Unsubscribe` et `List-Unsubscribe-Post`, le pied et le lien de désinscription dans `back/functional/reminders/src/Notifications/ReviewReminderNotification.php` et `back/functional/reminders/lang/fr/notifications.php`
- [X] T073 [US4] Créer l'écouteur `RecordPushDelivery` de `NotificationSent` (met à jour `last_delivered_at`) dans `back/functional/reminders/src/Listeners/RecordPushDelivery.php` ; vérifier que le canal du paquet supprime l'abonnement sur 404 ou 410
- [X] T074 [US4] Créer l'interface `RecordsEmailBounce` et son implémentation SMTP `SmtpBounceRecorder` (5xx → compteur, 3e refus → `bounced`, envoi accepté → compteur à 0) dans `back/functional/reminders/src/Support/`, appelées par `back/functional/reminders/src/Jobs/SendReviewReminder.php` (research R8)
- [X] T075 [US4] Créer `useUnsubscribe` dans `web/functional/Reminders/app/composables/useUnsubscribe.ts` (appel par `useApiFetch`, comme les endpoints hors lomkit de la 001) et la page publique `web/functional/Reminders/app/pages/rappels/desinscription.vue`
- [X] T076 [US4] Afficher dans `web/functional/Reminders/app/components/AccountReminders.vue` l'état « coupé par le lien » ou « désactivé après des refus » de l'email, avec la réactivation ; textes dans `web/functional/Reminders/i18n/locales/fr.json`

**Checkpoint**: les quatre stories fonctionnent indépendamment.

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: données de démonstration, documentation des repos, qualité et validation de bout en
bout.

- [X] T077 [P] Créer `RemindersSeeder` (quelques comptes avec l'email activé, un journal sur plusieurs jours) dans `back/functional/reminders/database/seeders/RemindersSeeder.php`, lancé par `./vendor/bin/sail artisan osdd:seed`. Pas d'appareil factice : un faux abonnement ferait partir chaque envoi de développement vers un service push inexistant, et seul un navigateur peut créer un vrai abonnement
- [X] T078 [P] Mettre à jour `back/CLAUDE.md` : couche `reminders` dans la liste, `schedule:work` et `webpush:vapid` dans « Run »
- [X] T079 [P] Mettre à jour `web/CLAUDE.md` : la notification push ne se teste qu'en build de production (`pnpm build`), et `NUXT_PUBLIC_VAPID_PUBLIC_KEY` doit reprendre la clé publique du back
- [X] T080 Lancer `./vendor/bin/sail artisan test` et `./vendor/bin/sail bin pint --dirty --format agent` dans `back/`, puis `pnpm test`, `pnpm lint` et `pnpm exec prettier --check .` dans `web/`
- [X] T081 Vérifier de 360 à 1440 px, sans défilement horizontal ni libellé tronqué et sans violation axe (script `a11y.cjs` de la 001), `web/functional/Reminders/app/components/ReminderProposalDialog.vue`, `web/functional/Reminders/app/components/AccountReminders.vue` et `web/functional/Reminders/app/pages/rappels/desinscription.vue`
- [ ] T082 Dérouler les scénarios manuels 1 à 10 de `specs/002-review-reminders/quickstart.md` et consigner les résultats à la fin de `specs/002-review-reminders/tasks.md`

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)** : aucune dépendance.
- **Foundational (Phase 2)** : après le Setup ; bloque toutes les stories.
- **US1 (Phase 3)** et **US2 (Phase 4)** : après la Phase 2. US2 n'a besoin d'US1 que pour
  activer un canal en conditions réelles ; ses tests créent les réglages par les factories.
- **US3 (Phase 5)** : après US2, car elle ajoute une condition à `ReminderEligibility`.
- **US4 (Phase 6)** : après US2, car elle complète l'email et la tâche d'envoi ; ses parties web
  dépendent d'US1 (`AccountReminders.vue`).
- **Polish (Phase 7)** : après les stories livrées.

### Ordre entre repos

- Dans chaque story, les tâches `back/` passent avant les tâches `web/` qui consomment leurs
  endpoints.
- La merge request `back` est fusionnée en premier ; la merge request `web` reste ouverte tant que
  le back n'est pas fusionné et déployé (plan, « Affected Repos »).

### Within Each User Story

- Tests écrits et en échec avant l'implémentation.
- Modèles, puis domaine, puis ressources et actions, puis routes ; côté web, composables avant
  composants, composants avant leur intégration dans une page.

### Parallel Opportunities

- Setup : T004, T005 et T006 en parallèle après T001 à T003.
- Foundational : T010, T012, T013, T014 en parallèle après T007 à T009 ; T019 et T023, T024 à tout
  moment de la phase.
- US1 : T025 à T030 ensemble ; puis T036, T037, T041 et T046 en parallèle du reste.
- US2 : T047 à T052 ensemble ; T055 en parallèle de T054.
- US3 : T060 et T061 ensemble.
- US4 : T064 à T069 ensemble.
- Polish : T077, T078 et T079 ensemble.

---

## Parallel Example: User Story 1

```bash
# Tests d'abord, tous ensemble :
Task: "ReminderSettingsResourceTest dans back/functional/reminders/tests/Feature/ReminderSettingsResourceTest.php"
Task: "PushSubscriptionsResourceTest dans back/functional/reminders/tests/Feature/PushSubscriptionsResourceTest.php"
Task: "DismissProposalTest dans back/functional/reminders/tests/Feature/DismissProposalTest.php"
Task: "ReminderProposal.nuxt.spec.ts dans web/functional/Reminders/tests/"
Task: "AccountReminders.nuxt.spec.ts dans web/functional/Reminders/tests/"
Task: "deviceLabel.spec.ts dans web/functional/Reminders/tests/"

# Puis, en parallèle de l'implémentation back :
Task: "Messages d'erreur dans back/technical/osdd/lang/fr/errors.php"
Task: "deviceLabel() dans web/functional/Reminders/app/utils/deviceLabel.ts"
Task: "ReminderDeviceRow.vue dans web/functional/Reminders/app/components/"
```

---

## Implementation Strategy

### MVP First (US1 + US2)

1. Phase 1 : Setup.
2. Phase 2 : Foundational (bloque tout).
3. Phase 3 : US1, activer et régler.
4. Phase 4 : US2, recevoir le rappel du jour.
5. **STOP and VALIDATE** : scénarios manuels 1 à 6 de quickstart.md.

Les deux stories P1 forment ensemble le plus petit produit utile : l'une sans l'autre n'envoie
rien ou n'est jamais activée.

### Incremental Delivery

1. Setup + Foundational → fondations prêtes.
2. US1 + US2 → MVP, démontrable.
3. US3 → espacement, scénario 7.
4. US4 → désinscription et échecs d'envoi, scénarios 8 et 9. **À livrer avant toute mise en
   production** : un email de rappel sans lien de désinscription n'est pas conforme (FR-014).
5. Polish → scénario 10 et validation complète.

---

## Notes

- [P] = fichiers différents, aucune dépendance sur une tâche non terminée.
- Chaque exigence fonctionnelle est couverte par au moins un test (principe IV) : FR-001 (T015),
  FR-002 (T025, T027, T028), FR-003 (T025, T029), FR-004 et FR-005 (T029), FR-006 (T026, T029),
  FR-007 (T049), FR-008 (T047, T049), FR-009 (T049, T050), FR-010 (T052), FR-011 (T050),
  FR-012 et FR-013 (T060, T061), FR-014 (T064, T066), FR-015 et FR-016 (T064), FR-017 (T067),
  FR-018 (T067, T069), FR-019 (T047, T049), FR-020 (T049).
- Commit après chaque tâche ou groupe logique, dans le repo concerné (`git -C back`, `git -C web`),
  message à l'impératif en anglais, sans mention d'IA.

---

## Résultats de validation (T080 à T082)

Relevés le 2026-09-29, sur les branches `002-review-reminders`.

### Tests automatisés (T080)

- `back/` : 259 tests, 761 assertions, tous verts ; `pint --test` propre sur tout le projet.
  `NextReminderSlot`, `ReminderSpacing` et `ReminderEligibility` sont couverts à 100 % (pcov).
- `web/` : 114 tests verts ; ESLint et Prettier propres ; `pnpm build` réussit et le `sw.js`
  généré importe `sw-push.js`.

### Scénarios du quickstart (T082)

Déroulés sur la pile de développement (Sail, SMTP vers Mailpit, API sur le port 8090) :

| Scénario | Résultat |
|---|---|
| 1. Proposition | ✅ dans le navigateur : ouverte après le premier « Apprendre ce sujet », email seul sous `pnpm dev` (pas de service worker), 19:00 ; « Activer » enregistre et confirme |
| 2. Section « Rappels » | ✅ email coché, heure passée à 08:00 → prochain rappel en base le lendemain 06:00 UTC ; entrée « Rappels de révision » ajoutée au menu du compte, qui n'avait aucun lien vers `/compte` sur ordinateur |
| 4. Rappel du jour | ✅ `reminders:dispatch --now` : un email « 23 cartes à réviser aujourd’hui » dans Mailpit, lien `/revisions/seance?sujets=1` ; la même minute relancée n'envoie rien |
| 8. Désinscription | ✅ en-têtes `List-Unsubscribe` et `List-Unsubscribe-Post` présents ; `POST` sans session → 204, deux fois ; signature altérée → 403 `invalid_link` ; le compte passe à `email_enabled = false`, `unsubscribed`, sans prochain rappel. Page web sans session : le lien de l'email confirme, un lien altéré affiche « Ce lien n’est pas valide. » |
| 3, 5, 7 | Couverts par les tests automatisés (iOS hors PWA, rien à réviser, espacement) |
| 6, 9 | ⏳ push dans un vrai navigateur, sur le build de production : à dérouler par le développeur |
| 10. Mise en page | ✅ voir T081 |

### Mise en page (T081)

Mesurée dans le navigateur intégré à 360, 768 et 1440 px : défilement horizontal de la page,
libellés tronqués et cibles de moins de 44 px. Aucun écart sur la proposition (360 px), la
section « Rappels » (360, 768, 1440 px) et `/rappels/desinscription` (360 et 1440 px, états
confirmé et lien invalide). Le script `a11y.cjs` (axe) de la 001 n'étant dans aucun repo, axe n'a
pas été passé.

### Écarts relevés pendant la validation

- Le dialogue proposait les notifications là où elles ne pouvaient pas marcher (sans service
  worker, iPhone hors PWA), et un échec de l'appareil bloquait aussi l'email : corrigé.
- La tâche d'envoi construisait le canal push même sans appareil : une configuration VAPID
  invalide empêchait l'email. Corrigé, avec un test.
- Le script `a11y.cjs` cité par le quickstart n'existe dans aucun repo : la vérification de
  mise en page se fait à la main dans le navigateur.
