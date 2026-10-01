# Implementation Plan: Révision hors ligne

**Branch**: `006-offline-review` | **Date**: 2026-10-01 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/006-offline-review/spec.md`

**Maquette** : aucune. Les écrans « Mes révisions » et séance de la 001 gagnent une indication
hors ligne, un compteur de réponses à envoyer et un remplacement des images ; la déconnexion gagne
un dialogue de confirmation existant (`ConfirmDialog`).

## Summary

Permettre de réviser sans réseau : tant qu'elle est en ligne, la PWA garde sur l'appareil les
cartes des 7 prochains jours ; hors ligne, « Mes révisions » et la séance fonctionnent depuis ces
données ; les réponses restent sur l'appareil et partent au retour du réseau, en comptant à l'heure
où elles ont été données.

**Approche technique** :

- **Back** (`back/functional/learning`) : une instruction lomkit `upcoming` sur `card-progress`
  fournit le paquet. L'action `answer` accepte `answer_id`, `answered_at` et `due_on`, et applique
  chaque réponse par un rejeu de l'historique de la carte. Ce rejeu donne l'ordre des réponses
  données, la première réponse par échéance, le traitement des dates incohérentes et
  l'idempotence. `review_answers` gagne `answer_id`, `due_on` et `status`.
- **Back** (`back/functional/users`) : `GET /api/user` expose le fuseau du compte.
- **Web** (`web/functional/Learning`) : toute réponse, en ligne comme hors ligne, est calculée
  localement par les règles Leitner portées en TypeScript, écrite dans IndexedDB (`idb`), puis
  envoyée une à une.
- **Web** (`web/technical/Pwa`, `web/technical/ApiClient`) : une coquille SPA précachée sert
  `/revisions*` hors ligne, et le store de session distingue « serveur injoignable » de « non
  connecté ».
- **Web** (`web/functional/Account`) : la déconnexion avertit des réponses en attente et efface
  les données de l'appareil.

Les décisions et leurs alternatives sont dans [research.md](research.md).

## Affected Repos

Le back est le fournisseur et se fusionne en premier ; le web suit (constitution 2.0.0, flux de
travail). Les nouveaux champs de `answer` sont facultatifs, donc le web en place continue de
fonctionner entre les deux fusions.

<!-- speckit-multirepo:begin -->
| Repo | Rôle | Ordre de fusion |
|---|---|---|
| back | API Laravel : instruction `upcoming`, action `answer` datée et rejouée, `timezone` dans `/user` | 1 |
| web | PWA Nuxt : paquet et file IndexedDB, séance hors ligne, coquille `/revisions` précachée, avertissement à la déconnexion | 2 |
<!-- speckit-multirepo:end -->

## Technical Context

**Language/Version**: PHP 8.5 (Laravel 13) côté `back/` ; TypeScript 6 (Nuxt 4, Vue 3) côté `web/`

**Primary Dependencies**: côté web, `idb` 8 (nouvelle, directe) et `fake-indexeddb` 6 (nouvelle,
développement). `@vite-pwa/nuxt` (Workbox, `generateSW`) et `laravel-raom-nuxt` sont existants.
Aucune nouvelle dépendance côté back.

**Storage**: PostgreSQL : trois colonnes sur `review_answers`. Navigateur : une base IndexedDB
`cinq-offline-review` (magasins `pack` et `answers`) et le précache du service worker pour la
coquille `/revisions`.

**Testing**: PHPUnit 12 par `./vendor/bin/sail artisan test` (`back/`) ; Vitest et
`@nuxt/test-utils` (`web/`), avec `fake-indexeddb` pour IndexedDB ; Pint, ESLint et Prettier.

**Target Platform**: API Linux sous Docker ; navigateurs récents qui gèrent la PWA (Chrome,
Firefox, Safari iOS 16.4+), installée ou non.

**Project Type**: application web, soit une API et une PWA, dans deux repos distincts

**Performance Goals**: séance hors ligne lancée en moins de 5 s depuis l'ouverture (SC-001) ;
file vidée en moins d'une minute après le retour du réseau (SC-002).

**Constraints**: aucune réponse perdue ni appliquée deux fois (FR-008, FR-016) ; boîte et date
identiques entre l'appareil et le serveur hors cas FR-013 à FR-015 (SC-003) ; rien du compte sur
l'appareil après déconnexion (SC-004) ; aucune image en cache (FR-002) ; l'API n'est jamais
servie depuis un cache du service worker (règle existante).

**Scale/Scope**: 4 user stories, 21 exigences fonctionnelles. Un paquet de quelques centaines de
cartes en texte, de l'ordre du Mo au plus. 2 pages web modifiées et 1 dialogue.

Aucune inconnue ne reste ouverte : toutes sont tranchées dans [research.md](research.md). Un point
est à vérifier en tâche : le prérendu de la coquille `ssr: false` (R8, avec son repli).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principe | Vérification | Avant conception | Après conception |
|---|---|---|---|
| I. La spec d'abord | Spec validée et poussée ; chemins préfixés. FR-013 et FR-015 sont précisés par `due_on`, sans changer leur sens (R3, R4) | ✅ | ✅ |
| II. Couches OSDD | Back : `learning` (paquet, réponses), `users` (fuseau). Web : `Learning` porte paquet, file et règles ; `Account` n'utilise que `useOfflineReview()` exposé par `Learning` ; `Pwa` et `ApiClient` restent techniques | ✅ | ✅ R9 |
| III. Paquets imposés, API par contrat | Paquet et réponses par lomkit (instruction, action) et modèles raom ; aucun `fetch` direct vers lomkit ; `/user` reste une route de compte par `useApiFetch` ; back fusionné avant web, champs facultatifs | ✅ | ✅ R2, R5 |
| IV. Tests obligatoires | Une ligne de test par exigence dans [quickstart.md](quickstart.md) ; table des cas Leitner testée en PHP et en TypeScript (SC-005) ; coupure pendant l'envoi testée (SC-002) | ✅ | ✅ |
| V. Accessibilité, mobile d'abord | Textes dans les fichiers de traduction, vouvoiement ; compteur et indication hors ligne annoncés (`role="status"`) ; écrans existants à 360 px | ✅ | ✅ |
| VI. Sécurité, données personnelles | Données liées à un compte, effacées à la déconnexion, à la demande de suppression et au changement de compte ; aucune réponse de l'action ne révèle l'existence d'une carte d'autrui ; HTML gardé tel qu'assaini par l'API ; brouillons et sujets retirés ou retenus exclus du paquet | ✅ | ✅ R4, R9 |
| VII. Simplicité | Une action élargie plutôt qu'un endpoint de synchronisation ; pas de Background Sync ; horizon en constante ; `idb` justifiée face à l'API IndexedDB brute (R7) | ✅ | ✅ |

Aucun écart à justifier.

## Project Structure

### Documentation (this feature)

```text
specs/006-offline-review/
├── spec.md
├── plan.md              # ce fichier
├── research.md          # décisions techniques (Phase 0)
├── data-model.md        # review_answers, rejeu, IndexedDB (Phase 1)
├── quickstart.md        # guide de validation (Phase 1)
├── contracts/
│   └── api.md           # /user, instruction upcoming, action answer (Phase 1)
├── checklists/
│   └── requirements.md
└── tasks.md             # Phase 2, créé par /speckit-tasks
```

### Source Code (repository root)

```text
back/
├── functional/learning/
│   ├── database/migrations/                  # + answer_id, due_on, status sur review_answers
│   ├── src/
│   │   ├── Enums/AnswerStatus.php            # + applied, discarded
│   │   ├── Domain/LeitnerSchedule.php        # inchangé
│   │   ├── Domain/CardAnswerReplay.php       # + rejeu de l'historique d'une carte (R3, R4)
│   │   ├── Models/ReviewAnswer.php           # + champs, cast du statut
│   │   ├── Queries/DueCardsQuery.php         # borne paramétrable : aujourd'hui ou + 7 jours
│   │   ├── Rest/Actions/AnswerCard.php       # champs facultatifs, délègue au rejeu, plus de 409
│   │   ├── Rest/Instructions/UpcomingCardsInstruction.php   # +
│   │   └── Rest/Resources/CardProgressResource.php          # + instruction upcoming
│   └── tests/{Unit,Feature}/                 # rejeu, upcoming, idempotence, table Leitner
├── functional/users/
│   ├── src/Http/Controllers/CurrentUserController.php      # + timezone
│   └── tests/Feature/LoginTest.php
└── functional/reminders/tests/               # espacement depuis une réponse envoyée plus tard

