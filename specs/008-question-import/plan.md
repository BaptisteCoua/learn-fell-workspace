# Implementation Plan: Import de questions

**Branch**: `008-question-import` | **Date**: 2026-10-01 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/008-question-import/spec.md`

**Maquette** : aucune. L'écran d'import reprend les styles de l'éditeur de sujet : boutons, notices
`AccountNotice`, cartes de question et `RichTextView`.

## Summary

Permettre à l'auteur d'un sujet, et à un administrateur, d'ajouter d'un coup des questions à ce
sujet. La source est un texte collé depuis un tableur, Anki ou Quizlet, ou un fichier CSV ou XLSX.
L'auteur voit d'abord un aperçu des questions rendues, avec les erreurs et les avertissements ligne
par ligne. L'import se fait ensuite en tout ou rien.

**Approche technique** :

- **Back** (`back/functional/catalog`) :
  - un lecteur de source détecte l'encodage et le séparateur pour le CSV et le texte collé (sur
    `fgetcsv`), et lit le XLSX avec `openspout/openspout` ;
  - `MarkdownCell` convertit chaque cellule avec `league/commonmark`, dans un environnement
    restreint à la mise en forme autorisée, puis le nettoyage de l'éditeur s'applique ;
  - l'aperçu ne garde rien. La confirmation relit la source, verrouille le sujet et crée les
    questions une à une dans une transaction, avec un `import_id` pour ne jamais importer deux fois ;
  - chaque question créée déclenche le chemin existant vers la boîte 1 des apprenants ;
  - le modèle XLSX est produit par le back, à partir des traductions.
- **Back** (`back/functional/learning`) : une route `has_learners` pour l'avertissement.
- **Web** (`web/functional/Authoring`) :
  - une page `/sujets/[id]/importer`, en deux temps : la source (coller ou fichier), puis l'aperçu
    et la confirmation ;
  - `useUploadRequest` accepte des champs en plus du fichier.

Les décisions et leurs alternatives sont dans [research.md](research.md).

## Affected Repos

Le back est le fournisseur et se fusionne en premier ; le web suit (constitution 2.0.0, flux de
travail).

<!-- speckit-multirepo:begin -->
| Repo | Rôle | Ordre de fusion |
|---|---|---|
| back | API Laravel : lecture CSV/XLSX/texte, Markdown restreint, aperçu, import atomique et idempotent, modèle XLSX, `has_learners` | 1 |
| web | PWA Nuxt : page d'import (coller, fichier, aperçu, confirmation), bouton sur l'éditeur, champs en plus dans `useUploadRequest` | 2 |
<!-- speckit-multirepo:end -->

## Technical Context

**Language/Version**: PHP 8.5 (Laravel 13) côté `back/` ; TypeScript 6 (Nuxt 4, Vue 3) côté `web/`

**Primary Dependencies**:
- `openspout/openspout` 4.x : **nouvelle dépendance du back**, à valider par le développeur avec
  ce plan (`back/AGENTS.md`).
- `league/commonmark` 2.x : déjà installé par `laravel/framework`, ajouté en dépendance directe de
  la couche catalog puisqu'on s'en sert directement.
- `stevebauman/purify` : inchangé.
- Aucune dépendance nouvelle côté web.

**Storage**: PostgreSQL, colonne `questions.import_id` (uuid, nul, indexée avec `subject_id`).
Aucune table nouvelle. La source n'est jamais conservée.

**Testing**: PHPUnit 12 par `./vendor/bin/sail artisan test` (`back/`), avec des fixtures CSV et
XLSX qui reproduisent les sorties d'Excel en français, de LibreOffice, de Google Sheets, d'Anki et de
Quizlet ; Vitest et `@nuxt/test-utils` (`web/`) ; Pint, ESLint et Prettier.

**Target Platform**: API Linux sous Docker ; navigateurs récents, PWA installée sur Android et iOS.

**Project Type**: application web, soit une API et une PWA, dans deux repos distincts

**Performance Goals**:
- aperçu de 500 lignes en moins de 3 s (SC-004) ;
- 100 questions importées en moins de 2 min de bout en bout (SC-001).

**Constraints**:
- tout ou rien (SC-003) ;
- une seule fois par confirmation (FR-017) ;
- aperçu identique au rendu enregistré (FR-027) ;
- rien d'exécutable (SC-006) ;
- aucun symbole ordinaire pris pour une mise en forme (SC-007) ;
- 360 px sans défilement horizontal (FR-023) ;
- import indisponible hors ligne.

**Scale/Scope**:
- 4 user stories et 27 exigences fonctionnelles ;
- côté back, 4 routes nouvelles (3 dans catalog, 1 dans learning) et 1 colonne ;
- côté web, 1 page nouvelle, 3 composants, 1 composable, 2 composables modifiés et l'éditeur
  modifié.

Aucune inconnue ne reste ouverte. Le vrai comportement des exports de chaque tableur est à vérifier
avec de vrais fichiers ([quickstart.md](quickstart.md), parcours 1 à 3).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principe | Vérification | Avant conception | Après conception |
|---|---|---|---|
| I. La spec d'abord | Spec validée, deux clarifications du développeur intégrées, poussée ; chemins préfixés | ✅ | ✅ |
| II. Couches OSDD | L'import vit dans `catalog` (back) et `Authoring` (web). « Sujet appris ? » est servi par `learning`, qui seule voit sujets et apprenants, sans cycle catalog → learning (R8, comme la 004) | ✅ | ✅ R8 |
| III. Paquets imposés, API par contrat | Les questions restent créées par le modèle `Question`, avec le même cast et les mêmes événements. Les routes hors lomkit sont justifiées : fichier, aperçu calculé, flux binaire (R9). Côté web : `useUploadRequest` et `useApiFetch` ; le modèle est un lien de navigation (R10). Le back est fusionné avant le web. Accès par `Gate` et `subjects.moderate`, jamais par un nom de rôle | ✅ | ✅ R9, R10, R12 |
| IV. Tests obligatoires | Un test par exigence dans [quickstart.md](quickstart.md). Tout ou rien testé sur chaque cause d'échec (SC-003). Jeu d'essai Markdown (SC-007). Règle Leitner de la question ajoutée couverte par le chemin existant, plus un test d'import sur un sujet appris | ✅ | ✅ |
| V. Accessibilité, mobile d'abord | Aperçu en cartes, sans tableau, à 360 px. Erreurs en `role="alert"` avec lien vers chaque ligne. Bouton désactivé hors ligne avec explication. Tous les textes dans les traductions, modèle XLSX compris (R10), en vouvoyant | ✅ | ✅ R10, R11 |
| VI. Sécurité, données personnelles | Le Markdown est rendu sans HTML brut (`html_input: escape`), puis nettoyé par le même Purify que l'éditeur. Les liens dangereux sont refusés. Le contenu du fichier est contrôlé, pas son nom. Les droits sont revérifiés à la confirmation. `has_learners` n'expose qu'un booléen. La source n'est pas conservée | ✅ | ✅ R3, R4, R8 |
| VII. Simplicité | Aucune table : une colonne pour l'idempotence. Aperçu sans état. Chemin Leitner existant réutilisé. `useFilePicker` justifié par un second usage réel. La dépendance OpenSpout est justifiée face à une lecture XLSX écrite à la main (R1) | ✅ | ✅ R1, R5, R6, R7 |

Aucun écart. La nouvelle dépendance n'en est pas un (VII l'admet quand elle est justifiée), mais
`back/AGENTS.md` demande l'accord du développeur : il est demandé avec ce plan.

## Project Structure

### Documentation (this feature)

```text
specs/008-question-import/
├── spec.md
├── plan.md              # ce fichier
├── research.md          # décisions techniques (Phase 0)
├── data-model.md        # colonne import_id, lignes lues, codes (Phase 1)
├── quickstart.md        # guide de validation (Phase 1)
├── contracts/
│   └── api.md           # aperçu, import, modèle, has_learners (Phase 1)
├── checklists/
│   └── requirements.md
└── tasks.md             # Phase 2, créé par /speckit-tasks
```

### Source Code (repository root)

```text
back/
├── composer.json, composer.lock                         # + openspout/openspout, league/commonmark
├── technical/osdd/lang/fr/
│   ├── errors.php                                       # + import_*, question_limit_exceeded
│   └── import.php                                       # + lignes, notices, avertissements, modèle
├── functional/catalog/
│   ├── composer.json                                    # + openspout/openspout, league/commonmark
│   ├── config/catalog.php                               # + import.max_kilobytes, max_lines, extensions
│   ├── database/migrations/                             # + questions.import_id, index (subject_id, import_id)
│   ├── routes/api.php                                   # + preview, import, template
│   ├── src/
│   │   ├── Casts/SanitizedHtml.php                      # clean() statique, partagé (R4)
│   │   ├── Models/Question.php                          # import_id fillable et caché
│   │   ├── Import/SourceDecoder.php                     # + BOM, UTF-8 ou Windows-1252, séparateur (R2)
│   │   ├── Import/SourceReader.php                      # + texte, CSV (fgetcsv), XLSX (OpenSpout) → enregistrements (R1)
│   │   ├── Import/MarkdownCell.php                      # + environnement CommonMark restreint (R3)
│   │   ├── Import/ImportedRow.php                       # + ligne lue
│   │   ├── Import/QuestionImportPreview.php             # + lignes, erreurs, doublons, limite, can_confirm
│   │   ├── Import/TemplateWorkbook.php                  # + modèle XLSX (R10)
│   │   ├── Actions/ImportQuestions.php                  # + transaction, verrou, idempotence, création (R6)
│   │   ├── Http/Requests/QuestionImportRequest.php      # + file xor text, import_id
│   │   └── Http/Controllers/{PreviewQuestionImport,StoreQuestionImport,QuestionImportTemplate}Controller.php
│   └── tests/
│       ├── fixtures/import/                             # + excel-fr.csv, libreoffice.csv, google-sheets.csv, *.xlsx, anki.txt, quizlet.txt, corrupt.xlsx
│       ├── Unit/{SourceReader,MarkdownCell}Test.php
│       └── Feature/{QuestionImportPreview,QuestionImport,QuestionImportTemplate}Test.php
└── functional/learning/
    ├── routes/api.php                                   # + learning/subjects/{subject}/learners
    ├── src/Http/Controllers/SubjectLearnersController.php
    └── tests/Feature/{SubjectLearners,ImportedQuestionsReachLearners}Test.php

