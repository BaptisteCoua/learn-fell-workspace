---

description: "Tâches de la feature 005 — Inscription neutre (CINQ)"
---

# Tasks: Inscription neutre

**Input**: Documents de conception dans `specs/005-neutral-registration/`

**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md),
[data-model.md](data-model.md), [contracts/api.md](contracts/api.md), [quickstart.md](quickstart.md)

**Tests**: obligatoires (constitution, principe IV). Dans chaque story, les tests sont écrits en
premier et doivent échouer avant l'implémentation. La table des cas est dans
[quickstart.md](quickstart.md).

**Organization**: une phase par user story ; le back avant le web.

## Format: `[ID] [P?] [Story] Description`

- **[P]** : parallélisable (fichiers différents, aucune dépendance sur une tâche non terminée)
- **[Story]** : story concernée (US1 à US3, voir spec.md)
- Tous les chemins commencent par leur repo : `back/…` ou `web/…`

## Conventions de chemins

- **Back** : couche `back/functional/users/` (`src/`, espace de noms `Functional\Users\`, `routes/`,
  `lang/fr/`, `config/`, `tests/Feature`). Commandes via `./vendor/bin/sail`.
- **Web** : couche `web/functional/Account/` (`app/{pages,composables}`, `i18n/locales/fr.json`,
  `tests/`). Commandes via `pnpm`.
- Les branches `005-neutral-registration` de `back/` et `web/` sont créées par le hook
  `speckit.multirepo.branch` avant la première tâche.

---

## Phase 1: Setup

Aucune : ni couche, ni dépendance, ni migration (plan, « Primary Dependencies »).

---

## Phase 2: Foundational (prérequis bloquants)

**Purpose**: reprendre la route d'inscription à Fortify, sans encore changer son comportement
visible pour une adresse libre.

- [ ] T001 Retirer `Features::registration()` de `back/functional/users/config/fortify.php` ; vérifier avec `./vendor/bin/sail artisan route:list --path=api/register` que Fortify ne sert plus la route (research R1)
- [ ] T002 Créer `back/functional/users/src/Http/Controllers/RegisterController.php` (invocable) et la route `POST register`, nommée `users.register`, dans `back/functional/users/routes/account.php` (middleware `web`, préfixe `/api`, comme les autres routes de compte) ; il délègue à `RegisterAccount` (T003) et répond `201 { message: __('users::account.registered') }` sans ouvrir de session
- [ ] T003 Créer `back/functional/users/src/Actions/RegisterAccount.php` avec les règles de validation de l'actuel `CreateNewUser` : « nom de 2 à 60 caractères », « adresse valide de 255 caractères au plus », « mot de passe de 8 caractères au moins, confirmé », « fuseau valide, `Europe/Paris` par défaut », sans règle `unique` (research R5) ; pour une adresse libre, créer le compte puis `event(new Registered($user))`
- [ ] T004 Supprimer `back/functional/users/src/Actions/CreateNewUser.php`, `back/functional/users/src/Http/Responses/RegisteredResponse.php` et leurs liaisons (`Fortify::createUsersUsing`, `RegisterResponse`) dans `back/functional/users/src/Providers/FortifyServiceProvider.php` (research R7)

**Checkpoint**: `./vendor/bin/sail artisan test functional/users` : l'inscription d'une adresse libre et la confirmation passent comme avant ; seuls les deux tests d'adresse prise de `RegistrationTest` échouent, ils sont réécrits en US1 et US2.

---

## Phase 3: User Story 1 - S'inscrire sans apprendre si l'adresse est prise (Priority: P1) 🎯 MVP

**Goal**: même réponse et même écran, quelle que soit l'adresse.

**Independent Test**: trois inscriptions identiques sauf l'adresse (libre, confirmée, en attente)
donnent la même réponse et le même écran.

### Tests (écrits d'abord, en échec)

- [ ] T005 [P] [US1] Dans `back/functional/users/tests/Feature/RegistrationTest.php`, remplacer `test_an_address_of_an_active_account_invites_to_log_in` et `test_an_address_waiting_for_confirmation_offers_a_new_link` par : quatre inscriptions avec la même saisie valide et des adresses libre, confirmée, en cours de suppression (`pendingDeletion()`) et en attente donnent le même statut `201` et le même JSON (FR-001, SC-001) ; adresse libre : compte inactif créé, notification de confirmation envoyée, `assertGuest('web')` (FR-002) ; adresse avec d'autres majuscules traitée comme la même (FR-006)
- [ ] T006 [P] [US1] Test `back/functional/users/tests/Feature/RegistrationOnTakenAddressTest.php` : pour un compte confirmé et pour un compte en cours de suppression, nom, mot de passe, fuseau, `deletion_requested_at` et `keeps_published_subjects` inchangés, aucune notification envoyée (`Notification::assertNothingSent()`), aucun compte en plus (FR-003, SC-003)
- [ ] T007 [P] [US1] Tests web : `web/functional/Account/tests/RegisterPage.nuxt.spec.ts` (après un `201`, navigation vers `/inscription/confirmation` ; plus aucun message « Cette adresse a déjà un compte » ni « Un compte en attente… », FR-009) ; `web/functional/Account/tests/ConfirmationPage.nuxt.spec.ts` (texte « Si cette adresse peut être utilisée… », liens « Se connecter » vers `/connexion` et « Mot de passe oublié » vers `/mot-de-passe-oublie`, FR-008, FR-010)

### Implémentation

- [ ] T008 [US1] Dans `back/functional/users/src/Actions/RegisterAccount.php`, ne rien faire pour une adresse dont le compte a `email_verified_at` renseigné, en cours de suppression ou non (FR-003) ; la recherche se fait sur l'adresse en minuscules (FR-006)
- [ ] T009 [US1] Remplacer le message `registered` de `back/functional/users/lang/fr/account.php` par « Si cette adresse peut être utilisée, un lien de confirmation vient d'y être envoyé. » (contrat) ; retirer `email_taken` et `email_pending_verification` de `back/technical/osdd/lang/fr/errors.php`
- [ ] T010 [US1] Retirer l'état `emailConflict` de `web/functional/Account/app/composables/useRegisterForm.ts` et le bloc `account-form__conflict` de `web/functional/Account/app/pages/inscription/index.vue` ; retirer les clés « this address already has an account. » et « an account waiting for confirmation already exists for this address. » de `web/functional/Account/i18n/locales/fr.json` (FR-009)
- [ ] T011 [US1] Dans `web/functional/Account/app/pages/inscription/confirmation.vue`, remplacer « Nous avons envoyé un lien de confirmation à » par « Si cette adresse peut être utilisée, vous allez recevoir un lien de confirmation à » et ajouter, sous les points, « Rien reçu ? Vous avez peut-être déjà un compte » avec les liens « Se connecter » et « Mot de passe oublié » ; clés dans `web/functional/Account/i18n/locales/fr.json`, vouvoiement (FR-008, FR-010)

**Checkpoint**: US1 se démontre seule : plus aucune réponse ne distingue une adresse prise.

---

## Phase 4: User Story 2 - Retrouver un compte resté en attente (Priority: P1)

**Goal**: une nouvelle inscription sur un compte en attente renvoie son lien, sans rien changer
d'autre, au plus une fois par minute.

**Independent Test**: un compte en attente « Camille », une inscription « Dominique » à la même
adresse ; un nouveau lien arrive, l'ancien est refusé, le compte confirmé s'appelle « Camille ».

- [ ] T012 [P] [US2] Test `back/functional/users/tests/Feature/RegistrationOnPendingAccountTest.php` : nouvelle inscription sur un compte en attente → une `VerifyEmailNotification` envoyée au compte existant ; l'ancien lien refusé et le nouveau accepté ; nom, mot de passe, fuseau et `created_at` inchangés (FR-004) ; deux inscriptions dans la même minute → une seule notification, mêmes réponses, et une troisième 61 secondes plus tard en envoie une (FR-005) ; un compte en attente créé il y a 8 jours reste élagué par `model:prune` malgré un renvoi la veille (US2 scénario 4)
- [ ] T013 [US2] Dans `back/functional/users/src/Actions/RegisterAccount.php`, pour une adresse dont le compte a `email_verified_at` nul : `RateLimiter::attempt('registration-link:'.$user->getKey(), 1, fn () => $user->sendEmailVerificationNotification(), 60)`, sans modifier le compte (research R3, R4)

**Checkpoint**: US1 et US2 ensemble : les trois cas d'adresse sont traités et indiscernables.

---

## Phase 5: User Story 3 - Corriger une saisie (Priority: P2)

**Goal**: les erreurs de saisie restent affichées, identiques pour une adresse libre ou prise.

- [ ] T014 [P] [US3] Dans `back/functional/users/tests/Feature/RegistrationTest.php`, garder `test_two_different_passwords_are_refused_under_the_confirmation` et `test_a_password_needs_eight_characters`, et ajouter : un mot de passe trop court et un nom trop court donnent les mêmes erreurs de champ pour une adresse libre et pour une adresse confirmée, et ne créent ni n'envoient rien (FR-007)
- [ ] T015 [P] [US3] Dans `web/functional/Account/tests/RegisterPage.nuxt.spec.ts`, vérifier que les erreurs de champ d'un `422` s'affichent toujours sous les champs, et que deux mots de passe différents sont refusés sans appel à l'API (comportement de la 001 conservé)

---

## Phase 6: Polish & vérifications transverses

- [ ] T016 [P] Mettre à jour la note d'écart dans `specs/004-account-deletion/spec.md` (hypothèse « Adresse email pendant le délai ») et le Complexity Tracking de `specs/004-account-deletion/plan.md` : l'écart au principe VI est résolu par la feature 005
- [ ] T017 Lancer `./vendor/bin/sail bin pint --dirty --format agent` et `./vendor/bin/sail artisan test --compact` dans `back/` ; `pnpm lint`, `pnpm exec prettier --check .` et `pnpm test` dans `web/`
- [ ] T018 Dérouler les trois parcours manuels de [quickstart.md](quickstart.md) à 360 px et noter le résultat dans ce fichier

---

## Dependencies & Execution Order

- **Phase 2** bloque tout : la route doit être reprise à Fortify avant de changer son comportement.
- **US1** dépend de la Phase 2 ; c'est le MVP.
- **US2** dépend de la Phase 2 et de T008 (même action) ; elle se fait juste après US1.
- **US3** dépend de la Phase 2 seulement.
- **Polish** après les trois stories.
- Dans chaque story, le back passe avant le web ; le back est fusionné sur `main` avant le web.

### Parallel Opportunities

- T005, T006 et T007 en parallèle (fichiers différents).
- T014 et T015 en parallèle, et en parallèle d'US2.

## Implementation Strategy

1. Phase 2, puis US1 : l'inscription ne révèle plus rien.
2. US2 : le compte en attente retrouve son lien.
3. US3, puis les finitions et la fusion sur `main`, back puis web.
