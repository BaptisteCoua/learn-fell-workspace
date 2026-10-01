# Feature Specification: Révision hors ligne

**Feature Branch**: `006-offline-review`

**Created**: 2026-10-01

**Status**: Draft

**Input**: User description: "Révision hors ligne : la plus grosse feature, celle qui justifie la PWA. Mettre en cache les cartes dues, enregistrer les réponses en local, puis synchroniser. Choix du développeur : l'app garde les cartes dues aujourd'hui et dans les 7 jours ; texte seulement, sans les images ; une réponse hors ligne compte à la date où elle a été donnée ; si la même carte est révisée sur deux appareils, la première réponse compte."

## Contexte

CINQ n'a pas d'application native : la PWA est son unique expérience mobile. Or on révise souvent là où le réseau manque (métro, train, avion, zone blanche), et la méthode Leitner ne pardonne pas un jour manqué. Aujourd'hui, sans connexion, CINQ affiche seulement « Vous êtes hors ligne » (feature 001, FR-039) : impossible de réviser.

Cette feature rend la révision possible sans réseau. Tant qu'elle est en ligne, l'app garde sur l'appareil les cartes que la personne aura à réviser dans les jours qui viennent. Hors ligne, elle peut ouvrir « Mes révisions », lancer une séance et répondre comme d'habitude. Ses réponses restent sur l'appareil, puis partent d'elles-mêmes au retour du réseau, en comptant à la date où elles ont été données.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Réviser sans réseau (Priority: P1)

Une apprenante ouvre CINQ dans le métro, sans réseau. Elle voit « Mes révisions » avec ses cartes du jour, lance une séance et répond comme en ligne : recto, verso, « Je savais » ou « Je ne savais pas », nouvelle boîte affichée après chaque réponse, bilan en fin de séance.

**Why this priority**: C'est la valeur de la feature : ne plus manquer un jour de révision faute de réseau.

**Independent Test**: Avec 12 cartes dues sur 2 sujets, l'appareil passe hors ligne ; « Mes révisions » montre les 2 sujets et leurs 12 cartes ; une séance se lance et se termine par un bilan, sans aucune erreur de réseau.

**Acceptance Scenarios**:

1. **Given** une apprenante qui a ouvert CINQ en ligne sur cet appareil depuis moins de 7 jours, **When** elle l'ouvre sans réseau, **Then** « Mes révisions » s'affiche avec ses sujets appris, leurs cartes à réviser aujourd'hui et la répartition dans les boîtes, accompagnés de l'indication « Hors ligne — cartes à jour du {date} ».
2. **Given** « Mes révisions » hors ligne, **When** elle sélectionne des sujets et lance une séance, **Then** la séance contient les cartes à réviser aujourd'hui de ces sujets, des plus en retard aux plus récentes, comme en ligne (feature 001, FR-044).
3. **Given** une séance hors ligne, **When** elle affiche le verso puis répond, **Then** la carte change de boîte selon les règles de la feature 001 (FR-046), la nouvelle boîte et la date de retour s'affichent (FR-047), et la carte n'est pas reposée dans la séance.
4. **Given** une séance hors ligne terminée, **When** la dernière carte reçoit sa réponse, **Then** le bilan s'affiche (FR-049), comme en ligne.
5. **Given** une carte dont le recto contient une image, **When** elle s'affiche hors ligne, **Then** l'image est remplacée par sa description, signalée comme « Image non disponible hors ligne ».
6. **Given** une carte due dans 3 jours d'après les données gardées, **When** l'apprenante ouvre CINQ hors ligne ce jour-là, **Then** la carte fait partie des cartes à réviser du jour.

---

### User Story 2 - Retrouver ses réponses sur le serveur (Priority: P1)

De retour en ligne, les réponses données hors ligne partent d'elles-mêmes. Elles comptent à la date où elles ont été données : une carte révisée lundi hors ligne et envoyée mercredi revient selon la date de lundi.

**Why this priority**: Sans synchronisation fidèle, réviser hors ligne ne servirait à rien, ou fausserait le calendrier Leitner.

**Independent Test**: Lundi, hors ligne, 5 réponses ; mercredi, retour du réseau ; en moins d'une minute, les 5 réponses sont sur le serveur, et chaque carte revient à la date calculée depuis lundi, sur tous les appareils.

**Acceptance Scenarios**:

1. **Given** des réponses données hors ligne, **When** l'appareil retrouve le réseau, l'app étant ouverte, **Then** les réponses sont envoyées sans action de l'apprenante, et l'indicateur « {n} réponses à envoyer » disparaît une fois l'envoi terminé.
2. **Given** des réponses données hors ligne et l'app fermée, **When** l'apprenante rouvre CINQ en ligne, **Then** les réponses partent avant tout affichage des révisions, pour que les chiffres soient justes.
3. **Given** une carte en boîte 2 révisée « Je savais » lundi hors ligne et envoyée mercredi, **When** le serveur l'enregistre, **Then** elle passe en boîte 3 avec une prochaine révision le vendredi suivant, soit lundi plus 4 jours.
4. **Given** des réponses envoyées, **When** l'apprenante consulte « Mes révisions » sur un autre appareil, **Then** elle y voit la progression à jour.
5. **Given** un envoi interrompu par une nouvelle coupure, **When** le réseau revient, **Then** l'envoi reprend sans perdre ni doubler aucune réponse.

---

### User Story 3 - Une carte révisée sur deux appareils (Priority: P2)

L'apprenant révise une carte hors ligne sur son téléphone, puis la même carte en ligne sur son ordinateur. Seule la première réponse compte, comme en ligne, où une carte n'est révisable qu'une fois par échéance.

**Why this priority**: Le cas est rare, mais il ne doit ni fausser la progression ni afficher d'erreur incompréhensible.

**Independent Test**: Une carte due mardi, révisée « Je ne savais pas » à 8 h hors ligne sur le téléphone et « Je savais » à 12 h en ligne sur l'ordinateur ; au retour du réseau du téléphone à 18 h, la carte est en boîte 1 : la réponse de 8 h compte et celle de 12 h est annulée.

**Acceptance Scenarios**:

1. **Given** une carte révisée sur deux appareils pour la même échéance, **When** les deux réponses sont sur le serveur, **Then** seule la plus ancienne compte, et la progression de la carte est celle qu'elle donne.
2. **Given** une réponse hors ligne ignorée parce qu'une réponse plus ancienne existe déjà, **When** la synchronisation se termine, **Then** aucune erreur ne s'affiche, et les chiffres de l'appareil sont remis à jour depuis le serveur.
3. **Given** une réponse hors ligne sur une carte qui n'existe plus (question supprimée, sujet arrêté, dépublié, retiré, supprimé ou retenu), **When** elle est envoyée, **Then** elle est ignorée sans erreur, comme le serait une réponse en ligne sur cette carte.

---

### User Story 4 - Garder les données hors ligne à jour et à soi (Priority: P2)

Tant qu'elle est en ligne, l'app garde d'elle-même les cartes des 7 prochains jours, sans action de l'apprenante. Ces données appartiennent au compte connecté : se déconnecter les efface de l'appareil, en prévenant s'il reste des réponses non envoyées.

**Why this priority**: Sans mise à jour, les cartes gardées vieillissent ; sans effacement, un appareil partagé exposerait la progression de quelqu'un d'autre.

**Independent Test**: Un apprenant apprend un nouveau sujet en ligne, passe hors ligne : ses cartes sont disponibles. Il se déconnecte : plus aucune carte ni réponse de son compte ne reste sur l'appareil.

**Acceptance Scenarios**:

1. **Given** un apprenant en ligne, **When** il ouvre CINQ, apprend un sujet, arrête d'en apprendre un ou termine une séance, **Then** les cartes gardées sur l'appareil sont mises à jour.
2. **Given** des réponses non envoyées, **When** l'apprenant choisit de se déconnecter, **Then** un message indique « {n} réponses ne sont pas encore envoyées » et propose d'attendre le réseau ou de se déconnecter quand même en les perdant.
3. **Given** une déconnexion, **When** elle a lieu, **Then** les cartes gardées et les réponses de ce compte sont effacées de l'appareil.
4. **Given** un autre compte qui se connecte sur le même appareil, **When** il ouvre « Mes révisions », **Then** il ne voit que ses propres cartes.
5. **Given** des cartes gardées depuis plus de 7 jours sans retour en ligne, **When** l'apprenant ouvre CINQ hors ligne, **Then** il ne peut réviser que les cartes dont l'échéance tombe dans les 7 jours qui suivent la dernière mise à jour, et un message l'invite à se reconnecter au réseau pour la suite.

---

### Edge Cases