web/
├── technical/ApiClient/
│   ├── app/composables/useUploadRequest.ts              # upload(path, file, fields?)
│   └── tests/uploadRequest.nuxt.spec.ts
└── functional/Authoring/
    ├── app/pages/sujets/[id]/importer.vue               # + page d'import
    ├── app/pages/sujets/[id]/modifier.vue               # + bouton « Importer des questions »
    ├── app/components/QuestionImportSource.vue          # + onglets Coller / Fichier, lien du modèle
    ├── app/components/QuestionImportPreview.vue         # + résumé, notices, avertissement d'apprentissage
    ├── app/components/QuestionImportRow.vue             # + une ligne : recto, verso, erreurs, avertissements
    ├── app/composables/useQuestionImport.ts             # + source, import_id, aperçu, confirmation, has_learners
    ├── app/composables/useFilePicker.ts                 # useImagePicker généralisé (accept en paramètre)
    ├── app/components/QuestionImagesField.vue           # useFilePicker
    ├── i18n/locales/fr.json
    └── tests/{QuestionImportPage,SubjectEditor}.nuxt.spec.ts
```

**Structure Decision**: l'import est une façon d'écrire des questions. Il vit dans la couche
`catalog` côté back, qui possède `Question`, son nettoyage et ses limites, et dans la couche
`Authoring` côté web, qui possède l'éditeur. Seule la question « ce sujet est-il appris ? » vit
dans `learning`. Aucune couche nouvelle.