web/
├── package.json                              # + idb, + fake-indexeddb (dev)
├── technical/ApiClient/
│   ├── app/stores/useSessionStore.ts         # + timezone, + isUnreachable
│   ├── app/middleware/auth.ts                # laisse passer meta.availableOffline si injoignable
│   └── tests/
├── technical/Pwa/
│   ├── nuxt.config.ts                        # précache revisions/index.html, règle de navigation /revisions*
│   ├── app/pages/hors-ligne.vue              # mention de la révision hors ligne
│   └── i18n/locales/fr.json
├── functional/Learning/
│   ├── nuxt.config.ts                        # routeRules ssr: false, prérendu /revisions
│   ├── app/utils/leitner.ts                  # + arrivalBox, nextReviewOn, localDay(tz)
│   ├── app/offline/                          # + base IndexedDB : pack, file de réponses
│   ├── app/composables/useOfflineReview.ts   # + pendingCount, flush, discard, refreshPack (exposé)
│   ├── app/composables/useRevisions.ts       # paquet hors ligne, attend flush en ligne
│   ├── app/composables/useReviewSession.ts   # réponse locale + file, cartes du paquet hors ligne
│   ├── app/plugins/offline-review.client.ts  # ouverture, événement online, changement de compte
│   ├── app/components/{OfflineStatus,PendingAnswers,OfflineImageNotice}.vue   # +
│   ├── app/components/SessionCard.vue        # description au lieu des images hors ligne
│   ├── app/pages/revisions/{index,seance}.vue   # meta.availableOffline
│   ├── i18n/locales/fr.json
│   └── tests/
└── functional/Account/
    ├── app/composables/useAccountMenu.ts     # avertissement, flush puis discard
    ├── app/composables/useAuth.ts            # discard à la demande de suppression
    ├── i18n/locales/fr.json
    └── tests/
```

**Structure Decision**: la révision hors ligne est un parcours de révision. Elle vit dans la
couche `learning` côté back et dans la couche `Learning` côté web, y compris le stockage
IndexedDB, qui n'a pas d'autre usage (principe VII). Les couches techniques ne gagnent que ce qui
relève de l'infrastructure : la coquille précachée et l'état « injoignable » de la session.
