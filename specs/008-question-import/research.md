# Research: Import de questions

Décisions techniques de la feature 008, avec leurs raisons et les alternatives écartées. Les
chemins renvoient à l'état du code au 2026-10-01.

## R1. `openspout/openspout` pour lire le XLSX, un lecteur maison pour le CSV et le texte collé

**Decision**: ajouter `openspout/openspout` (version stable la plus récente, 4.x) à la couche
`back/functional/catalog` (son `composer.json` et celui de la racine). Il ne sert qu'à lire le
XLSX : première feuille, valeurs de cellule. Le CSV et le texte collé passent par un même lecteur
écrit dans la couche, sur `fgetcsv` d'un flux `php://temp`, avec `escape: ""` (RFC 4180 : guillemets
doublés, cellules multilignes).

**Rationale**: aucune bibliothèque de tableur n'est installée (`back/composer.json`). OpenSpout lit
le XLSX en flux, sans charger le classeur en mémoire, et sous licence MIT. PhpSpreadsheet ferait la
même chose, mais en beaucoup plus lourd, et ses fonctions d'écriture et de calcul sont inutiles ici.
Pour le CSV, il faut de toute façon détecter soi-même l'encodage et le séparateur (R2) : OpenSpout
demande qu'on les lui donne. `fgetcsv` sait déjà lire les guillemets et les retours à la ligne dans
une cellule, et le texte collé est un CSV dont le séparateur est la tabulation. Un seul lecteur sert
donc aux deux. `back/AGENTS.md` demande l'accord du développeur pour toute nouvelle dépendance : la
validation de ce plan vaut cet accord.

