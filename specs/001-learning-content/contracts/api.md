# Contrat d'API : Comptes, contenu et révision Leitner (001)

Le back (`back/`) fournit l'API ; le web (`web/`) la consomme. Base : `https://api.<domaine>/api`
(en développement `http://localhost:8090/api`). Toutes les réponses sont en JSON. Les messages
d'erreur destinés à l'utilisateur viennent des fichiers de traduction français du back.

## 1. Authentification (Sanctum SPA + Fortify)

Hors lomkit. Le web appelle d'abord `GET /sanctum/csrf-cookie`, puis envoie le cookie de session
et l'en-tête `X-XSRF-TOKEN` à chaque requête qui modifie quelque chose.

| Méthode et chemin | Corps | Réponse | Exigences |
|---|---|---|---|
| `POST /register` | `display_name`, `email`, `password`, `password_confirmation`, `timezone` | 201 ; compte inactif, email de confirmation envoyé ; aucune session ouverte ; 422 `email_taken` (compte actif) ou `email_pending_verification` (compte en attente) | FR-001, FR-002, scénario 6 |
| `POST /email/verification-notification` | `email` | 202, même réponse que l'adresse existe ou non | FR-002, FR-006 |
| `GET /email/verify/{id}/{hash}?expires&nonce&signature` (lien signé, 24 h) | — | 204 et session ouverte ; 403 `link_expired` si expiré, déjà utilisé ou remplacé par un lien plus récent | FR-002 |
| `POST /login` | `email`, `password`, `timezone` | 204 et session de 30 jours ; 422 `invalid_credentials` (message générique) ; 422 `email_not_verified` (seulement si le mot de passe est correct) ; 429 `locked` avec `retry_after` après 5 échecs, verrou de 15 min | FR-003, FR-004, FR-006 |
| `POST /logout` | — | 204 | FR-003 |
| `POST /forgot-password` | `email` | 202, même réponse que l'adresse existe ou non ; lien valable 60 min | FR-005, FR-006 |
| `POST /reset-password` | `token`, `email`, `password`, `password_confirmation` | 204 et fermeture des autres sessions ; 422 `link_expired` | FR-005 |
| `GET /user` | — | l'utilisateur connecté : `id`, `display_name`, `email`, `permissions[]` ; 401 si déconnecté | — |

## 2. Ressources lomkit

Chaque ressource expose les routes lomkit standard : `POST /<ressource>/search`,
`POST /<ressource>/mutate`, `DELETE /<ressource>`, `GET /<ressource>/details`, et
`POST /<ressource>/actions/<action>`. Les périmètres d'accès sont décrits dans
[data-model.md](../data-model.md). Un élément hors périmètre est absent des résultats et répond 404
en accès direct.

### categories

- **Champs** : `id`, `name`, `position`. **Relations** : `subjects` (comptage `subjects_count`).
- **Lecture** : tout le monde. **Écriture** : permission `categories.manage`.
- **Validation** : nom obligatoire, unique après normalisation (erreur `category_name_taken`, FR-009).
- **Suppression** : refusée si des sujets y sont rangés, 422 `category_not_empty` avec leur nombre
  (FR-010).
- **Action `reorder`** : champ `ids[]` dans le nouvel ordre (FR-008).

### tags

- **Champs** : `id`, `name`. Lecture seule, pour les suggestions pendant la saisie (FR-011) ; filtre
  `name like`.

### subjects

- **Champs** : `id`, `title`, `description`, `status`, `published_at`, `retired_reason`,
  `retired_at`, `created_at`, `updated_at`. **Relations** : `author` (champs publics `id`,
  `display_name`), `category`, `tags`, `questions` (comptage `questions_count`).
- **Instruction `search`** : champ `q` (2 caractères minimum), recherche sur le titre, la
  description et les tags, sans tenir compte de la casse ni des accents (FR-025).
- **Tri** : `published_at desc` par défaut dans le catalogue (FR-024). Pagination par 20.
- **Création** (`mutate`, opération `create`) : inscrit à l'email confirmé ; statut `draft`
  (FR-012, FR-013). Les tags passent par l'action `sync-tags` (ci-dessous).
- **Modification** : auteur, ou `subjects.moderate` ; interdite à l'auteur sur un sujet `retired`.
- **Suppression** : auteur, confirmation côté web ; supprime questions, apprentissages et
  progressions, clôt les signalements en attente (FR-019).
- **Actions** :

Chaque action vise le sujet par `search.filters` (`id`), comme toute action lomkit.

| Action | Champs | Qui | Effet | Erreurs |
|---|---|---|---|---|
| `publish` | — | auteur | `draft` → `published` (FR-016) | 422 `subject_has_no_question`, 422 `subject_retired` |
| `unpublish` | — | auteur | `published` → `draft` (FR-017) | |
| `sync-tags` | `names[]` | auteur | remplace les tags par ces noms : normalisés, doublons ignorés, créés au besoin, 10 au plus de 30 caractères (FR-011) | 422 `subject_retired` |

Retirer et rétablir passent par `moderation-decisions` ; apprendre et arrêter d'apprendre passent
par `learnings`. Ainsi, aucune couche n'ajoute d'action sur la ressource d'une autre couche.

