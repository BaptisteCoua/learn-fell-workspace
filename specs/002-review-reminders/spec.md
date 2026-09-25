# Feature Specification: Rappels de révision

**Feature Branch**: `002-review-reminders`

**Created**: 2026-09-25

**Status**: Draft

**Input**: User description: "Feature 002 de CINQ : rappels de révision. Un utilisateur inscrit qui apprend des sujets peut recevoir un rappel quand il a des cartes à réviser. Canaux au choix, cumulables : notification push de la PWA et email. Rien n'est activé par défaut : on propose d'activer les rappels juste après le premier « Apprendre ce sujet », et on les règle dans une section « Rappels » de la page Compte (canaux, heure). Un rappel part au plus une fois par jour, à l'heure choisie (19 h par défaut), dans le fuseau de l'utilisateur, seulement s'il a au moins une carte à réviser ce jour-là dans des sujets publiés. Contenu : le nombre de cartes à réviser et un lien qui ouvre la séance sur les sujets concernés. Sans révision, les rappels s'espacent : chaque jour, puis tous les 2 jours, puis une fois par semaine ; une révision remet le rythme quotidien. Chaque email porte un lien signé de désinscription en un clic, sans connexion. L'utilisateur peut couper le push d'un appareil ; plusieurs appareils peuvent recevoir le push. Un utilisateur qui n'apprend plus aucun sujet ne reçoit rien."

## Contexte

La méthode Leitner de CINQ (feature 001) ne fonctionne que si l'on revient réviser le bon jour : une carte de la boîte 1 est due le lendemain, une carte de la boîte 5 seize jours plus tard. Aujourd'hui, rien ne prévient l'apprenant qu'il a des cartes à réviser ; il doit penser à ouvrir « Mes révisions ».

Cette feature ajoute des rappels : une fois par jour au plus, à l'heure qu'il choisit, CINQ prévient l'apprenant du nombre de cartes qui l'attendent, par une notification sur ses appareils, par email, ou les deux. Les rappels sont un service rendu à l'apprenant, jamais une sollicitation : rien n'est envoyé sans son accord, ils s'espacent s'il ne révise plus, et un clic suffit pour les couper.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Activer et régler ses rappels (Priority: P1)

Juste après avoir choisi « Apprendre ce sujet » pour la première fois, l'apprenant se voit proposer de recevoir un rappel quand il a des cartes à réviser. Il choisit les notifications sur cet appareil, l'email, ou les deux, et l'heure du rappel. Il retrouve et modifie ces réglages à tout moment dans la section « Rappels » de sa page Compte.

**Why this priority**: Sans activation, aucun rappel ne part ; c'est aussi là que se joue le consentement de l'utilisateur.

**Independent Test**: Un inscrit apprend son premier sujet, voit la proposition, active l'email à 19 h, puis retrouve ce réglage dans sa page Compte, le passe à 8 h et ajoute les notifications sur son appareil.

**Acceptance Scenarios**:

1. **Given** un inscrit qui n'a jamais appris de sujet ni réglé de rappel, **When** il choisit « Apprendre ce sujet », **Then** une proposition d'activer les rappels s'affiche, avec les canaux notification et email non cochés et l'heure préréglée à 19 h.
2. **Given** la proposition affichée, **When** il choisit « Plus tard », **Then** aucun rappel n'est activé et la proposition ne lui est plus présentée ; les réglages restent accessibles dans la page Compte.
3. **Given** la proposition affichée, **When** il coche l'email et valide, **Then** les rappels par email sont activés à 19 h dans son fuseau horaire, et une confirmation l'indique.
4. **Given** un inscrit dans sa page Compte, **When** il active les notifications sur son appareil, **Then** le navigateur lui demande l'autorisation ; s'il accepte, cet appareil est enregistré et le réglage l'indique ; s'il refuse, un message explique comment l'autoriser plus tard dans les réglages du navigateur, et rien n'est enregistré.
5. **Given** un inscrit sur iPhone qui n'a pas installé l'application, **When** il veut activer les notifications, **Then** un message lui explique qu'il doit d'abord installer CINQ sur son écran d'accueil, avec le lien vers les instructions d'installation.
6. **Given** des rappels activés, **When** il change l'heure de 19 h à 8 h, **Then** le prochain rappel part à 8 h.
7. **Given** des notifications activées sur un téléphone et un ordinateur, **When** il consulte la section « Rappels », **Then** il voit les deux appareils et peut couper les notifications de chacun séparément.