**Alternatives considered**: `phpoffice/phpspreadsheet` (lourd, lecture en mémoire) ;
`maatwebsite/excel` (surcouche de PhpSpreadsheet, pensée pour des imports en file d'attente) ;
`league/csv` (n'apporte rien à `fgetcsv` pour deux colonnes, et ne lit pas le XLSX) ; lire le XLSX
à la main avec `ZipArchive` et SimpleXML (chaînes partagées, styles, dates : du code sans second
usage, principe VII).

## R2. Encodage et séparateur détectés par le back

**Decision**:
- **Encodage** : la marque d'en-tête UTF-8 (BOM) est retirée. Un contenu valide en UTF-8
  (`mb_check_encoding`) est gardé tel quel. Sinon, il est converti depuis `Windows-1252`, que
  `mbstring` sait lire et qui est l'encodage d'Excel en français. Une conversion qui échoue laisse
  le caractère de remplacement `�` visible dans l'aperçu (edge case de la spec).
- **Séparateur** : sur les 20 premières lignes non vides, lues avec chacun des candidats
  (tabulation, point-virgule, virgule), on retient celui qui donne au moins deux colonnes sur le plus
  de lignes. À égalité, l'ordre de préférence est tabulation, point-virgule, virgule. Le texte collé
  est toujours lu avec la tabulation (FR-005). S'il n'en contient aucune, il est refusé avec
  `import_single_column` (US3-4).
- **En-têtes Anki** : les lignes de tête de la forme `#clé:valeur` (`#separator:tab`, `#html:true`,
  `#notetype column:…`) sont ignorées.
- **Fins de ligne** : `\r\n`, `\r` et `\n` sont acceptées.

**Rationale**: un CSV d'Excel en français est en Windows-1252 et séparé par des points-virgules.
Google Sheets et LibreOffice produisent de l'UTF-8 séparé par des virgules. Un texte en UTF-8 valide
qui ne serait pas de l'UTF-8 est très improbable, d'où l'ordre de test. Le test sur plusieurs lignes
évite de prendre pour séparateur la virgule d'une phrase.

**Alternatives considered**: demander l'encodage et le séparateur à l'auteur (justement ce que la
spec veut éviter) ; détecter dans le navigateur (logique en double et non testable côté back).

## R3. Markdown léger : `league/commonmark` dans un environnement restreint

**Decision**: une classe `MarkdownCell` de la couche catalog convertit chaque cellule en HTML avec
`league/commonmark`, qu'on ajoute en dépendance directe (il est déjà installé par
`laravel/framework`). On n'utilise pas `CommonMarkCoreExtension`. Seuls ces éléments sont
enregistrés dans un `Environment` :
- **blocs** : paragraphes, listes à puces et numérotées, blocs de code délimités par trois
  accents graves (fenced code) ;
- **en ligne** : emphase et gras (`*`, `_`), code, liens `[texte](url)` et liens automatiques
  `<https://…>`, échappements par `\`, entités.

Tout le reste n'est pas reconnu et reste donc du texte (FR-024) : titres `#`, citations `>`,
séparateurs, code indenté de quatre espaces, HTML. Les images `![…](…)` sont lues par le même
analyseur que les liens. Elles reçoivent donc un rendu qui réécrit leur source telle quelle, sous
forme de texte. Le reste de la configuration :
- `html_input: escape` et `allow_unsafe_links: false` ;
- les sauts de ligne simples sont rendus en `<br>`, pour garder les retours à la ligne des
  cellules (edge case de la spec).

Le HTML produit passe ensuite par le même nettoyage que l'éditeur (R4).

**FR-025** : les règles d'emphase de CommonMark couvrent déjà presque tout. Un symbole n'ouvre une
emphase que s'il est collé au texte qui suit et qu'un symbole fermant existe : `2 * 3 * 4`,
`_début sans fin` et `prix : 5 € *` restent du texte. Le tiret bas à l'intérieur d'un mot
(`nom_de_variable`) ne fait jamais d'emphase. Reste l'astérisque à l'intérieur d'un mot, que
CommonMark accepte (`5*3*2` donnerait `5<em>3</em>2`). Avant la conversion, tout `*` placé entre
deux lettres ou chiffres est donc échappé, ce qui respecte FR-025. Un jeu d'essai de textes courants
vérifie SC-007.

**Rationale**: CommonMark est la référence du Markdown, et ses règles d'emphase sont justement
faites pour ne pas interpréter les symboles isolés. Un environnement restreint en dit plus qu'un
nettoyage après coup : un titre non reconnu reste `# Titre`, alors que retirer la balise `<h1>`
après conversion ferait perdre le `#`.

**Alternatives considered**: `Str::markdown()` avec l'environnement complet, puis HTMLPurifier
(les titres et les images seraient avalés au lieu de rester du texte, ce que FR-024 interdit) ;
`erusev/parsedown` (moins strict sur l'emphase, et une dépendance de plus) ; une conversion par
expressions régulières (fragile, et justement source des erreurs que la spec veut éviter).

## R4. Le même nettoyage pour l'aperçu et l'enregistrement

**Decision**: le contenu de `SanitizedHtml::set()` (`Purify::clean()` puis `rel="noopener nofollow
ugc"` sur les liens) devient une méthode statique `SanitizedHtml::clean()`. L'aperçu l'appelle sur
le HTML converti, et le cast l'appelle à l'enregistrement. Les longueurs sont contrôlées par
`VisibleTextLength::of()` (1 à 5 000 caractères visibles) sur ce HTML nettoyé, et par
`QuestionResource::MAX_HTML_LENGTH` (20 000 caractères de balisage), dont le dépassement est signalé
comme un texte trop long.

**Rationale**: FR-027 demande que l'aperçu montre exactement ce qui sera enregistré. Si le chemin
de nettoyage est le même, rien ne peut diverger entre les deux.

**Alternatives considered**: nettoyer dans le navigateur pour l'aperçu (deux nettoyeurs, deux
résultats possibles) ; renvoyer le Markdown au web pour qu'il le convertisse (même problème).

## R5. Deux appels sans état, la source renvoyée à la confirmation

**Decision**: l'aperçu (`POST …/question-import/preview`) lit la source, construit les lignes et
ne garde rien. La confirmation (`POST …/question-import`) reçoit de nouveau la même source (le web
garde le `File` ou le texte en mémoire), la relit et la revalide entièrement. Si tout est valide,
elle importe. Sinon, elle répond `422 import_has_errors` et n'importe rien.

**Rationale**: la spec ne conserve pas la source (Key Entities). Relire à la confirmation est
justement ce que demande FR-014 (« vérifier de nouveau les droits et les limites »). Le web n'a rien
à garder qu'il n'ait déjà, et aucun nettoyage d'aperçus abandonnés n'est à prévoir. Un fichier de
5 Mo envoyé deux fois reste acceptable.

**Alternatives considered**: garder l'aperçu analysé côté serveur, dans une table ou en cache, et
confirmer par son identifiant (une table, une durée de vie et un nettoyage pour éviter un second
envoi, principe VII) ; renvoyer à la confirmation les lignes en JSON (un second format d'entrée à
valider, alors que la source est déjà là).

## R6. Tout ou rien, et une seule fois

**Decision**: une action `ImportQuestions` fait tout dans une `DB::transaction` :
1. Elle verrouille le sujet (`lockForUpdate`).
2. Elle vérifie `Gate::authorize('update', $subject)` et `RetiredSubjectLock::ensureEditable()`,
   comme `QuestionResource::placeNewQuestion()`.
3. Elle recompte les questions face à `Subject::MAX_QUESTIONS`.
4. Elle crée chaque question comme un modèle (`Question::create`), aux positions `max + 1`,
   `max + 2`, etc.

L'idempotence (FR-017) reprend le motif de `AnswerCard` (feature 006) :
- le web génère un `import_id` (UUID) à chaque aperçu, et l'envoie à la confirmation ;
- chaque question importée porte cet `import_id` ;
- si des questions de ce sujet portent déjà cet `import_id`, la confirmation répond `200` avec leur
  nombre, sans rien ajouter.

**Rationale**:
- Le verrou du sujet supprime la course entre le comptage, la position maximale et l'insertion,
  que la création une par une laisse ouverte aujourd'hui.
- Une colonne nullable sur `questions` suffit à reconnaître une confirmation rejouée (double clic,
  ou nouvel essai après une coupure de réseau). Après une coupure, l'auteur apprend ainsi si
  l'import avait réussi (edge case « hors ligne »).
- La colonne n'est jamais exposée, et une question importée reste une question ordinaire (FR-016).

**Alternatives considered**: une table `question_imports` (une table, et un nettoyage, pour une
seule information) ; un verrou en cache Redis (rien ne le relie à la transaction : un import annulé
pourrait bloquer le nouvel essai) ; se fier au bouton désactivé du web (ne couvre pas un nouvel
essai après une coupure).

## R7. Les apprenants reçoivent les questions par le chemin existant

**Decision**: chaque `Question::create` déclenche `QuestionCreated`. Le chemin existant prend
alors le relais : `QueueQuestionForLearners`, puis le job `AddQuestionToLearners`, qui place la
carte en boîte 1, à réviser le jour même dans le fuseau de chaque apprenant. Aucun nouveau code dans
la couche learning pour FR-020.

**Rationale**: FR-020 exige « exactement comme une question ajoutée à la main », et ce chemin est
déjà testé (`ContentChangesTest`). 500 jobs pour un import maximal restent raisonnables. Si la
transaction est annulée, les jobs trouvent une question absente et sont supprimés
(`deleteWhenMissingModels`), sans rien créer.

**Alternatives considered**: un événement `QuestionsImported` et une insertion groupée de
`card_progress`, comme `CreateCardsForLearning` (plus rapide, mais une seconde règle de mise en
boîte à garder alignée sur la première, sans besoin de performance avéré, principe VII).

## R8. « Ce sujet est-il appris ? » est demandé à la couche learning

**Decision**: la couche learning expose `GET /api/learning/subjects/{subject}/learners`, qui répond
`{ "has_learners": bool }` à qui peut modifier le sujet. Le web l'appelle en ouvrant l'écran
d'import, et affiche l'avertissement de FR-019 si la réponse est vraie. L'auteur compte parmi les
apprenants s'il apprend son propre sujet, puisque ses cartes entrent aussi en boîte 1.

**Rationale**: learning dépend de catalog, et l'inverse créerait un cycle (principe II). C'est déjà
la réponse de la feature 004 (`AuthoredSubjectsSummaryController`, R10 de la 004) : learning est la
seule couche qui voit à la fois les sujets et les apprenants. Seul un booléen est exposé, aucun
nombre ni aucune identité d'apprenant.

**Alternatives considered**: ajouter `has_learners` à la réponse de l'aperçu (catalog devrait lire
`learnings`) ; un contrat ou un événement de catalog que learning implémenterait (une abstraction
pour un seul usage, principe VII).

