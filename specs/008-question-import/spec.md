# Feature Specification: Import de questions

**Feature Branch**: `008-question-import`

**Created**: 2026-10-01

**Status**: Draft

**Input**: User description: "Feature 008 — Import de questions par l'auteur. Tout auteur peut importer d'un coup des questions dans son propre sujet (un administrateur aussi, puisqu'il peut modifier n'importe quel sujet, cf. 001 FR-032). Deux façons d'importer : coller deux colonnes copiées depuis un tableur (format tabulé, compatible avec les exports Anki et Quizlet), ou envoyer un fichier CSV ou XLSX, formats gratuits que produisent Google Sheets et LibreOffice. Deux colonnes : recto et verso. Un modèle de fichier est téléchargeable. Pour le CSV, le séparateur (virgule, point-virgule, tabulation) est détecté automatiquement, et l'encodage UTF-8 comme Windows-1252 est accepté, pour que les accents d'un export Excel en français ne soient pas cassés. Une ligne d'en-tête est reconnue et ignorée. Les cellules sont du texte brut. Point ouvert : accepte-t-on du Markdown léger, converti dans la mise en forme autorisée (gras, italique, listes, code, liens) puis nettoyé comme une saisie normale (001 FR-015) ? Avant de valider, l'auteur voit un aperçu des questions lues, avec les erreurs ligne par ligne (recto ou verso vide, plus de 5 000 caractères, dépassement de la limite de 500 questions par sujet). L'import se fait en tout ou rien : tant qu'il reste une erreur, rien n'est importé. Les questions importées sont ajoutées à la fin du sujet, dans l'ordre du fichier. L'import est possible dans un brouillon. Dans un sujet publié et déjà appris, chaque question importée entre en boîte 1 pour tous les apprenants (001 FR-051) : l'auteur en est averti avant de confirmer. Point ouvert : signaler ou ignorer une question dont le recto existe déjà dans le sujet ? Hors périmètre : les images (feature 003), l'export de questions, l'import de plusieurs sujets à la fois, le remplissage du catalogue par l'équipe au lancement."

## Contexte

Depuis la feature 001, un auteur ajoute les questions de son sujet une par une : un recto, un verso, enregistrer, recommencer. C'est adapté à un sujet de 10 questions. Ça ne l'est plus pour 200 verbes irréguliers, une liste de capitales ou les fiches qu'un auteur a déjà rédigées dans un tableur, dans Anki ou dans Quizlet. Il ne les retapera jamais une à une, et ce contenu n'arrive pas dans CINQ.

Cette feature permet à l'auteur d'importer d'un coup des questions dans son sujet. Il colle deux colonnes copiées depuis un tableur, ou il envoie un fichier de tableur aux formats gratuits (CSV, XLSX). Avant que quoi que ce soit ne soit enregistré, il voit ce qui a été lu et ce qui pose problème. Ensuite, tout est importé, ou rien.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Importer des questions depuis un fichier (Priority: P1)

Une autrice a préparé ses questions dans un tableur, une question par ligne, le recto dans la première colonne et le verso dans la seconde. Depuis son sujet, elle choisit « Importer des questions », envoie son fichier, voit l'aperçu des questions lues, puis confirme. Les questions s'ajoutent à la fin de son sujet.

**Why this priority**: C'est la valeur de la feature : passer de plusieurs heures de saisie à quelques secondes.

**Independent Test**: Une autrice ouvre un brouillon de 3 questions, envoie un fichier XLSX de 50 lignes plus un en-tête, voit 50 questions dans l'aperçu, confirme, et retrouve 53 questions dans son sujet : les 3 anciennes, puis les 50 importées dans l'ordre du fichier.

**Acceptance Scenarios**:

1. **Given** une autrice sur son brouillon, **When** elle envoie un fichier XLSX de deux colonnes, **Then** l'aperçu affiche chaque question lue (numéro de ligne, recto, verso) et le nombre total de questions qui seront ajoutées.
2. **Given** un aperçu sans erreur, **When** l'autrice confirme, **Then** toutes les questions sont ajoutées à la fin du sujet, dans l'ordre du fichier, et un message indique combien ont été ajoutées.
3. **Given** un aperçu, **When** l'autrice l'abandonne sans confirmer, **Then** son sujet n'a pas changé.
4. **Given** un fichier CSV exporté par Excel en français (séparateur point-virgule, accents en Windows-1252), **When** l'autrice l'envoie, **Then** l'aperçu affiche les accents correctement et sépare bien le recto du verso.
5. **Given** un fichier CSV en UTF-8 séparé par des virgules, avec une cellule entre guillemets qui contient une virgule et un retour à la ligne, **When** l'autrice l'envoie, **Then** cette cellule est lue en entier, retour à la ligne compris, dans la bonne colonne.
6. **Given** un fichier dont la première ligne est « Recto ; Verso » (ou « Question ; Réponse »), **When** il est lu, **Then** cette ligne n'est pas importée comme une question.
7. **Given** une autrice qui ne sait pas comment préparer son fichier, **When** elle ouvre « Importer des questions », **Then** elle peut télécharger un modèle de fichier déjà rempli de deux exemples, et lire en quelques lignes le format attendu.
8. **Given** un fichier qui n'est ni un CSV ni un XLSX lisible (PDF, image, XLSX corrompu), **When** l'autrice l'envoie, **Then** il est refusé avec un message qui indique les formats acceptés, et rien n'est importé.
9. **Given** une cellule « Le passé de *go* est **went** », **When** elle est lue, **Then** l'aperçu affiche « go » en italique et « went » en gras, exactement comme la question s'affichera une fois importée.
10. **Given** des cellules « 2 * 3 * 4 = 24 », « nom_de_variable », « prix : 5 € * » ou « _début sans fin », **When** elles sont lues, **Then** l'aperçu les affiche telles qu'elles sont écrites, sans aucune mise en forme.
11. **Given** une cellule qui contient une mise en forme que CINQ n'autorise pas (titre « # Titre », image, tableau, balise HTML), **When** elle est lue, **Then** elle est conservée comme du texte, telle qu'elle est écrite, et n'est jamais interprétée.

---

### User Story 2 - Corriger les erreurs avant d'importer (Priority: P1)

Le fichier d'un auteur contient quelques lignes incomplètes ou trop longues. L'aperçu les lui montre, ligne par ligne. Il corrige son fichier et le renvoie. Tant qu'une erreur reste, rien n'est importé.

**Why this priority**: Sans contrôle préalable, un import de 200 lignes laisserait le sujet à moitié rempli, ou rempli de cartes vides, et l'auteur devrait les retrouver une par une.

**Independent Test**: Un fichier de 100 lignes dont la ligne 12 a un verso vide et la ligne 40 un recto de 6 000 caractères ; l'aperçu signale ces deux lignes, la confirmation est impossible, et le sujet reste inchangé. Le même fichier corrigé s'importe en entier.

**Acceptance Scenarios**:

1. **Given** un fichier dont une ligne a un recto ou un verso vide, **When** il est lu, **Then** l'aperçu signale cette ligne, avec son numéro dans le fichier et le motif « recto vide » ou « verso vide ».
2. **Given** un fichier dont une cellule dépasse 5 000 caractères, **When** il est lu, **Then** l'aperçu signale la ligne et la colonne concernées, avec la limite.
3. **Given** un sujet de 450 questions et un fichier de 80 lignes, **When** il est lu, **Then** l'aperçu indique que le sujet ne peut recevoir que 50 questions de plus, sur les 500 autorisées.
4. **Given** un aperçu qui contient au moins une erreur, **When** l'auteur regarde le bouton de confirmation, **Then** il est indisponible, et un message indique le nombre de lignes à corriger.
5. **Given** un fichier dont des lignes sont entièrement vides, **When** il est lu, **Then** ces lignes sont ignorées sans erreur.
6. **Given** un fichier qui a plus de deux colonnes (par exemple un export Anki avec une colonne de tags), **When** il est lu, **Then** seules les deux premières colonnes sont utilisées, et l'aperçu le signale une fois, sans bloquer l'import.
7. **Given** un fichier qui ne contient aucune question une fois les lignes vides et l'en-tête ignorés, **When** il est lu, **Then** un message l'indique, et rien ne peut être importé.
8. **Given** une ligne dont le recto est identique à celui d'une question déjà dans le sujet, ou d'une autre ligne de la source, **When** elle est lue, **Then** l'aperçu la signale comme un doublon probable, en indiquant la question ou la ligne en double, mais l'import reste possible.

