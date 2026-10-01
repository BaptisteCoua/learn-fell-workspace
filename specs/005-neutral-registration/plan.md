# Implementation Plan: Inscription neutre

**Branch**: `005-neutral-registration` | **Date**: 2026-10-01 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/005-neutral-registration/spec.md`

**Maquette** : aucune. Les écrans d'inscription de la 001 perdent un bloc et changent de texte.

## Summary

Rendre l'inscription muette sur l'existence d'un compte : toute saisie valide reçoit la même
réponse et le même écran. Selon l'adresse, un compte est créé (adresse libre), un nouveau lien part
(compte en attente, au plus un par minute) ou rien ne se passe (compte confirmé, y compris en cours
de suppression).

**Approche technique** : la couche `back/functional/users` reprend la route `POST /api/register`
à Fortify, dont le contrôleur connecte le compte qu'il reçoit. Une action `RegisterAccount` traite
les trois cas et répond toujours `201` avec un message neutre ; le renvoi de lien réutilise
`sendEmailVerificationNotification()`, qui invalide déjà le lien précédent. Côté web, la couche
`web/functional/Account` retire les messages d'adresse prise et met l'écran « Vérifiez vos
emails » au conditionnel. Les décisions et leurs alternatives sont dans [research.md](research.md).

## Affected Repos

Le back est le fournisseur et se fusionne en premier ; le web suit (constitution 2.0.0, flux de
travail).

<!-- speckit-multirepo:begin -->
| Repo | Rôle | Ordre de fusion |
|---|---|---|
| back | API Laravel : route d'inscription neutre, renvoi de lien limité | 1 |
| web | PWA Nuxt : formulaire sans message d'adresse prise, écran « Vérifiez vos emails » au conditionnel | 2 |
<!-- speckit-multirepo:end -->

## Technical Context

**Language/Version**: PHP 8.5 (Laravel 13) côté `back/` ; TypeScript 6 (Nuxt 4, Vue 3) côté `web/`

**Primary Dependencies**: aucune nouvelle. `laravel/fortify` garde la connexion et la
réinitialisation du mot de passe ; l'inscription passe à la couche `users`.

**Storage**: PostgreSQL, sans changement de schéma ; cache pour le nonce des liens et le limiteur.

**Testing**: PHPUnit 12 par `./vendor/bin/sail artisan test` (`back/`), Vitest et
`@nuxt/test-utils` (`web/`) ; Pint, ESLint et Prettier.

**Target Platform**: API Linux sous Docker ; navigateurs récents.

**Project Type**: application web, soit une API et une PWA, dans deux repos distincts

**Performance Goals**: aucun nouveau ; l'égalisation du temps de réponse est hors périmètre.

**Constraints**: réponses identiques mot pour mot pour les quatre cas d'adresse (SC-001) ; aucune
session ouverte par l'inscription ; contrat `POST /api/register` inchangé pour la requête.

**Scale/Scope**: 3 user stories et 10 exigences fonctionnelles ; 2 écrans web modifiés.

Aucune inconnue ne reste ouverte : toutes sont tranchées dans [research.md](research.md).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principe | Vérification | Avant conception | Après conception |
|---|---|---|---|
| I. La spec d'abord | Spec validée et poussée ; chemins préfixés | ✅ | ✅ |
| II. Couches OSDD | Tout dans `users` (back) et `Account` (web) | ✅ | ✅ |
| III. Paquets imposés, API par contrat | Route de compte hors lomkit, comme la vérification d'email de la 001 ; web par `useApiFetch` pour les routes de compte ; back fusionné avant web | ✅ | ✅ Fortify reste pour la connexion et la réinitialisation (research R1) |
| IV. Tests obligatoires | Un test par exigence, listés dans [quickstart.md](quickstart.md) ; SC-001 compare les quatre réponses | ✅ | ✅ |
| V. Accessibilité, mobile d'abord | Textes dans les fichiers de traduction, vouvoiement ; écrans existants | ✅ | ✅ |
| VI. Sécurité, données personnelles | Corrige l'écart de la 001 : plus aucune réponse ne révèle un compte ; aucune session ouverte sur un compte existant ; renvoi limité par compte | ✅ | ✅ |
| VII. Simplicité | Aucune table ni dépendance ; on retire `CreateNewUser`, `RegisteredResponse` et deux codes d'erreur | ✅ | ✅ |

Aucun écart. L'écart suivi dans le Complexity Tracking de la 004 est résolu par cette feature.

## Project Structure

### Documentation (this feature)

```text
specs/005-neutral-registration/
├── spec.md
├── plan.md              # ce fichier
├── research.md          # décisions techniques (Phase 0)
├── data-model.md        # aucun schéma, traitement par état du compte (Phase 1)
├── quickstart.md        # guide de validation (Phase 1)
├── contracts/
│   └── api.md           # POST /api/register et emails (Phase 1)
├── checklists/
│   └── requirements.md
└── tasks.md             # Phase 2, créé par /speckit-tasks
```

### Source Code (repository root)

```text
back/
├── technical/osdd/lang/fr/errors.php     # − email_taken, email_pending_verification
└── functional/users/
    ├── config/fortify.php                # − Features::registration()
    ├── routes/account.php                # + POST register
    ├── lang/fr/account.php               # message neutre
    ├── src/
    │   ├── Actions/                      # + RegisterAccount ; − CreateNewUser
    │   ├── Http/Controllers/             # + RegisterController
    │   ├── Http/Responses/               # − RegisteredResponse
    │   └── Providers/FortifyServiceProvider.php
    └── tests/Feature/RegistrationTest.php

web/
└── functional/Account/
    ├── app/composables/useRegisterForm.ts     # − emailConflict
    ├── app/pages/inscription/index.vue        # − bloc d'adresse prise
    ├── app/pages/inscription/confirmation.vue # texte au conditionnel, liens connexion et mot de passe
    ├── i18n/locales/fr.json
    └── tests/{RegisterPage,ConfirmationPage}.nuxt.spec.ts
```

**Structure Decision**: l'inscription est un parcours de compte ; elle reste dans la couche `users`
côté back et `Account` côté web. Aucune couche nouvelle.
