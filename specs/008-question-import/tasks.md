---

description: "Tâches de la feature 008 — Import de questions (CINQ)"
---

# Tasks: Import de questions

**Input**: Documents de conception dans `specs/008-question-import/`

**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md),
[data-model.md](data-model.md), [contracts/api.md](contracts/api.md), [quickstart.md](quickstart.md)

**Tests**: obligatoires (constitution, principe IV). Dans chaque story, les tests sont écrits en
premier et doivent échouer avant l'implémentation. La table des cas par exigence est dans
[quickstart.md](quickstart.md).

**Organization**: une phase par user story ; dans chaque phase, le back avant le web.

## Format: `[ID] [P?] [Story] Description`

- **[P]** : parallélisable (fichiers différents, aucune dépendance sur une tâche non terminée)
- **[Story]** : story concernée (US1 à US4, voir spec.md)
- Tous les chemins commencent par leur repo : `back/…` ou `web/…`

## Conventions de chemins

- **Back** : couche `back/functional/catalog/`, avec :
  - `src/`, espace de noms `Functional\Catalog\` ;
  - `database/migrations/`, `config/catalog.php`, `routes/api.php` ;
  - `tests/{Unit,Feature,fixtures}` et le helper `tests/Concerns/WritesCatalog.php`.

  La couche `back/functional/learning/` sert à `has_learners`. Les codes d'erreur vont dans
  `back/technical/osdd/lang/fr/errors.php`, les autres textes dans
  `back/technical/osdd/lang/fr/import.php`. Fichiers générés par les commandes `osdd:*` avec
  `--layer=functional/<couche>`. Commandes via `./vendor/bin/sail`. Lire `back/CLAUDE.md` et
  `back/AGENTS.md` avant la première tâche.
- **Web** : couche `web/functional/Authoring/`, avec :
  - `app/{pages,components,composables}`, `i18n/locales/fr.json` ;
  - `tests/`, dont le support d'API `tests/support/authoringApi.ts` et le double d'envoi
    `tests/support/uploadDouble.ts`.

  `web/technical/ApiClient/` sert au transport. Commandes via `pnpm`. Lire `web/CLAUDE.md` avant la
  première tâche.
- Clés de traduction web : phrases anglaises en minuscules, triées, valeurs françaises vouvoyées.
- Les branches `008-question-import` de `back/` et `web/` sont créées par le hook
  `speckit.multirepo.branch` avant la première tâche. Le travail non commité des deux repos (`back/` :
  `Dockerfile`, `.dockerignore`, `docker-entrypoint.sh`, `railway.json` ; `web/` : `package.json`,
  `technical/ApiClient/nuxt.config.ts`) appartient au développeur. Il n'est ni commité ni modifié
  par ces tâches.

---

## Phase 1: Setup

- [X] T001 Dans `back/`, ajouter `openspout/openspout` et `league/commonmark` en dépendances
  directes, en version stable la plus récente (`./vendor/bin/sail composer require
  openspout/openspout league/commonmark`, attendues en 4.x et 2.x, research R1 et R3). Les déclarer
  aussi dans `back/functional/catalog/composer.json`. Vérifier que seuls `back/composer.json`,
  `back/composer.lock` et `back/functional/catalog/composer.json` changent.

---

## Phase 2: Foundational (prérequis bloquants)

**Purpose**: la colonne `import_id`, le nettoyage partagé, la conversion Markdown, le décodage et
la lecture en enregistrements, la ligne lue, et le transport multipart à champs. Rien de visible ne
change encore.

### Back

- [X] T002 Créer une migration dans `back/functional/catalog/database/migrations/` qui ajoute à
  `questions` la colonne `import_id` « uuid, nul », sans valeur par défaut en base, et un index
  `(subject_id, import_id)`. Dans `back/functional/catalog/src/Models/Question.php`, ajouter
  `import_id` au `#[Fillable]` et au `#[Hidden]`. Vérifier que
  `back/functional/catalog/src/Rest/Resources/QuestionResource.php` ne l'expose pas dans ses
  `fields()` (data-model).
- [X] T003 [P] Dans `back/functional/catalog/src/Casts/SanitizedHtml.php`, extraire le corps de
  `set()` (`Purify::clean()` puis `<a rel="noopener nofollow ugc" href=`) en méthode statique
  publique `clean(?string $html): string`, que `set()` appelle. Le comportement ne change pas :
  `./vendor/bin/sail artisan test --compact functional/catalog/tests/Feature/QuestionContentTest.php`
  reste vert (research R4).
- [X] T004 [P] Dans `back/functional/catalog/config/catalog.php`, ajouter le bloc `import` :
  `max_kilobytes` = 5120, `max_lines` = 2000, `extensions` = `['csv', 'tsv', 'txt', 'xlsx']`
  (data-model).
