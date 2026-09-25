# Implementation Plan: Rappels de révision

**Branch**: `002-review-reminders` | **Date**: 2026-09-25 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/002-review-reminders/spec.md`

**Maquette** : aucune. Les écrans reprennent les composants et la direction visuelle de la 001
(brutalisme jaune et bleu).

## Summary

Prévenir l'apprenant, au plus une fois par jour et à l'heure qu'il choisit, du nombre de cartes qui
l'attendent :

- par notification push sur chacun de ses appareils, par email, ou les deux ;
- désactivé par défaut, proposé une seule fois après le premier « Apprendre ce sujet », réglable
  dans une section « Rappels » de la page Compte ;
- de moins en moins souvent s'il ne révise plus (chaque jour, puis un jour sur deux, puis chaque
  semaine) ;
- coupé en un clic depuis un email, sans connexion.

**Approche technique** : une nouvelle couche OSDD `back/functional/reminders` porte les réglages,
les appareils et le journal des rappels. Une commande planifiée chaque minute choisit les comptes
dont l'heure est venue (colonne `next_reminder_at` en UTC, indexée), puis une tâche en file vérifie
l'éligibilité et envoie une notification Laravel sur les canaux `mail` et `webpush`
(`laravel-notification-channels/webpush`, clés VAPID). Côté web, une couche
`web/functional/Reminders` gère la proposition, la section du Compte, l'abonnement push par
l'API Push du navigateur et la page de désinscription. Le service worker généré garde Workbox et
importe un petit script qui affiche la notification et ouvre la séance. Les décisions et leurs
alternatives sont dans [research.md](research.md).

## Affected Repos

Le back est le fournisseur et se fusionne en premier ; le web consomme ses endpoints et reste
ouvert tant que la merge request du back n'est pas fusionnée et déployée.

<!-- speckit-multirepo:begin -->
| Repo | Rôle | Ordre de fusion |
|---|---|---|
| back | API Laravel : réglages, appareils, envoi planifié, désinscription | 1 |
| web | PWA Nuxt : proposition, section « Rappels », abonnement push, page de désinscription | 2 |
<!-- speckit-multirepo:end -->

## Technical Context

**Language/Version**: PHP 8.5 (Laravel 13) côté `back/` ; TypeScript 6 (Nuxt 4, Vue 3) côté `web/`

**Primary Dependencies**:

- `back/` :
  - déjà installés : `xefi/laravel-osdd`, `lomkit/laravel-rest-api`,
    `lomkit/laravel-access-control`, `laravel/sanctum`, `laravel/fortify` ;
  - à ajouter : `laravel-notification-channels/webpush` ^13.0 (Laravel 13.13 et plus,
    `minishlink/web-push` ^11).
- `web/` : aucune nouvelle dépendance. L'API Push et l'API Notifications du navigateur suffisent ;
  `@vite-pwa/nuxt` importe le script du service worker.

**Storage**: PostgreSQL, trois nouvelles tables (`reminder_settings`, `push_subscriptions`,
`reminder_sends`) ; Redis pour la file d'attente

**Testing**: PHPUnit 12 par `./vendor/bin/sail artisan test` (`back/`), Vitest et
`@nuxt/test-utils` avec happy-dom (`web/`) ; Pint, ESLint et Prettier. L'horloge est figée dans
les tests (`$this->travelTo`) pour couvrir fuseaux, changements d'heure et espacement.

**Target Platform**: API Linux sous Docker ; navigateurs récents. Push : Chrome, Edge et Firefox
sur ordinateur et Android ; Safari sur macOS ; iOS et iPadOS 16.4 et plus, seulement une fois la
PWA installée (FR-005).

**Project Type**: application web, soit une API et une PWA, dans deux repos distincts

**Performance Goals**: 99 % des rappels partis dans les 5 minutes qui suivent l'heure choisie
(SC-002). La commande de chaque minute lit un index, sans parcourir tous les comptes.

**Constraints**:

- au plus un rappel par jour et par compte, garanti par une contrainte d'unicité en base
  (FR-007, SC-003) ;
- heure locale respectée malgré les changements d'heure (edge cases) ;
- désinscription sans connexion, et sans rien révéler pour un lien invalide (FR-014 à FR-016,
  principe VI) ;
- aucune donnée personnelle dans la notification push au-delà du nombre de cartes (FR-011) ;
- 360 px minimum, aucun libellé de bouton tronqué, du 360 au 1440 px.

**Scale/Scope**: 4 user stories et 20 exigences fonctionnelles ; 3 surfaces web (proposition,
section du Compte, page de désinscription), un email et une notification

Aucune inconnue ne reste ouverte : toutes sont tranchées dans [research.md](research.md).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principe | Vérification | Avant conception | Après conception |
|---|---|---|---|
| I. La spec d'abord | Spec validée et poussée ; chemins préfixés par `back/` ou `web/` | ✅ | ✅ |
| II. Couches OSDD | Nouvelle couche `back/functional/reminders`, qui dépend de `users` et de `learning`, sans l'inverse ; nouvelle couche `web/functional/Reminders` ; le script push du service worker reste dans `web/technical/Pwa` | ✅ | ✅ `users` n'est pas modifié : le destinataire des notifications est le modèle `ReminderSetting` (research R3) |
| III. Paquets imposés, API par contrat | Réglages et appareils par des ressources lomkit avec `Control` ; web uniquement par modèles raom ; back fusionné avant web | ✅ | ✅ Seule la désinscription est hors lomkit, car elle se fait sans session, par lien signé, comme la confirmation d'email de la 001 |
| IV. Tests obligatoires | Un test par exigence ; les règles d'éligibilité, d'espacement et de prochain créneau sont des classes du domaine couvertes à 100 % | ✅ | ✅ Cas listés dans [quickstart.md](quickstart.md) |
| V. Accessibilité, mobile d'abord | Dialogue et section dans les composants Vuetify thémés de la 001 ; i18n dans la couche ; libellés complets de 360 à 1440 px | ✅ | ✅ |
| VI. Sécurité, données personnelles | Consentement explicite ; lien signé ; réponse neutre pour un lien invalide ; données supprimées avec le compte ; journal élagué après 90 jours | ✅ | ✅ |
| VII. Simplicité | Pas de service externe d'envoi push (Web Push standard, VAPID) ; pas de worker dédié : scheduler et file Redis existants ; une seule dépendance ajoutée | ✅ | ✅ |

Aucun écart non justifié. La dépendance ajoutée est tracée dans Complexity Tracking.

## Project Structure

### Documentation (this feature)

```text
specs/002-review-reminders/
├── spec.md
├── plan.md              # ce fichier
├── research.md          # décisions techniques (Phase 0)
├── data-model.md        # tables, règles d'éligibilité et d'espacement (Phase 1)
├── quickstart.md        # guide de validation (Phase 1)
├── contracts/
│   └── api.md           # contrat back ↔ web, email et notification (Phase 1)
├── checklists/
│   └── requirements.md
└── tasks.md             # Phase 2, créé par /speckit-tasks
```

### Source Code (repository root)

```text
back/
├── composer.json                      # + laravel-notification-channels/webpush
├── .env.example                       # + VAPID_PUBLIC_KEY, VAPID_PRIVATE_KEY, VAPID_SUBJECT
├── routes/console.php                 # + reminders:dispatch chaque minute
└── functional/
    └── reminders/                     # nouvelle couche
        ├── config/webpush.php         # modèle d'abonnement de la couche, TTL, urgence
        ├── routes/api.php             # Rest::resource(...) et POST reminders/unsubscribe/{id}
        ├── lang/fr/                   # textes de l'email et de la notification
        ├── src/
        │   ├── Models/                # ReminderSetting (Notifiable), PushSubscription, ReminderSend
        │   ├── Domain/                # ReminderEligibility, ReminderSpacing, NextReminderSlot
        │   ├── Rest/{Resources,Actions}
        │   ├── Access/Controls
        │   ├── Policies
        │   ├── Console/               # DispatchDueRemindersCommand
        │   ├── Jobs/                  # SendReviewReminder
        │   ├── Notifications/         # ReviewReminderNotification (mail, webpush)
        │   ├── Listeners/             # création des réglages, fuseau modifié, compte supprimé, livraison push
        │   ├── Http/Controllers/      # UnsubscribeController
        │   ├── Support/               # UnsubscribeLink
        │   └── Providers/
        ├── database/{migrations,factories,seeders}
        └── tests/{Feature,Unit}