## R9. Routes hors lomkit dans la couche catalog

**Decision**: trois routes dans `back/functional/catalog/routes/api.php`. Un commentaire explique
qu'il s'agit de fichiers et d'un flux binaire, que lomkit ne transporte pas :
- `POST /api/subjects/{subject}/question-import/preview` ;
- `POST /api/subjects/{subject}/question-import` ;
- `GET /api/question-import/template`.

Les deux `POST` sont sous `auth:sanctum` et `verified`, avec `throttle:30,1` pour l'aperçu et
`throttle:10,1` pour la confirmation. Ils acceptent soit un fichier `file` (multipart), soit un
champ `text`. Le modèle (template) est public, avec `throttle:30,1`.

**Rationale**: même motif que `POST /api/question-images` (feature 003). Les actions lomkit ne
transportent ni un fichier ni un aperçu calculé.

**Alternatives considered**: une action lomkit `questions/actions/import` pour le texte collé (deux
chemins pour une même lecture).

## R10. Le modèle XLSX est produit par le back

**Decision**: `GET /api/question-import/template` écrit avec OpenSpout un classeur
`modele-questions.xlsx` : un en-tête « Recto », « Verso », et deux lignes d'exemple, dont une avec
du gras et une liste. Tous ces textes viennent des traductions du back. Le web l'offre par un lien
de navigation (`<a href download>`), pas par un appel `fetch`.