---

### User Story 2 - Recevoir le rappel du jour (Priority: P1)

À l'heure choisie, si l'apprenant a des cartes à réviser dans des sujets publiés, il reçoit un rappel sur chaque canal activé : « 12 cartes à réviser aujourd'hui ». Le rappel ouvre directement la séance de révision sur les sujets concernés.

**Why this priority**: C'est la valeur de la feature : revenir réviser au bon moment.

**Independent Test**: Un apprenant avec 12 cartes dues et l'email activé à 19 h reçoit un email à 19 h qui annonce 12 cartes ; le lien ouvre une séance de 12 cartes. Le lendemain, s'il n'a aucune carte due, il ne reçoit rien.

**Acceptance Scenarios**:

1. **Given** un apprenant avec l'email activé à 19 h et 12 cartes à réviser aujourd'hui, réparties sur 2 sujets, **When** il est 19 h dans son fuseau horaire, **Then** il reçoit un email annonçant 12 cartes à réviser aujourd'hui, avec un bouton qui ouvre la séance sur ces 2 sujets.
2. **Given** les notifications activées sur 2 appareils, **When** le rappel part, **Then** chacun des 2 appareils reçoit une notification ; la toucher ouvre la séance sur les sujets concernés.
3. **Given** un apprenant qui a déjà révisé toutes ses cartes du jour avant l'heure du rappel, **When** l'heure arrive, **Then** aucun rappel ne part.
4. **Given** un apprenant dont toutes les cartes dues appartiennent à des sujets dépubliés ou retirés, **When** l'heure arrive, **Then** aucun rappel ne part.
5. **Given** un rappel déjà envoyé aujourd'hui, **When** l'apprenant change l'heure pour une heure plus tardive le même jour, **Then** il ne reçoit pas de second rappel ce jour-là.
6. **Given** un apprenant qui n'apprend plus aucun sujet, **When** l'heure arrive, **Then** il ne reçoit rien.
7. **Given** un apprenant qui ouvre le lien d'un rappel sans être connecté, **When** il se connecte, **Then** il arrive sur la séance des sujets concernés.

---

### User Story 3 - Des rappels qui s'espacent (Priority: P2)

Si l'apprenant ne révise plus, les rappels s'espacent pour ne pas devenir envahissants : tous les jours d'abord, puis tous les deux jours, puis une fois par semaine. Dès qu'il révise à nouveau, le rythme quotidien reprend.

**Why this priority**: Un rappel quotidien ignoré pendant des semaines pousse à tout couper, ou à marquer les emails comme indésirables. L'espacement protège l'apprenant et la réputation des envois de CINQ.

**Independent Test**: Un apprenant qui ne révise plus reçoit un rappel les 7 premiers jours, puis un jour sur deux jusqu'au 21e jour, puis un par semaine ; il révise une carte et reçoit de nouveau un rappel le lendemain.

**Acceptance Scenarios**:

1. **Given** un apprenant qui a révisé pour la dernière fois il y a 3 jours et a des cartes à réviser, **When** l'heure arrive, **Then** il reçoit le rappel (rythme quotidien).
2. **Given** un apprenant qui n'a pas révisé depuis 10 jours, **When** l'heure arrive un jour où il a reçu un rappel la veille, **Then** il ne reçoit rien ce jour-là (rythme d'un jour sur deux).
3. **Given** un apprenant qui n'a pas révisé depuis 30 jours, **When** l'heure arrive moins de 7 jours après son dernier rappel, **Then** il ne reçoit rien (rythme hebdomadaire).
4. **Given** un apprenant au rythme hebdomadaire, **When** il révise au moins une carte, **Then** il reçoit de nouveau un rappel dès le jour suivant où il a des cartes à réviser.

---

### User Story 4 - Couper les rappels (Priority: P2)

L'apprenant peut arrêter les rappels à tout moment : un clic sur le lien de désinscription d'un email coupe les emails de rappel, sans avoir à se connecter. Il peut aussi couper les notifications d'un appareil, ou tous les rappels, depuis sa page Compte.

**Why this priority**: La désinscription simple est une obligation légale pour les emails et une condition de confiance. Elle vient après l'activation et l'envoi, sans lesquels il n'y a rien à couper.

**Independent Test**: Depuis un email de rappel, un clic sur « Ne plus recevoir ces rappels » affiche une confirmation sans connexion ; le lendemain, plus aucun email de rappel n'arrive, alors que les notifications continuent si elles étaient activées.

**Acceptance Scenarios**:

1. **Given** un email de rappel, **When** le destinataire clique sur « Ne plus recevoir ces rappels », **Then** une page confirme que les emails de rappel sont coupés, sans demander de connexion, et plus aucun email de rappel ne part pour lui.
2. **Given** un email de rappel ouvert dans une messagerie qui propose un bouton de désinscription, **When** il l'utilise, **Then** les emails de rappel sont coupés de la même façon.
3. **Given** un lien de désinscription modifié ou inventé, **When** quelqu'un l'ouvre, **Then** aucun réglage n'est modifié et une page indique que le lien n'est pas valide, sans révéler à quel compte il se rapporte.
4. **Given** des notifications activées sur 2 appareils, **When** l'apprenant coupe celles de son ordinateur, **Then** seul son téléphone reçoit encore les notifications.
5. **Given** des rappels coupés par le lien d'un email, **When** l'apprenant ouvre sa page Compte, **Then** il voit l'email de rappel désactivé et peut le réactiver.
6. **Given** un appareil dont l'utilisateur a retiré l'autorisation de notifications dans son navigateur, **When** le prochain rappel part, **Then** cet appareil est retiré de la liste, sans erreur visible pour l'utilisateur, et les autres canaux continuent.

---

### Edge Cases

- **Heure déjà passée le jour de l'activation** : si l'apprenant active les rappels à 20 h avec une heure réglée à 19 h, le premier rappel part le lendemain à 19 h.
- **Changement d'heure (heure d'été ou d'hiver)** : le rappel part à l'heure locale choisie, quel que soit le décalage du jour ; un jour où cette heure n'existe pas, il part à l'heure suivante.
- **Changement de fuseau horaire** : le fuseau est celui du compte, mis à jour à chaque connexion (feature 001) ; le rappel suit le nouveau fuseau dès le jour suivant, sans jamais partir deux fois le même jour.
- **Nombre de cartes au moment de l'envoi** : le nombre annoncé est celui des cartes à réviser au moment de l'envoi, pas au moment du réglage.
- **Email refusé par la messagerie du destinataire** : après des refus définitifs répétés, les emails de rappel de ce compte sont désactivés, et la page Compte l'indique.
- **Notification non délivrée** : un appareil qui ne peut plus recevoir de notifications (autorisation retirée, application désinstallée) est retiré de la liste ; les autres appareils et l'email ne sont pas affectés.
- **Plusieurs rappels possibles le même jour** : un seul rappel part par jour et par compte, même s'il a plusieurs canaux ; chaque canal activé reçoit ce rappel une fois.
- **Le lien du rappel ouvert le lendemain** : la séance porte sur les cartes à réviser au moment où elle s'ouvre ; s'il n'y en a plus, l'écran « Rien à réviser » de la feature 001 s'affiche.
- **Sujet arrêté entre le réglage et l'envoi** : ses cartes ne comptent plus ; si plus rien n'est à réviser, aucun rappel ne part.
- **Compte non confirmé ou supprimé** : il ne reçoit aucun rappel.

## Requirements *(mandatory)*

### Functional Requirements

**Activation et réglages**

- **FR-001**: Les rappels DOIVENT être désactivés par défaut pour tout compte ; aucun rappel ne part sans une action explicite de l'utilisateur.
- **FR-002**: Après le premier « Apprendre ce sujet » d'un compte qui n'a jamais réglé ses rappels, le système DOIT proposer une seule fois d'activer les rappels, avec les canaux non cochés, l'heure préréglée à 19 h, et un choix « Plus tard » qui n'active rien.
- **FR-003**: La page Compte DOIT offrir une section « Rappels » où l'utilisateur active ou désactive chaque canal (notifications, email) et choisit l'heure du rappel, à l'heure pleine ou à la demi-heure.
- **FR-004**: L'activation des notifications DOIT demander l'autorisation du navigateur ; en cas de refus, rien n'est enregistré et un message explique comment l'autoriser plus tard.
- **FR-005**: Sur un appareil qui ne peut recevoir de notifications qu'une fois l'application installée (iPhone et iPad), le système DOIT l'expliquer et renvoyer vers les instructions d'installation au lieu de proposer l'activation.
- **FR-006**: Un compte DOIT pouvoir recevoir les notifications sur plusieurs appareils ; la section « Rappels » liste ces appareils et permet de couper les notifications de chacun séparément.

**Envoi**

- **FR-007**: Le système DOIT envoyer au plus un rappel par jour et par compte, à l'heure choisie dans le fuseau horaire du compte, sur chaque canal activé.
- **FR-008**: Un rappel NE DOIT partir que si le compte a au moins une carte à réviser ce jour-là dans des sujets publiés, selon la règle de la feature 001 ; le nombre annoncé est celui du moment de l'envoi.
- **FR-009**: Un rappel DOIT annoncer le nombre de cartes à réviser aujourd'hui et proposer un lien qui ouvre la séance de révision sur les sujets qui ont des cartes à réviser.
- **FR-010**: Le lien d'un rappel ouvert par une personne non connectée DOIT la mener à la connexion, puis à la séance demandée.
- **FR-011**: Les rappels DOIVENT être rédigés en français, en vouvoyant l'utilisateur, et ne DOIVENT contenir aucune autre information personnelle que le nom affiché et le nombre de cartes.

**Espacement**

- **FR-012**: Le rythme des rappels DOIT dépendre du nombre de jours écoulés depuis la dernière révision du compte (ou depuis l'activation s'il n'a jamais révisé) : un rappel par jour jusqu'au 7e jour, un jour sur deux du 8e au 21e jour, puis un par semaine.
- **FR-013**: Toute réponse donnée en séance DOIT remettre le rythme à un rappel par jour.

**Désinscription et arrêt**

- **FR-014**: Chaque email de rappel DOIT contenir un lien de désinscription propre au compte, infalsifiable, qui coupe les emails de rappel en un clic, sans connexion, et affiche une confirmation.
- **FR-015**: Chaque email de rappel DOIT permettre la désinscription en un clic depuis la messagerie du destinataire, lorsque celle-ci le propose.
- **FR-016**: Un lien de désinscription invalide NE DOIT modifier aucun réglage ni révéler l'existence d'un compte.
- **FR-017**: Un appareil qui ne peut plus recevoir de notifications DOIT être retiré de la liste des appareils du compte au premier échec définitif d'envoi.
- **FR-018**: Après des refus définitifs répétés d'une adresse email, le système DOIT désactiver les emails de rappel de ce compte et l'indiquer dans la section « Rappels ».
- **FR-019**: Un compte qui n'apprend aucun sujet, dont l'adresse n'est pas confirmée, ou dont tous les canaux sont désactivés NE DOIT recevoir aucun rappel.

**Suivi**

- **FR-020**: Le système DOIT conserver, pour chaque compte, la date du dernier rappel envoyé et les canaux qui l'ont reçu, pour garantir l'unicité quotidienne et l'espacement.

### Key Entities

- **Réglages de rappel** : pour chaque compte, les canaux activés (email, notifications), l'heure du rappel, la date d'activation, l'état « proposition déjà faite », et l'éventuelle désactivation automatique de l'email.
- **Appareil de notification** : un appareil ou navigateur autorisé à recevoir les notifications d'un compte, avec un nom lisible (« Chrome sur Android »), sa date d'enregistrement et sa dernière réception réussie.
- **Rappel envoyé** : pour chaque compte et chaque jour, le rappel envoyé, le nombre de cartes annoncé et les canaux qui l'ont reçu ; il sert à l'unicité quotidienne et au calcul de l'espacement.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Un utilisateur active ses rappels en moins de 30 secondes à partir de la proposition qui suit son premier « Apprendre ce sujet ».
- **SC-002**: 99 % des rappels partent dans les 5 minutes qui suivent l'heure choisie par l'utilisateur.
- **SC-003**: Aucun compte ne reçoit plus d'un rappel par jour, et aucun rappel ne part pour un compte sans carte à réviser. Ceci est vérifié par des tests qui couvrent chaque cas de FR-007, FR-008 et FR-019.
- **SC-004**: 100 % des clics sur un lien de désinscription valide coupent les emails de rappel dès l'envoi suivant, sans connexion.
- **SC-005**: Le rythme d'espacement est respecté à chaque jour d'inactivité, du 1er au 60e jour. Ceci est vérifié par des tests qui couvrent chaque palier de FR-012 et la reprise de FR-013.
- **SC-006**: Le nombre de cartes annoncé par un rappel est égal au nombre de cartes proposées par la séance ouverte depuis ce rappel au même moment.
- **SC-007**: La part des apprenants qui reviennent réviser le jour d'un rappel reçu est mesurable, pour juger de l'utilité des rappels après la mise en service.

## Assumptions

- **Dépend de la feature 001** : comptes, fuseau horaire du compte mis à jour à la connexion, apprentissages, cartes à réviser, séance sur plusieurs sujets, installation de la PWA.
- **Hors périmètre** : les rappels par SMS, les rappels pour un sujet précis, plusieurs rappels par jour, les résumés hebdomadaires, les notifications pour d'autres événements (modération, nouveaux sujets d'un auteur), la personnalisation du texte du rappel.
- **Heure du rappel** : choisie par tranches de 30 minutes, de 6 h à 23 h 30. Le rappel de 19 h par défaut correspond au moment où l'on dispose le plus souvent de quelques minutes.
- **Paliers d'espacement** : 7 jours de rythme quotidien, puis un jour sur deux jusqu'au 21e jour, puis une fois par semaine, compté depuis la dernière réponse donnée en séance. Ces paliers sont une valeur par défaut raisonnable, ajustable après mesure (SC-007).
- **Refus d'email répétés** : 3 refus définitifs consécutifs désactivent les emails de rappel du compte.
- **Nom d'un appareil** : il est déduit du navigateur et du système (« Safari sur iPhone », « Chrome sur Windows ») ; l'utilisateur ne le saisit pas.
- **Notifications hors ligne** : une notification arrive même si l'application est fermée, dans les limites du système de l'appareil (mode concentration, économie d'énergie).
- **Données personnelles** : les réglages et appareils de notification sont supprimés avec le compte, quand la suppression de compte sera livrée (feature dédiée prévue avant l'ouverture au public).
