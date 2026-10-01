# Feature Specification: Suppression de compte

**Feature Branch**: `004-account-deletion`

**Created**: 2026-10-01

**Status**: Draft

**Input**: User description: "Suppression de compte (RGPD), prérequis à l'ouverture au public de CINQ. Une personne inscrite peut supprimer son compte depuis la page Compte. Confirmation par mot de passe ; le compte est désactivé immédiatement (déconnexion de tous les appareils, rappels arrêtés) et l'effacement définitif a lieu 30 jours plus tard ; se reconnecter pendant ce délai annule la suppression. Un email confirme la demande. Au moment de supprimer, la personne choisit entre tout effacer ou laisser ses sujets publiés au catalogue sous « Auteur supprimé » ; brouillons et sujets non publiés toujours effacés. Effacés à la fin du délai : identité, progression, réglages et journal des rappels. Signalements et décisions de modération conservés mais anonymisés. Le dernier compte d'administration ne peut pas se supprimer. Hors périmètre : export des données, suppression par un administrateur, suspension, transfert de sujets."

## Contexte

La constitution de CINQ (principe VI) impose que la suppression de compte soit livrée avant l'ouverture au public. Aujourd'hui, une personne inscrite n'a aucun moyen d'effacer ses données : son identité, sa progression Leitner, ses réglages de rappels et les sujets qu'elle a écrits restent indéfiniment.

Cette feature lui donne ce droit, depuis sa page Compte. La suppression laisse un délai de réflexion de 30 jours : le compte est fermé tout de suite, mais rien n'est effacé avant la fin du délai, et se reconnecter suffit pour revenir sur sa décision. Les sujets publiés sont un cas à part, car d'autres personnes les apprennent : l'auteur choisit de les emporter avec lui ou de les laisser à la communauté, sans son nom.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Demander la suppression de son compte (Priority: P1)

Depuis sa page Compte, la personne ouvre « Supprimer mon compte ». Un écran lui explique ce qui sera effacé, ce qui sera conservé sans son nom, et le délai de 30 jours. Si elle a publié des sujets, elle choisit ce qu'ils deviennent. Elle confirme en saisissant son mot de passe. Elle est déconnectée, reçoit un email de confirmation, et ne reçoit plus aucun rappel.

**Why this priority**: C'est l'obligation légale et la condition de l'ouverture au public.

**Independent Test**: Un inscrit sans sujet publié demande la suppression avec son mot de passe ; il est déconnecté sur tous ses appareils, reçoit l'email qui donne la date d'effacement, et aucun rappel ne part pour lui le soir même.

**Acceptance Scenarios**:

1. **Given** une personne connectée sur sa page Compte, **When** elle ouvre « Supprimer mon compte », **Then** un écran liste ce qui sera effacé (nom affiché, adresse email, progression, réglages de rappels, brouillons), ce qui sera conservé sans son nom (signalements déposés, décisions de modération), et indique que l'effacement aura lieu 30 jours plus tard sauf si elle se reconnecte.
2. **Given** cet écran, **When** elle saisit un mot de passe erroné et confirme, **Then** rien ne change et un message indique que le mot de passe est incorrect.
3. **Given** cet écran, **When** elle saisit son mot de passe et confirme, **Then** son compte est désactivé, elle est déconnectée sur cet appareil et sur tous les autres, et elle arrive sur une page qui confirme la demande, donne la date d'effacement et explique comment annuler.
4. **Given** une demande de suppression confirmée, **When** la demande est enregistrée, **Then** un email part à son adresse, rappelle la date d'effacement définitif et explique qu'il suffit de se reconnecter avant cette date pour annuler.
5. **Given** une demande de suppression confirmée, **When** l'heure de ses rappels arrive, **Then** aucun rappel ne part, ni par email ni en notification.
6. **Given** une demande de suppression confirmée, **When** quelqu'un consulte un sujet que la personne apprenait, **Then** rien n'indique son nom ni sa progression.

---

### User Story 2 - Décider du sort de ses sujets publiés (Priority: P1)

Une autrice qui a publié des sujets choisit, avant de confirmer la suppression, entre deux options : tout effacer avec son compte, ou laisser ses sujets publiés au catalogue, attribués à « Auteur supprimé », pour que les personnes qui les apprennent puissent continuer. Ses brouillons et ses sujets non publiés sont effacés dans tous les cas.