- [X] T005 [P] Ajouter à `back/technical/osdd/lang/fr/errors.php` les codes `import_unreadable`,
  `import_too_large`, `import_single_column`, `import_empty`, `question_limit_exceeded` (avec
  `:remaining` et `:max`) et `import_has_errors`. Créer `back/technical/osdd/lang/fr/import.php`
  avec les messages de ligne (`recto_empty`, `verso_empty`, `recto_too_long`, `verso_too_long`,
  avec `:max`), d'avertissement (`duplicate_in_subject` avec `:position`, `duplicate_in_source`
  avec `:line`), de notice (`extra_columns_ignored`, `first_sheet_only`, `header_ignored`), les
  en-têtes reconnus et les textes du modèle. Tout est en français, en vouvoyant (contrat).
- [X] T006 [P] Test unitaire `back/functional/catalog/tests/Unit/MarkdownCellTest.php`, en échec.
  Il couvre FR-024, FR-025 et SC-007 :
  - sont convertis : `**gras**`, `*italique*`, `_italique_`, listes `-` et `1.`, code en ligne et
    bloc délimité par trois accents graves, `[texte](https://…)`, `<https://…>` ;
  - restent en texte tel quel : `# Titre`, `> citation`, `---`, `![img](x.png)`, `<b>gras</b>`,
    `<script>`, code indenté de 4 espaces ;
  - le lien `[x](javascript:alert(1))` n'est pas un lien actif ;
  - jeu d'essai sans aucune mise en forme : `2 * 3 * 4 = 24`, `5*3*2`, `nom_de_variable`,
    `prix : 5 € *`, `_début sans fin`, `**`, `a_b_c`, `C:\dossier\*` ;
  - `\*` donne `*` ;
  - un retour à la ligne simple donne `<br>`.

  Chaque sortie passe par `SanitizedHtml::clean()`.
- [X] T007 Créer `back/functional/catalog/src/Import/MarkdownCell.php` pour faire passer T006
  (research R3) :
  - un `Environment` de `league/commonmark` **sans** `CommonMarkCoreExtension`, qui n'enregistre
    que les paragraphes, les listes, le code délimité, l'emphase (`*`, `_`), le code en ligne, les
    liens, les liens automatiques, les échappements et les entités ;
  - les images rendues comme leur source littérale ;
  - `html_input: escape`, `allow_unsafe_links: false`, saut de ligne simple rendu en `<br>` ;
  - avant la conversion, tout `*` placé entre deux lettres ou chiffres
    (`(?<=[\p{L}\p{N}])\*(?=[\p{L}\p{N}])`) est échappé ;
  - une méthode `toHtml(string $cell): string` qui renvoie le résultat de
    `SanitizedHtml::clean()`.
- [X] T008 [P] Test unitaire `back/functional/catalog/tests/Unit/SourceDecoderTest.php`, en échec :
  - le BOM UTF-8 est retiré ;
  - l'UTF-8 valide est gardé ;
  - le Windows-1252 (« é », « œ », « € ») est converti en UTF-8 ;
  - un octet illisible donne `�` ;
  - les fins de ligne `\r\n`, `\r` et `\n` sont acceptées ;
  - séparateur détecté sur les 20 premières lignes non vides (`;` pour un CSV d'Excel FR, `,`
    pour Google Sheets, tabulation) ;
  - une phrase qui contient des virgules, dans un CSV à `;`, garde `;` ;
  - à égalité, l'ordre est tabulation, `;`, `,` (FR-004, research R2).
- [X] T009 Créer `back/functional/catalog/src/Import/SourceDecoder.php` pour faire passer T008.
  Il expose `decode(string $bytes): string` et `detectSeparator(string $text): ?string`, qui
  renvoie `null` si aucun candidat ne donne deux colonnes.
- [X] T010 [P] Test unitaire `back/functional/catalog/tests/Unit/SourceReaderTest.php`, partie
  « texte délimité », en échec. `records(string $text, string $separator)` lit avec `fgetcsv` sur
  `php://temp` et `escape: ""` :
  - guillemets doublés ;
  - cellule entre guillemets avec séparateur et retour à la ligne ;
  - numéro de ligne de la source de chaque enregistrement (une cellule multiligne compte pour sa
    première ligne) ;
  - lignes de tête Anki `#clé:valeur` ignorées ;
  - lignes vides ignorées ;
  - espaces des bords retirés ;
  - plus de deux colonnes signalées (FR-003, FR-005, FR-008).
- [X] T011 Créer `back/functional/catalog/src/Import/SourceReader.php` (partie texte délimité) et
  `back/functional/catalog/src/Import/ImportedRow.php`, pour faire passer T010.
  - Le lecteur renvoie des enregistrements `{ line, recto, verso }` et les notices
    `extra_columns_ignored` et `header_ignored`.
  - L'en-tête est reconnu sur la première ligne non vide : `recto`/`verso` ou
    `question`/`réponse`, sans tenir compte de la casse ni des accents (FR-007).
  - Au-delà de `catalog.import.max_lines` lignes, il lève `BusinessRuleException('import_too_large')`.
  - `ImportedRow` porte `line`, `recto_html`, `verso_html`, `errors` et `warnings`, et sait se
    sérialiser au format du contrat.

### Web

- [X] T012 [P] Test dans `web/technical/ApiClient/tests/uploadRequest.nuxt.spec.ts`, en échec :
  `upload(path, file, { import_id: '…' })` envoie `file` et `import_id` dans le `FormData`. Sans
  troisième argument, l'envoi est inchangé (research R12).
