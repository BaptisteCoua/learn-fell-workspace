# Feature Specification: Comptes et contenu d'apprentissage

**Feature Branch**: `001-learning-content`

**Created**: 2026-09-24

**Status**: Draft

**Input**: User description: "Espace public d'apprentissage (Learn Fell) — feature 001 : comptes et contenu. Visiteurs, inscrits (auteurs) et administrateurs. Inscription par email et mot de passe. Catégories à un niveau gérées par les administrateurs, tags libres posés par l'auteur. Sujets en brouillon puis publiés, contenant des questions recto/verso en texte mis en forme. Lecture libre sans compte, parcours par catégorie et recherche. Signalement des sujets, traité par les administrateurs. La révision Leitner est hors périmètre (feature 002)."

## Contexte

Learn Fell est un espace public où chacun peut apprendre n'importe quel sujet par la méthode Leitner. Cette première feature pose les fondations : des comptes, et un catalogue de sujets composés de questions et de réponses, que la communauté écrit et que tout le monde peut consulter.

La révision Leitner elle-même (5 boîtes, intervalles de 1, 2, 4, 8 et 16 jours, auto-évaluation « je savais / je ne savais pas ») fera l'objet de la feature 002. Ici, le contenu est seulement structuré pour l'accueillir : chaque question est une carte avec un recto (la question) et un verso (la réponse).

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Consulter le catalogue sans compte (Priority: P1)

Un visiteur arrive sur Learn Fell sans compte. Il parcourt les catégories, ouvre un sujet publié et lit ses questions et leurs réponses. Il peut aussi chercher un sujet par mot-clé.

**Why this priority**: C'est la vitrine du produit. Sans contenu lisible, il n'y a aucune raison de créer un compte. Cette story se démontre à partir d'un contenu préparé par un administrateur.

**Independent Test**: Avec des sujets publiés et des brouillons préparés à l'avance, un visiteur non connecté peut trouver un sujet par catégorie et par recherche, puis en lire toutes les questions et réponses, sans jamais voir un brouillon.

**Acceptance Scenarios**:

1. **Given** la catégorie « Langues » contient 3 sujets publiés et 1 brouillon, **When** un visiteur ouvre cette catégorie, **Then** il voit les 3 sujets publiés, avec pour chacun son titre, sa description, ses tags, son auteur et son nombre de questions, et il ne voit pas le brouillon.
2. **Given** un sujet publié de 20 questions, **When** un visiteur l'ouvre, **Then** il voit les 20 questions dans l'ordre défini par l'auteur, et peut afficher la réponse de chacune.
3. **Given** un sujet publié intitulé « Verbes irréguliers anglais » avec le tag « grammaire », **When** un visiteur cherche « irreguliers » ou « grammaire », **Then** ce sujet apparaît dans les résultats.
4. **Given** un brouillon, **When** un visiteur tente d'y accéder par son adresse directe, **Then** il reçoit une page « contenu introuvable », sans indication que le sujet existe.
5. **Given** une recherche sans aucun résultat, **When** le visiteur la lance, **Then** un message l'indique et l'invite à parcourir les catégories.

---

### User Story 2 - Créer un compte et se connecter (Priority: P1)

Un visiteur crée un compte avec son adresse email et un mot de passe, se connecte, se déconnecte, et peut réinitialiser son mot de passe s'il l'a oublié.

**Why this priority**: C'est un prérequis à toute création de contenu (story 3), au signalement (story 5) et à la révision Leitner de la feature 002.

**Independent Test**: Créer un compte, confirmer l'email, se déconnecter, se reconnecter, puis réinitialiser le mot de passe via le lien reçu par email.

**Acceptance Scenarios**:

1. **Given** un visiteur, **When** il s'inscrit avec un nom affiché, une adresse email valide et un mot de passe conforme, **Then** son compte est créé, il est connecté, et il reçoit un email de confirmation.
2. **Given** un compte existe déjà avec cette adresse email, **When** un visiteur tente de s'inscrire avec la même adresse, **Then** l'inscription est refusée avec un message qui l'invite à se connecter ou à réinitialiser son mot de passe.
3. **Given** un utilisateur inscrit, **When** il saisit un mauvais mot de passe, **Then** la connexion est refusée avec un message générique qui ne révèle pas si l'adresse email existe.
4. **Given** 5 échecs de connexion consécutifs sur un même compte, **When** une sixième tentative est faite, **Then** elle est bloquée temporairement et un message indique quand réessayer.
5. **Given** un utilisateur a oublié son mot de passe, **When** il demande une réinitialisation, **Then** il reçoit un lien valable 60 minutes et à usage unique, et le message affiché est le même que l'adresse existe ou non.
6. **Given** un utilisateur connecté, **When** il se déconnecte, **Then** il ne peut plus accéder aux pages réservées aux inscrits sans se reconnecter.

