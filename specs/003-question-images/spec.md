# Feature Specification: Images dans les questions

**Feature Branch**: `003-question-images`

**Created**: 2026-09-29

**Status**: Draft

**Input**: User description: "Feature 003 — Images dans les questions. Aujourd'hui (feature 001), une question est une carte avec un recto et un verso en texte mis en forme. On permet à l'auteur d'un sujet d'ajouter des images au RECTO d'une question uniquement (l'image sert de question : « qu'est-ce que c'est ? », le verso reste du texte). Formats acceptés : JPEG, PNG, WebP ; pas de GIF animé ni de SVG. Au plus 5 Mo par image, au plus 4 images sur le recto. Un texte alternatif (description de l'image) est obligatoire pour chaque image : impossible d'enregistrer une image sans lui. Les images s'affichent partout où le recto s'affiche : lecture publique d'un sujet, séance de révision Leitner, édition par l'auteur, et dans la PWA sur mobile. La modération reste au niveau du sujet (signalement du sujet, retrait par un administrateur) : un sujet retiré, dépublié ou en brouillon ne doit pas exposer ses images à qui ne peut pas le voir. Supprimer une image, une question ou un sujet supprime définitivement les images concernées. Remplacer une image d'une question apprise ne change pas sa boîte Leitner (une question modifiée garde sa boîte, cf. 001). Point encore ouvert : les images s'insèrent-elles dans le texte mis en forme du recto à l'endroit choisi, ou s'affichent-elles en bloc au-dessus du texte ? Hors périmètre : images au verso, GIF/vidéo/audio, retrait d'une image seule par un administrateur, filtrage automatique des images, édition d'image (recadrage, rotation)."

## Contexte

Dans CINQ (feature 001), une question est une carte : un recto (la question) et un verso (la réponse), tous deux en texte mis en forme. Beaucoup de sujets s'apprennent pourtant d'abord par l'image : reconnaître un tableau, un oiseau, un panneau, une molécule, un drapeau, une région sur une carte. Aujourd'hui, l'auteur doit décrire en mots ce qu'il voudrait montrer, et la carte perd l'essentiel de son intérêt.

Cette feature permet à l'auteur de placer des images sur le recto d'une question. L'image devient la question — « Qu'est-ce que c'est ? » — et le verso reste la réponse en texte. Chaque image est accompagnée d'une description, pour que la carte reste utilisable par une personne qui ne la voit pas. Les images suivent exactement la visibilité de leur sujet : ce qu'on ne peut pas lire, on ne peut pas non plus en voir les images.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Ajouter des images au recto d'une question (Priority: P1)

En rédigeant ou en modifiant une question, l'auteur ajoute une ou plusieurs images au recto, depuis son ordinateur ou depuis la galerie ou l'appareil photo de son téléphone. Pour chaque image, il saisit une description. Il peut changer l'ordre des images, en remplacer une ou en retirer une, puis il enregistre la question.

**Why this priority**: Sans ajout d'images, rien d'autre de la feature n'existe.

**Independent Test**: Un auteur ouvre un brouillon, ajoute 2 images avec leur description au recto d'une question, inverse leur ordre, enregistre, recharge la page et retrouve les 2 images dans le nouvel ordre avec leurs descriptions.

**Acceptance Scenarios**:

1. **Given** un auteur qui rédige une question, **When** il ajoute une image JPEG de 2 Mo et saisit sa description, **Then** un aperçu de l'image s'affiche sur le recto, et la question s'enregistre avec cette image.
2. **Given** une image ajoutée sans description, **When** l'auteur tente d'enregistrer la question, **Then** l'enregistrement est refusé, avec un message placé sur l'image concernée qui demande de la décrire.
3. **Given** un recto qui porte déjà 4 images, **When** l'auteur tente d'en ajouter une cinquième, **Then** l'ajout est refusé, avec un message qui indique la limite de 4 images.
4. **Given** un auteur qui choisit un fichier de 7 Mo, un GIF, un SVG ou un fichier qui n'est pas une image, **When** il tente de l'ajouter, **Then** le fichier est refusé avant d'être envoyé, avec un message qui indique les formats acceptés (JPEG, PNG, WebP) et la taille maximale (5 Mo).
5. **Given** une question dont le recto porte 3 images, **When** l'auteur déplace la troisième en première position et enregistre, **Then** le nouvel ordre est conservé et c'est lui qui s'affiche aux lecteurs.
6. **Given** une question enregistrée avec une image, **When** l'auteur remplace cette image par une autre et enregistre, **Then** seule la nouvelle image s'affiche, et l'ancienne n'est plus accessible nulle part.
7. **Given** une question enregistrée avec 2 images, **When** l'auteur en retire une et enregistre, **Then** la question ne porte plus qu'une image, et l'image retirée n'est plus accessible nulle part.
8. **Given** un auteur sur son téléphone, **When** il ajoute une image, **Then** il peut la choisir dans sa galerie ou la prendre avec l'appareil photo.
9. **Given** un auteur qui rédige une question, **When** il ajoute une image avec sa description, laisse le texte du recto vide, remplit le verso et enregistre, **Then** la question s'enregistre comme une carte « image seule ».
10. **Given** une question « image seule », **When** l'auteur retire sa dernière image et tente d'enregistrer, **Then** l'enregistrement est refusé, avec un message qui demande un texte ou une image au recto.
11. **Given** un administrateur qui modifie une question d'un sujet dont il n'est pas l'auteur (feature 001), **When** il ajoute, remplace ou retire une image, **Then** la modification est enregistrée comme pour l'auteur.

---

### User Story 2 - Voir les images en lisant et en révisant (Priority: P1)

Un lecteur qui consulte un sujet publié voit les images au recto des questions. Un apprenant en séance de révision voit l'image du recto, cherche la réponse, puis retourne la carte. Sur son téléphone, il peut agrandir une image pour en voir les détails. Une personne qui utilise un lecteur d'écran entend la description de chaque image.

**Why this priority**: C'est la valeur de la feature pour l'apprenant ; une image que l'on ne voit qu'en édition ne sert à rien.

**Independent Test**: Un visiteur non connecté ouvre un sujet publié dont une question porte 2 images et les voit au recto. Un inscrit qui apprend ce sujet lance une séance, voit les 2 images au recto de la carte, en agrandit une en plein écran, puis affiche le verso et répond.

**Acceptance Scenarios**:

1. **Given** un sujet publié dont une question porte 2 images au recto, **When** un visiteur l'ouvre, **Then** il voit les 2 images au recto de cette question, dans l'ordre choisi par l'auteur.
2. **Given** une séance de révision dont une carte porte une image au recto, **When** la carte s'affiche, **Then** l'image est visible avant que le verso ne soit affiché.
3. **Given** un apprenant sur un écran de 360 px de large, **When** une carte à image s'affiche, **Then** l'image tient dans la largeur de l'écran sans défilement horizontal, et les boutons de réponse restent accessibles.
4. **Given** une image affichée sur un recto, **When** le lecteur la touche ou clique dessus, **Then** elle s'affiche en grand, et il peut revenir à la carte en un geste ou avec la touche Échap.
5. **Given** un lecteur qui utilise un lecteur d'écran, **When** il arrive sur une image du recto, **Then** la description saisie par l'auteur lui est lue.
6. **Given** une image qui ne se charge pas (connexion lente ou interrompue), **When** la carte s'affiche, **Then** la description de l'image apparaît à sa place, et la séance peut continuer.
7. **Given** une question sans image, **When** elle s'affiche, **Then** son recto est identique à ce qu'il était avant cette feature.

---

### User Story 3 - Des images qui suivent la visibilité de leur sujet (Priority: P1)

