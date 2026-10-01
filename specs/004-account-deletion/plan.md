# Implementation Plan: Suppression de compte

**Branch**: `004-account-deletion` | **Date**: 2026-10-01 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/004-account-deletion/spec.md`

**Maquette** : aucune. L'écran de suppression reprend les composants de la page Compte de la 001
(`AccountField`, `AccountNotice`, boutons Vuetify thémés) et la direction visuelle brutaliste.

## Summary

Permettre à une personne inscrite de supprimer son compte depuis sa page Compte :

- confirmation par mot de passe, déconnexion immédiate de tous ses appareils, arrêt des rappels,
  nom masqué partout, email qui donne la date d'effacement ;
- choix, pour une personne qui a publié, entre laisser ses sujets publiés sous « Auteur supprimé »
  et tout effacer ;
- annulation par simple reconnexion pendant 30 jours ;
- effacement complet et atomique au bout du délai, en gardant signalements et décisions de
  modération sans nom.

**Approche technique** : la demande est portée par deux colonnes sur `users`
(`deletion_requested_at`, `keeps_published_subjects`). La couche `back/functional/users` expose
`GET` et `POST api/account/deletion`, ferme les sessions, annule la demande dans l'action de
connexion de Fortify, et déclenche `AccountDeletionRequested` / `AccountDeletionCancelled`. Les
sujets à effacer passent au nouveau statut `withheld`, ce qui les retire d'un coup du catalogue,
des séances et des rappels. L'effacement réutilise l'élagage quotidien de `User` (`Prunable`),
dans une transaction, et chaque couche efface ou détache ses données sur `eloquent.deleting: User`,
selon le modèle de la 001 et de la 002. Côté web, la couche `web/functional/Account` porte l'écran,
la page de confirmation et le message d'annulation ; `Catalog` et `Moderation` affichent
« Auteur supprimé » et « Compte supprimé ». Les décisions et leurs alternatives sont dans
[research.md](research.md).

## Affected Repos

Le back est le fournisseur et se fusionne en premier ; le web consomme ses endpoints et reste
ouvert tant que la merge request du back n'est pas fusionnée et déployée.

<!-- speckit-multirepo:begin -->
| Repo | Rôle | Ordre de fusion |
|---|---|---|
| back | API Laravel : demande, annulation à la connexion, statut `withheld`, élagage et effacement par couche | 1 |
| web | PWA Nuxt : écran de suppression, page de confirmation, message d'annulation, libellés « Auteur supprimé » et « Compte supprimé » | 2 |
<!-- speckit-multirepo:end -->

## Technical Context

**Language/Version**: PHP 8.5 (Laravel 13) côté `back/` ; TypeScript 6 (Nuxt 4, Vue 3) côté `web/`

**Primary Dependencies**: aucune nouvelle. `back/` : `xefi/laravel-osdd`, `lomkit/laravel-rest-api`,
`lomkit/laravel-access-control`, `spatie/laravel-permission`, `laravel/sanctum`, `laravel/fortify`.
`web/` : Vuetify, `@nuxtjs/i18n`, `laravel-raom-nuxt`, `useApiFetch` pour les routes de compte.

**Storage**: PostgreSQL. Aucune table nouvelle : deux colonnes sur `users`, quatre clés étrangères
rendues nullables (`subjects.author_id`, `question_images.uploader_id`, `reports.reporter_id`,
`moderation_decisions.admin_id`), un cas ajouté à `SubjectStatus`. Sessions en base
(`SESSION_DRIVER=database`), file Redis pour l'email.

**Testing**: PHPUnit 12 par `./vendor/bin/sail artisan test` (`back/`), Vitest et
`@nuxt/test-utils` avec happy-dom (`web/`) ; Pint, ESLint et Prettier. L'horloge est figée
(`$this->travelTo`) pour le délai de 30 jours et l'élagage.

**Target Platform**: API Linux sous Docker ; navigateurs récents, PWA installée ou non.

**Project Type**: application web, soit une API et une PWA, dans deux repos distincts

**Performance Goals**: effacement au plus 24 heures après le délai (FR-016), par l'élagage
quotidien existant ; un compte effacé par transaction.

**Constraints**:

- effacement tout ou rien (FR-022), fichiers d'images supprimés seulement après le commit ;
- aucune réponse ne distingue un compte en suppression d'un compte actif, ni un compte effacé
  d'une adresse jamais utilisée (FR-024, principe VI) ;
- dépendances de couches inchangées : `users` ne connaît ni `catalog` ni `learning` (principe II) ;
- 360 px minimum ; libellés complets de 360 à 1440 px.

**Scale/Scope**: 5 user stories et 25 exigences fonctionnelles ; 2 surfaces web nouvelles (écran de
suppression, page de confirmation), 1 message à la connexion, des libellés dans 2 couches ; 1 email.

Aucune inconnue ne reste ouverte : toutes sont tranchées dans [research.md](research.md).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principe | Vérification | Avant conception | Après conception |
|---|---|---|---|
| I. La spec d'abord | Spec validée et poussée ; corrigée avant le plan sur deux points révélés par le code (réponse d'inscription, signalements d'un sujet effacé), avec l'accord du développeur ; chemins préfixés | ✅ | ✅ |
| II. Couches OSDD | Demande et événements dans `users` ; réactions dans `catalog`, `learning`, `moderation`, `reminders` ; aucune dépendance inverse | ✅ | ✅ Le résumé des sujets et apprenants vit dans `learning`, seule couche qui voit les deux (research R10) |
| III. Paquets imposés, API par contrat | Accès par permissions spatie, dernier admin compris ; web par raom pour les ressources ; back fusionné avant web | ✅ | ✅ `account/deletion` est hors lomkit comme les routes de compte de la 001 : ré-authentification et fermeture des sessions (research R11) |
| IV. Tests obligatoires | Un test par exigence, listés dans [quickstart.md](quickstart.md) ; l'invisibilité des sujets `withheld` couverte à 100 % comme celle des brouillons | ✅ | ✅ |
| V. Accessibilité, mobile d'abord | Écran dans les composants de la page Compte ; textes dans les fichiers de traduction, back et web ; vouvoiement | ✅ | ✅ |
| VI. Sécurité, données personnelles | Ré-authentification par mot de passe ; sessions fermées ; nom masqué pendant le délai ; effacement complet ; réponses indiscernables | ✅ | ⚠️ L'inscription répond « adresse déjà utilisée » depuis la 001 : la 004 n'aggrave rien (même réponse pendant le délai), l'écart est suivi à part |
| VII. Simplicité | Pas de table, de commande ni de dépendance nouvelle ; délai en constante ; élagage existant | ✅ | ✅ |

Un écart est hérité de la 001 et non introduit ; il est tracé dans Complexity Tracking.

## Project Structure

### Documentation (this feature)

```text
specs/004-account-deletion/
├── spec.md
├── plan.md              # ce fichier
├── research.md          # décisions techniques (Phase 0)
├── data-model.md        # colonnes, statut withheld, états du compte (Phase 1)
├── quickstart.md        # guide de validation (Phase 1)
├── contracts/
│   └── api.md           # contrat back ↔ web et email (Phase 1)
├── checklists/
│   └── requirements.md
└── tasks.md             # Phase 2, créé par /speckit-tasks
```

### Source Code (repository root)

```text
back/
└── functional/
    ├── users/
    │   ├── routes/account.php              # + GET, POST account/deletion
    │   ├── lang/fr/notifications.php       # + email de demande
    │   ├── database/migrations/            # + deletion_requested_at, keeps_published_subjects
    │   ├── src/
    │   │   ├── Models/User.php             # display_name masqué, prunable() élargi, prune() transactionnel
    │   │   ├── Events/                     # AccountDeletionRequested, AccountDeletionCancelled
    │   │   ├── Actions/                    # RequestAccountDeletion ; AuthenticateUser annule
    │   │   ├── Support/                    # LastAdministratorGuard
    │   │   ├── Http/Controllers/           # AccountDeletionController
    │   │   ├── Http/Responses/             # LoginResponse (deletion_cancelled)
    │   │   ├── Listeners/                  # DeleteSessionsOfUser
    │   │   ├── Notifications/              # AccountDeletionRequestedNotification
    │   │   └── Rest/Resources/PublicUserResource.php
    │   └── tests/Feature
    ├── catalog/
    │   ├── database/migrations/            # author_id, uploader_id nullables
    │   ├── src/Enums/SubjectStatus.php     # + Withheld
    │   ├── src/Listeners/                  # WithholdSubjectsOfLeavingAuthor, RestoreWithheldSubjects, EraseSubjectsOfUser
    │   └── tests/Feature
    ├── learning/
    │   ├── routes/api.php                  # + GET learning/authored-subjects-summary
    │   ├── src/Http/Controllers/           # AuthoredSubjectsSummaryController
    │   ├── src/Listeners/                  # DeleteLearningsOfUser
    │   └── tests/Feature
    ├── moderation/
    │   ├── database/migrations/            # reporter_id, admin_id nullables
    │   ├── src/Listeners/                  # AnonymizeModerationOfUser
    │   └── tests/Feature
    └── reminders/
        ├── src/                            # éligibilité : exclut les comptes en suppression
        └── tests/Feature

