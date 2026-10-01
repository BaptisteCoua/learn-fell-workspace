# Data Model: Import de questions

Décisions dans [research.md](research.md), contrat dans [contracts/api.md](contracts/api.md).

## Back : `questions` (couche `back/functional/catalog`)

| Colonne | Type | Règle |
|---|---|---|
| `import_id` | uuid, nul | **nouvelle**. UUID de l'import qui a créé la question, nul pour une question saisie à la main. Index `(subject_id, import_id)`. Jamais exposé : ajouté à `#[Hidden]`, absent des champs de `QuestionResource` (R6). |

Aucune autre table. La source et l'aperçu ne sont pas conservés (spec, Key Entities ; R5).

## Back : configuration `catalog.import` (`back/functional/catalog/config/catalog.php`)

| Clé | Valeur | Source |
|---|---|---|
| `max_kilobytes` | 5120 | FR-009 |
| `max_lines` | 2000 | FR-009 (lignes lues, en-tête et lignes vides compris) |
| `extensions` | `csv`, `tsv`, `txt`, `xlsx` | FR-002, R12 |

Les limites des questions restent celles de la 001 : `Subject::MAX_QUESTIONS` (500),
`VisibleTextLength::MAX` (5 000), `QuestionResource::MAX_HTML_LENGTH` (20 000).

## Lecture : de la source aux lignes (en mémoire)

```text
source (fichier ou texte)
  → décodage (BOM retiré, UTF-8 ou Windows-1252)            R2
  → enregistrements (fgetcsv, séparateur détecté ; XLSX par OpenSpout, 1re feuille)   R1, R2
  → en-têtes Anki ignorés, lignes vides ignorées, espaces retirés, en-tête reconnu   FR-007, FR-008
  → cellules 1 et 2 → MarkdownCell → SanitizedHtml::clean()  R3, R4
  → lignes lues + validation + doublons
```

### Ligne lue (`ImportedRow`)

| Champ | Contenu |
|---|---|
| `line` | numéro de l'enregistrement dans la source, en partant de 1. Une cellule multiligne compte pour la ligne où elle commence. |
| `recto_html`, `verso_html` | HTML nettoyé, exactement celui qui sera enregistré (FR-027) |
| `errors` | liste de `{ field: recto\|verso, code }`. Elles bloquent l'import. |
| `warnings` | liste de `{ code, line?, position? }`. Elles ne bloquent pas. |

### Erreurs d'une ligne (bloquantes)

| Code | Quand |
|---|---|
| `recto_empty`, `verso_empty` | 0 caractère visible après conversion (FR-011) |
| `recto_too_long`, `verso_too_long` | plus de 5 000 caractères visibles, ou plus de 20 000 caractères de HTML (FR-011) |

### Avertissements d'une ligne (non bloquants)

| Code | Quand | Précise |
|---|---|---|
| `duplicate_in_subject` | recto normalisé égal à celui d'une question du sujet (FR-026) | `position` de cette question |
| `duplicate_in_source` | recto normalisé égal à celui d'une ligne précédente de la source | `line` de cette ligne |

**Normalisation du recto** pour les doublons :
1. retirer les balises et décoder les entités ;
2. passer en minuscules et retirer les accents (`Str::ascii`) ;
3. réduire les espaces à un seul et retirer ceux des bords.

### Notices de la source (non bloquantes)

| Code | Quand |
|---|---|
| `extra_columns_ignored` | au moins une ligne a plus de deux colonnes (FR-003), signalé une seule fois |
| `first_sheet_only` | le XLSX a plusieurs feuilles (FR-006) |
| `header_ignored` | une ligne d'en-tête a été reconnue (FR-007) |

### Erreurs de la source (bloquantes, l'aperçu n'a pas de lignes)

| Code | HTTP | Quand |
|---|---|---|
| `import_unreadable` | 422 | fichier qui n'est ni un CSV ni un XLSX lisible (FR-009, US1-8) |
| `import_too_large` | 422 | plus de 5 Mo ou plus de 2 000 lignes (FR-009) |
| `import_single_column` | 422 | texte collé sans tabulation, ou source dont aucune ligne n'a deux colonnes (US3-4) |

### Erreurs de l'ensemble (bloquantes, l'aperçu garde ses lignes)

| Code | Quand |
|---|---|
| `import_empty` | aucune question après les lignes vides et l'en-tête (US2-7, FR-012) |
| `question_limit_exceeded` | questions du sujet + lignes lues > 500 ; précise `remaining` (FR-011, US2-3) |

## Règles de la confirmation (`ImportQuestions`)

1. Tout se passe dans une transaction, avec le sujet verrouillé (`lockForUpdate`).
2. Si des questions du sujet portent déjà cet `import_id` : réponse `200` avec leur nombre, rien
   n'est ajouté (FR-017).
3. Les droits sont vérifiés par `Gate::authorize('update')` et
   `RetiredSubjectLock::ensureEditable()` (FR-001).
4. La source est relue et revalidée (FR-014). S'il reste une erreur, de ligne, de source ou
   d'ensemble : `422 import_has_errors`, rien n'est ajouté.
5. Les positions vont de `max(position) + 1` à `max(position) + n`, dans l'ordre de la source
   (FR-015).
6. Chaque `Question::create` déclenche `QuestionCreated`, donc la boîte 1 pour chaque apprenant
   (FR-020, R7).

## Web : état de l'écran d'import (`useQuestionImport`)

| État | Contenu |
|---|---|
| `source` | `{ kind: 'text', text }` ou `{ kind: 'file', file }` |
| `importId` | UUID, renouvelé à chaque aperçu |
| `preview` | réponse de l'aperçu, ou `null` |
| `hasLearners` | réponse de `learning/subjects/{id}/learners` |
| `phase` | `source` → `preview` → `importing` → retour à l'éditeur |
