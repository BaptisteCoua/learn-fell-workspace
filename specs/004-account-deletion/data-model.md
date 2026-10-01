# Data Model: Suppression de compte

Les décisions sont justifiées dans [research.md](research.md). Aucune nouvelle table.

## Compte (`users`, couche `back/functional/users`)

Colonnes ajoutées :

| Colonne | Type | Règle |
|---|---|---|
| `deletion_requested_at` | `timestamp` nullable, indexé | Renseignée à la demande, remise à nul à l'annulation |
| `keeps_published_subjects` | `boolean` nullable | `true` : « Laisser mes sujets publiés » ; `false` : « Tout effacer » ; nul si le compte n'avait aucun sujet publié à la demande |

Valeur dérivée : **date d'effacement** = `deletion_requested_at` + 30 jours (constante
`User::DELETION_GRACE_DAYS`, sans option de configuration : principe VII).

Accesseur ajouté : `public_name`, égal à `display_name`, ou nul quand `deletion_requested_at` est
renseigné. `PublicUserResource` l'expose sous le nom `display_name`.

### États

```text
                demande (mot de passe)                 délai écoulé (model:prune)
   actif ─────────────────────────────▶ en suppression ─────────────────────────▶ effacé
     ▲                                        │
     └──────────── connexion réussie ─────────┘
```

| État | `deletion_requested_at` | Sessions | Nom public | Rappels |
|---|---|---|---|---|
| actif | nul | ouvertes | affiché | selon réglages |
| en suppression | renseigné | toutes supprimées | nul | aucun |
| effacé | ligne supprimée | aucune | relation nulle | aucun |

### Règles de la demande

- Refusée si le mot de passe est faux (`password` invalide, 422, aucun changement).
- Refusée si le compte est le dernier compte actif à détenir l'une des permissions
  d'administration (`categories.manage`, `subjects.moderate`, `reports.review`,
  `moderation.history.view`) : `last_admin`, 422.
- `keeps_published_subjects` est obligatoire si le compte a au moins un sujet `published`
  (`subject_choice_required`, 422), et ignoré sinon.
- Un compte déjà en suppression ne peut pas refaire de demande : il est déconnecté.

### Élagage

`User::prunable()` couvre désormais deux cas : comptes non confirmés depuis 7 jours (inchangé), et
comptes dont `deletion_requested_at` est antérieur à maintenant moins 30 jours. `User::prune()`
exécute `pruning()` et `delete()` dans une transaction.

## Sujet (`subjects`, couche `back/functional/catalog`)

- `author_id` devient **nullable** (la contrainte `restrictOnDelete` reste) : nul pour un sujet
  conservé après l'effacement de son auteur.
- `SubjectStatus` gagne le cas **`Withheld`** (`withheld`), libellé « Retenu » : un sujet publié
  dont l'auteur a choisi « Tout effacer » et dont la suppression est en cours.

### Transitions ajoutées

| De | Vers | Déclencheur |
|---|---|---|
| `published` | `withheld` | `AccountDeletionRequested` avec `keeps_published_subjects = false` |
| `withheld` | `published` | `AccountDeletionCancelled` |
| `withheld` | `retired` | décision de modération, comme pour un sujet publié |

- `published_at` n'est pas modifié par ces transitions.
- Un sujet `withheld` est invisible pour tout compte sans `subjects.moderate`, et n'est pas
  modifiable (son auteur est déconnecté).
- Un sujet dont `author_id` est nul n'est modifiable par personne (`SubjectControl` n'autorise
  l'écriture que sur ses propres sujets) ; la modération peut encore le retirer.

## Image de question (`question_images`, couche `catalog`)

- `uploader_id` devient **nullable** : nul pour une image d'un sujet conservé.

## Signalement (`reports`) et décision (`moderation_decisions`), couche `back/functional/moderation`

- `reports.reporter_id` et `moderation_decisions.admin_id` deviennent **nullables** : nul après
  l'effacement du compte qui a signalé ou décidé.
- Inchangé : les signalements d'un sujet supprimé partent avec lui
  (`DeleteReportsOfDeletedSubject`) ; les décisions gardent `subject_title`.

## Rappels (couche `back/functional/reminders`)

- Aucune colonne ajoutée. L'éligibilité exclut les comptes dont `deletion_requested_at` est
  renseigné ; `next_reminder_at` n'est pas modifié, le rappel suivant l'annulation part au créneau
  normal.

## Effacement : ce que chaque couche fait sur `eloquent.deleting: User`

| Couche | Données | Effet |
|---|---|---|
| `catalog` | sujets, questions, images | voir [research.md, R8](research.md#r8-ordre-de-leffacement-par-couche) |
| `learning` | `learnings`, `card_progress`, `review_answers` du compte | supprimés |
| `moderation` | `reports.reporter_id`, `moderation_decisions.admin_id` | mis à nul |
| `reminders` | `reminder_settings`, `push_subscriptions`, `reminder_sends` | supprimés (existant) |
| `users` | `sessions`, `password_reset_tokens`, rôles et permissions | supprimés |
