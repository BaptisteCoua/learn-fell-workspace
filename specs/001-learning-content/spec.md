# Feature Specification: Comptes, contenu et révision Leitner

**Feature Branch**: `001-learning-content`

**Created**: 2026-09-24

**Status**: Draft

**Maquette**: [Learn Fell — Design 001](https://claude.ai/artifact/DxQK5ap9dAK6UYGuZURLsP), validée le 2026-09-24. 43 écrans (ordinateur, mobile, PWA), style brutaliste jaune #FACC15 et bleu #3B82F6 sur crème, sans le design system Xefi.

**Input**: User description: "Espace public d'apprentissage (Learn Fell) — feature 001 : comptes et contenu. Visiteurs, inscrits (auteurs) et administrateurs. Inscription par email et mot de passe. Catégories à un niveau gérées par les administrateurs, tags libres posés par l'auteur. Sujets en brouillon puis publiés, contenant des questions recto/verso en texte mis en forme. Lecture libre sans compte, parcours par catégorie et recherche. Signalement des sujets, traité par les administrateurs. Élargie après la maquette : révision Leitner avec auto-évaluation (5 boîtes, 1, 2, 4, 8 et 16 jours), lancée par « Apprendre ce sujet », séance sur un ou plusieurs sujets choisis."

## Contexte

Learn Fell est un espace public où chacun peut apprendre n'importe quel sujet par la méthode Leitner. Cette première feature livre le produit de bout en bout : des comptes, un catalogue de sujets composés de questions et de réponses, que la communauté écrit et que tout le monde peut consulter, et la révision de ces questions par la méthode Leitner.

Chaque question est une carte avec un recto (la question) et un verso (la réponse). Un inscrit choisit les sujets qu'il veut apprendre ; leurs cartes circulent alors entre 5 boîtes, révisées tous les 1, 2, 4, 8 et 16 jours, selon qu'il répond « je savais » ou « je ne savais pas ».

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

Un visiteur crée un compte avec son adresse email et un mot de passe saisi deux fois, puis confirme son adresse en cliquant sur le lien reçu par email. Tant que l'adresse n'est pas confirmée, le compte est inutilisable. Une fois le compte actif, il se connecte, se déconnecte, et peut réinitialiser son mot de passe s'il l'a oublié.

**Why this priority**: C'est un prérequis à toute création de contenu (story 3), au signalement (story 5) et à la révision Leitner (story 6).

**Independent Test**: Créer un compte, vérifier que la connexion est refusée avant la confirmation, confirmer l'email, se déconnecter, se reconnecter, puis réinitialiser le mot de passe via le lien reçu par email.

**Acceptance Scenarios**:

1. **Given** un visiteur, **When** il s'inscrit avec un nom affiché, une adresse email valide et un mot de passe conforme saisi deux fois à l'identique, **Then** son compte est créé mais inactif, il n'est pas connecté, il reçoit un email de confirmation, et un écran l'invite à vérifier ses emails.
2. **Given** un visiteur saisit deux mots de passe différents, **When** il valide l'inscription, **Then** l'inscription est refusée avec un message sous le champ de confirmation.
3. **Given** un compte inactif, **When** son propriétaire clique sur le lien de confirmation dans les 24 heures, **Then** le compte devient actif, il est connecté, et il voit la confirmation « Adresse confirmée ».
4. **Given** un compte inactif, **When** son propriétaire tente de se connecter, **Then** la connexion est refusée avec un message qui l'invite à confirmer son adresse et lui propose de renvoyer le lien.
5. **Given** un lien de confirmation expiré ou déjà utilisé, **When** l'utilisateur l'ouvre, **Then** un message l'indique et lui propose de recevoir un nouveau lien.
6. **Given** un compte existe déjà avec cette adresse email, **When** un visiteur tente de s'inscrire avec la même adresse, **Then** l'inscription est refusée avec un message qui l'invite à se connecter ou à réinitialiser son mot de passe. Si ce compte existant est encore inactif, le message propose à la place de renvoyer le lien de confirmation.
7. **Given** un utilisateur inscrit, **When** il saisit un mauvais mot de passe, **Then** la connexion est refusée avec un message générique qui ne révèle pas si l'adresse email existe.
8. **Given** 5 échecs de connexion consécutifs sur un même compte, **When** une sixième tentative est faite, **Then** elle est bloquée temporairement et un message indique quand réessayer.
9. **Given** un utilisateur a oublié son mot de passe, **When** il demande une réinitialisation, **Then** il reçoit un lien valable 60 minutes et à usage unique, et le message affiché est le même que l'adresse existe ou non.
10. **Given** un utilisateur connecté, **When** il se déconnecte, **Then** il ne peut plus accéder aux pages réservées aux inscrits sans se reconnecter.

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

### User Story 6 - Apprendre un sujet avec la méthode Leitner (Priority: P1)

Un inscrit choisit « Apprendre ce sujet » : toutes les cartes du sujet entrent dans sa boîte 1. Chaque jour, il ouvre « Mes révisions », sélectionne un ou plusieurs sujets et lance une séance. Pour chaque carte, il lit le recto, affiche le verso, puis s'auto-évalue : « Je savais » fait monter la carte d'une boîte, « Je ne savais pas » la renvoie en boîte 1.

**Why this priority**: C'est la promesse du produit : apprendre et retenir. Sans révision, Learn Fell n'est qu'un catalogue de fiches.

**Independent Test**: Un inscrit apprend un sujet de 5 questions, fait une séance (3 « je savais », 2 « je ne savais pas »), puis vérifie la répartition dans les boîtes et la date de la prochaine révision de chaque carte.

**Acceptance Scenarios**:

1. **Given** un inscrit sur un sujet publié qu'il n'apprend pas, **When** il choisit « Apprendre ce sujet », **Then** toutes les cartes du sujet entrent dans sa boîte 1, à réviser le jour même, et le sujet apparaît dans « Mes révisions ».
2. **Given** un visiteur sans compte, **When** il consulte un sujet, **Then** le bouton l'invite à créer un compte ou à se connecter pour apprendre ce sujet.
3. **Given** « Mes révisions » affiche 3 sujets avec des cartes à réviser, **When** l'inscrit en sélectionne 2 et lance la séance, **Then** la séance contient toutes les cartes à réviser aujourd'hui de ces 2 sujets, et seulement elles.
4. **Given** une carte en boîte 2, **When** l'inscrit répond « Je savais », **Then** elle passe en boîte 3 et revient dans 4 jours ; l'écran l'indique avant la carte suivante.
5. **Given** une carte en boîte 4, **When** l'inscrit répond « Je ne savais pas », **Then** elle retourne en boîte 1 et revient le lendemain ; elle n'est pas reposée dans la séance en cours.
6. **Given** une carte en boîte 5, **When** l'inscrit répond « Je savais », **Then** elle reste en boîte 5 et revient dans 16 jours.
7. **Given** une séance en cours, **When** l'inscrit la quitte avant la fin, **Then** les réponses déjà données sont conservées, et les cartes non vues restent à réviser.
8. **Given** la dernière carte d'une séance, **When** l'inscrit y répond, **Then** un bilan affiche le nombre de « je savais » et de « je ne savais pas », la nouvelle répartition dans les boîtes et la date de la prochaine révision.
9. **Given** aucun des sujets appris n'a de carte à réviser aujourd'hui, **When** l'inscrit ouvre « Mes révisions », **Then** un message l'indique, avec la date de la prochaine révision.
10. **Given** un sujet appris, **When** l'inscrit choisit « Arrêter d'apprendre » et confirme, **Then** le sujet quitte « Mes révisions » et sa progression est supprimée.

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
- **Email non confirmé** : le compte reste inactif. Il ne permet aucune connexion, et donc aucune création ni aucun signalement. Le lien de confirmation peut être renvoyé depuis l'écran de vérification et depuis le message de connexion refusée. Chaque nouvel envoi invalide le lien précédent.
- **Compte jamais confirmé** : un compte resté inactif 7 jours est supprimé, ce qui libère l'adresse email pour une nouvelle inscription.
- **Sujet dépublié ou retiré pendant qu'un visiteur le lit** : son prochain chargement affiche « contenu introuvable ».
- **Échéances** : une carte est à réviser à partir de sa date prévue, au jour près, dans le fuseau de l'utilisateur. Une carte en retard reste à réviser, sans pénalité, jusqu'à ce qu'il y réponde.
- **L'auteur ajoute une question à un sujet appris** : elle entre en boîte 1 pour tous ceux qui l'apprennent. Une question modifiée garde sa boîte. Une question supprimée disparaît de leur progression.
- **Sujet appris dépublié, retiré ou supprimé** : ses cartes ne sont plus proposées en révision. S'il est republié ou rétabli puis republié, elles reprennent là où elles en étaient. S'il est supprimé, la progression l'est aussi.
- **Séance interrompue** (fermeture, perte de connexion) : chaque réponse est enregistrée dès qu'elle est donnée. Une réponse qui n'a pas pu être enregistrée est signalée, et la carte reste à réviser.
- **Double clic sur une réponse** : une carte ne change de boîte qu'une fois par présentation.

## Requirements *(mandatory)*

### Functional Requirements

**Comptes**

- **FR-001**: Le système DOIT permettre à un visiteur de créer un compte avec un nom affiché, une adresse email unique et un mot de passe d'au moins 8 caractères, saisi deux fois. L'inscription est refusée si les deux saisies diffèrent.
- **FR-002**: Le système DOIT envoyer à l'inscription un email contenant un lien de confirmation, valable 24 heures et à usage unique, et permettre de le renvoyer. Un compte n'est actif qu'après ce clic. Un compte inactif NE DOIT PAS pouvoir se connecter, et il est supprimé au bout de 7 jours.
- **FR-003**: Le système DOIT permettre à un utilisateur de se connecter et de se déconnecter.
- **FR-004**: Le système DOIT limiter les tentatives de connexion : après 5 échecs consécutifs sur un même compte, la connexion est bloquée pendant 15 minutes, et le message propose de réinitialiser le mot de passe.
- **FR-005**: Le système DOIT permettre de réinitialiser le mot de passe via un lien envoyé par email, valable 60 minutes et à usage unique. Le nouveau mot de passe est saisi deux fois, et sa validation ferme les sessions ouvertes sur les autres appareils.
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
- **FR-016**: Le système DOIT permettre à l'auteur de publier un brouillon qui contient au moins une question.
- **FR-017**: Le système DOIT permettre à l'auteur de dépublier son sujet publié, qui redevient un brouillon.
- **FR-018**: Le système DOIT empêcher de supprimer la dernière question d'un sujet publié.
- **FR-019**: Le système DOIT permettre à l'auteur de supprimer définitivement son sujet, après confirmation.
- **FR-020**: Le système DOIT fournir à chaque inscrit la liste de ses propres sujets, avec leur statut et, pour un sujet retiré, le motif du retrait.
- **FR-021**: Chaque question DOIT être un élément identifiable de façon stable et distinct au sein de son sujet, pour qu'une progression Leitner par utilisateur puisse s'y rattacher.

**Consultation**

- **FR-022**: Le système DOIT permettre à quiconque, connecté ou non, de consulter les sujets publiés et toutes leurs questions et réponses.
- **FR-023**: Le système NE DOIT exposer un brouillon qu'à son auteur et aux administrateurs, et un sujet retiré qu'à son auteur (en lecture, avec le motif) et aux administrateurs. Pour toute autre personne, ces sujets sont introuvables.
- **FR-024**: Le système DOIT permettre de parcourir les sujets publiés par catégorie, des plus récemment publiés aux plus anciens.
- **FR-025**: Le système DOIT permettre une recherche texte sur le titre, la description et les tags des sujets publiés, sans tenir compte de la casse ni des accents, à partir de 2 caractères.
- **FR-026**: Chaque sujet listé DOIT afficher son titre, sa description, sa catégorie, ses tags, le nom affiché de son auteur et son nombre de questions.

**Signalement et modération**

- **FR-027**: Le système DOIT permettre à un utilisateur inscrit de signaler un sujet publié dont il n'est pas l'auteur, avec un motif pris dans une liste fermée (contenu inapproprié, contenu erroné, spam, droits d'auteur, autre) et un commentaire facultatif d'au plus 500 caractères.
- **FR-028**: Le système DOIT refuser un second signalement d'un même utilisateur sur un même sujet tant que le premier est en attente.
- **FR-029**: Un sujet signalé DOIT rester visible tant qu'aucun administrateur n'a statué.
- **FR-030**: Le système DOIT fournir aux administrateurs une file des sujets qui ont des signalements en attente. Ils y sont regroupés par sujet et triés du plus ancien signalement au plus récent.
- **FR-031**: Un administrateur DOIT pouvoir ignorer les signalements d'un sujet (ils sont clos et le sujet reste publié), ou retirer le sujet en saisissant un motif (le sujet passe au statut retiré et ses signalements sont clos).
- **FR-032**: Un administrateur DOIT pouvoir modifier ou retirer n'importe quel sujet, même en dehors de la file, et rétablir un sujet retiré, qui redevient alors un brouillon.
- **FR-033**: L'auteur d'un sujet retiré NE DOIT PAS pouvoir le republier.
- **FR-034**: Le système DOIT conserver l'historique des décisions de modération : qui, quand, quelle décision, quel motif.

**Révision Leitner**

- **FR-041**: Le système DOIT permettre à un inscrit d'apprendre un sujet publié : toutes ses cartes entrent dans la boîte 1 de cet inscrit, à réviser le jour même. Un visiteur sans compte est invité à se connecter ou à créer un compte.
- **FR-042**: Chaque carte apprise DOIT avoir, pour chaque inscrit, une boîte (de 1 à 5) et une date de prochaine révision. L'intervalle dépend de la boîte d'arrivée : 1 jour (boîte 1), 2 jours (boîte 2), 4 jours (boîte 3), 8 jours (boîte 4), 16 jours (boîte 5).
- **FR-043**: Le système DOIT fournir une page « Mes révisions » qui liste les sujets appris, avec pour chacun le nombre de cartes à réviser aujourd'hui, la répartition dans les 5 boîtes et la date de la prochaine révision.
- **FR-044**: Le système DOIT permettre de sélectionner un ou plusieurs sujets appris et de lancer une séance, qui contient toutes les cartes à réviser aujourd'hui de ces sujets, des plus en retard aux plus récentes.
- **FR-045**: Pendant une séance, le système DOIT montrer le recto, puis le verso à la demande, et proposer deux réponses : « Je savais » et « Je ne savais pas ». La réponse n'est possible qu'une fois le verso affiché.
- **FR-046**: « Je savais » DOIT faire monter la carte d'une boîte (une carte en boîte 5 y reste). « Je ne savais pas » DOIT la renvoyer en boîte 1. La carte n'est pas reposée dans la même séance.
- **FR-047**: Après chaque réponse, le système DOIT indiquer la nouvelle boîte de la carte et quand elle reviendra.
- **FR-048**: Chaque réponse DOIT être enregistrée dès qu'elle est donnée. Quitter une séance ne perd aucune réponse déjà donnée.
- **FR-049**: En fin de séance, le système DOIT afficher un bilan : le nombre de « je savais » et de « je ne savais pas », la répartition dans les boîtes et la date de la prochaine révision.
- **FR-050**: Le système DOIT permettre d'arrêter d'apprendre un sujet, après confirmation. Sa progression est alors supprimée.
- **FR-051**: Le système DOIT tenir la progression à jour quand l'auteur modifie un sujet appris : une question ajoutée entre en boîte 1, une question supprimée sort de la progression. Les cartes d'un sujet dépublié ou retiré sont mises en pause, puis reprennent à sa republication.

**Transverse**

- **FR-035**: Toute l'interface DOIT être en français, en vouvoyant l'utilisateur.
- **FR-036**: Le produit DOIT être utilisable sur mobile et sur ordinateur, et installable sur l'écran d'accueil d'un téléphone.
- **FR-052**: Sur téléphone, les textes DOIVENT s'adapter à la largeur de l'écran : les titres rétrécissent sur un écran étroit sans déborder, un mot trop long passe à la ligne avec une césure au lieu de sortir de l'écran, le texte courant ne descend jamais sous 12 px, et le réglage de taille de texte du téléphone est respecté.
- **FR-037**: Le produit DOIT proposer son installation sur l'écran d'accueil : une invitation que l'on peut accepter ou repousser sur les navigateurs qui le permettent, et des instructions pas à pas sur iPhone. L'installation reste accessible à tout moment depuis le menu du compte.
- **FR-038**: Une fois installé, le produit DOIT s'ouvrir en plein écran, sans la barre du navigateur, sur un écran de lancement aux couleurs de Learn Fell, puis sur le catalogue.
- **FR-039**: Sans connexion, le produit DOIT afficher un écran « Vous êtes hors ligne » avec un bouton pour réessayer. Si la connexion se perd sur une page déjà affichée, un bandeau l'indique, les actions qui enregistrent (enregistrer, publier, signaler) sont suspendues, et un message confirme le retour en ligne.
- **FR-040**: Quand une nouvelle version du produit est disponible, l'utilisateur DOIT en être informé et pouvoir mettre à jour tout de suite ou plus tard.

### Key Entities

- **Utilisateur** : une personne inscrite, avec un nom affiché, une adresse email (confirmée ou non) et un rôle (inscrit ou administrateur). Elle est l'auteur de zéro ou plusieurs sujets.
- **Catégorie** : une rubrique de premier niveau, avec un nom unique et une position d'affichage. Elle contient des sujets.
- **Tag** : un mot-clé libre et normalisé, partagé entre les sujets. Un sujet en porte de 0 à 10.
- **Sujet** : un ensemble de questions sur un thème, avec un titre, une description, un auteur, une catégorie, des tags, un statut (brouillon, publié ou retiré), une date de publication et, s'il est retiré, un motif de retrait.
- **Question (carte)** : un élément d'un sujet, avec un recto, un verso et une position dans le sujet. C'est l'unité que la révision Leitner fait circuler entre les boîtes.
- **Signalement** : l'alerte d'un utilisateur sur un sujet, avec un motif, un commentaire facultatif, une date et un état (en attente ou clos).
- **Décision de modération** : l'action d'un administrateur sur un sujet (ignorer, retirer, rétablir), avec son auteur, sa date et son motif.
- **Apprentissage** : le fait qu'un inscrit apprend un sujet, avec la date à laquelle il a commencé.
- **Progression d'une carte** : pour un inscrit et une question, la boîte actuelle (1 à 5), la date de prochaine révision et la date de la dernière réponse.
- **Réponse de révision** : une auto-évaluation donnée pendant une séance (« je savais » ou « je ne savais pas »), avec sa date, la boîte de départ et la boîte d'arrivée. Elle sert au bilan de fin de séance.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Un visiteur trouve un sujet et lit sa première réponse en moins de 30 secondes à partir de la page d'accueil, par la recherche ou par une catégorie.
- **SC-002**: Un nouvel utilisateur crée son compte en moins de 2 minutes.
- **SC-003**: Un auteur crée un sujet de 10 questions et le publie en moins de 15 minutes.
- **SC-004**: 90 % des utilisateurs qui testent le produit publient leur premier sujet sans aide.
- **SC-005**: La recherche et l'ouverture d'un sujet affichent leur résultat en moins de 2 secondes pour 95 % des requêtes, avec un catalogue de 10 000 sujets et 500 000 questions.
- **SC-006**: Aucun brouillon ni sujet retiré n'est accessible à une personne non autorisée. Ceci est vérifié par des tests qui couvrent chaque accès (liste, recherche, adresse directe).
- **SC-007**: Un signalement apparaît dans la file de modération dès qu'il est envoyé, et 100 % des décisions de modération sont retrouvables dans l'historique.
- **SC-008**: Les parcours principaux (consulter, s'inscrire, créer, publier, signaler, réviser) fonctionnent sur un téléphone de 360 px de large, sans défilement horizontal.
- **SC-009**: Une séance de 20 cartes se termine en moins de 5 minutes, sans temps d'attente perceptible entre deux cartes.
- **SC-010**: 100 % des réponses donnent la boîte et la date de prochaine révision prévues par FR-042 et FR-046. Ceci est vérifié par des tests qui couvrent chaque boîte et chaque réponse.

## Assumptions

- **Hors périmètre, pour une feature ultérieure** : les rappels et notifications de révision, l'annulation d'une réponse donnée, les statistiques d'apprentissage au-delà du bilan de séance, les favoris et abonnements à un sujet, les commentaires et notes de sujet, les images dans les questions, l'import et l'export de questions, les notifications par email à l'auteur lors d'une modération, la suspension de comptes.
- **Connexion avec Google** : elle est prévue dans une feature ultérieure. Dans la 001, les écrans d'inscription et de connexion affichent déjà un bouton « Continuer avec Google », marqué « Bientôt » et désactivé.
- **Suppression de compte** : elle n'est pas couverte ici, mais elle est obligatoire au regard du RGPD avant l'ouverture au public. Elle sera traitée dans une feature dédiée avant la mise en production.
- **Premier administrateur** : il est créé à la mise en service, par l'équipe technique. Le produit ne fournit aucun écran pour promouvoir un utilisateur administrateur dans cette feature.
- **Catégories initiales** : un jeu de catégories est fourni à la mise en service, pour que les auteurs puissent créer des sujets dès le premier jour.
- **Un sujet a un seul auteur** : la co-écriture n'est pas prévue.
- **Nom affiché** : il est public et visible sur les sujets de l'auteur. L'adresse email n'est jamais affichée publiquement.
- **Mot de passe** : 8 caractères minimum, sans autre règle de composition, conformément aux recommandations actuelles. Une connexion reste ouverte 30 jours sur un même appareil, sauf déconnexion.
- **Consultation hors ligne** : l'installation sur l'écran d'accueil est attendue, mais la consultation hors ligne ne l'est pas dans cette feature. Hors ligne, seule la page déjà affichée reste lisible (FR-039).
