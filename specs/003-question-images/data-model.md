# Modèle de données : Images dans les questions (003)

Une nouvelle table dans la couche `back/functional/catalog`. Aucune table existante ne change de
schéma. La règle de validation du recto de `questions` change (R7).

## Table `question_images`

| Colonne | Type | Contraintes | Rôle |
|---|---|---|---|
| `id` | bigint | clé primaire | |
| `question_id` | bigint, nullable | FK `questions`, `restrictOnDelete` ; index | nul tant que l'image est en attente |
| `uploader_id` | bigint | FK `users`, `restrictOnDelete` ; index | personne qui a envoyé le fichier |
| `alt` | string(250), nullable | obligatoire dès le rattachement | description (FR-004) |
| `position` | smallint, nullable | 0 à 3, obligatoire dès le rattachement | ordre sur le recto (FR-005) |
| `width` | smallint | | largeur de la plus grande variante, en px |
| `height` | smallint | | hauteur de la plus grande variante, en px |
| `variant_widths` | json | liste d'entiers, sous-ensemble de `[480, 960, 1600]` | variantes réellement produites (une petite image n'est jamais agrandie) |
| `created_at`, `updated_at` | timestamps | index composé (`question_id`, `created_at`) | élagage des images en attente |

- **Fichiers** : `question-images/{id}/{largeur}.webp` sur le disque `catalog.images.disk`.
  Aucun fichier d'origine n'est conservé (research R3).
- **Pas d'unicité en base sur (`question_id`, `position`)** : le rattachement réécrit toutes les
  positions d'un coup. L'unicité et la continuité (0..n-1) sont validées sur le payload (R7).
- **Clés étrangères sans cascade**, comme partout dans le projet : les suppressions passent par
  les modèles (R9).

## Modèle `QuestionImage`

- `belongsTo question`, `belongsTo uploader` (User) ; `Question hasMany images`, triées par
  `position`.
- `$dispatchesEvents` : `deleted` → `QuestionImageDeleted`.
- `Prunable` : `whereNull('question_id')->where('created_at', '<', now()->subDay())`.
- `isPending()` : `question_id` nul.
- `HasControl` avec `QuestionImageControl` (lecture par `search` et `include`), qui reprend
  les périmètres de `QuestionControl` à travers `question` :
  - `subjects.moderate` : tout ;
  - lecture : images des questions de sujets publiés, et celles des sujets dont on est
    l'auteur ;
  - les images en attente ne sortent jamais par `search`.

## États d'une image

```text
            envoi (POST /question-images)
                      │
                      ▼
               ┌─────────────┐   24 h sans rattachement (Prunable)
               │ en attente  │ ─────────────────────────────────► supprimée
               └─────────────┘
                      │ mutate de la question (update dans images)
                      ▼
               ┌─────────────┐   absente de la liste envoyée, question
               │  rattachée  │   supprimée, ou sujet supprimé
               └─────────────┘ ─────────────────────────────────► supprimée
```

- Une image rattachée ne revient jamais en attente et ne change jamais de question.
- « Supprimée » signifie ligne et fichiers effacés définitivement (FR-017).

## Règles de visibilité (FR-016)

| Image | Visiteur | Autre inscrit | Envoyeur (image en attente) | Auteur du sujet | Modérateur (`subjects.moderate`) |
|---|---|---|---|---|---|
| en attente | ✗ | ✗ | ✓ | — | ✗ |
| sujet `published` | ✓ | ✓ | — | ✓ | ✓ |
| sujet `draft` (brouillon ou dépublié) | ✗ | ✗ | — | ✓ | ✓ |
| sujet `retired` | ✗ | ✗ | — | ✓ | ✓ |
| supprimée | ✗ | ✗ | ✗ | ✗ | ✗ |

✗ signifie `404`, sans différence avec une image inexistante. Un apprenant d'un sujet repassé en
brouillon ou retiré n'en voit plus les images, comme il n'en voit plus les cartes (001, FR-051).

## Règles de rattachement (mutate de `questions`)

- `relations.images` porte la liste complète des images du recto ; au plus 4.
- Chaque élément : `operation: update`, `key` (id de l'image), `attributes.alt` (1 à 250
  caractères après rognage), `attributes.position` (0 à 3, toutes distinctes et continues).
- Une image est rattachable si elle est en attente et que `uploader_id` est l'utilisateur
  courant, ou si elle appartient déjà à cette question (`QuestionImagePolicy::attach`).
- Le recto est valide s'il a de 1 à 5 000 caractères visibles, ou s'il est vide avec au moins
  une image. Sinon, l'erreur est `recto_empty`.
- Les images de la question absentes de la liste sont supprimées dans la même transaction, et
  leurs fichiers après la validation.
- `RetiredSubjectLock` et la limite de 500 questions de la 001 s'appliquent sans changement.
- Aucun effet sur `card_progress` (FR-019).

## Configuration de la couche (`back/functional/catalog/config/catalog.php`)

| Clé | Valeur par défaut | Rôle |
|---|---|---|
| `images.disk` | `env('QUESTION_IMAGES_DISK', 'local')` | disque des fichiers |
| `images.max_per_recto` | 4 | FR-003 |
| `images.max_kilobytes` | 5120 | FR-002 |
| `images.max_side` | 8000 | FR-002 |
| `images.variant_widths` | `[480, 960, 1600]` | R3 |
| `images.pending_hours` | 24 | FR-018 |