### questions

- **Champs** : `id`, `subject_id`, `recto_html`, `verso_html`, `position`.
- **Lecture** : selon le périmètre du sujet. **Écriture** : auteur du sujet, ou `subjects.moderate` ;
  jamais sur un sujet `retired` pour l'auteur.
- **Validation** : recto et verso de 1 à 5 000 caractères de texte visible, HTML assaini à
  l'enregistrement (FR-014, FR-015) ; 500 questions au plus par sujet.
- **Suppression** : refusée pour la dernière question d'un sujet `published`, 422
  `last_question_of_published_subject` (FR-018).
- **Création** : ajoutée en dernière position ; `position` est fixée par l'API. 422 `question_limit_reached` au-delà de 500.
- **Action `reorder`** (autonome) : champs `subject_id`, `ids[]` — exactement les questions du sujet, dans le nouvel ordre (FR-014).

### reports

- **Champs** : `id`, `subject_id`, `reason`, `comment`, `status`, `created_at`. **Relations** :
  `subject`, `reporter` (`display_name`).
- **Création** : inscrit, sur un sujet publié dont il n'est pas l'auteur (FR-027) ; 409
  `report_already_pending` (FR-028).
- **Lecture** : `reports.review` ; filtre `status = pending`, groupé par sujet côté web, tri du plus
  ancien au plus récent (FR-030).
- Ignorer les signalements d'un sujet passe par `moderation-decisions` (décision `ignored`).

### moderation-decisions

- **Champs** : `id`, `subject_id`, `subject_title`, `decision`, `reason`, `created_at`.
  **Relations** : `admin` (`display_name`).
- **Création** (permission `subjects.moderate`) : c'est l'acte de modération lui-même.

| `decision` | Champs | Effet | Erreurs |
|---|---|---|---|
| `ignored` | `subject_id` | clôt les signalements en attente du sujet, qui reste publié (FR-031) | |
| `retired` | `subject_id`, `reason` | sujet `retired` avec son motif, signalements clos (FR-031, FR-032) | 422 `reason_required` |
| `restored` | `subject_id` | sujet `retired` → `draft` (FR-032) | |

- **Lecture** : permission `moderation.history.view` ; filtre par `decision` ; jamais modifiée ni
  supprimée (FR-034).

### learnings

- **Champs** : `id`, `subject_id`, `created_at`. **Relations** : `subject`. Comptages calculés :
  `due_today_count`, `box_1_count` à `box_5_count`, `next_review_on` (le plus proche).
- **Lecture** : l'utilisateur connecté, ses apprentissages uniquement (FR-043).
- **Création** (`mutate`, `subject_id`) : apprendre un sujet publié ; une progression en boîte 1,
  due aujourd'hui, par question (FR-041) ; 409 `already_learning`, 422 `subject_not_published`.
- **Suppression** : arrêter d'apprendre ; supprime les progressions et les réponses (FR-050).

### card-progress

- **Champs** : `id`, `subject_id`, `question_id`, `box`, `next_review_on`, `last_answered_at`.
  **Relations** : `question` (recto et verso), `subject` (titre).
- **Instruction `due`** : champ `subject_ids[]` ; ne garde que les cartes dues aujourd'hui dans le
  fuseau de l'utilisateur, de sujets publiés, triées de la plus en retard à la plus récente
  (FR-044).
- **Action `answer`** (autonome) : champs `card_progress_id`, `known` (booléen). Applique la règle
  Leitner (FR-046) et enregistre la réponse (FR-048). Comme toute action lomkit, elle renvoie le
  nombre d'éléments touchés : le web relit la carte (`box`, `next_review_on`) pour afficher
  `from_box`, `to_box` et la prochaine date (FR-047). 409 `card_not_due` si la carte n'est pas due, ce qui couvre le double clic.
- **Bilan de séance** : le web l'établit à partir des réponses de la séance et d'une recherche
  `learnings` sur les sujets sélectionnés (FR-049).

## 3. Codes d'erreur métier

Chaque refus métier renvoie `{ "code": "<code>", "message": "<texte français>" }`. Le web affiche
`message` et peut réagir à `code`.

| Code | HTTP | Exigence |
|---|---|---|
| `invalid_credentials` | 422 | FR-006 |
| `email_taken` | 422 | FR-001, scénario 6 |
| `email_pending_verification` | 422 | FR-002, scénario 6 |
| `email_not_verified` | 422 | FR-002 |
| `locked` | 429 | FR-004 |
| `link_expired` | 403 / 422 | FR-002, FR-005 |
| `category_name_taken` | 422 | FR-009 |
| `category_not_empty` | 422 | FR-010 |
| `subject_has_no_question` | 422 | FR-016 |
| `subject_retired` | 422 | FR-033 |
| `question_limit_reached` | 422 | FR-014 |
| `last_question_of_published_subject` | 422 | FR-018 |
| `reason_required` | 422 | FR-031 |
| `report_already_pending` | 409 | FR-028 |
| `subject_not_published` | 422 | FR-041 |
| `already_learning` | 409 | FR-041 |
| `card_not_due` | 409 | FR-046 |