web/
├── .env.example                       # + NUXT_PUBLIC_VAPID_PUBLIC_KEY
├── technical/
│   └── Pwa/
│       ├── nuxt.config.ts             # workbox.importScripts: ['sw-push.js']
│       └── public/sw-push.js          # événements push et notificationclick
└── functional/
    ├── Reminders/                     # nouvelle couche
    │   ├── nuxt.config.ts             # runtimeConfig.public.vapidPublicKey
    │   ├── app/
    │   │   ├── models/                # ReminderSetting, PushSubscription
    │   │   ├── components/            # ReminderProposalDialog, AccountReminders, ReminderDeviceRow
    │   │   ├── composables/           # useReminderSettings, usePushDevice, useReminderProposal, useUnsubscribe
    │   │   └── pages/rappels/desinscription.vue
    │   ├── i18n/locales/fr.json
    │   └── tests/
    ├── Learning/                      # LearnSubjectPanel : ouvre la proposition après le premier apprentissage
    └── Account/                       # compte.vue : section « Rappels »
```

**Structure Decision**: les rappels forment un domaine à part, avec son propre cycle de vie
(consentement, envoi, désinscription). Ils deviennent donc une couche fonctionnelle dans chaque
repo plutôt qu'une extension de `learning` ou de `users`. La couche back lit `learning` (cartes
dues, dernière réponse) et `users` (fuseau, adresse confirmée) ; ni l'une ni l'autre ne la
connaît. Le script du service worker est générique (il affiche le titre, le texte et le lien reçus)
et reste donc dans la couche technique `Pwa`.

## Complexity Tracking

> Dépendances ajoutées qui ne figurent pas dans les listes approuvées Xefi.

| Ajout | Pourquoi il est nécessaire | Alternative plus simple écartée parce que |
|---|---|---|
| `laravel-notification-channels/webpush` | Chiffrement des messages Web Push (RFC 8291) et signature VAPID (RFC 8292), canal de notification Laravel, modèle d'abonnement (FR-006, FR-017) | Chiffrer soi-même les messages est risqué ; Firebase Cloud Messaging ajoute un service tiers, un SDK web et un compte Google pour un besoin couvert par le standard |