- [X] T013 Dans `web/technical/ApiClient/app/composables/useUploadRequest.ts`, ajouter le
  paramètre facultatif `fields?: Record<string, string>`, ajouté au `FormData` après `file`, pour
  faire passer T012.
- [X] T014 [P] Renommer `web/functional/Authoring/app/composables/useImagePicker.ts` en
  `web/functional/Authoring/app/composables/useFilePicker.ts`, avec `accept` reçu en paramètre.
  Mettre à jour `web/functional/Authoring/app/components/QuestionImagesField.vue`, qui passe
  `image/jpeg,image/png,image/webp`. Les tests existants des images restent verts.

**Checkpoint**: `./vendor/bin/sail artisan test --compact functional/catalog` et `pnpm test`
verts.

---

## Phase 3: User Story 1 - Importer des questions depuis un fichier (Priority: P1) 🎯 MVP

**Goal**: envoyer un fichier CSV ou XLSX, voir l'aperçu, confirmer, et retrouver les questions à
la fin du sujet.

**Independent Test**: un brouillon de 3 questions et un XLSX de 50 lignes plus un en-tête donnent
50 questions dans l'aperçu, puis 53 questions dans le sujet, les 50 nouvelles dans l'ordre du
fichier.

### Tests (écrits d'abord, en échec)

- [X] T015 [P] [US1] Créer les fixtures dans `back/functional/catalog/tests/fixtures/import/`.
  Chacune reproduit la sortie du tableur nommé :
  - `excel-fr.csv` : Windows-1252, `;`, CRLF, accents et « € » ;
  - `google-sheets.csv` : UTF-8 sans BOM, `,`, une cellule entre guillemets avec virgule et retour
    à la ligne ;
  - `libreoffice.csv` : UTF-8 avec BOM, `,` ;
  - `excel.xlsx`, `libreoffice.xlsx` et `google-sheets.xlsx`, avec en-tête « Recto | Verso » et
    50 lignes : chaînes partagées pour Excel, chaînes en ligne pour LibreOffice, produits par le
    writer d'OpenSpout ou un script de test documenté ;
  - `two-sheets.xlsx`, avec une formule et un nombre entier ;
  - `corrupt.xlsx` ;
  - `not-a-sheet.pdf`.
- [X] T016 [P] [US1] Test unitaire, partie fichiers, dans
  `back/functional/catalog/tests/Unit/SourceReaderTest.php`, en échec :
  - chaque CSV de T015 donne les mêmes 50 enregistrements, accents justes (SC-002) ;
  - chaque XLSX aussi ;
  - `two-sheets.xlsx` : première feuille seulement, notice `first_sheet_only`, formule lue par sa
    valeur, nombre entier sans `.0` (FR-006) ;
  - `corrupt.xlsx` lève `import_unreadable`.
- [X] T017 [P] [US1] Test `back/functional/catalog/tests/Feature/QuestionImportPreviewTest.php`,
  partie accès et fichier, en échec. Il couvre `POST /api/subjects/{subject}/question-import/preview` :
  - auteur → `200` avec `rows`, `notices`, `errors`, `question_count`, `error_line_count`,
    `can_confirm` (contrat) ;
  - administrateur sur le sujet d'un autre, même retiré → `200` ;
  - autre inscrit → `403` ou `404` ;
  - auteur d'un sujet retiré → `422 subject_retired` ;
  - visiteur → `401` ;
  - email non confirmé → `403` ;
  - PDF ou `corrupt.xlsx` → `422 import_unreadable` ;
  - fichier de 5 121 Ko ou de 2 001 lignes → `422 import_too_large` ;
  - ni `file` ni `text`, ou les deux → `422` sur `file` ;
  - après l'aperçu, le sujet est inchangé (FR-001, FR-002, FR-009, FR-010, FR-013).
- [X] T018 [P] [US1] Test `back/functional/catalog/tests/Feature/QuestionImportTest.php`, partie
  nominale, en échec. Il couvre `POST /api/subjects/{subject}/question-import` :
  - sujet de 3 questions + XLSX de 50 lignes → `201 { data: { imported: 50 } }`, positions 4 à 53
    dans l'ordre du fichier, les 3 premières intactes ;
  - chaque question porte l'`import_id` envoyé ;
  - même `import_id` renvoyé → `200`, toujours 53 questions ;
  - `import_id` absent ou non UUID → `422` ;
  - une question importée se modifie, se déplace (`questions/actions/reorder`) et se supprime par
    lomkit ;
  - `import_id` absent des réponses de `questions/search` ;
  - le `recto_html` de l'aperçu est identique à celui enregistré (FR-014 à FR-018, FR-027).
- [X] T019 [P] [US1] Test `back/functional/catalog/tests/Feature/QuestionImportTemplateTest.php`,
  en échec :
  - `GET /api/question-import/template` → `200`, type
    `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`, `attachment;
    filename="modele-questions.xlsx"` ;
  - le classeur, lu par OpenSpout, contient l'en-tête et 2 exemples des traductions ;
  - le modèle renvoyé à l'aperçu donne 2 questions, aucune erreur et la notice `header_ignored`
    (FR-021).

