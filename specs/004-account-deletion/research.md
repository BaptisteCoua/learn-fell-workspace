# Research: Suppression de compte

Décisions techniques de la feature 004, avec leurs raisons et les alternatives écartées. Les
chemins renvoient à l'état du code au 2026-10-01.

## R1. Où vit la demande de suppression

**Decision**: deux colonnes nullables sur `users`, portées par la couche `back/functional/users` :
`deletion_requested_at` (horodatage UTC) et `keeps_published_subjects` (booléen, nul pour un compte
sans sujet publié au moment de la demande). La date d'effacement se calcule : demande + 30 jours.

**Rationale**: une demande n'existe que pour un compte et disparaît avec lui ou à l'annulation ;
elle n'a ni historique ni cycle de vie propre (principe VII). La couche `users` possède déjà le
compte, la connexion et l'élagage des comptes non confirmés (`User::prunable()`).

**Alternatives considered**: une table `account_deletion_requests` (un modèle et une relation pour
deux champs, sans second usage) ; un `SoftDeletes` sur `User` (masquerait le compte de toutes les
requêtes Eloquent, y compris celles qui doivent encore le voir pendant le délai, comme la connexion
qui annule, et changerait le sens de `delete()` partout).

## R2. Comment les autres couches réagissent

**Decision**: la couche `users` déclenche deux événements de domaine,
`AccountDeletionRequested(User $user)` et `AccountDeletionCancelled(User $user)`, et l'effacement
passe par `delete()` du modèle, qui déclenche déjà `eloquent.deleting: User`. Chaque couche écoute
ce dont elle a besoin, comme `reminders` le fait déjà (`DeleteRemindersOfUser`,
`RemindersServiceProvider.php:52`).

**Rationale**: `users` est la couche la plus basse ; `catalog`, `learning`, `moderation` et
`reminders` dépendent d'elle et non l'inverse (principe II). Le modèle « chaque couche efface ses
dépendances sur l'événement `deleting` » est celui de la 001 et de la 002 ; aucune clé étrangère
ne cascade (`restrictOnDelete` partout).

**Alternatives considered**: un service d'effacement dans `users` qui connaîtrait toutes les tables
(dépendance inverse, interdite) ; des `cascadeOnDelete` en base (contraire à la règle de l'équipe,
et les fichiers d'images ne seraient pas supprimés).

## R3. Masquer les sujets publiés pendant le délai (« Tout effacer »)

**Decision**: un nouveau statut `SubjectStatus::Withheld` (`withheld`). À la demande, la couche
`catalog` passe les sujets publiés de l'auteur de `published` à `withheld` ; à l'annulation, elle
les repasse à `published` sans toucher à `published_at`. `isPublic()` reste vrai seulement pour
`Published`. La modération (`subjects.moderate`) voit déjà tous les statuts
(`SubjectControl.php:29-32`) et garde donc l'accès aux sujets retenus.

**Rationale**: six endroits filtrent aujourd'hui sur `status = published` (`SubjectControl:46`,
`SubjectResource:138`, `QuestionControl:48`, `QuestionResource:195`, `DueCardsQuery:42`,
`LearningResource:127`). Un statut distinct les rend tous corrects sans les modifier : catalogue,
recherche, adresse directe, séances et rappels (FR-009). Les apprenants gardent leur progression,
rétablie telle quelle si l'auteur annule.