**Why this priority**: Sans ce choix, la suppression d'un auteur casserait la progression d'autres personnes, ou au contraire conserverait du contenu contre son gré.

**Independent Test**: Une autrice avec 2 sujets publiés et 1 brouillon choisit « Laisser mes sujets publiés » ; dès la demande, les 2 sujets restent consultables et apprenables, signés « Auteur supprimé », et le brouillon a disparu de partout. Une autre autrice choisit « Tout effacer » ; ses sujets disparaissent aussitôt du catalogue.

**Acceptance Scenarios**:

1. **Given** une personne sans aucun sujet publié, **When** elle ouvre « Supprimer mon compte », **Then** aucun choix sur les sujets ne lui est proposé.
2. **Given** une autrice avec des sujets publiés, **When** elle ouvre « Supprimer mon compte », **Then** elle voit le nombre de ses sujets publiés et le nombre de personnes qui les apprennent, et doit choisir entre « Laisser mes sujets publiés, sans mon nom » et « Tout effacer » avant de pouvoir confirmer.
3. **Given** le choix « Laisser mes sujets publiés », **When** la demande est confirmée, **Then** ses sujets publiés restent au catalogue, signés « Auteur supprimé », avec leurs questions et leurs images, et les personnes qui les apprennent gardent leur progression.
4. **Given** le choix « Tout effacer », **When** la demande est confirmée, **Then** ses sujets publiés deviennent introuvables pour tout le monde (catalogue, recherche, adresse directe), et leurs cartes ne sont plus proposées en séance ni comptées dans les rappels des personnes qui les apprenaient.
5. **Given** l'un ou l'autre choix, **When** la demande est confirmée, **Then** ses brouillons, ses sujets dépubliés et ses sujets retirés par la modération deviennent introuvables pour tout le monde.

---

### User Story 3 - Revenir sur sa décision (Priority: P1)

Pendant les 30 jours, la personne peut changer d'avis : il lui suffit de se connecter. Tout est rétabli à l'identique, y compris ses sujets s'ils avaient été masqués, ses rappels et sa progression.

**Why this priority**: Le délai de réflexion protège d'une erreur irréversible ; il n'a de valeur que si l'annulation est simple et complète.

**Independent Test**: Une personne qui a demandé la suppression avec « Tout effacer » se reconnecte 10 jours plus tard ; elle voit un message qui confirme l'annulation, retrouve ses sujets au catalogue, sa progression et ses rappels tels qu'ils étaient.

**Acceptance Scenarios**:

1. **Given** une demande de suppression en cours depuis 10 jours, **When** la personne se connecte avec son email et son mot de passe, **Then** la demande est annulée, un message le lui confirme, et son compte fonctionne comme avant.
2. **Given** une demande annulée, **When** elle consulte ses révisions, **Then** elle retrouve sa progression telle qu'elle était au moment de la demande ; les cartes devenues dues pendant le délai sont à réviser, sans pénalité, selon la règle de la feature 001.
3. **Given** une demande annulée avec le choix « Tout effacer », **When** quiconque consulte le catalogue, **Then** ses sujets publiés y sont de nouveau, sous son nom.
4. **Given** une demande annulée avec le choix « Laisser mes sujets publiés », **When** quiconque consulte ces sujets, **Then** ils sont de nouveau signés de son nom.
5. **Given** une demande annulée, **When** l'heure de ses rappels arrive, **Then** ses rappels reprennent avec ses réglages et appareils d'avant ; un appareil qui ne peut plus recevoir de notifications est retiré selon la règle de la feature 002.
6. **Given** une demande en cours, **When** la personne a oublié son mot de passe et le réinitialise, **Then** la connexion qui suit annule la demande de la même façon.

---

### User Story 4 - L'effacement définitif (Priority: P1)