---

### User Story 3 - Créer et publier un sujet (Priority: P1)

Un utilisateur inscrit crée un sujet : un titre, une description, une catégorie et des tags. Il y ajoute des questions, chacune avec un recto et un verso mis en forme, les réordonne, les modifie, puis publie le sujet quand il est prêt. Il peut le dépublier plus tard.

**Why this priority**: C'est le cœur de la valeur communautaire : sans auteurs, le catalogue reste vide.

**Independent Test**: Un inscrit crée un brouillon, y ajoute 3 questions, vérifie qu'un visiteur ne le voit pas, le publie, vérifie qu'un visiteur le voit, puis le dépublie.

**Acceptance Scenarios**:

1. **Given** un utilisateur inscrit, **When** il crée un sujet avec un titre, une description, une catégorie et 2 tags, **Then** le sujet est enregistré en brouillon et lui seul peut le voir.
2. **Given** un brouillon, **When** l'auteur ajoute une question dont le recto contient du gras et le verso une liste et un bloc de code, **Then** la mise en forme s'affiche telle quelle à la lecture.
3. **Given** un sujet de 5 questions, **When** l'auteur déplace la question 5 en première position, **Then** le nouvel ordre est conservé et c'est lui qui s'affiche aux lecteurs.
4. **Given** un brouillon sans aucune question, **When** l'auteur tente de le publier, **Then** la publication est refusée avec un message qui demande d'ajouter au moins une question.
5. **Given** un brouillon d'au moins une question, **When** l'auteur le publie, **Then** il apparaît dans sa catégorie, dans la recherche, et à l'adresse directe du sujet.
6. **Given** un sujet publié, **When** l'auteur modifie une question, **Then** la modification est visible des lecteurs dès l'enregistrement.
7. **Given** un sujet publié, **When** l'auteur le dépublie, **Then** il redevient un brouillon, invisible de tous sauf de lui.
8. **Given** un sujet publié dont il ne reste qu'une question, **When** l'auteur tente de supprimer cette dernière question, **Then** la suppression est refusée tant que le sujet est publié.
9. **Given** l'auteur saisit un tag déjà utilisé ailleurs, **When** il tape les premières lettres, **Then** les tags existants correspondants lui sont proposés.
10. **Given** un utilisateur inscrit, **When** il ouvre la liste « Mes sujets », **Then** il voit tous ses sujets (brouillons, publiés, retirés) avec leur statut.

---

### User Story 4 - Gérer les catégories (Priority: P2)

Un administrateur crée, renomme, réordonne et supprime les catégories dans lesquelles les auteurs rangent leurs sujets.

**Why this priority**: Sans catégorie, aucun sujet ne peut être créé. En revanche, un jeu initial de catégories peut être fourni à la mise en service, ce qui rend cette story moins urgente que les stories 1 à 3.

**Independent Test**: Un administrateur crée une catégorie, la renomme, vérifie qu'elle est proposée aux auteurs, puis tente de la supprimer alors qu'elle contient des sujets.

**Acceptance Scenarios**:

1. **Given** un administrateur, **When** il crée la catégorie « Histoire », **Then** elle apparaît dans la liste des catégories publique et dans le choix proposé aux auteurs.
2. **Given** une catégorie du même nom existe déjà (sans tenir compte de la casse ni des accents), **When** un administrateur en crée une autre sous ce nom, **Then** la création est refusée.
3. **Given** une catégorie qui contient au moins un sujet, quel que soit son statut, **When** un administrateur tente de la supprimer, **Then** la suppression est refusée et le nombre de sujets concernés est affiché.
4. **Given** un utilisateur qui n'est pas administrateur, **When** il tente de créer, modifier ou supprimer une catégorie, **Then** l'action est refusée.

---

### User Story 5 - Signaler un sujet et modérer (Priority: P2)

Un utilisateur inscrit signale un sujet publié qu'il juge inapproprié, en donnant un motif. Les administrateurs traitent les signalements depuis une file : ils ignorent un signalement ou retirent le sujet. Un administrateur peut aussi modifier ou retirer directement n'importe quel sujet.

**Why this priority**: Un contenu public écrit par la communauté doit pouvoir être modéré. Au lancement, le volume de contenu reste faible, d'où la priorité P2.

