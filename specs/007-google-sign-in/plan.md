# Implementation Plan: Connexion avec Google

**Branch**: `007-google-sign-in` | **Date**: 2026-10-01 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/007-google-sign-in/spec.md`

**Maquette** : aucune. Le bouton existe déjà ; s'ajoutent une page de retour (message ou nom affiché)
et un bouton « Confirmer avec Google » sur l'écran de suppression, dans les styles des écrans de
compte.

## Summary

Activer « Continuer avec Google » : créer un compte confirmé sans email de confirmation, se
connecter, lier un compte existant de même adresse, et confirmer une suppression de compte sans mot
de passe.

**Approche technique** :

- **Back** (`back/functional/users`) :
  - `laravel/socialite` mène un flux par redirection : `GET /api/auth/google/redirect`, puis
    `GET /api/auth/google/callback`.
  - Une action `SignInWithGoogle` décide, selon le profil reçu, de connecter, lier, refuser ou
    demander le nom affiché.
  - Le lien est une colonne `users.google_id`, et `password` devient nul.
  - La fin de connexion (fuseau, annulation d'une suppression) est partagée avec la connexion par
    email.
- **Web** (`web/functional/Account`) :
  - un bouton `GoogleButton` remplace `GoogleSoonButton` ;
  - une page `/connexion-google` traite chaque résultat (message, ou formulaire du nom affiché) ;
  - l'écran de suppression propose « Confirmer avec Google » pour un compte lié.

Les décisions et leurs alternatives sont dans [research.md](research.md).

## Affected Repos

Le back est le fournisseur et se fusionne en premier ; le web suit (constitution 2.0.0, flux de
travail). Le bouton du web reste « Bientôt » tant que le back n'est pas en place, puisqu'il change
dans le même commit que la page de retour.

<!-- speckit-multirepo:begin -->
| Repo | Rôle | Ordre de fusion |
|---|---|---|
| back | API Laravel : flux Google par Socialite, lien `google_id`, mot de passe nul, suppression confirmée par Google | 1 |
| web | PWA Nuxt : bouton Google actif, page `/connexion-google` et nom affiché, confirmation de suppression par Google | 2 |
<!-- speckit-multirepo:end -->

## Technical Context

**Language/Version**: PHP 8.5 (Laravel 13) côté `back/` ; TypeScript 6 (Nuxt 4, Vue 3) côté `web/`

**Primary Dependencies**: `laravel/socialite` 5.x, **nouvelle dépendance du back**, à valider par le
développeur avec ce plan (`back/AGENTS.md`). Fortify et Sanctum sont inchangés. Aucune dépendance
nouvelle côté web.

**Storage**: PostgreSQL : `users.google_id`, `users.google_linked_at`, `users.password` devient nul.
Session (table `sessions`) pour le parcours et le profil Google en attente.

**Testing**: PHPUnit 12 par `./vendor/bin/sail artisan test` (`back/`), avec un double de Socialite ;
Vitest et `@nuxt/test-utils` (`web/`) ; Pint, ESLint et Prettier.

**Target Platform**: API Linux sous Docker ; navigateurs récents, PWA installée sur Android et iOS.

**Project Type**: application web, soit une API et une PWA, dans deux repos distincts

**Performance Goals**: entrer connecté en moins de 30 s et 3 gestes depuis l'inscription (SC-001).

**Constraints**:
- jamais deux comptes pour une adresse (SC-002) ;
- aucun retour rejoué ou modifié n'ouvre de session (SC-004) ;
- aucun jeton Google gardé (FR-012) ;
- aucune réponse qui révèle un compte à qui n'a pas prouvé l'adresse (FR-014) ;
- un compte client Google (identifiant et secret) est nécessaire pour les parcours manuels, mais pas
  pour les tests automatisés.

**Scale/Scope**: 4 user stories et 16 exigences fonctionnelles ; 5 routes back (2 nouvelles
redirections, 2 nouvelles routes JSON, 1 route modifiée) ; 1 page web nouvelle et 4 modifiées.

Aucune inconnue ne reste ouverte. Le comportement de la PWA installée sur iOS est à vérifier sur un
appareil ([research.md](research.md), R9).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principe | Vérification | Avant conception | Après conception |
|---|---|---|---|
| I. La spec d'abord | Spec validée, deux clarifications du développeur intégrées, poussée ; chemins préfixés | ✅ | ✅ |
| II. Couches OSDD | Tout dans `users` (back) et `Account` (web). La page `/compte-supprime` utilise `useOfflineReview()` exposé par `Learning` | ✅ | ✅ |
| III. Paquets imposés, API par contrat | Routes de compte hors lomkit, comme l'inscription et la confirmation d'email ; web par `useApiFetch` pour `pending` et `register`, et par lien de navigation pour la redirection ; back fusionné avant web | ✅ | ✅ R2 |
| IV. Tests obligatoires | Un test par exigence dans [quickstart.md](quickstart.md) ; Google remplacé par un double ; `state` rejoué testé (SC-004) ; combinaisons email / Google testées (SC-002) | ✅ | ✅ |
| V. Accessibilité, mobile d'abord | Textes dans les traductions, vouvoiement ; bouton et formulaire à 360 px ; désactivation hors ligne expliquée | ✅ | ✅ |
| VI. Sécurité, données personnelles | Portées minimales, aucun jeton gardé, `state` et PKCE, profil en attente côté serveur, suppression toujours possible, lien effacé avec le compte, mot de passe d'un compte en attente retiré | ✅ | ✅ R3, R4, R7 |
| VII. Simplicité | Une colonne plutôt qu'une table d'identités ; un seul fournisseur ; une seule page de retour. La dépendance `laravel/socialite` est justifiée face à un flux OpenID écrit à la main (R1) | ✅ | ✅ |

Aucun écart. La nouvelle dépendance n'en est pas un (VII l'admet quand elle est justifiée), mais
`back/AGENTS.md` demande l'accord du développeur : il est demandé avec ce plan.

## Project Structure

### Documentation (this feature)

```text
specs/007-google-sign-in/
├── spec.md
├── plan.md              # ce fichier
├── research.md          # décisions techniques (Phase 0)
├── data-model.md        # colonnes users, session, table de décision (Phase 1)
├── quickstart.md        # guide de validation (Phase 1)
├── contracts/
│   └── api.md           # routes Google, suppression, login (Phase 1)
├── checklists/
│   └── requirements.md
└── tasks.md             # Phase 2, créé par /speckit-tasks
```

### Source Code (repository root)

```text
back/
├── composer.json, composer.lock                      # + laravel/socialite
├── .env.example                                      # + GOOGLE_CLIENT_ID, GOOGLE_CLIENT_SECRET, GOOGLE_REDIRECT_URI
└── functional/users/
    ├── database/migrations/                          # + google_id, google_linked_at ; password nul
    ├── database/factories/UserFactory.php            # + état withGoogle()
    ├── routes/account.php                            # + auth/google/{redirect,callback,pending,register}
    ├── lang/fr/account.php                           # messages
    ├── src/
    │   ├── Actions/CompleteSignIn.php                # + fuseau, annulation de suppression (repris d'AuthenticateUser)
    │   ├── Actions/SignInWithGoogle.php              # + table de décision (R4)
    │   ├── Actions/RegisterWithGoogle.php            # + création après le nom affiché (R5)
    │   ├── Actions/AuthenticateUser.php              # compte sans mot de passe = mot de passe faux
    │   ├── Actions/RequestAccountDeletion.php        # preuve : mot de passe ou identité Google
    │   ├── Http/Controllers/GoogleSignInController.php      # + redirect, callback
    │   ├── Http/Controllers/GoogleRegistrationController.php # + pending, register
    │   ├── Http/Controllers/AccountDeletionController.php   # + confirmations
    │   ├── Models/User.php                           # google_id caché, password nul
    │   └── Providers/UsersServiceProvider.php        # services.google
    └── tests/Feature/{GoogleSignIn,GoogleRegistration,GoogleAccountDeletion}Test.php

web/
└── functional/Account/
    ├── app/components/GoogleButton.vue               # + remplace GoogleSoonButton.vue (supprimé)
    ├── app/composables/useGoogleSignIn.ts            # + URL du bouton, résultat, nom affiché
    ├── app/pages/connexion-google.vue                # + page de retour
    ├── app/pages/{connexion,inscription/index}.vue   # GoogleButton
    ├── app/pages/supprimer-mon-compte.vue            # « Confirmer avec Google » selon confirmations
    ├── app/pages/compte-supprime.vue                 # efface les données de révision de l'appareil
    ├── app/composables/useAccountDeletion.ts         # confirmations, retour ?google=
    ├── i18n/locales/fr.json
    └── tests/{GoogleReturnPage,LoginPage,RegisterPage,AccountDeletionPage}.nuxt.spec.ts
```

**Structure Decision**: la connexion avec Google est un parcours de compte. Elle vit dans la couche
`users` côté back et dans la couche `Account` côté web. Aucune couche nouvelle.