---

### User Story 3 - Coller depuis un tableur (Priority: P2)

Un auteur sélectionne deux colonnes dans Google Sheets, LibreOffice ou Excel, les copie, et les colle dans la zone prévue de « Importer des questions ». Il passe ensuite par le même aperçu que pour un fichier. Le même geste fonctionne avec le texte exporté par Anki ou Quizlet (une question par ligne, recto et verso séparés par une tabulation).

**Why this priority**: C'est le chemin le plus rapide pour quelques dizaines de questions, sans fichier à enregistrer ni question d'encodage, et le seul chemin naturel sur téléphone. Mais l'import par fichier (US1) couvre déjà le besoin, d'où la P2.

**Independent Test**: Copier 20 lignes de deux colonnes dans un tableur, les coller, voir 20 questions dans l'aperçu, confirmer, et les retrouver à la fin du sujet.

**Acceptance Scenarios**:

1. **Given** deux colonnes copiées depuis un tableur, **When** l'auteur les colle et demande l'aperçu, **Then** chaque ligne devient une question, la première colonne en recto et la seconde en verso.
2. **Given** un texte exporté par Anki ou Quizlet (recto et verso séparés par une tabulation), **When** l'auteur le colle, **Then** il est lu de la même façon.
3. **Given** une cellule copiée depuis un tableur qui contient un retour à la ligne, **When** elle est collée, **Then** elle reste une seule cellule, retour à la ligne compris.
4. **Given** un texte collé sans aucune tabulation, **When** l'auteur demande l'aperçu, **Then** un message explique que le recto et le verso doivent être dans deux colonnes, et rien ne peut être importé.
5. **Given** un auteur sur un téléphone de 360 px de large, **When** il colle du texte et consulte l'aperçu, **Then** l'aperçu se lit sans défilement horizontal.

---

### User Story 4 - Importer dans un sujet déjà publié et appris (Priority: P2)

Un auteur enrichit un sujet publié que d'autres apprennent déjà. Avant de confirmer, il est averti que les questions importées vont arriver dans la boîte 1 de ces personnes, à réviser dès le jour même.

**Why this priority**: Un import de 200 questions dans un sujet appris remplit d'un coup la séance du lendemain de chaque apprenant. L'auteur doit le savoir avant, sans que l'import lui soit pour autant interdit.

**Independent Test**: Un sujet publié appris par 2 inscrits ; un import de 30 questions affiche l'avertissement avant la confirmation ; après confirmation, les 30 questions sont à réviser aujourd'hui en boîte 1 chez chacun des 2 inscrits, et leurs autres cartes n'ont pas changé de boîte.

**Acceptance Scenarios**:

1. **Given** un sujet publié appris par au moins un inscrit, **When** l'auteur arrive à l'aperçu, **Then** un avertissement indique que les questions importées entreront en boîte 1 chez les personnes qui apprennent ce sujet, à réviser dès aujourd'hui.
2. **Given** cet avertissement, **When** l'auteur confirme, **Then** chaque question importée entre en boîte 1 pour chaque apprenant, exactement comme une question ajoutée à la main (feature 001, FR-051), et les cartes déjà apprises ne changent pas.
3. **Given** un sujet publié que personne n'apprend, ou un brouillon, **When** l'auteur arrive à l'aperçu, **Then** aucun avertissement n'est affiché.
4. **Given** un sujet publié, **When** l'import est confirmé, **Then** les nouvelles questions sont visibles des lecteurs aussitôt, comme une question ajoutée à la main.

---

### Edge Cases