### Implémentation — back

- [X] T020 [US1] Compléter `back/functional/catalog/src/Import/SourceReader.php` avec :
  - `fromFile(UploadedFile $file)` : XLSX par le reader d'OpenSpout, première feuille, valeurs de
    cellule, entiers sans décimale, dates au format `d/m/Y`, exception d'OpenSpout traduite en
    `import_unreadable` ;
  - le CSV, le TSV et le TXT par `SourceDecoder`, puis `records()` ;
  - `import_single_column` si `detectSeparator()` renvoie `null`.

  Fait passer T016.
- [X] T021 [US1] Créer `back/functional/catalog/src/Import/QuestionImportPreview.php`. Il
  transforme les enregistrements en `ImportedRow` (recto et verso par `MarkdownCell::toHtml()`),
  rassemble les notices, et calcule `question_count`, `error_line_count` et `can_confirm` (vrai si
  aucune erreur et au moins une question). Les règles d'erreur viennent en US2.
- [X] T022 [US1] Créer `back/functional/catalog/src/Http/Requests/QuestionImportRequest.php` :
  - `file` : `required_without:text`, `prohibits:text`, `file`, `extensions:csv,tsv,txt,xlsx`,
    `mimes:csv,txt,xlsx`, `max:` + `catalog.import.max_kilobytes`, avec `bail` ;
  - `text` : `required_without:file`, `string`, `max:5242880` ;
  - `import_id` : `required`, `uuid`, seulement sur la route de confirmation.

  Les dépassements de taille répondent `import_too_large` (contrat).
- [X] T023 [US1] Créer `back/functional/catalog/src/Actions/ImportQuestions.php` (research R6) :
  1. `DB::transaction` ;
  2. `Subject::query()->lockForUpdate()->findOrFail()` ;
  3. si des questions du sujet portent déjà l'`import_id` : renvoyer leur nombre, avec un
     indicateur « déjà importé » ;
  4. `Gate::authorize('update', $subject)` et `RetiredSubjectLock::ensureEditable($subject)` ;
  5. relire la source par `SourceReader` et `QuestionImportPreview` ; si `can_confirm` est faux :
     `BusinessRuleException('import_has_errors')` ;
  6. `Question::create([...])` pour chaque ligne, aux positions `max(position) + 1…n`, avec
     `import_id`.

  `QuestionCreated` part pour chaque question (R7).