web/
└── functional/
    ├── Account/
    │   ├── app/pages/supprimer-mon-compte.vue            # écran, choix, mot de passe
    │   ├── app/pages/compte-supprime.vue   # confirmation et date (sans session)
    │   ├── app/composables/                # useAccountDeletion ; useAuth (deleteAccount) ; useLoginForm (annulation)
    │   ├── app/components/AccountMenuPanel.vue          # + entrée « Supprimer mon compte »
    │   ├── i18n/locales/fr.json
    │   └── tests/
    ├── Catalog/                            # « Auteur supprimé » quand l'auteur ou son nom est nul
    └── Moderation/                         # « Compte supprimé » ; statut « Retenu »
```

**Structure Decision**: la suppression de compte n'est pas un domaine à part : c'est un état du
compte, donc dans `users`, et un effet dans chaque couche qui possède des données d'un compte.
Aucune couche nouvelle, ni côté back ni côté web. Seule exception à la règle « chaque couche
parle d'elle-même » : le résumé avant suppression, qui combine sujets et apprenants, vit dans
`learning` parce que c'est la seule couche qui dépend des deux.

## Complexity Tracking

| Écart | Pourquoi il est accepté | Alternative écartée parce que |
|---|---|---|
| L'inscription répond « adresse déjà utilisée » (principe VI), écart de la 001 — **résolu par la feature 005** | La 004 garde la même réponse pendant le délai, donc rien ne distingue un compte en suppression ; corriger l'inscription change un parcours de la 001 et mérite sa propre spec | Corriger ici élargirait la 004 à un parcours sans lien avec la suppression |