**Alternatives considered**: un filtre « auteur non en cours de suppression » ajouté aux six
requêtes (facile à oublier dans une requête future, et la couche `learning` devrait connaître la
règle) ; repasser les sujets en brouillon (l'annulation ne saurait plus lesquels étaient publiés).

## R4. Masquer le nom pendant le délai et après l'effacement

**Decision**: un accesseur sur `display_name` lui-même, dans le modèle `User`, le rend nul quand
`deletion_requested_at` est renseigné ; `PublicUserResource` est inchangé. Lomkit sérialise par
`attributesToArray()` et ne sait pas renommer un champ, d'où l'accesseur sur la colonne plutôt
qu'un `public_name` exposé sous un autre nom. Les emails que reçoit le compte lui-même lisent son
nom brut par `ownDisplayName()`. Après l'effacement, les colonnes
`subjects.author_id`, `question_images.uploader_id`, `reports.reporter_id` et
`moderation_decisions.admin_id` deviennent nullables, et la relation est nulle. Le web affiche
« Auteur supprimé » pour un sujet et « Compte supprimé » dans la modération quand la relation ou
le nom est nul.

**Rationale**: un seul point d'exposition du nom d'un tiers (`PublicUserResource`, utilisé par
`SubjectResource:53`, `ReportResource:43`, `ModerationDecisionResource:41`). Le libellé dépend du
contexte d'affichage, il appartient donc au web et à ses fichiers de traduction (principe V).

**Alternatives considered**: remplacer le nom par « Auteur supprimé » côté back (le même compte
serait « auteur » dans le catalogue et « compte » dans la modération) ; un compte fantôme
« Auteur supprimé » auquel rattacher les contenus (une ligne `users` factice qui fausse les
comptages et les droits).

## R5. Désactiver le compte immédiatement

**Decision**: la demande supprime toutes les lignes `sessions` du compte, comme le fait déjà
`ResetUserPassword.php:28`, puis invalide la session courante. Les rappels s'arrêtent parce que
l'éligibilité de la couche `reminders` exclut les comptes dont `deletion_requested_at` est
renseigné ; leurs réglages et appareils restent intacts pour l'annulation (FR-015).

**Rationale**: l'authentification est une session Sanctum SPA (`SESSION_DRIVER=database`), sans
jeton d'API ; sans session, toute action reçoit 401 (FR-006). Garder les réglages évite de devoir
les reconstruire à l'annulation.

**Alternatives considered**: une colonne « désactivé » vérifiée par un middleware à chaque
requête (inutile une fois les sessions supprimées, puisque la seule façon d'en rouvrir une est la
connexion, qui annule) ; supprimer les réglages de rappels à la demande (l'annulation ne
rétablirait pas « à l'identique »).

## R6. Annuler à la connexion

**Decision**: `AuthenticateUser` (l'action `authenticateUsing` de Fortify) annule la demande
après une authentification réussie : remise à nul des deux colonnes, puis
`AccountDeletionCancelled`. Une réponse de connexion personnalisée (`LoginResponse` de Fortify)
renvoie `{ "deletion_cancelled": true }` dans ce seul cas, pour que le web affiche le message.

**Rationale**: toute connexion réussie passe par cette action, y compris celle qui suit une
réinitialisation du mot de passe (FR-014). Le verrouillage et les erreurs d'identifiants restent
inchangés, donc un compte en cours de suppression répond comme un compte actif (FR-024).

**Alternatives considered**: un lien « Annuler la suppression » dans l'email (une seconde façon de
s'authentifier, signée, pour un gain faible) ; annuler à la réinitialisation du mot de passe
(la personne n'est pas encore connectée et ne verrait pas la confirmation).

## R7. L'effacement définitif

**Decision**: étendre `User::prunable()` aux comptes dont `deletion_requested_at` date de plus de
30 jours. `model:prune` tourne déjà chaque jour (`routes/console.php`), ce qui tient FR-016
(au plus 24 heures après le délai). `User` redéfinit `prune()` pour exécuter `pruning()` et
`delete()` dans une transaction : si un écouteur échoue, rien n'est effacé et le compte est repris
le lendemain (FR-022). Les fichiers d'images ne sont supprimés qu'après le commit, comme
aujourd'hui (`DeleteQuestionImageFiles`, `DB::afterCommit`).

**Rationale**: la rétention par `Prunable` est la règle de l'équipe et le mécanisme déjà utilisé
pour les comptes non confirmés ; il hydrate chaque modèle, donc les événements `deleting` de toutes
les couches se déclenchent.

**Alternatives considered**: une commande planifiée dédiée `accounts:erase` (une seconde entrée de
planning et une logique de sélection hors du modèle) ; un job différé de 30 jours à la demande
(perdu si la file est vidée, et à annuler à la connexion).

## R8. Ordre de l'effacement par couche

**Decision**: sur `eloquent.deleting: User`, chaque couche efface ou détache ce qui la concerne :

| Couche | Effet |
|---|---|
| `catalog` | « Tout effacer » : supprime tous les sujets de l'auteur, ce qui déclenche la chaîne existante (questions, images, fichiers, apprentissages des autres, signalements). « Laisser » : met `author_id` à nul sur ses sujets `published`, et `uploader_id` à nul sur leurs images ; supprime ses autres sujets. Images envoyées mais jamais rattachées : supprimées. |
| `learning` | Supprime les apprentissages du compte, ce qui supprime réponses et progression (`DeleteCardsOfLearning`). |
| `moderation` | Met `reporter_id` et `admin_id` à nul sur ses signalements et décisions. |
| `reminders` | Inchangé : `DeleteRemindersOfUser` supprime appareils, journal et réglages. |
| `users` | Supprime les sessions et les jetons de réinitialisation de mot de passe ; `HasRoles` détache rôles et permissions. |

**Rationale**: chaque couche ne touche que ses tables. La chaîne de suppression d'un sujet existe
déjà et couvre les apprenants des autres comptes (FR-018) ainsi que les signalements (FR-021).

**Alternatives considered**: rattacher les sujets conservés à un compte système (voir R4).

## R9. Dernier compte d'administration

**Decision**: la couche `users` refuse la demande (`BusinessRuleException('last_admin')`, 422)
quand le compte détient au moins une des permissions `categories.manage`, `subjects.moderate`,
`reports.review`, `moderation.history.view`, et qu'aucun autre compte actif (sans demande en
cours) ne détient la même permission. Les permissions viennent de spatie, jamais d'un nom de rôle
(principe III).

**Rationale**: une administration partielle (un compte qui seul gère les catégories) laisserait
aussi un domaine sans responsable. `users:grant-admin` reste le seul moyen de nommer un
administrateur, ce que la spec note dans ses hypothèses.

**Alternatives considered**: tester seulement le rôle `admin` (interdit par la constitution).

## R10. Aperçu avant la confirmation

**Decision**: deux lectures côté web avant l'écran de confirmation. `GET api/account/deletion`
(couche `users`) renvoie la date d'effacement prévue et `can_request` / `blocked_reason`. Les
nombres de sujets publiés et d'apprenants viennent d'un endpoint de la couche `learning`,
`GET api/learning/authored-subjects-summary`, qui compte les sujets publiés de l'auteur et les
apprenants distincts de ces sujets.

**Rationale**: `users` ne peut pas compter les sujets ni les apprenants sans dépendre de
`catalog` et `learning` (principe II). `learning` dépend déjà de `catalog` et `users`, c'est donc
la seule couche qui voit les deux nombres.

**Alternatives considered**: un endpoint d'aperçu unique dans `users` alimenté par des événements
de collecte (une abstraction sans second usage) ; deux recherches lomkit côté web (le comptage des
apprenants d'autrui est hors de portée de `LearningControl`, qui ne montre que ses propres
apprentissages).

## R11. Endpoints hors lomkit

**Decision**: `GET` et `POST api/account/deletion` sont des contrôleurs de la couche `users`,
déclarés dans `functional/users/routes/account.php` à côté de la vérification d'email ; le web les
appelle par `useApiFetch`, comme les autres routes de compte (`useAuth.ts`).

**Rationale**: la demande n'est pas une ressource CRUD : elle exige le mot de passe, ferme les
sessions et renvoie une date. Les routes de compte de la 001 (Fortify, vérification d'email)
suivent déjà ce modèle, ce que le plan de la 002 a admis pour la désinscription.

**Alternatives considered**: une action lomkit sur `users` (lomkit ne gère pas la ré-authentification
par mot de passe ni la fermeture des sessions, et exposerait la ressource `users` en écriture).

## R12. Email de confirmation

**Decision**: une notification `AccountDeletionRequestedNotification` (canal `mail`) dans la couche
`users`, textes dans `functional/users/lang/fr/notifications.php`, envoyée après la mise à jour du
compte. Un échec d'envoi n'annule pas la demande (edge case) : la notification est mise en file et
la page affichée donne les mêmes informations (FR-013).

**Rationale**: c'est le modèle des emails existants (`VerifyEmailNotification`,
`ResetPasswordNotification`).

**Alternatives considered**: un `Mailable` (aucun dans le projet).
