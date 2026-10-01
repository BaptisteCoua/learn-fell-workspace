# Contrat d'API : Import de questions (008)

Complète le contrat de la 001 ([specs/001-learning-content/contracts/api.md](../../001-learning-content/contracts/api.md)).
Ce sont des routes hors lomkit, parce qu'elles transportent un fichier et un flux binaire (research R9).
Les messages sont en français, tirés des traductions du back.

## Source : deux formes acceptées

Les deux `POST` ci-dessous reçoivent la source sous l'une de ces formes, jamais les deux :

| Forme | Corps | Règle |
|---|---|---|
| Fichier | `multipart/form-data`, champ `file` | requis ; extensions `csv`, `tsv`, `txt`, `xlsx` ; contenu reconnu (`mimes:csv,txt,xlsx`) ; 5 120 Ko au plus |
| Texte collé | JSON `{ "text": "…" }` | chaîne non vide ; 5 Mo au plus ; lue avec la tabulation pour séparateur (FR-005) |

Un corps qui porte les deux formes, ou aucune, reçoit `422` avec une erreur sur `file`.

## `POST /api/subjects/{subject}/question-import/preview` (nouvelle)

Middleware `auth:sanctum`, `verified`, `throttle:30,1`. Rien n'est enregistré.

Accès (FR-001) : `Gate::authorize('update', $subject)` et `RetiredSubjectLock::ensureEditable()`.
- Un sujet invisible pour la personne donne `404`.
- Un sujet retiré, pour son auteur, donne `422 subject_retired`.

**`200`** :

```json
{
  "data": {
    "rows": [
      {
        "line": 2,
        "recto_html": "Le passé de <em>go</em>",
        "verso_html": "<strong>went</strong>",
        "errors": [],
        "warnings": []
      },
      {
        "line": 12,
        "recto_html": "Capitale du Pérou",
        "verso_html": "",
        "errors": [{ "field": "verso", "code": "verso_empty", "message": "Le verso est vide." }],
        "warnings": [{ "code": "duplicate_in_subject", "position": 4, "message": "Ce recto existe déjà dans le sujet (question 4)." }]
      }
    ],
    "notices": [{ "code": "header_ignored", "message": "La première ligne est un en-tête : elle n'est pas importée." }],
    "errors": [{ "code": "question_limit_exceeded", "remaining": 50, "message": "Ce sujet ne peut plus recevoir que 50 questions, sur les 500 autorisées." }],
    "question_count": 80,
    "error_line_count": 1,
    "can_confirm": false
  }
}
```

- `rows` : dans l'ordre de la source. Les codes des erreurs et des avertissements sont dans
  [data-model.md](../data-model.md).
- `errors` : erreurs d'ensemble (`import_empty`, `question_limit_exceeded`).
- `can_confirm` : vrai seulement si aucune ligne n'a d'erreur, si `errors` est vide et si
  `question_count` est d'au moins 1 (FR-012).

**`422`** :
- `{ "code": "import_unreadable" | "import_too_large" | "import_single_column", "message": "…" }` ;
- erreurs de champ de la forme Laravel (`errors.file`, `errors.text`).

## `POST /api/subjects/{subject}/question-import` (nouvelle)

Middleware `auth:sanctum`, `verified`, `throttle:10,1`. Même source que l'aperçu, plus :

| Champ | Règle |
|---|---|
| `import_id` | requis, UUID ; généré par le web à chaque aperçu (R6) |

Pour un fichier, `import_id` est un champ du `multipart`. Pour un texte, c'est une clé du JSON.

- **`201 { "data": { "imported": 80 } }`** : toutes les questions ont été ajoutées à la fin du
  sujet (FR-014, FR-015, FR-018).
- **`200 { "data": { "imported": 80 } }`** : cet `import_id` a déjà été importé dans ce sujet.
  Rien n'est ajouté (FR-017).
- **`422 { "code": "import_has_errors", "message": "…" }`** : la relecture trouve une erreur, de
  ligne, de source ou d'ensemble, par exemple une limite atteinte entre-temps. Rien n'est ajouté.
- **`422`** `import_unreadable`, `import_too_large`, `import_single_column`, `subject_retired`,
  `subject_withheld` ou `subject_authorless`, comme pour l'aperçu.
- **`403` ou `404`** : comme l'aperçu.

## `GET /api/question-import/template` (nouvelle)

Public, `throttle:30,1`. Ouvert par un lien de navigation du web (R10), jamais par `fetch`.

**`200`**, `Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`,
`Content-Disposition: attachment; filename="modele-questions.xlsx"`. Une feuille :

| Recto | Verso |
|---|---|
| Quelle est la capitale du Pérou ? | Lima |
| Le passé de \*\*go\*\* ? | \*\*went\*\*, comme dans : \n- I went home |

Tous les textes viennent de `back/technical/osdd/lang/fr/import.php`.

## `GET /api/learning/subjects/{subject}/learners` (nouvelle, couche learning)

Middleware `auth:sanctum`. Accès : `Gate::authorize('update', $subject)`. Sinon `403`, ou `404`
si le sujet est invisible.

**`200 { "has_learners": true }`** : au moins un inscrit, auteur compris, apprend ce sujet
(FR-019, R8). Ni nombre ni identité.

## Inchangé

- `POST /api/questions/mutate` : `import_id` n'est ni accepté ni renvoyé.
- `QuestionCreated` → `AddQuestionToLearners` : déclenché par chaque question importée (R7).