Une image n'est jamais visible par quelqu'un qui ne peut pas lire son sujet. Tant que le sujet est un brouillon, seuls son auteur et les administrateurs en voient les images. Un sujet dépublié, retiré ou supprimé cesse d'exposer les siennes. Une image supprimée disparaît définitivement.

**Why this priority**: La constitution exige qu'un brouillon ou un sujet retiré soit introuvable pour qui n'y a pas droit (principe VI). Une image restée accessible par son adresse suffirait à rompre cette règle, et à laisser en ligne un contenu retiré par la modération.

**Independent Test**: Un auteur ajoute une image à un brouillon et note l'adresse de l'image. Un visiteur, puis un autre inscrit, ouvrent cette adresse et n'obtiennent rien. L'auteur publie le sujet : le visiteur voit l'image. Un administrateur retire le sujet : l'adresse ne répond plus pour le visiteur.

**Acceptance Scenarios**:

1. **Given** une image sur un brouillon, **When** une personne qui n'est ni l'auteur ni un administrateur ouvre l'adresse de l'image, **Then** elle n'obtient rien, exactement comme pour une image qui n'existe pas.
2. **Given** un sujet publié dont les images sont visibles, **When** l'auteur le dépublie, **Then** ses images ne sont plus visibles que de l'auteur et des administrateurs.
3. **Given** un sujet publié, **When** un administrateur le retire, **Then** ses images ne sont plus visibles que de l'auteur (en lecture, avec le motif) et des administrateurs.
4. **Given** un sujet retiré puis rétabli et republié, **When** un visiteur l'ouvre, **Then** il voit de nouveau ses images.
5. **Given** un sujet supprimé, une question supprimée ou une image retirée, **When** quiconque, administrateur compris, ouvre l'adresse d'une de ces images, **Then** il n'obtient rien, et l'image n'est plus conservée.
6. **Given** un auteur qui ajoute une image puis abandonne la question sans l'enregistrer, **When** l'abandon est constaté, **Then** l'image n'est rattachée à aucune question, n'est visible de personne d'autre que lui, et n'est pas conservée.

---

### Edge Cases