**Rationale**: la constitution (V) interdit le texte destiné à l'utilisateur codé en dur. Un fichier
binaire figé dans `web/public/` porterait des textes hors des traductions. OpenSpout écrit aussi le
XLSX : aucune dépendance de plus. Un lien de navigation n'est pas un appel d'API au sens du principe
III, comme la redirection Google de la 007.

**Alternatives considered**: un fichier statique dans `web/functional/Authoring/public/` (textes
hors traductions, et un binaire à régénérer à la main) ; un modèle CSV (que justement Excel en
français ouvre mal, comme le note la spec).

## R11. Écran d'import : une page de la couche Authoring

**Decision**: une page `/sujets/[id]/importer` dans `web/functional/Authoring`, ouverte par un
bouton « Importer des questions » de la page `/sujets/[id]/modifier`. Ce bouton est caché quand la
page est en lecture seule (`isReadOnly`). La page a deux temps :
- **la source** : un onglet « Coller » avec une zone de texte, un onglet « Fichier » avec un
  sélecteur `.csv`, `.tsv`, `.txt` ou `.xlsx`, et un lien vers le modèle ;
- **l'aperçu** : un résumé en tête (nombre de questions, lignes à corriger avec des liens vers
  chacune, notices et avertissement d'apprentissage), puis une carte par ligne (« Ligne 12 », recto
  et verso rendus par `RichTextView`, erreurs en `role="alert"`, avertissements).

Le bouton « Importer N questions » est désactivé quand il y a une erreur ou hors ligne
(`useConnectionStatus`). Après l'import, la page revient sur `/sujets/[id]/modifier` avec un toast
« N questions ajoutées ».

**Rationale**:
- Un aperçu de 500 cartes n'entre pas dans un dialogue, surtout à 360 px (FR-023).
- Une page laisse la place de corriger, de recharger la source et de recommencer.
- Les cartes plutôt qu'un tableau évitent le défilement horizontal.

**Alternatives considered**: un dialogue ou une feuille du bas, comme le signalement (trop étroit
pour l'aperçu) ; un tableau (déborde à 360 px).

## R12. Envoi du fichier : `useUploadRequest` accepte des champs en plus

**Decision**: `upload(path, file, fields?)` dans `web/technical/ApiClient` ajoute au `FormData`
les champs donnés (`import_id`). Le texte collé passe par `useApiFetch()` en JSON, comme les autres
routes hors lomkit. Avant l'envoi, le web contrôle l'extension (`.csv`, `.tsv`, `.txt`, `.xlsx`) et
la taille (5 Mo). Le type MIME n'est pas contrôlé, car il varie selon le système pour un CSV. Le
sélecteur caché de `useImagePicker` est généralisé en `useFilePicker`, qui reçoit `accept` en
paramètre. Les images en sont le premier usage, les fichiers d'import le second.

**Rationale**: `useUploadRequest` est le seul transport multipart avec progression et annulation,
deux choses qu'un fichier de 5 Mo sur téléphone justifie. Le back reste juge du contenu
(`mimes:csv,txt,xlsx` et lecture réelle).

**Alternatives considered**: lire le fichier dans le navigateur et l'envoyer comme texte (le XLSX
demanderait une bibliothèque côté web) ; `$fetch` multipart direct (interdit par le principe III,
et sans progression).