- **Mise en forme dans les cellules** : une cellule peut porter du Markdown léger, limité à la mise en forme autorisée par la feature 001 (FR-015) : gras, italique, listes, code en ligne et en bloc, liens. Un symbole n'est une mise en forme que s'il en a la forme complète : un astérisque ou un tiret bas isolé, entouré d'espaces, à l'intérieur d'un mot ou sans symbole fermant reste du texte. Ce qui n'est pas autorisé (titres, images, tableaux, citations, balises HTML) reste du texte tel qu'il est écrit. Dans tous les cas, rien ne peut s'exécuter chez un lecteur.
- **Recto déjà présent** : une ligne dont le recto est identique (sans tenir compte de la casse, des accents, des espaces ni de la mise en forme) à celui d'une question du sujet ou d'une autre ligne de la source est signalée comme doublon probable. Ce signalement n'est pas une erreur : il ne bloque pas l'import, et la ligne est importée si l'auteur confirme. Pour l'écarter, il la retire de sa source et recommence.
- **Symbole voulu tel quel** : un auteur qui veut écrire un astérisque ou un tiret bas qui serait sinon lu comme une mise en forme le fait précéder d'une barre oblique inverse (`\*`), comme en Markdown. L'aperçu lui montre le résultat avant toute confirmation.
- **Espaces** : les espaces en début et en fin de cellule sont retirés. Une cellule qui ne contient que des espaces est vide.
- **Retours à la ligne** : un retour à la ligne dans une cellule est conservé dans le recto ou le verso.
- **Fichier XLSX à plusieurs feuilles** : seule la première feuille est lue, et l'aperçu le signale. Une formule est lue par sa valeur affichée. La mise en forme du tableur (couleurs, gras de la cellule) est ignorée.
- **Fichier trop gros** : un fichier de plus de 5 Mo, ou de plus de 2 000 lignes, est refusé avant la lecture, avec un message qui indique la limite.
- **Encodage illisible** : un CSV qui n'est ni en UTF-8 ni en Windows-1252 est lu au mieux. Les caractères impossibles à lire apparaissent dans l'aperçu, pour que l'auteur s'en rende compte avant de confirmer.
- **Le sujet change entre l'aperçu et la confirmation** (autre onglet, administrateur) : à la confirmation, les règles sont vérifiées de nouveau. Si la limite de 500 questions serait dépassée, ou si l'auteur n'a plus le droit de modifier le sujet, rien n'est importé, et un message l'indique.
- **Double confirmation** (double clic, connexion lente) : les questions ne sont ajoutées qu'une fois.
- **Sujet retiré** : son auteur ne peut pas y importer, comme il ne peut plus le modifier. Un administrateur le peut.
- **Hors ligne** : l'import est indisponible, comme toute action qui enregistre (feature 001, FR-039) ; un message l'explique. Une connexion perdue pendant la confirmation n'importe rien à moitié : soit tout est ajouté, soit rien, et l'auteur en est informé.
- **Dernier enregistrement** : un import n'écrase aucune question existante. Il ne fait qu'en ajouter à la fin.

## Requirements *(mandatory)*

### Functional Requirements

**Accès**

- **FR-001**: Le système DOIT permettre d'importer des questions dans un sujet à toute personne qui peut y ajouter une question à la main : son auteur, sauf si le sujet est retiré, et un administrateur, quel que soit le statut du sujet (feature 001, FR-014 et FR-032). Pour toute autre personne, l'import est refusé.
- **FR-002**: L'import DOIT être proposé depuis l'édition d'un sujet, et accepter deux sources : un texte collé, ou un fichier CSV ou XLSX.

**Lecture**

- **FR-003**: Chaque ligne non vide DOIT donner une question : la première colonne en recto, la seconde en verso. Les colonnes suivantes sont ignorées, et l'aperçu le signale.
- **FR-004**: Pour un CSV, le système DOIT reconnaître de lui-même le séparateur (virgule, point-virgule ou tabulation) et l'encodage (UTF-8, avec ou sans marque d'en-tête, ou Windows-1252), et lire correctement les cellules entre guillemets, y compris celles qui contiennent un séparateur ou un retour à la ligne.
- **FR-005**: Un texte collé DOIT être lu comme des colonnes séparées par des tabulations, avec les mêmes règles de guillemets et de retours à la ligne que celles des tableurs et des exports Anki et Quizlet.
- **FR-006**: Pour un XLSX, le système DOIT lire la première feuille seulement, et prendre la valeur affichée de chaque cellule.
- **FR-007**: Une première ligne dont les deux cellules sont « recto » et « verso », ou « question » et « réponse » (sans tenir compte de la casse ni des accents), DOIT être reconnue comme un en-tête et ignorée.
- **FR-008**: Le système DOIT ignorer les lignes entièrement vides, et retirer les espaces en début et en fin de cellule.
- **FR-009**: Le système DOIT refuser un fichier qui n'est pas un CSV ou un XLSX lisible, de plus de 5 Mo, ou de plus de 2 000 lignes, avec un message qui indique les formats et les limites.