**Independent Test**: Un inscrit signale un sujet. Un administrateur voit le signalement dans la file et retire le sujet. Le sujet disparaît du catalogue, et son auteur voit qu'il a été retiré, avec le motif.

**Acceptance Scenarios**:

1. **Given** un utilisateur inscrit qui lit un sujet publié d'un autre auteur, **When** il le signale en choisissant un motif (contenu inapproprié, contenu erroné, spam, droits d'auteur, autre) et, s'il le souhaite, un commentaire, **Then** le signalement est enregistré, le sujet reste visible, et l'utilisateur voit une confirmation.
2. **Given** un utilisateur a déjà un signalement en attente sur un sujet, **When** il tente de signaler à nouveau ce sujet, **Then** l'action est refusée avec un message qui indique que son signalement est déjà en cours d'examen.
3. **Given** un auteur, **When** il consulte son propre sujet, **Then** l'option de signalement ne lui est pas proposée.
4. **Given** des signalements en attente, **When** un administrateur ouvre la file de modération, **Then** il voit les sujets signalés, regroupés par sujet, avec le nombre de signalements, les motifs, les commentaires et la date du plus ancien signalement, du plus ancien au plus récent.
5. **Given** un sujet signalé, **When** l'administrateur ignore les signalements, **Then** ils sont clos, le sujet reste publié, et il quitte la file.
6. **Given** un sujet signalé, **When** l'administrateur retire le sujet en saisissant un motif, **Then** le sujet n'est plus visible du public, tous ses signalements en attente sont clos, et son auteur voit le statut « retiré » et le motif dans « Mes sujets ».
7. **Given** un sujet retiré, **When** son auteur tente de le republier, **Then** l'action est refusée. Seul un administrateur peut rétablir le sujet.
8. **Given** un administrateur, **When** il modifie une question d'un sujet dont il n'est pas l'auteur, **Then** la modification est enregistrée.

---

### Edge Cases

- **Un auteur supprime un sujet** : la suppression est définitive, après confirmation. Le sujet et ses questions disparaissent partout, et ses signalements en attente sont clos.
- **Deux onglets modifient le même sujet** : le dernier enregistrement l'emporte. Une modification de question ne touche que cette question, et n'écrase pas les autres.
- **Contenu mis en forme malveillant** (script, lien piégé, balise non prévue) : seule la mise en forme autorisée est conservée (gras, italique, listes, code, liens). Aucun contenu saisi ne peut s'exécuter chez un lecteur.
- **Limites de taille** : titre de 3 à 120 caractères, description de 2 000 caractères maximum, 10 tags maximum par sujet, de 30 caractères chacun, recto et verso de 1 à 5 000 caractères chacun, 500 questions maximum par sujet. Un dépassement est refusé avec un message qui indique la limite.
- **Tags** : ils sont normalisés en minuscules et sans espaces superflus, si bien que « Grammaire » et « grammaire  » donnent le même tag. Un tag en double sur un même sujet est ignoré.
- **La connexion est perdue pendant la rédaction** : une question non enregistrée n'est pas perdue en silence. L'utilisateur est prévenu que l'enregistrement a échoué, et sa saisie reste à l'écran pour qu'il puisse réessayer.
- **Recherche** : elle ne tient compte ni de la casse ni des accents. Une recherche vide ou d'un seul caractère n'est pas lancée.
- **Pagination** : les listes de catégories, les résultats de recherche et la file de modération sont paginés, à 20 éléments par page.
- **Lien de réinitialisation expiré ou déjà utilisé** : un message l'indique et propose d'en demander un nouveau.
- **Email non confirmé** : l'utilisateur peut créer et préparer des brouillons, mais doit confirmer son email pour publier un sujet ou en signaler un. Le lien de confirmation peut être renvoyé.
- **Sujet dépublié ou retiré pendant qu'un visiteur le lit** : son prochain chargement affiche « contenu introuvable ».

## Requirements *(mandatory)*

### Functional Requirements

**Comptes**

- **FR-001**: Le système DOIT permettre à un visiteur de créer un compte avec un nom affiché, une adresse email unique et un mot de passe d'au moins 8 caractères.
- **FR-002**: Le système DOIT envoyer un email de confirmation à l'inscription, et permettre de le renvoyer.
- **FR-003**: Le système DOIT permettre à un utilisateur de se connecter et de se déconnecter.
- **FR-004**: Le système DOIT limiter les tentatives de connexion : un blocage temporaire s'applique après 5 échecs consécutifs sur un même compte.
- **FR-005**: Le système DOIT permettre de réinitialiser le mot de passe via un lien envoyé par email, valable 60 minutes et à usage unique.
- **FR-006**: Le système NE DOIT PAS révéler si une adresse email possède un compte, ni lors de la connexion ni lors de la demande de réinitialisation.
- **FR-007**: Le système DOIT distinguer deux rôles : utilisateur inscrit et administrateur.