À la fin du délai de 30 jours, sans reconnexion, le compte est effacé : il ne reste rien qui permette d'identifier la personne. Ce qu'elle a laissé à la communauté (sujets publiés si elle l'a choisi, signalements, décisions de modération) demeure, sans son nom.

**Why this priority**: C'est l'effacement lui-même, ce que le RGPD exige.

**Independent Test**: 30 jours après une demande avec « Laisser mes sujets publiés », plus aucune donnée ne contient le nom, l'email ou la progression de la personne ; ses sujets sont toujours apprenables sous « Auteur supprimé » ; son adresse email permet de créer un nouveau compte.

**Acceptance Scenarios**:

1. **Given** une demande de suppression vieille de 30 jours, **When** l'effacement a lieu, **Then** le nom affiché, l'adresse email, le mot de passe, la progression Leitner, l'historique des réponses, les réglages de rappels, les appareils de notification et le journal des rappels envoyés de la personne sont effacés.
2. **Given** l'effacement fait avec le choix « Tout effacer », **When** il a lieu, **Then** ses sujets, leurs questions et leurs images sont effacés, ainsi que la progression des personnes qui apprenaient ces sujets.
3. **Given** l'effacement fait avec le choix « Laisser mes sujets publiés », **When** il a lieu, **Then** ses sujets publiés restent au catalogue sous « Auteur supprimé » avec leurs questions et leurs images, et la progression des personnes qui les apprennent est intacte.
4. **Given** des signalements déposés par la personne, **When** l'effacement a lieu, **Then** ils restent dans la file et l'historique de modération, attribués à « Compte supprimé ».
5. **Given** une administratrice qui a pris des décisions de modération, **When** son compte est effacé, **Then** ces décisions restent dans l'historique, attribuées à « Compte supprimé ».
6. **Given** un compte effacé, **When** quelqu'un s'inscrit avec la même adresse email, **Then** l'inscription se déroule comme pour une adresse jamais utilisée, et le nouveau compte ne retrouve rien de l'ancien.
7. **Given** un compte effacé, **When** quelqu'un tente de se connecter avec ses anciens identifiants, **Then** il reçoit la réponse habituelle d'identifiants incorrects.

---

### User Story 5 - Protéger la dernière administration (Priority: P2)

Le dernier compte qui dispose de droits d'administration (modération ou gestion des catégories) ne peut pas se supprimer, car aucun écran ne permet de nommer quelqu'un d'autre : CINQ resterait sans modération.

**Why this priority**: Le cas est rare, mais il laisserait le catalogue sans personne pour traiter les signalements.

**Independent Test**: Avec une seule administratrice, elle ouvre « Supprimer mon compte » et voit un message qui explique pourquoi la suppression n'est pas possible ; avec deux administrateurs, l'une d'elles peut se supprimer.

**Acceptance Scenarios**:

1. **Given** une seule personne disposant d'un droit d'administration, **When** elle ouvre « Supprimer mon compte », **Then** un message explique que le dernier compte d'administration ne peut pas être supprimé, et la confirmation n'est pas proposée.
2. **Given** deux personnes disposant d'un droit d'administration, **When** l'une demande la suppression, **Then** la demande est acceptée ; l'autre devient le dernier compte d'administration et ne peut plus se supprimer tant que la demande de la première est en cours.
3. **Given** une administratrice avec une demande en cours qui se reconnecte, **When** la demande est annulée, **Then** elle retrouve ses droits d'administration.

---

### Edge Cases

- **Demande pendant une séance de révision sur un autre appareil** : la réponse suivante envoyée depuis cet appareil est refusée comme celle d'une personne déconnectée ; elle n'est pas enregistrée.
- **Signalement en attente sur un sujet de la personne, avec « Tout effacer »** : le sujet devient introuvable ; le signalement reste dans la file de modération, rattaché à « Sujet supprimé », et peut être classé.
- **Sujet de la personne retiré par la modération pendant le délai, avec « Laisser mes sujets publiés »** : le retrait s'applique comme pour tout sujet ; à l'effacement, un sujet retiré n'est pas conservé, puisque seuls les sujets publiés le sont.
- **Personne qui apprend son propre sujet** : sa progression est effacée avec le reste ; le sujet suit le choix fait pour ses sujets publiés.
- **Lien de rappel ou de désinscription reçu avant la demande** : le lien de rappel mène à la connexion, et se connecter annule la demande ; le lien de désinscription d'un compte en cours de suppression ou effacé affiche la page de lien non valide de la feature 002, sans révéler l'état du compte.
- **Inscription avec l'adresse d'un compte en cours de suppression** : elle reçoit la réponse neutre habituelle, comme pour toute adresse déjà prise ; aucun message ne révèle qu'une suppression est en cours.
- **Mot de passe oublié pendant le délai** : la réinitialisation fonctionne ; la connexion qui suit annule la demande (User Story 3).
- **Échec de l'email de confirmation** : la demande reste enregistrée et la page de confirmation à l'écran donne la même information que l'email.
- **Effacement qui échoue en cours de route** : rien n'est effacé à moitié ; l'effacement est repris à la passe suivante jusqu'à réussir, et l'échec est signalé à l'équipe technique.
- **Sujet signé « Auteur supprimé » signalé plus tard** : la modération le traite comme tout autre sujet ; aucune notification n'est possible vers son auteur.

## Requirements *(mandatory)*

### Functional Requirements

**Demande**

- **FR-001**: La page Compte DOIT proposer « Supprimer mon compte » à toute personne connectée.
- **FR-002**: Avant la confirmation, le système DOIT présenter ce qui sera effacé, ce qui sera conservé sans nom, et la date d'effacement définitif, 30 jours après la demande.
- **FR-003**: La demande DOIT être confirmée par la saisie du mot de passe du compte ; un mot de passe erroné ne change rien et affiche un message d'erreur.
- **FR-004**: Une personne qui a au moins un sujet publié DOIT choisir entre « Laisser mes sujets publiés, sans mon nom » et « Tout effacer » avant de pouvoir confirmer ; l'écran indique le nombre de ses sujets publiés et le nombre de personnes qui les apprennent. Sans sujet publié, aucun choix n'est demandé.
- **FR-005**: Le dernier compte disposant d'un droit d'administration (modération des sujets ou gestion des catégories) NE DOIT PAS pouvoir demander la suppression ; un message en explique la raison. Un compte d'administration dont la demande est en cours ne compte pas comme compte d'administration disponible.

**Effets immédiats de la demande**

- **FR-006**: Dès la demande confirmée, le système DOIT déconnecter la personne sur tous ses appareils et refuser toute action de ce compte jusqu'à une nouvelle connexion.
- **FR-007**: Dès la demande confirmée, aucun rappel NE DOIT partir pour ce compte, sur aucun canal.
- **FR-008**: Dès la demande confirmée, les brouillons, sujets dépubliés et sujets retirés de la personne DOIVENT devenir introuvables pour tout le monde, au même titre qu'un brouillon pour une personne non autorisée.
- **FR-009**: Avec « Tout effacer », dès la demande confirmée, ses sujets publiés DOIVENT devenir introuvables pour tout le monde ; leurs cartes ne sont plus proposées en séance ni comptées dans les rappels de quiconque.
- **FR-010**: Avec « Laisser mes sujets publiés », dès la demande confirmée, ses sujets publiés DOIVENT rester consultables et apprenables, attribués à « Auteur supprimé » partout où le nom de l'auteur apparaît.
- **FR-011**: Le nom de la personne NE DOIT plus apparaître nulle part, pour personne, entre la demande confirmée et une éventuelle annulation.
- **FR-012**: Le système DOIT envoyer un email qui confirme la demande, donne la date d'effacement définitif et explique qu'une connexion avant cette date annule la suppression.
- **FR-013**: La page affichée après la confirmation DOIT donner les mêmes informations que l'email.

**Annulation**

- **FR-014**: Toute connexion réussie au compte pendant le délai, y compris après une réinitialisation du mot de passe, DOIT annuler la demande et afficher un message qui le confirme.
- **FR-015**: L'annulation DOIT rétablir le compte à l'identique : nom, sujets et leur attribution, progression, réglages de rappels, appareils de notification et droits d'administration.

**Effacement définitif**

- **FR-016**: Le système DOIT effacer le compte au plus tard 24 heures après la fin du délai de 30 jours, sans action de quiconque.
- **FR-017**: L'effacement DOIT supprimer le nom affiché, l'adresse email, le mot de passe, la progression Leitner, l'historique des réponses, les réglages de rappels, les appareils de notification et le journal des rappels envoyés de la personne.
- **FR-018**: Avec « Tout effacer », l'effacement DOIT supprimer ses sujets, leurs questions et leurs images, ainsi que la progression de toutes les personnes sur ces sujets.
- **FR-019**: Avec « Laisser mes sujets publiés », l'effacement DOIT conserver ses sujets publiés, leurs questions et leurs images, attribués à « Auteur supprimé », sans toucher à la progression des personnes qui les apprennent ; ses autres sujets sont supprimés.
- **FR-020**: Les signalements déposés par la personne et les décisions de modération qu'elle a prises DOIVENT être conservés, attribués à « Compte supprimé ».
- **FR-021**: Un signalement ou une décision de modération qui porte sur un sujet effacé DOIT rester dans l'historique, rattaché à « Sujet supprimé ».
- **FR-022**: L'effacement DOIT être complet ou ne pas avoir lieu ; un effacement interrompu est repris jusqu'à réussir.
- **FR-023**: Après l'effacement, l'adresse email DOIT redevenir libre pour une nouvelle inscription, sans lien avec l'ancien compte.

**Confidentialité**

- **FR-024**: Aucune réponse du système (inscription, connexion, mot de passe oublié, lien de désinscription) NE DOIT révéler qu'un compte est en cours de suppression ou a été effacé.
- **FR-025**: Les textes de l'écran de suppression, de la page de confirmation et de l'email DOIVENT être en français, en vouvoyant la personne.

### Key Entities

- **Demande de suppression** : pour un compte, la date de la demande, la date d'effacement prévue (30 jours plus tard) et le choix fait pour les sujets publiés (laisser ou tout effacer). Elle disparaît à l'annulation ou avec le compte à l'effacement.
- **Auteur supprimé** : l'attribution affichée pour un sujet publié conservé après la suppression de son auteur.
- **Compte supprimé** : l'attribution affichée dans la modération pour un signalement ou une décision dont l'auteur a été effacé.
- **Compte** (existant) : désormais dans l'un de ces trois états : actif, en cours de suppression, effacé.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Une personne demande la suppression de son compte en moins de 2 minutes depuis sa page Compte.
- **SC-002**: Dès la demande confirmée, 100 % des sessions du compte sont fermées et aucun rappel ne part. Ceci est vérifié par des tests qui couvrent FR-006 et FR-007.
- **SC-003**: 30 jours et 24 heures après une demande non annulée, aucune donnée ne permet de retrouver le nom, l'adresse email ou la progression de la personne. Ceci est vérifié par des tests qui couvrent chaque donnée de FR-017.
- **SC-004**: Une annulation rétablit 100 % de ce qui existait au moment de la demande. Ceci est vérifié par des tests qui couvrent chaque élément de FR-015.
- **SC-005**: Avec « Laisser mes sujets publiés », 100 % des personnes qui apprenaient ces sujets gardent leur progression, de la demande à l'effacement et au-delà.
- **SC-006**: Aucune réponse du système ne permet de distinguer un compte en cours de suppression ou effacé d'une adresse inconnue ou déjà prise. Ceci est vérifié par des tests qui couvrent chaque cas de FR-024.

## Assumptions

- **Dépend des features 001, 002 et 003** : comptes et connexion, sujets et leur visibilité, apprentissages et séances, modération ; réglages, appareils et journal des rappels ; images des questions.
- **Hors périmètre** : l'export des données (portabilité, feature séparée), la suppression d'un compte par un administrateur, la suspension de comptes, le transfert des sujets à un autre auteur, la purge des comptes jamais confirmés.
- **Délai de 30 jours** : un délai d'usage pour ce type de service, qui laisse le temps de changer d'avis sans conserver les données plus longtemps que nécessaire.
- **Aucun email à l'effacement définitif** : une fois le compte effacé, CINQ n'a plus de raison de contacter la personne ; l'email de la demande donne déjà la date.
- **Dernier compte d'administration** : la règle tient tant qu'aucun écran ne permet de nommer un administrateur ; elle sera revue si un tel écran est livré.
- **Adresse email pendant le délai** : elle reste prise ; une inscription avec cette adresse reçoit la réponse neutre de la feature 001.
- **Personnes qui apprenaient un sujet effacé** : elles ne reçoivent aucune notification ; le sujet disparaît de leurs révisions comme un sujet dépublié (feature 001).
- **Contenu laissé à la communauté** : un sujet signé « Auteur supprimé » ne peut plus être modifié par personne ; seule la modération peut encore le retirer.