**Mise en forme**

- **FR-024**: Le système DOIT interpréter dans chaque cellule le Markdown léger correspondant à la mise en forme autorisée par la feature 001 (FR-015) : gras, italique, listes, code en ligne et en bloc, liens. Toute autre syntaxe (titres, images, tableaux, citations, HTML) DOIT être conservée comme du texte, telle qu'elle est écrite.
- **FR-025**: Un symbole de mise en forme NE DOIT être interprété que s'il en a la forme complète, ouvrant et fermant. Un astérisque ou un tiret bas isolé, entouré d'espaces, placé à l'intérieur d'un mot ou sans symbole fermant, DOIT rester du texte. Une barre oblique inverse devant un symbole DOIT le garder tel quel.
- **FR-027**: L'aperçu DOIT afficher le recto et le verso déjà mis en forme, exactement comme ils s'afficheront une fois importés, après le nettoyage de la feature 001 (FR-015).

**Aperçu et validation**

- **FR-010**: Avant tout enregistrement, le système DOIT afficher un aperçu : pour chaque question lue, son numéro de ligne dans la source, son recto et son verso, ainsi que le nombre de questions qui seront ajoutées.
- **FR-011**: L'aperçu DOIT signaler, ligne par ligne, chaque recto ou verso vide et chaque cellule de plus de 5 000 caractères. Il DOIT aussi signaler un dépassement de la limite de 500 questions par sujet, en indiquant combien de questions le sujet peut encore recevoir.
- **FR-012**: La confirmation DOIT être impossible tant que l'aperçu contient une erreur ou aucune question.
- **FR-026**: L'aperçu DOIT signaler, sans en faire une erreur, chaque ligne dont le recto est identique, sans tenir compte de la casse, des accents, des espaces ni de la mise en forme, à celui d'une question du sujet ou d'une autre ligne de la source, en indiquant laquelle. Ces lignes restent importables.
- **FR-013**: Abandonner l'aperçu NE DOIT rien modifier dans le sujet.

**Enregistrement**

- **FR-014**: À la confirmation, le système DOIT vérifier de nouveau les droits et les limites, puis ajouter toutes les questions ou aucune. Un import ne laisse jamais un sujet à moitié rempli.
- **FR-015**: Les questions importées DOIVENT être ajoutées à la fin du sujet, dans l'ordre de la source, sans modifier ni réordonner les questions existantes.
- **FR-016**: Une question importée DOIT être en tout point une question ordinaire : modifiable, déplaçable et supprimable ensuite comme les autres, et soumise au même nettoyage de contenu (feature 001, FR-015 ; constitution VI).
- **FR-017**: Une même confirmation NE DOIT ajouter les questions qu'une seule fois, même si elle est envoyée plusieurs fois.
- **FR-018**: Après l'import, le système DOIT indiquer le nombre de questions ajoutées.

**Révision**

- **FR-019**: Dans un sujet appris par au moins un inscrit, l'aperçu DOIT avertir l'auteur que les questions importées entreront en boîte 1 chez les personnes qui apprennent ce sujet, à réviser dès le jour même.
- **FR-020**: Chaque question importée DOIT entrer dans la progression des apprenants exactement comme une question ajoutée à la main (feature 001, FR-051), sans changer la boîte des autres cartes.

**Aide**

- **FR-021**: Le système DOIT proposer un modèle de fichier XLSX à télécharger, avec l'en-tête et deux exemples de questions, et une courte explication du format attendu.

**Transverse**

- **FR-022**: Tous les textes DOIVENT être en français, en vouvoyant.
- **FR-023**: L'import par texte collé et l'aperçu DOIVENT être utilisables sur un téléphone de 360 px de large, sans défilement horizontal.