- **Question apprise dont on change les images** : ajouter, remplacer, réordonner ou retirer une image est une modification de la question ; elle garde sa boîte et sa date de prochaine révision pour tous ceux qui l'apprennent (feature 001).
- **Recto sans texte** : un recto qui porte au moins une image peut avoir un texte vide ; la carte est alors une « image seule ». Si l'auteur retire la dernière image d'un recto sans texte, l'enregistrement est refusé avec un message qui demande un texte ou une image.
- **Emplacement des images sur le recto** : les images s'affichent en bloc, au-dessus du texte du recto, jamais au milieu du texte mis en forme.
- **Envoi interrompu** (perte de connexion pendant l'ajout) : l'auteur est prévenu que l'image n'a pas été envoyée, les autres éléments de sa saisie restent à l'écran, et il peut réessayer. Hors ligne, l'ajout d'image est suspendu comme les autres actions qui enregistrent (feature 001, FR-039).
- **Envoi lent** : pendant l'envoi, l'image affiche sa progression ; la question ne peut pas être enregistrée tant qu'un envoi est en cours, et l'auteur peut annuler un envoi.
- **Fichier dont l'extension ment** (un autre contenu renommé en `.jpg`) ou image illisible : il est refusé avec le message des formats acceptés.
- **Image aux dimensions démesurées** : une image de plus de 8 000 pixels de côté est refusée, avec un message qui indique la limite.
- **Photo prise au téléphone** : elle s'affiche dans le bon sens, quelle que soit l'orientation de l'appareil au moment de la prise de vue.
- **Informations cachées dans le fichier** : la position GPS, l'appareil et la date de prise de vue qu'une photo contient ne sont jamais transmis aux lecteurs.
- **Description** : de 1 à 250 caractères, en texte simple, sans mise en forme. Un dépassement est refusé avec un message qui indique la limite.
- **Image venant d'ailleurs** : le texte mis en forme du recto ou du verso ne peut pas afficher une image hébergée sur un autre site ; seules les images envoyées sur CINQ s'affichent.
- **Deux onglets modifient la même question** : le dernier enregistrement l'emporte, images comprises (feature 001) ; les images qu'il ne conserve pas ne sont plus accessibles.
- **Signalement pour droits d'auteur** : le motif « droits d'auteur » de la feature 001 couvre les images ; l'administrateur retire le sujet entier.
- **Recherche** : la recherche par mot-clé ne porte pas sur les descriptions d'images.
- **Question dupliquée ou sujet copié** : hors périmètre, aucune de ces actions n'existe dans CINQ.

## Requirements *(mandatory)*

### Functional Requirements

**Ajout et gestion des images**

- **FR-001**: Le système DOIT permettre à l'auteur d'un sujet, et à un administrateur, d'ajouter des images au recto d'une question, quel que soit le statut du sujet. Le verso ne peut pas porter d'image.
- **FR-002**: Le système DOIT accepter uniquement les images JPEG, PNG et WebP, d'au plus 5 Mo et d'au plus 8 000 pixels de côté, et refuser tout autre fichier avec un message qui indique les formats et les limites acceptés. Le contrôle porte sur le contenu réel du fichier, pas sur son nom.
- **FR-003**: Un recto NE DOIT pas porter plus de 4 images.
- **FR-004**: Chaque image DOIT avoir une description de 1 à 250 caractères, en texte simple. Une question dont une image n'a pas de description NE DOIT pas pouvoir être enregistrée.
- **FR-005**: Le système DOIT permettre à l'auteur de réordonner, de remplacer et de retirer les images d'un recto ; l'ordre choisi est celui qui s'affiche aux lecteurs.
- **FR-006**: Sur téléphone, le système DOIT permettre de choisir l'image dans la galerie ou de la prendre avec l'appareil photo.
- **FR-007**: Pendant l'envoi d'une image, le système DOIT afficher sa progression, permettre de l'annuler, et empêcher l'enregistrement de la question tant qu'un envoi est en cours. Un envoi échoué DOIT être signalé sans perdre le reste de la saisie.
- **FR-008**: Un recto DOIT porter du texte (1 à 5 000 caractères, feature 001) ou au moins une image, ou les deux. Un recto sans texte ni image NE DOIT pas pouvoir être enregistré. Le verso reste obligatoire.

**Affichage**

- **FR-009**: Les images DOIVENT s'afficher, dans l'ordre choisi, partout où le recto s'affiche : lecture d'un sujet, séance de révision, édition, et à 360 px de large sans défilement horizontal.
- **FR-010**: Les images d'un recto DOIVENT s'afficher en bloc, au-dessus du texte mis en forme, et non à l'intérieur de celui-ci.
- **FR-011**: Le lecteur DOIT pouvoir afficher une image en grand, puis revenir à la carte en un geste, un clic ou avec la touche Échap.
- **FR-012**: La description de chaque image DOIT être restituée par les lecteurs d'écran, et affichée à la place de l'image quand celle-ci ne se charge pas.
- **FR-013**: Une image DOIT s'afficher dans le bon sens, et être servie dans une taille adaptée à l'écran qui l'affiche, pour limiter les données consommées sur mobile.
- **FR-014**: Les informations contenues dans un fichier image (position, appareil, date de prise de vue) NE DOIVENT jamais être transmises aux lecteurs.
- **FR-015**: Le texte mis en forme d'un recto ou d'un verso NE DOIT pas pouvoir afficher une image hébergée ailleurs que sur CINQ.

**Visibilité et suppression**

- **FR-016**: Une image DOIT être visible exactement des personnes qui peuvent lire la question qui la porte : tout le monde pour un sujet publié, l'auteur et les administrateurs pour un brouillon, un sujet dépublié ou un sujet retiré (feature 001, FR-023). Pour toute autre personne, une image est introuvable, y compris à son adresse directe, et rien ne la distingue d'une image qui n'existe pas.
- **FR-017**: Retirer ou remplacer une image, supprimer une question ou supprimer un sujet DOIT supprimer définitivement les images concernées ; elles ne sont plus accessibles à personne, administrateurs compris.
- **FR-018**: Une image envoyée mais jamais rattachée à une question enregistrée NE DOIT être visible que de la personne qui l'a envoyée, et NE DOIT pas être conservée au-delà de 24 heures.

**Révision**

- **FR-019**: Ajouter, remplacer, réordonner ou retirer une image DOIT être traité comme une modification de la question : sa boîte et sa date de prochaine révision ne changent pour aucun apprenant (feature 001, FR-051).

### Key Entities

- **Image de question** : une image rattachée au recto d'une question, avec sa description, sa position parmi les images du recto, son format, ses dimensions et sa date d'envoi. Elle hérite de la visibilité du sujet de sa question et disparaît avec elle.
- **Image en attente** : une image envoyée par une personne mais pas encore rattachée à une question enregistrée ; elle n'est visible que de cette personne et expire après 24 heures.
- **Question (carte)** *(feature 001, étendue)* : son recto porte désormais, en plus du texte mis en forme, de 0 à 4 images ordonnées.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Un auteur ajoute une photo et sa description au recto d'une question en moins de 30 secondes, depuis un téléphone comme depuis un ordinateur.
- **SC-002**: Une image de 5 Mo est envoyée en moins de 10 secondes sur une connexion mobile 4G ordinaire.
- **SC-003**: Sur une connexion mobile 4G ordinaire, l'image d'une carte en séance s'affiche en moins de 2 secondes, et la page d'une carte à image ne consomme pas plus de 500 Ko de données sur un écran de 360 px.
- **SC-004**: 100 % des cas de visibilité de FR-016 et FR-017 sont couverts par des tests : pour chaque statut de sujet (brouillon, publié, dépublié, retiré, supprimé) et chaque profil (visiteur, autre inscrit, auteur, administrateur), l'accès à l'image par son adresse directe donne le résultat attendu.
- **SC-005**: Aucune image affichée à un lecteur ne contient d'information de position, d'appareil ou de date de prise de vue.
- **SC-006**: 100 % des images publiées ont une description.
- **SC-007**: La part des sujets publiés qui contiennent au moins une image est mesurable, pour juger de l'usage de la feature après sa mise en service.

## Assumptions

- **Dépend de la feature 001** : sujets et leurs statuts, questions à recto et verso mis en forme, droits de l'auteur et de l'administrateur, visibilité des brouillons et des sujets retirés, séance de révision, bandeau hors ligne.
- **Hors périmètre** : les images au verso, les GIF animés, les SVG, la vidéo et l'audio, le retrait d'une image seule par un administrateur, le filtrage automatique des images inappropriées, l'édition d'image (recadrage, rotation, filtres), la recherche dans les descriptions d'images, l'affichage hors ligne des images.
- **Modération** : elle reste au niveau du sujet. On signale un sujet, pas une image, et le retrait d'un sujet masque toutes ses images.
- **Droits sur les images** : l'auteur est responsable des images qu'il envoie ; le motif de signalement « droits d'auteur » de la feature 001 permet de traiter les abus.
- **Pas de quota par auteur** : les limites sont de 4 images par recto et de 500 questions par sujet (feature 001). Un quota de stockage par compte pourra être ajouté par une feature ultérieure si les volumes l'exigent.
- **Limite de description** : 250 caractères suffisent à décrire une image sans réécrire la question.
- **Images en attente** : 24 heures laissent à un auteur le temps de finir une question commencée, sans garder indéfiniment des fichiers orphelins.
- **Données personnelles** : les images d'un auteur sont supprimées avec ses sujets ; leur sort à la suppression du compte sera fixé par la feature de suppression de compte, prévue avant l'ouverture au public.