**Catégories et tags**

- **FR-008**: Le système DOIT permettre aux seuls administrateurs de créer, renommer, réordonner et supprimer des catégories, sur un seul niveau.
- **FR-009**: Le système DOIT refuser deux catégories de même nom, sans tenir compte de la casse ni des accents.
- **FR-010**: Le système DOIT refuser la suppression d'une catégorie qui contient au moins un sujet, quel que soit son statut.
- **FR-011**: Le système DOIT permettre à l'auteur de poser des tags libres sur son sujet, normalisés en minuscules, et lui proposer les tags existants pendant la saisie.

**Sujets et questions**

- **FR-012**: Le système DOIT permettre à tout utilisateur inscrit de créer un sujet, avec un titre, une description, une seule catégorie et jusqu'à 10 tags.
- **FR-013**: Un sujet DOIT avoir l'un de ces trois statuts : brouillon, publié ou retiré. Un sujet nouvellement créé est un brouillon.
- **FR-014**: Le système DOIT permettre à l'auteur d'ajouter, modifier, réordonner et supprimer les questions de son sujet, quel que soit son statut. Chaque question a un recto (la question) et un verso (la réponse).
- **FR-015**: Le recto et le verso DOIVENT accepter du texte mis en forme, limité au gras, à l'italique, aux listes, au code (en ligne et en bloc) et aux liens. Toute autre mise en forme ou tout contenu exécutable est retiré à l'enregistrement.
- **FR-016**: Le système DOIT permettre à l'auteur de publier un brouillon qui contient au moins une question, sous réserve que son email soit confirmé.
- **FR-017**: Le système DOIT permettre à l'auteur de dépublier son sujet publié, qui redevient un brouillon.
- **FR-018**: Le système DOIT empêcher de supprimer la dernière question d'un sujet publié.
- **FR-019**: Le système DOIT permettre à l'auteur de supprimer définitivement son sujet, après confirmation.
- **FR-020**: Le système DOIT fournir à chaque inscrit la liste de ses propres sujets, avec leur statut et, pour un sujet retiré, le motif du retrait.
- **FR-021**: Chaque question DOIT être un élément identifiable de façon stable et distinct au sein de son sujet, pour que la feature 002 puisse y rattacher une progression Leitner par utilisateur.

**Consultation**

- **FR-022**: Le système DOIT permettre à quiconque, connecté ou non, de consulter les sujets publiés et toutes leurs questions et réponses.
- **FR-023**: Le système NE DOIT exposer un brouillon qu'à son auteur et aux administrateurs, et un sujet retiré qu'à son auteur (en lecture, avec le motif) et aux administrateurs. Pour toute autre personne, ces sujets sont introuvables.
- **FR-024**: Le système DOIT permettre de parcourir les sujets publiés par catégorie, des plus récemment publiés aux plus anciens.
- **FR-025**: Le système DOIT permettre une recherche texte sur le titre, la description et les tags des sujets publiés, sans tenir compte de la casse ni des accents, à partir de 2 caractères.
- **FR-026**: Chaque sujet listé DOIT afficher son titre, sa description, sa catégorie, ses tags, le nom affiché de son auteur et son nombre de questions.

**Signalement et modération**

- **FR-027**: Le système DOIT permettre à un utilisateur inscrit dont l'email est confirmé de signaler un sujet publié dont il n'est pas l'auteur, avec un motif pris dans une liste fermée (contenu inapproprié, contenu erroné, spam, droits d'auteur, autre) et un commentaire facultatif d'au plus 500 caractères.
- **FR-028**: Le système DOIT refuser un second signalement d'un même utilisateur sur un même sujet tant que le premier est en attente.
- **FR-029**: Un sujet signalé DOIT rester visible tant qu'aucun administrateur n'a statué.
- **FR-030**: Le système DOIT fournir aux administrateurs une file des sujets qui ont des signalements en attente. Ils y sont regroupés par sujet et triés du plus ancien signalement au plus récent.
- **FR-031**: Un administrateur DOIT pouvoir ignorer les signalements d'un sujet (ils sont clos et le sujet reste publié), ou retirer le sujet en saisissant un motif (le sujet passe au statut retiré et ses signalements sont clos).
- **FR-032**: Un administrateur DOIT pouvoir modifier ou retirer n'importe quel sujet, même en dehors de la file, et rétablir un sujet retiré, qui redevient alors un brouillon.
- **FR-033**: L'auteur d'un sujet retiré NE DOIT PAS pouvoir le republier.
- **FR-034**: Le système DOIT conserver l'historique des décisions de modération : qui, quand, quelle décision, quel motif.