### Key Entities

- **Source d'import** : un texte collé ou un fichier (CSV ou XLSX), lu le temps de l'aperçu. Elle n'est pas conservée après l'import.
- **Ligne lue** : pour l'aperçu, son numéro dans la source, un recto et un verso mis en forme, ses éventuelles erreurs, qui bloquent, et ses éventuels avertissements (doublon probable, colonnes ignorées), qui ne bloquent pas. Elle devient une question à la confirmation.
- **Question** (existante, feature 001) : une question importée ne se distingue en rien d'une question saisie à la main.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Un auteur ajoute 100 questions à un sujet depuis un fichier préparé en moins de 2 minutes, contre plus d'une heure en saisie une par une.
- **SC-002**: Un fichier produit par Excel en français, par LibreOffice ou par Google Sheets, au format CSV comme XLSX, s'importe avec 100 % des accents corrects et des colonnes bien séparées. Ceci est vérifié par des tests sur chacun de ces formats.
- **SC-003**: Aucun import ne laisse un sujet à moitié rempli : dans 100 % des cas d'erreur (ligne invalide, limite dépassée, droits perdus, connexion coupée), le sujet est identique à ce qu'il était avant. Ceci est vérifié par des tests.
- **SC-004**: L'aperçu d'un fichier de 500 lignes s'affiche en moins de 3 secondes.
- **SC-005**: 90 % des auteurs qui testent l'import réussissent leur premier import sans aide, en partant du modèle ou d'un tableur existant.
- **SC-006**: Aucun contenu importé ne s'exécute chez un lecteur. Ceci est vérifié par des tests qui importent des balises et des scripts.
- **SC-007**: Aucun symbole ordinaire n'est pris pour une mise en forme : 100 % des cas d'un jeu d'essai de textes courants (calculs, noms avec tiret bas, prix, symboles isolés, symboles sans fermeture) s'affichent tels qu'ils sont écrits. Ceci est vérifié par des tests.

## Assumptions

- **Dépend des features 001 et 006** : sujets, questions, limites et nettoyage du contenu, droits de l'auteur et de l'administrateur, progression Leitner (001) ; actions indisponibles hors ligne (001 FR-039, 006).
- **Hors périmètre** : les images (feature 003), l'export de questions, l'import de plusieurs sujets à la fois, la création d'un sujet à partir d'un fichier (l'import se fait dans un sujet existant), le format natif des fichiers Anki (`.apkg`), la mise à jour de questions existantes par import, et le remplissage du catalogue par l'équipe au lancement.
- **Formats gratuits** : CSV et XLSX sont produits par Google Sheets et LibreOffice, gratuits, ainsi que par Excel. Le format ODS de LibreOffice n'est pas accepté. LibreOffice enregistre aussi en XLSX et en CSV.
- **Modèle en XLSX** : un CSV ouvert dans Excel en français casse les accents et les colonnes, alors qu'un XLSX s'ouvre correctement dans les trois tableurs.
- **Limites de fichier** (5 Mo, 2 000 lignes) : choix par défaut. 500 questions de 2 × 5 000 caractères tiennent dans 5 Mo, et 2 000 lignes laissent de la place aux lignes vides et aux erreurs, sans permettre de faire tourner l'aperçu sur un fichier démesuré.
- **Tout ou rien** : choix du développeur. Un import partiel obligerait l'auteur à retrouver, dans son sujet, ce qui est passé et ce qui ne l'est pas.
- **Markdown léger** (FR-024, FR-025) : choix du développeur, le 2026-10-01. Les sujets techniques ont besoin du code et du gras. Pour éviter les erreurs d'interprétation, seule une forme complète est interprétée, et l'aperçu montre le rendu avant toute confirmation.
- **Doublons signalés sans bloquer** (FR-026) : choix du développeur, le 2026-10-01. Deux questions au même recto peuvent avoir des versos différents et être voulues. Les écarter d'office ferait perdre du contenu, et les importer sans rien dire laisserait passer un fichier réimporté par erreur.
- **Pas d'annulation** : un import confirmé n'est pas annulable d'un geste. L'auteur supprime les questions une par une, comme toute question.
