# Modèle de données : Comptes, contenu et révision Leitner (001)

Base PostgreSQL du back. Chaque table appartient à une couche OSDD ; les migrations vivent dans
`back/functional/<couche>/database/migrations/`. Aucune clé étrangère en cascade (`ON DELETE
RESTRICT` par défaut) : les suppressions liées sont faites par l'application (voir research R8).
Les statuts sont des chaînes contrôlées par des enums PHP, pas des enums de base.

## Couche `functional/users`

### users

| Colonne | Type | Règles |
|---|---|---|
| id | bigint PK | |
| display_name | varchar(60) | obligatoire, 2 à 60 caractères, public (FR-001) |
| email | varchar(255) | unique (insensible à la casse), jamais exposé publiquement |
| password | varchar | hash bcrypt, 8 caractères minimum, saisi deux fois (FR-001) |
| email_verified_at | timestamp null | nul = compte inactif (FR-002) |
| timezone | varchar(64) | fuseau IANA, défaut `Europe/Paris` (research R6) |
| remember_token | varchar null | |
| created_at, updated_at | timestamps | |

- **Prunable** : sans `email_verified_at` et créé il y a plus de 7 jours (research R3).
- **Relations** : a plusieurs `subjects` (auteur), `reports`, `learnings`, `card_progress`.
- Rôles et permissions via les tables de `spatie/laravel-permission`. Données de référence insérées
  par migration : rôles `member` et `admin` ; permissions `categories.manage`,
  `subjects.moderate`, `reports.review`, `moderation.history.view`, toutes données au rôle `admin`.
- `sessions` (pilote `database`) et `password_reset_tokens` : tables standard de Laravel.

## Couche `functional/catalog`

### categories

| Colonne | Type | Règles |
|---|---|---|
| id | bigint PK | |
| name | varchar(60) | obligatoire |
| name_normalized | varchar(60) | unique ; minuscules, sans accents ni espaces superflus (FR-009) |
| position | integer | ordre d'affichage, unique |

- Suppression refusée si au moins un sujet y est rangé, quel que soit son statut (FR-010).

### tags

| Colonne | Type | Règles |
|---|---|---|
| id | bigint PK | |
| name | varchar(30) | unique, normalisé en minuscules, espaces réduits (FR-011) |

### subject_tag

Pivot `subject_id`, `tag_id`, clé primaire composée. 10 tags au plus par sujet.

### subjects

| Colonne | Type | Règles |
|---|---|---|
| id | bigint PK | |
| author_id | FK users | obligatoire |
| category_id | FK categories | obligatoire |
| title | varchar(120) | 3 à 120 caractères |
| description | text | 2 000 caractères au plus |
| status | varchar(16) | `draft`, `published`, `retired` (FR-013) |
| published_at | timestamp null | renseigné à chaque publication |
| retired_reason | text null | motif de la modération, obligatoire au retrait |
| retired_at | timestamp null | |
| search_document | text | titre + description + tags normalisés, index GIN `pg_trgm` (research R5) |
| created_at, updated_at | timestamps | |

**Transitions de statut** :

| De | Vers | Qui | Condition |
|---|---|---|---|
| (création) | draft | auteur | email confirmé (compte actif) |
| draft | published | auteur | au moins une question (FR-016) |
| published | draft | auteur | (dépublication, FR-017) |
| published | retired | admin (`subjects.moderate`) | motif obligatoire ; clôt les signalements en attente (FR-031) |
| retired | draft | admin (`subjects.moderate`) | rétablissement (FR-032) |
| retired | published | — | interdit, y compris pour l'auteur (FR-033) |

**Visibilité** (périmètres de la `SubjectControl`, FR-022 et FR-023) :

- visiteur et inscrit : `status = published` ;
- auteur : ses propres sujets, quel que soit leur statut ;
- admin (`subjects.moderate`) : tous les sujets.

Un sujet hors périmètre répond 404, jamais 403, pour ne pas révéler son existence.