- [X] T024 [US1] Créer les contrôleurs invocables
  `back/functional/catalog/src/Http/Controllers/PreviewQuestionImportController.php`,
  `back/functional/catalog/src/Http/Controllers/StoreQuestionImportController.php` (`201` ou
  `200` selon l'indicateur de T023) et
  `back/functional/catalog/src/Http/Controllers/QuestionImportTemplateController.php`. Ce dernier
  s'appuie sur `back/functional/catalog/src/Import/TemplateWorkbook.php`, qui écrit le classeur
  avec le writer d'OpenSpout à partir de `lang/fr/import.php` (research R10).
- [X] T025 [US1] Dans `back/functional/catalog/routes/api.php`, déclarer les trois routes du
  contrat. Un commentaire indique que lomkit ne transporte ni un fichier ni un flux binaire.
  - `POST subjects/{subject}/question-import/preview` : `auth:sanctum`, `verified`, `throttle:30,1` ;
  - `POST subjects/{subject}/question-import` : `auth:sanctum`, `verified`, `throttle:10,1` ;
  - `GET question-import/template` : public, `throttle:30,1`.

  Faire passer T017, T018 et T019.

### Implémentation — web

- [X] T026 [P] [US1] Test `web/functional/Authoring/tests/QuestionImportPage.nuxt.spec.ts`, partie
  fichier, en échec (stubs `stubAuthoringApi` et `fakeUploadRequest`) :
  - `/sujets/{id}/importer` affiche l'onglet « Fichier » et le lien du modèle vers
    `/api/question-import/template` ;
  - un `.pdf` ou un fichier de plus de 5 Mo est refusé avant l'envoi, avec un message en
    `role="alert"` ;
  - un `.xlsx` est envoyé à `subjects/{id}/question-import/preview` ;
  - l'aperçu affiche « Ligne N », le recto et le verso rendus, et le nombre de questions ;
  - « Importer 50 questions » envoie le même fichier avec `import_id` à
    `subjects/{id}/question-import` ;
  - après `201`, retour à `/sujets/{id}/modifier` et toast « 50 questions ajoutées » ;
  - « Abandonner » revient à la source sans appel d'import ;
  - le bouton d'import est désactivé hors ligne.
- [X] T027 [P] [US1] Test dans `web/functional/Authoring/tests/SubjectEditor.nuxt.spec.ts`, en
  échec : le bouton « Importer des questions » mène à `/sujets/{id}/importer`, et il est caché quand
  `isReadOnly`.
- [X] T028 [US1] Créer `web/functional/Authoring/app/composables/useQuestionImport.ts` :
  - état `source`, `importId` (`crypto.randomUUID()` renouvelé à chaque aperçu), `preview` et
    `phase` (`source` → `preview` → `importing`) ;
  - `previewFile(file)` et `confirm()` par `useUploadRequest().upload(path, file, { import_id })` ;
  - contrôle avant l'envoi : extension `.csv`, `.tsv`, `.txt`, `.xlsx` et taille ≤ 5 Mo ;
  - erreurs par `useApiError().toApiError()` ;
  - après l'import : `navigateTo('/sujets/{id}/modifier')` et `useToast().notify()`.
- [X] T029 [P] [US1] Créer `web/functional/Authoring/app/components/QuestionImportRow.vue`, une
  carte par ligne : « Ligne N », recto et verso par `RichTextView`, BEM et variables `--cinq-*`.
- [X] T030 [US1] Créer `web/functional/Authoring/app/components/QuestionImportSource.vue` (onglet
  « Fichier » par `useFilePicker` avec
  `accept=".csv,.tsv,.txt,.xlsx"`, lien `<a :href download>` vers le modèle, court texte d'aide du
  format) et `web/functional/Authoring/app/components/QuestionImportPreview.vue` (résumé avec le
  nombre de questions et les notices, liste de `QuestionImportRow`, boutons « Importer N
  questions » et « Abandonner », import désactivé si `!can_confirm` ou `isOffline` avec
  explication).
- [X] T031 [US1] Créer `web/functional/Authoring/app/pages/sujets/[id]/importer.vue`
  (`definePageMeta({ middleware: 'auth' })`) qui assemble T028 à T030. Dans
  `web/functional/Authoring/app/pages/sujets/[id]/modifier.vue`, ajouter le bouton « Importer des
  questions », caché quand `isReadOnly`. Ajouter les clés à
  `web/functional/Authoring/i18n/locales/fr.json`. Faire passer T026 et T027.

**Checkpoint**: US1 démontrable de bout en bout avec un fichier valide.

---

## Phase 4: User Story 2 - Corriger les erreurs avant d'importer (Priority: P1)

**Goal**: chaque problème est signalé ligne par ligne, rien n'est importé tant qu'il en reste un,
et les doublons sont signalés sans bloquer.

**Independent Test**: un fichier de 100 lignes, avec un verso vide à la ligne 12 et un recto de
6 000 caractères à la ligne 40, signale ces deux lignes et rend la confirmation impossible. Le sujet
reste inchangé. Corrigé, il s'importe en entier.

### Tests (écrits d'abord, en échec)

- [X] T032 [P] [US2] Compléter `back/functional/catalog/tests/Feature/QuestionImportPreviewTest.php` :
  - verso vide en ligne 12 → `errors: [{ field: verso, code: verso_empty }]` sur la ligne 12 ;
  - recto de 5 001 caractères visibles → `recto_too_long` ;
  - cellule de moins de 5 000 caractères visibles mais de plus de 20 000 caractères de HTML →
    `*_too_long` ;
  - cellule faite d'espaces → vide ;
  - `error_line_count` juste et `can_confirm` faux ;
  - 450 questions + 80 lignes → `question_limit_exceeded` avec `remaining: 50` ;
  - source faite seulement d'un en-tête et de lignes vides → `import_empty` ;
  - trois colonnes → `extra_columns_ignored` une seule fois et `can_confirm` vrai (FR-003, FR-008,
    FR-011, FR-012, US2-1 à US2-7).
- [X] T033 [P] [US2] Compléter `back/functional/catalog/tests/Feature/QuestionImportPreviewTest.php`
  pour FR-026 :
  - recto égal à celui de la question en position 4, avec une casse, des accents, des espaces et
    une mise en forme différents → `duplicate_in_subject` avec `position: 4` ;
  - deux lignes au même recto → `duplicate_in_source` avec `line` de la première, porté par la
    seconde ;
  - `can_confirm` reste vrai.
- [X] T034 [P] [US2] Compléter `back/functional/catalog/tests/Feature/QuestionImportTest.php` pour
  SC-003 : dans chaque cas suivant, le sujet est identique avant et après, et aucun job
  `AddQuestionToLearners` n'aboutit.
  - confirmation d'une source qui contient une erreur → `422 import_has_errors` ;
  - limite atteinte entre l'aperçu et la confirmation (questions créées entre-temps) → `422
    import_has_errors` ;
  - sujet retiré entre-temps, pour son auteur → `422 subject_retired` ;
  - exception levée à la 30e création (double de `Question` ou écouteur qui lève) → transaction
    annulée, aucune question.

### Implémentation

- [X] T035 [US2] Dans `back/functional/catalog/src/Import/QuestionImportPreview.php`, ajouter les
  erreurs de ligne et d'ensemble :
  - `recto_empty`, `verso_empty` : `VisibleTextLength::of() === 0` ;
  - `recto_too_long`, `verso_too_long` : plus de `VisibleTextLength::MAX` (5 000) ou plus de
    `QuestionResource::MAX_HTML_LENGTH` (20 000) caractères de HTML ;
  - `import_empty` ;
  - `question_limit_exceeded` : questions du sujet + lignes > `Subject::MAX_QUESTIONS` (500), avec
    `remaining`.

  Faire passer T032 et T034.
- [X] T036 [US2] Dans le même fichier, ajouter les avertissements `duplicate_in_subject` et
  `duplicate_in_source`, sur le recto normalisé (data-model) : balises retirées, entités
  décodées, minuscules, `Str::ascii`, espaces réduits et bords retirés. Les rectos du sujet sont
  chargés en une requête. Faire passer T033.
- [X] T037 [P] [US2] Compléter `web/functional/Authoring/tests/QuestionImportPage.nuxt.spec.ts` :
  - un aperçu avec des erreurs affiche en tête « N lignes à corriger » et un lien vers chacune
    (ancre `#ligne-12`) ;
  - chaque erreur s'affiche en `role="alert"` sur sa ligne ;
  - les avertissements et les notices s'affichent sans bloquer ;
  - les erreurs d'ensemble (`question_limit_exceeded`, `import_empty`) s'affichent en tête ;
  - « Importer » est désactivé tant que `can_confirm` est faux ;
  - « Choisir un autre fichier » revient à la source ;
  - une réponse `422 import_has_errors` à la confirmation réaffiche l'aperçu avec le message.
- [X] T038 [US2] Dans `web/functional/Authoring/app/components/QuestionImportPreview.vue` et
  `web/functional/Authoring/app/components/QuestionImportRow.vue`, ajouter :
  - le résumé des lignes à corriger, avec les ancres `id="ligne-N"` ;
  - les erreurs d'ensemble ;
  - les erreurs de ligne en `role="alert"` sous la cellule concernée ;
  - les avertissements dans un ton distinct, sans compter sur la couleur seule.

  Ajouter les clés à `web/functional/Authoring/i18n/locales/fr.json`. Faire passer T037.

**Checkpoint**: US1 et US2 forment la première livraison utile.

---

## Phase 5: User Story 3 - Coller depuis un tableur (Priority: P2)

**Goal**: coller deux colonnes d'un tableur, d'Anki ou de Quizlet, et passer par le même aperçu.

**Independent Test**: 20 lignes copiées d'un tableur, collées, donnent 20 questions dans l'aperçu,
puis à la fin du sujet.

### Tests (écrits d'abord, en échec)

- [X] T039 [P] [US3] Créer `back/functional/catalog/tests/fixtures/import/anki.txt`
  (`#separator:tab`, `#html:true`, puis des lignes tabulées) et
  `back/functional/catalog/tests/fixtures/import/quizlet.txt`. Compléter
  `back/functional/catalog/tests/Feature/QuestionImportPreviewTest.php` et
  `back/functional/catalog/tests/Feature/QuestionImportTest.php` pour `{ "text": … }` :
  - texte tabulé → une question par ligne ;
  - Anki et Quizlet lus pareil ;
  - cellule multiligne entre guillemets, comme la copie de Google Sheets → une seule cellule ;
  - texte sans tabulation → `422 import_single_column` ;
  - confirmation par JSON avec `import_id` → `201` (FR-005, US3-1 à US3-4).
- [X] T040 [P] [US3] Compléter `web/functional/Authoring/tests/QuestionImportPage.nuxt.spec.ts` :
  - l'onglet « Coller » envoie `{ text }` à l'aperçu par `useApiFetch`, puis
    `{ text, import_id }` à l'import ;
  - un texte vide n'est pas envoyé ;
  - `import_single_column` affiche son message sous la zone de texte.

### Implémentation

- [X] T041 [US3] Dans `back/functional/catalog/src/Import/SourceReader.php`, ajouter
  `fromText(string $text)` : `SourceDecoder::decode()`, séparateur tabulation imposé,
  `import_single_column` si aucune ligne n'a de tabulation. Brancher `text` dans
  `PreviewQuestionImportController` et `ImportQuestions`. Faire passer T039.
- [X] T042 [US3] Dans `web/functional/Authoring/app/composables/useQuestionImport.ts`, ajouter
  `previewText(text)` et la confirmation par `useApiFetch()` (`POST`, JSON). Dans
  `web/functional/Authoring/app/components/QuestionImportSource.vue`, ajouter l'onglet
  « Coller », avec une zone de texte (`v-textarea`) et l'aide « copiez deux colonnes depuis votre
  tableur, Anki ou Quizlet ». Ajouter les clés à `web/functional/Authoring/i18n/locales/fr.json`.
  Faire passer T040.

**Checkpoint**: les deux sources mènent au même aperçu.

---

## Phase 6: User Story 4 - Importer dans un sujet déjà publié et appris (Priority: P2)

**Goal**: l'auteur est averti avant de confirmer que les questions arriveront en boîte 1 chez les
apprenants, et elles y arrivent.

**Independent Test**: un sujet publié appris par 2 inscrits affiche l'avertissement. Un import de
30 questions place 30 cartes en boîte 1, à réviser aujourd'hui chez chacun, sans changer les autres
cartes.

### Tests (écrits d'abord, en échec)

- [X] T043 [P] [US4] Test `back/functional/learning/tests/Feature/SubjectLearnersTest.php` sur
  `GET /api/learning/subjects/{subject}/learners` :
  - sujet appris par un autre inscrit → `{ has_learners: true }` ;
  - sujet appris seulement par son auteur → `true` ;
  - sujet que personne n'apprend → `false` ;
  - administrateur → `200` ;
  - autre inscrit → `403` ou `404` ;
  - visiteur → `401` ;
  - aucun nombre ni aucune identité dans la réponse (FR-019, R8).
- [X] T044 [P] [US4] Test
  `back/functional/learning/tests/Feature/ImportedQuestionsReachLearnersTest.php` : un sujet
  publié appris par 2 inscrits, dans deux fuseaux différents, reçoit 30 questions par
  `POST /api/subjects/{subject}/question-import`. Résultat attendu :
  - 30 `CardProgress` par inscrit, en `LeitnerSchedule::FIRST_BOX`, avec `next_review_on` =
    aujourd'hui dans le fuseau de chacun ;
  - les cartes existantes gardent leur boîte et leur date ;
  - un import rejeté (`import_has_errors`) ne crée aucune carte (FR-020, US4-2).
- [X] T045 [P] [US4] Compléter `web/functional/Authoring/tests/QuestionImportPage.nuxt.spec.ts` :
  - `learning/subjects/{id}/learners` est appelé à l'ouverture ;
  - si `has_learners` est vrai, l'aperçu affiche l'avertissement « Les questions importées
    entreront en boîte 1 chez les personnes qui apprennent ce sujet, à réviser dès aujourd'hui » ;
  - si c'est faux, aucun avertissement ;
  - en cas d'échec de cet appel, aucun avertissement, et l'import reste possible.

### Implémentation

- [X] T046 [US4] Créer
  `back/functional/learning/src/Http/Controllers/SubjectLearnersController.php`
  (`Gate::authorize('update', $subject)`, puis `Learning::query()->where('subject_id',
  …)->exists()`), et déclarer `Route::get('learning/subjects/{subject}/learners', …)` dans le
  groupe `auth:sanctum` de `back/functional/learning/routes/api.php`. Faire passer T043. T044 doit
  passer sans autre code, par le chemin `QuestionCreated` → `AddQuestionToLearners` (R7). Sinon,
  corriger `ImportQuestions`, pas le chemin Leitner.
- [X] T047 [US4] Dans `web/functional/Authoring/app/composables/useQuestionImport.ts`, lire
  `hasLearners` par `useApiFetch('/learning/subjects/{id}/learners')`. Dans
  `web/functional/Authoring/app/components/QuestionImportPreview.vue`, afficher l'avertissement
  avec `AccountNotice` (`tone="info"`). Ajouter la clé à
  `web/functional/Authoring/i18n/locales/fr.json`. Faire passer T045.

**Checkpoint**: les quatre stories sont en place.

---

## Phase 7: Polish & vérifications transverses

- [X] T048 [P] Test de performance dans `back/functional/catalog/tests/Feature/QuestionImportPreviewTest.php` :
  l'aperçu d'un XLSX de 500 lignes, chacune avec du gras et une liste, répond en moins de 3 s
  (SC-004).
- [X] T049 [P] Test dans `back/functional/catalog/tests/Feature/QuestionImportTest.php` pour
  SC-006 : des cellules `<script>alert(1)</script>`, `<img src=x onerror=alert(1)>`,
  `[x](javascript:alert(1))` et `<a href="javascript:…">` sont importées comme du texte ou
  retirées. Aucun `<script>`, `onerror` ni `javascript:` dans `recto_html` ou `verso_html`
  enregistrés.
- [ ] T050 [P] Vérifier à 360 px, dans `web/functional/Authoring/app/components/QuestionImport*.vue`,
  qu'il n'y a aucun défilement horizontal : une cellule de code longue passe à la ligne ou défile
  dans son propre bloc, et les liens et URL longs sont coupés. Vérifier aussi les cibles tactiles
  d'au moins 44 px et le focus visible (FR-023, constitution V).
- [ ] T051 Lancer `./vendor/bin/sail bin pint --dirty` et `./vendor/bin/sail artisan test` dans
  `back/`, puis `pnpm lint`, `pnpm exec prettier --check .` et `pnpm test` dans `web/`. Tout doit
  être vert.
- [ ] T052 Dérouler les parcours manuels 1 à 8 de `specs/008-question-import/quickstart.md` à
  360 px, avec de vrais exports d'Excel en français, de LibreOffice, de Google Sheets, d'Anki et de
  Quizlet. Noter les résultats sous une section « Résultats des parcours manuels » de ce fichier.
  Tout écart est corrigé dans le code ou remonté dans la spec, jamais contourné.
- [ ] T053 Fusionner en avance rapide, back puis web (constitution, flux de travail). Vérifier
  d'abord les principes III, IV et VI. Si `origin/main` a avancé et que l'avance rapide est
  impossible, s'arrêter et le signaler.

### État des finitions (2026-10-02, 00 h 15)

- **T050** : vérifié à la lecture du CSS, pas dans un navigateur. Les cartes de l'aperçu passent
  en une colonne sous 600 px, sans largeur minimale. `RichTextView` coupe les mots longs
  (`overflow-wrap: anywhere`) et fait défiler le code dans son propre bloc. Les boutons sont en
  `size="large"`. Reste à le voir dans un navigateur à 360 px.
- **T051** : Pint, ESLint et Prettier sont verts. Le web passe 270 tests sur 270, le back 576
  sur 579. Les 3 échecs sont des tests préexistants de la couche learning :
  - `test_the_learnings_count_their_cards_and_belong_to_their_learner` ;
  - `test_the_due_cards_are_those_of_the_chosen_subjects_most_overdue_first` ;
  - `test_the_card_shows_where_it_went_after_an_answer`.

  Ils échouent aussi sur `origin/main`, vérifié dans un worktree : entre minuit et 2 h, heure de
  Paris, la date UTC n'est pas encore celle de Paris. Ce n'est pas un effet de l'import, mais la
  suite doit être verte avant T053.
- **T052** : non fait. Ce parcours demande de vrais exports d'Excel, de LibreOffice, de Google
  Sheets, d'Anki et de Quizlet, ouverts dans un navigateur à 360 px.

### Écarts entre les tâches et l'implémentation

- **T001** : `openspout/openspout` s'installe en **5.12**, version stable la plus récente (le plan
  prévoyait 4.x). Son API de lecture diffère : cellules par `Row::$cells`, valeur calculée d'une
  formule par `FormulaCell::getComputedValue()`.