- **Jamais ouvert en ligne sur cet appareil** : hors ligne, l'écran « Vous êtes hors ligne » de la feature 001 s'affiche, avec la mention que la révision hors ligne sera possible après une première ouverture en ligne.
- **Appareil sans stockage disponible** (navigation privée, stockage plein ou refusé) : la révision hors ligne est indisponible ; l'app l'indique dans « Mes révisions » et fonctionne en ligne comme avant.
- **Contenu modifié par l'auteur pendant la coupure** : hors ligne, la carte s'affiche telle qu'elle était à la dernière mise à jour ; la réponse s'applique à la question, même modifiée entre-temps.
- **Nouvelle question ajoutée pendant la coupure** : elle apparaît à la prochaine mise à jour en ligne.
- **Date de l'appareil incohérente** : une réponse datée dans le futur ou avant l'échéance de la carte n'est pas acceptée telle quelle ; elle compte au moment de son envoi si la carte est due à ce moment, sinon elle est ignorée.
- **Réponses dans le désordre** : elles sont appliquées dans l'ordre où elles ont été données, pas dans l'ordre d'arrivée.
- **Rappels de révision** (feature 002) : une réponse envoyée plus tard compte pour l'espacement des rappels à sa date de réponse ; un rappel parti avant l'envoi annonce les cartes connues du serveur à ce moment.
- **Session expirée pendant la coupure** : les réponses restent sur l'appareil ; elles partent dès que le même compte se reconnecte, et sont effacées si un autre compte se connecte.
- **Compte en cours de suppression** (feature 004) : la demande ferme les sessions, donc l'app efface les données hors ligne à la déconnexion ; une reconnexion qui annule la demande les recrée.
- **Coupure en pleine séance en ligne** : la séance continue sans interruption ; les réponses suivantes sont gardées sur l'appareil et partent au retour du réseau.

## Requirements *(mandatory)*

### Functional Requirements

**Données gardées sur l'appareil**

- **FR-001**: Pour le compte connecté, l'app DOIT garder sur l'appareil les cartes de ses sujets appris publiés dont la prochaine révision tombe au plus tard 7 jours après la dernière mise à jour, avec leur recto, leur verso, leur boîte, leur échéance et le titre de leur sujet.
- **FR-002**: Les images des cartes NE DOIVENT PAS être gardées ; hors ligne, chaque image est remplacée par sa description, signalée comme indisponible hors ligne.
- **FR-003**: L'app DOIT mettre à jour ces données sans action de l'apprenant, à chaque ouverture en ligne, après chaque séance, et après chaque sujet appris ou arrêté.
- **FR-004**: Les données gardées et les réponses en attente DOIVENT être rattachées au compte connecté, effacées à sa déconnexion, et jamais montrées à un autre compte.

**Réviser hors ligne**

- **FR-005**: Hors ligne, « Mes révisions » DOIT afficher, depuis les données gardées, les sujets appris, leurs cartes à réviser ce jour-là, la répartition dans les boîtes, et la date de la dernière mise à jour.
- **FR-006**: Hors ligne, une séance DOIT suivre les règles de la feature 001 : cartes à réviser ce jour-là des sujets choisis, des plus en retard aux plus récentes (FR-044), verso avant réponse (FR-045), changement de boîte (FR-046), boîte et date de retour affichées (FR-047), bilan (FR-049).
- **FR-007**: « Ce jour-là » DOIT s'entendre dans le fuseau du compte, avec la date de l'appareil ; une carte gardée devient à réviser le jour de son échéance, même hors ligne.
- **FR-008**: Chaque réponse hors ligne DOIT être enregistrée sur l'appareil dès qu'elle est donnée, avec sa date et son heure, et ne DOIT jamais être perdue, même si l'app est fermée ou l'appareil redémarré.
- **FR-009**: L'app DOIT indiquer le nombre de réponses en attente d'envoi tant qu'il en reste.

**Synchroniser**