### questions

| Colonne | Type | Règles |
|---|---|---|
| id | bigint PK | identifiant stable, point d'attache de la progression (FR-021) |
| subject_id | FK subjects | obligatoire |
| recto_html | text | assaini (research R4), 1 à 5 000 caractères de texte visible |
| verso_html | text | idem |
| position | integer | ordre dans le sujet, unique par sujet |
| created_at, updated_at | timestamps | |

- 500 questions au plus par sujet.
- La dernière question d'un sujet publié ne peut pas être supprimée (FR-018).

## Couche `functional/moderation`

### reports

| Colonne | Type | Règles |
|---|---|---|
| id | bigint PK | |
| subject_id | FK subjects | sujet publié, dont le signaleur n'est pas l'auteur (FR-027) |
| reporter_id | FK users | |
| reason | varchar(24) | `inappropriate`, `incorrect`, `spam`, `copyright`, `other` |
| comment | varchar(500) null | |
| status | varchar(8) | `pending`, `closed` |
| closed_at | timestamp null | |
| created_at | timestamp | |

- Index unique partiel `(subject_id, reporter_id) WHERE status = 'pending'` : un seul signalement
  en attente par personne et par sujet (FR-028).

### moderation_decisions

| Colonne | Type | Règles |
|---|---|---|
| id | bigint PK | |
| subject_id | bigint | conservé même si le sujet est supprimé ensuite (pas de FK) |
| subject_title | varchar(120) | titre au moment de la décision, pour l'historique |
| admin_id | FK users | |
| decision | varchar(16) | `ignored`, `retired`, `restored` |
| reason | text null | obligatoire pour `retired` |
| created_at | timestamp | |

- Lecture seule, jamais modifiée ni supprimée (FR-034).

## Couche `functional/learning`

### learnings

| Colonne | Type | Règles |
|---|---|---|
| id | bigint PK | |
| user_id | FK users | |
| subject_id | FK subjects | |
| created_at | timestamp | date de début d'apprentissage |

- Unique `(user_id, subject_id)`. Arrêter d'apprendre supprime la ligne et ses progressions
  (FR-050).

### card_progress

| Colonne | Type | Règles |
|---|---|---|
| id | bigint PK | |
| learning_id | FK learnings | |
| user_id | FK users | redondant avec `learning.user_id`, pour filtrer vite |
| question_id | FK questions | |
| subject_id | FK subjects | redondant, pour sélectionner par sujet |
| box | smallint | 1 à 5 (FR-042) |
| next_review_on | date | dans le fuseau de l'utilisateur (research R6) |
| last_answered_at | timestamp null | |

- Unique `(user_id, question_id)`. Index `(user_id, subject_id, next_review_on)` pour la séance.
- **Carte due** : `next_review_on <= aujourd'hui (fuseau de l'utilisateur)` et sujet `published`.

### review_answers

| Colonne | Type | Règles |
|---|---|---|
| id | bigint PK | |
| card_progress_id | FK card_progress | |
| user_id | FK users | |
| known | boolean | « je savais » ou « je ne savais pas » |
| from_box | smallint | 1 à 5 |
| to_box | smallint | 1 à 5 |
| answered_at | timestamp | |

- Sert au bilan de fin de séance (FR-049). Supprimées avec la progression de la carte.

**Règle de transition Leitner** (FR-046, testée exhaustivement, SC-010) :

| Boîte de départ | Je savais | Je ne savais pas |
|---|---|---|
| 1 | boîte 2, dans 2 jours | boîte 1, demain |
| 2 | boîte 3, dans 4 jours | boîte 1, demain |
| 3 | boîte 4, dans 8 jours | boîte 1, demain |
| 4 | boîte 5, dans 16 jours | boîte 1, demain |
| 5 | boîte 5, dans 16 jours | boîte 1, demain |

Une carte qui vient d'être apprise entre en boîte 1, due le jour même.