- **T015** : seules les fixtures CSV, TXT et PDF sont des fichiers. Elles sont générées depuis
  `questions.json`, octet pour octet selon chaque tableur, et `.gitattributes` les garde en
  `-text` pour que Git ne touche pas à leurs fins de ligne. Les XLSX sont construits par
  `tests/Concerns/MakesImportSources.php` avec OpenSpout, en chaînes partagées ou en ligne, et la
  valeur calculée de la formule est ajoutée à la main, comme Excel l'écrit.
- **T022** : la taille et le contenu du fichier ne sont pas des règles de la requête.
  `SourceReader` les contrôle pour répondre avec les codes du contrat (`import_too_large`,
  `import_unreadable`) plutôt qu'avec des erreurs de champ. Un PDF renommé `.csv` est refusé sur
  son type réel.
- **T023** : le droit (`Gate`) est vérifié avant la reconnaissance d'un `import_id` déjà
  importé, et le verrou du sujet retiré après. Une personne sans droit n'apprend donc rien d'un
  `import_id`.
- **T041** : déjà en place avant US3. `fromText()` est venu avec T011, et le champ `text` avec
  T022. Les tests de T039 sont passés sans code de plus.
- **T037, T038** : le bouton qui revient à la source s'appelle « Changer de source », et non
  « Choisir un autre fichier », parce qu'il sert aussi au texte collé.