**Transverse**

- **FR-035**: Toute l'interface DOIT être en français, en vouvoyant l'utilisateur.
- **FR-036**: Le produit DOIT être utilisable sur mobile et sur ordinateur, et installable sur l'écran d'accueil d'un téléphone.

### Key Entities

- **Utilisateur** : une personne inscrite, avec un nom affiché, une adresse email (confirmée ou non) et un rôle (inscrit ou administrateur). Elle est l'auteur de zéro ou plusieurs sujets.
- **Catégorie** : une rubrique de premier niveau, avec un nom unique et une position d'affichage. Elle contient des sujets.
- **Tag** : un mot-clé libre et normalisé, partagé entre les sujets. Un sujet en porte de 0 à 10.
- **Sujet** : un ensemble de questions sur un thème, avec un titre, une description, un auteur, une catégorie, des tags, un statut (brouillon, publié ou retiré), une date de publication et, s'il est retiré, un motif de retrait.
- **Question (carte)** : un élément d'un sujet, avec un recto, un verso et une position dans le sujet. C'est l'unité que la révision Leitner de la feature 002 fera circuler entre les boîtes.
- **Signalement** : l'alerte d'un utilisateur sur un sujet, avec un motif, un commentaire facultatif, une date et un état (en attente ou clos).
- **Décision de modération** : l'action d'un administrateur sur un sujet (ignorer, retirer, rétablir), avec son auteur, sa date et son motif.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Un visiteur trouve un sujet et lit sa première réponse en moins de 30 secondes à partir de la page d'accueil, par la recherche ou par une catégorie.
- **SC-002**: Un nouvel utilisateur crée son compte en moins de 2 minutes.
- **SC-003**: Un auteur crée un sujet de 10 questions et le publie en moins de 15 minutes.
- **SC-004**: 90 % des utilisateurs qui testent le produit publient leur premier sujet sans aide.
- **SC-005**: La recherche et l'ouverture d'un sujet affichent leur résultat en moins de 2 secondes pour 95 % des requêtes, avec un catalogue de 10 000 sujets et 500 000 questions.
- **SC-006**: Aucun brouillon ni sujet retiré n'est accessible à une personne non autorisée. Ceci est vérifié par des tests qui couvrent chaque accès (liste, recherche, adresse directe).
- **SC-007**: Un signalement apparaît dans la file de modération dès qu'il est envoyé, et 100 % des décisions de modération sont retrouvables dans l'historique.
- **SC-008**: Les parcours principaux (consulter, s'inscrire, créer, publier, signaler) fonctionnent sur un téléphone de 360 px de large, sans défilement horizontal.

## Assumptions

- **Hors périmètre, pour la feature 002 et au-delà** : la révision Leitner et la progression par utilisateur, les favoris et abonnements à un sujet, les commentaires et notes de sujet, les images dans les questions, l'import et l'export de questions, les notifications par email à l'auteur lors d'une modération, la suspension de comptes, la connexion par un fournisseur externe (SSO).
- **Suppression de compte** : elle n'est pas couverte ici, mais elle est obligatoire au regard du RGPD avant l'ouverture au public. Elle sera traitée dans une feature dédiée avant la mise en production.
- **Premier administrateur** : il est créé à la mise en service, par l'équipe technique. Le produit ne fournit aucun écran pour promouvoir un utilisateur administrateur dans cette feature.
- **Catégories initiales** : un jeu de catégories est fourni à la mise en service, pour que les auteurs puissent créer des sujets dès le premier jour.
- **Un sujet a un seul auteur** : la co-écriture n'est pas prévue.
- **Nom affiché** : il est public et visible sur les sujets de l'auteur. L'adresse email n'est jamais affichée publiquement.
- **Mot de passe** : 8 caractères minimum, sans autre règle de composition, conformément aux recommandations actuelles. Une connexion reste ouverte 30 jours sur un même appareil, sauf déconnexion.
- **Consultation hors ligne** : l'installation sur l'écran d'accueil est attendue, mais la consultation hors ligne ne l'est pas dans cette feature.