- **FR-010**: Les réponses en attente DOIVENT partir sans action de l'apprenant dès que le réseau revient pendant que l'app est ouverte, et à chaque ouverture en ligne, avant l'affichage des révisions.
- **FR-011**: Le serveur DOIT appliquer chaque réponse à la date et à l'heure où elle a été donnée : la nouvelle boîte et la prochaine échéance se calculent depuis cette date (feature 001, FR-042).
- **FR-012**: Les réponses d'un compte DOIVENT être appliquées dans l'ordre où elles ont été données.
- **FR-013**: Pour une carte et une échéance données, seule la réponse la plus ancienne DOIT compter, qu'elle vienne d'un appareil hors ligne ou en ligne ; une réponse plus récente pour la même échéance est annulée, et la progression de la carte est recalculée depuis la plus ancienne.
- **FR-014**: Une réponse sur une carte qui n'existe plus ou n'est plus révisable (question supprimée, sujet arrêté, dépublié, retiré, retenu ou supprimé) DOIT être ignorée sans erreur visible.
- **FR-015**: Une réponse datée dans le futur, ou avant l'échéance de sa carte, NE DOIT PAS être appliquée à cette date : elle compte au moment de son envoi si la carte est due à ce moment, sinon elle est ignorée.
- **FR-016**: Une synchronisation interrompue DOIT pouvoir reprendre sans perdre ni appliquer deux fois une même réponse.
- **FR-017**: Après chaque synchronisation, l'app DOIT remettre à jour ses données gardées depuis le serveur.

**Déconnexion**

- **FR-018**: Si des réponses sont en attente, la déconnexion DOIT d'abord avertir l'apprenant de leur nombre, et proposer d'attendre le réseau ou de se déconnecter en les perdant.

**Limites et messages**

- **FR-019**: Si l'appareil ne permet pas de garder des données, l'app DOIT l'indiquer dans « Mes révisions » et fonctionner en ligne comme avant.
- **FR-020**: Hors ligne, les autres actions qui enregistrent (créer, publier, signaler, régler ses rappels) restent suspendues comme dans la feature 001 (FR-039) ; seules les réponses de révision sont gardées pour plus tard.
- **FR-021**: Tous les textes DOIVENT être en français, en vouvoyant l'apprenant.

### Key Entities

- **Paquet hors ligne** : pour un compte et un appareil, les cartes gardées (question, recto, verso, descriptions des images, sujet, boîte, échéance) et la date de la dernière mise à jour.
- **Réponse en attente** : une réponse donnée hors ligne, avec la carte, « Je savais » ou « Je ne savais pas », la date et l'heure, et un identifiant unique qui empêche de l'appliquer deux fois.
- **Réponse** (existante, feature 001) : désormais datée du moment où elle a été donnée, qui peut précéder son enregistrement sur le serveur.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Une apprenante qui a ouvert CINQ en ligne dans les 7 derniers jours lance une séance hors ligne en moins de 5 secondes depuis l'ouverture de l'app.
- **SC-002**: 100 % des réponses données hors ligne arrivent sur le serveur, une fois et une seule, dans la minute qui suit le retour du réseau avec l'app ouverte. Ceci est vérifié par des tests qui coupent et rétablissent la connexion pendant l'envoi.
- **SC-003**: Pour toute réponse hors ligne, la boîte et l'échéance calculées sur le serveur sont identiques à celles affichées hors ligne au moment de la réponse, sauf dans les cas de FR-013 à FR-015.
- **SC-004**: Après une déconnexion, aucune carte ni réponse du compte ne reste sur l'appareil.
- **SC-005**: Les règles Leitner restent couvertes à 100 % de leurs cas, en ligne et hors ligne, comme l'exige la constitution (principe IV).

## Assumptions

- **Dépend des features 001 à 005** : règles Leitner et séances (001), rappels et leur espacement (002), descriptions des images (003), sujets retenus et sessions fermées à la demande de suppression (004).
- **Lève une limite de la feature 001** : « la consultation hors ligne n'est pas prévue » ; seule la révision devient possible hors ligne, pas le catalogue, la création ni la modération.
- **Horizon de 7 jours** : choix du développeur ; il couvre une semaine sans réseau, et une carte en boîte 1 à 4 revient dans ce délai.
- **Sans images** : choix du développeur, pour garder les données légères ; les descriptions des images, obligatoires depuis la feature 003, permettent de réviser la plupart de ces cartes.
- **Date de la réponse** : choix du développeur ; la date de l'appareil fait foi dans les limites de FR-015.
- **Première réponse** : choix du développeur ; c'est la règle qui vaut déjà en ligne, où une carte n'est révisable qu'une fois par échéance.
- **Conservation par le navigateur** : un navigateur peut effacer les données d'un site peu utilisé (Safari les efface après 7 jours sans visite) ; les réponses en attente sont alors perdues, et l'app le signale à la prochaine ouverture si elle peut le détecter.
- **Navigateurs** : ceux qui gèrent déjà la PWA de CINQ, installée ou non, une fois le site ouvert une première fois en ligne.