- **Correctif trouvé en route** : `useUploadRequest` rejette déjà avec une `IApiError`. La
  repasser dans `toApiError` remplaçait le message de l'API par le message générique.
  `useQuestionImport` distingue les deux cas.

---

## Dependencies & Execution Order

- **Phase 1** (T001) avant toute tâche back qui utilise OpenSpout ou CommonMark (T007, T016, T020,
  T024).
- **Phase 2** bloque toutes les stories :
  - T002 avant T018 et T023 ;
  - T003 avant T007 ;
  - T006 → T007 ; T008 → T009 → T010 → T011 ;
  - T012 → T013 avant T028 ; T014 avant T030.
- **US1** (MVP) dépend de la Phase 2 :
  - côté back, T015 → T016 → T020 → T021 → T022 → T023 → T024 → T025 ;
  - côté web, T028 → T029/T030 → T031.
- **US2** dépend de T021 et T023 (back) et de T030 (web).
- **US3** dépend de T011, T022 et T023 (back) et de T028 et T030 (web). Elle est indépendante de
  US2.
- **US4** : T043 et T046 ne dépendent que de la Phase 2. T044 dépend de T025, et T047 de T030.
- **Polish** après les quatre stories.
- **Fusion** : le back (T001 à T011, T015 à T025, T032 à T036, T039, T041, T043, T044, T046) est
  fusionné et poussé sur `main` avant le web.

### Parallel Opportunities

- Phase 2 : T003, T004, T005, T006, T008 en parallèle ; T012 et T014 en parallèle entre eux et
  avec le back.
- US1 : T015 à T019 en parallèle ; T026, T027 et T029 en parallèle une fois T028 fait.
- US2 : T032 à T034 en parallèle ; T037 en parallèle du back.
- US3 : T039 et T040 en parallèle.
- US4 : T043 à T045 en parallèle ; T046 peut commencer dès la Phase 2.
- Polish : T048 à T050 en parallèle.

## Implementation Strategy

1. Phases 1 et 2 : dépendances, colonne `import_id`, nettoyage partagé, Markdown restreint,
   décodage et lecture.
2. US1 : fichier, aperçu, confirmation, modèle. C'est le MVP démontrable, mais il ne doit pas être
   livré seul : sans US2, une ligne vide ou trop longue passerait jusqu'à la confirmation, où le
   cast et la création la refuseraient au milieu de la transaction.
3. US2 : erreurs, doublons, tout ou rien vérifié. US1 + US2 forment la première livraison utile.
4. US3, puis US4 : collage, puis avertissement et vérification du chemin Leitner.
5. Finitions, parcours manuels avec de vrais exports, puis fusion sur `main`, back puis web.
