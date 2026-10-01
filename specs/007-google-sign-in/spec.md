# Feature Specification: Connexion avec Google

**Feature Branch**: `007-google-sign-in`

**Created**: 2026-10-01

**Status**: Draft

**Input**: User description: "Connexion avec Google : s'inscrire et se connecter à CINQ avec un compte Google, à la place du bouton « bientôt » déjà affiché sur les écrans de connexion et d'inscription."

## Contexte

Depuis la feature 001, les écrans de connexion et d'inscription affichent un bouton « Continuer avec Google », désactivé et marqué « Bientôt ». Aujourd'hui, seul un compte avec email et mot de passe permet d'entrer dans CINQ. L'inscription demande en plus de confirmer son adresse par un lien reçu par email, une étape que beaucoup abandonnent sur téléphone.

Cette feature active ce bouton. Une personne peut créer son compte et se connecter avec son compte Google, sans mot de passe CINQ ni email de confirmation, puisque Google a déjà vérifié son adresse. Le reste de CINQ ne change pas : sujets, révisions, rappels et suppression de compte fonctionnent de la même façon, quel que soit le mode de connexion.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Créer son compte avec Google (Priority: P1)

Une visiteuse ouvre « Créer un compte », choisit « Continuer avec Google », s'identifie chez Google et accepte de partager son nom et son adresse. Elle revient sur CINQ avec un compte actif et connecté, sans email de confirmation à ouvrir.

**Why this priority**: C'est la valeur de la feature : entrer dans CINQ en deux gestes, sans mot de passe à inventer ni email à attendre.

**Independent Test**: Une adresse Google jamais vue par CINQ ; après « Continuer avec Google », le compte est actif, la personne est connectée, son fuseau horaire est celui de son appareil, et aucun email de confirmation n'est parti.

**Acceptance Scenarios**:

1. **Given** une visiteuse dont l'adresse Google n'a pas de compte CINQ, **When** elle choisit « Continuer avec Google » et accepte chez Google, **Then** un compte actif est créé avec cette adresse, elle est connectée, et aucun lien de confirmation ne lui est envoyé.
2. **Given** une visiteuse revenue de Google sans compte CINQ, **When** elle arrive sur CINQ, **Then** un écran lui demande son nom affiché, prérempli avec son nom Google et accompagné de la mention qu'il sera visible sur ses sujets ; elle le garde ou le modifie, et le compte est créé quand elle valide.
3. **Given** ce nouveau compte, **When** il est créé, **Then** son fuseau horaire est celui de l'appareil, comme pour une inscription par email.
4. **Given** une visiteuse sur l'écran Google, **When** elle refuse ou ferme la fenêtre, **Then** elle revient sur l'écran de CINQ d'où elle est partie, sans compte créé, avec un message qui l'invite à réessayer ou à utiliser son email.
5. **Given** une visiteuse qui avait demandé une page réservée aux inscrits, **When** elle termine l'inscription avec Google, **Then** elle arrive sur cette page, comme après une connexion par email.
6. **Given** l'écran du nom affiché, **When** elle le quitte sans valider, **Then** aucun compte n'est créé.

---

### User Story 2 - Se connecter avec Google (Priority: P1)

Un inscrit qui a créé son compte avec Google revient sur CINQ, sur un autre appareil ou après s'être déconnecté. Il choisit « Continuer avec Google » sur l'écran de connexion et retrouve son compte, ses sujets et sa progression.

**Why this priority**: Sans connexion, le compte créé en US1 ne sert qu'une fois.

**Independent Test**: Un compte créé avec Google, déconnecté ; « Continuer avec Google » sur l'écran de connexion ouvre ce même compte, avec ses sujets et ses révisions.

**Acceptance Scenarios**:

1. **Given** un compte lié à un compte Google, **When** la personne choisit « Continuer avec Google » et s'identifie avec ce compte Google, **Then** elle est connectée à son compte CINQ, et son fuseau horaire est mis à jour comme à toute connexion.
2. **Given** une connexion avec Google, **When** elle réussit, **Then** elle reste ouverte 30 jours sur cet appareil, sauf déconnexion, comme une connexion par email.
3. **Given** un compte lié à Google dont la suppression a été demandée il y a moins de 30 jours, **When** la personne se connecte avec Google, **Then** la demande est annulée et un message le confirme, comme pour une connexion par email (feature 004, FR-014).
4. **Given** un compte lié à Google, **When** la personne change l'adresse ou le nom de son compte Google, **Then** elle se connecte toujours au même compte CINQ, dont l'adresse et le nom affiché ne changent pas.

---

### User Story 3 - Utiliser Google avec un compte déjà créé par email (Priority: P2)

Une inscrite a créé son compte CINQ avec son adresse Gmail et un mot de passe. Un jour, elle choisit « Continuer avec Google » avec cette même adresse.

**Why this priority**: C'est le cas le plus fréquent chez les premiers inscrits. Il ne doit ni créer un second compte, ni ouvrir un compte à quelqu'un d'autre que son titulaire.

**Independent Test**: Un compte confirmé créé par email avec l'adresse X ; « Continuer avec Google » avec le compte Google d'adresse X ouvre ce compte, avec ses sujets et sa progression, sans jamais créer de second compte avec X.

**Acceptance Scenarios**:

1. **Given** un compte confirmé créé par email avec l'adresse X, **When** la personne choisit « Continuer avec Google » avec un compte Google d'adresse X, **Then** le compte existant est lié à Google et ouvert directement, sans mot de passe, et un message lui indique qu'elle pourra désormais se connecter avec Google.
2. **Given** un compte créé par email et lié ensuite à Google, **When** la personne se connecte avec son mot de passe, **Then** la connexion fonctionne toujours : les deux modes ouvrent le même compte.
3. **Given** un compte en attente de confirmation, créé par email avec l'adresse X, **When** une personne choisit « Continuer avec Google » avec l'adresse X, **Then** le compte est confirmé et lié à Google, et le mot de passe saisi à l'inscription est retiré, car rien ne prouve qu'il a été choisi par la titulaire de l'adresse.

---

### User Story 4 - Supprimer un compte créé avec Google (Priority: P2)

Un inscrit qui n'a jamais eu de mot de passe CINQ veut supprimer son compte. La feature 004 demande de confirmer la suppression en saisissant son mot de passe ; il la confirme ici en s'identifiant de nouveau chez Google.

**Why this priority**: La suppression de compte est une obligation (RGPD, constitution VI) ; elle doit rester possible pour chaque compte.

**Independent Test**: Un compte créé avec Google, sans mot de passe ; il demande la suppression, confirme chez Google, et la suppression suit les règles de la feature 004 : déconnexion partout, email de confirmation, effacement au bout de 30 jours.

**Acceptance Scenarios**:

1. **Given** un compte sans mot de passe, **When** la personne ouvre « Supprimer mon compte », **Then** l'écran demande de confirmer avec Google au lieu de saisir un mot de passe.
2. **Given** cet écran, **When** elle confirme chez Google avec le compte Google lié, **Then** la demande de suppression est enregistrée comme dans la feature 004.
3. **Given** cet écran, **When** elle s'identifie chez Google avec un autre compte Google, ou annule, **Then** rien ne change et un message l'indique.
4. **Given** un compte qui a un mot de passe et qui est lié à Google, **When** la personne demande la suppression, **Then** elle peut confirmer avec son mot de passe ou avec Google.

---

### Edge Cases

- **Compte Google sans adresse vérifiée** : la connexion est refusée avec un message, aucun compte n'est créé ni ouvert.
- **Compte effacé** (feature 004) : son adresse est libre ; « Continuer avec Google » avec cette adresse crée un nouveau compte, sans rien de l'ancien.
- **Compte Google déjà lié à un autre compte CINQ** : il ouvre toujours ce compte-là, quelle que soit l'adresse actuelle du compte Google.
- **Mot de passe oublié sur un compte créé avec Google** : la demande de réinitialisation fonctionne comme pour tout compte (feature 001, FR-005) ; elle donne un mot de passe au compte, qui se connecte alors des deux façons.
- **Hors ligne** : le bouton Google est indisponible, comme toute connexion ; un message l'explique.
- **Google indisponible** ou réponse invalide : un message invite à réessayer ou à utiliser son email ; aucun compte n'est créé à moitié.
- **Tentative de rejouer ou de détourner le retour de Google** (lien déjà utilisé, lien ouvert sur un autre appareil, lien trafiqué) : la connexion est refusée, sans révéler si un compte existe.
- **Application installée (PWA)** : le parcours Google ramène dans l'application quand elle est installée, sans laisser la personne dans un onglet du navigateur.
- **Session de révision hors ligne en attente** (feature 006) : une connexion avec Google sur un appareil qui garde les réponses d'un autre compte les efface, comme une connexion par email.

## Requirements *(mandatory)*

### Functional Requirements

**Inscription et connexion**

- **FR-001**: Les écrans de connexion et d'inscription DOIVENT proposer « Continuer avec Google », actif, à la place du bouton « Bientôt ».
- **FR-002**: Pour un compte Google dont l'adresse vérifiée n'a pas de compte CINQ, le système DOIT créer un compte actif, sans lien de confirmation, avec cette adresse, le fuseau horaire de l'appareil, et le nom affiché retenu en FR-003.
- **FR-003**: Avant de créer un compte avec Google, le système DOIT demander le nom affiché, prérempli avec le nom du compte Google, en indiquant qu'il sera public ; ce nom DOIT respecter les règles des noms affichés (feature 001). Le compte n'est créé qu'à la validation de ce nom.
- **FR-004**: Un compte CINQ DOIT pouvoir être lié à un seul compte Google, et un compte Google à un seul compte CINQ ; le lien suit le compte Google, pas son adresse.
- **FR-005**: Une connexion avec Google DOIT produire la même session qu'une connexion par email : 30 jours sur l'appareil, fuseau mis à jour, annulation d'une demande de suppression en cours (feature 004, FR-014), retour à la page demandée.
- **FR-006**: Le système DOIT refuser un compte Google dont l'adresse n'est pas vérifiée par Google.

**Comptes existants**

- **FR-007**: Pour une adresse vérifiée par Google qui a déjà un compte confirmé, le système DOIT lier ce compte au compte Google et l'ouvrir directement, sans demander le mot de passe ni le nom affiché, et NE DOIT jamais créer un second compte avec la même adresse.
- **FR-008**: Pour une adresse vérifiée qui a un compte en attente de confirmation, le système DOIT confirmer ce compte, le lier à Google et retirer son mot de passe.
- **FR-009**: Un compte qui a un mot de passe DOIT continuer de se connecter par email et mot de passe après avoir été lié à Google.

**Suppression de compte**

- **FR-010**: Un compte sans mot de passe DOIT pouvoir confirmer sa demande de suppression en s'identifiant de nouveau avec le compte Google lié ; un autre compte Google ou une annulation ne change rien.
- **FR-011**: L'effacement d'un compte (feature 004, FR-017) DOIT aussi effacer son lien avec Google.

**Sécurité et confidentialité**

- **FR-012**: Le système NE DOIT recevoir de Google que l'identifiant du compte, l'adresse email et son statut de vérification, et le nom ; il NE DOIT conserver ni photo, ni contacts, ni accès aux autres services Google.
- **FR-013**: Un retour de Google rejoué, ouvert sur un autre appareil que celui qui l'a demandé, ou modifié, DOIT être refusé.
- **FR-014**: Aucun message du parcours Google NE DOIT révéler à une personne qui n'a pas prouvé la possession d'une adresse si cette adresse a un compte (feature 005, constitution VI).

**Messages**

- **FR-015**: Chaque échec du parcours Google (refus, annulation, Google indisponible, adresse non vérifiée, hors ligne) DOIT afficher un message qui propose de réessayer ou d'utiliser son email.
- **FR-016**: Tous les textes DOIVENT être en français, en vouvoyant.

### Key Entities

- **Lien Google** : pour un compte CINQ, l'identifiant stable de son compte Google et la date du lien. Un compte en a au plus un ; il disparaît avec le compte.
- **Utilisateur** (existant, feature 001) : peut désormais n'avoir aucun mot de passe, s'il a été créé avec Google.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Une personne sans compte CINQ entre dans l'application, connectée, en moins de 30 secondes et en 3 gestes au plus depuis l'écran d'inscription, si elle est déjà connectée à Google sur son appareil.
- **SC-002**: Aucune adresse email n'a jamais deux comptes CINQ, quel que soit l'ordre des inscriptions par email et par Google. Ceci est vérifié par des tests qui combinent les deux modes sur une même adresse.
- **SC-003**: 100 % des comptes, créés avec Google ou par email, peuvent demander leur suppression.
- **SC-004**: Un retour de Google rejoué ou modifié n'ouvre jamais de session. Ceci est vérifié par des tests automatisés.

## Assumptions

- **Dépend des features 001, 004, 005 et 006** : comptes et sessions de 30 jours (001), suppression et annulation par connexion (004), inscription neutre (005), effacement des données hors ligne d'un autre compte (006).
- **Google seulement** : aucun autre fournisseur (Apple, Microsoft) n'est prévu ; le parcours n'a pas besoin d'en accueillir d'autres.
- **Pas de dissociation** : retirer le lien avec Google depuis la page Compte n'est pas prévu. Un compte lié à Google peut obtenir un mot de passe par « Mot de passe oublié ».
- **Pas d'import de photo** : CINQ n'a pas de photo de profil ; les initiales restent l'avatar.
- **Lien automatique d'un compte existant** (FR-007) : choix du développeur ; Google a vérifié l'adresse, ce qui vaut la confirmation par email de la feature 001.
- **Nom affiché demandé à l'entrée** (FR-003) : choix du développeur ; le nom Google est souvent le vrai nom, et le nom affiché est public.
- **Mot de passe retiré d'un compte en attente** (FR-008) : un compte jamais confirmé n'a prouvé la possession de son adresse que par Google ; un mot de passe choisi par quelqu'un d'autre ne doit pas lui donner accès.
- **Conditions de Google** : CINQ est déclaré comme application auprès de Google, avec ses mentions légales et sa politique de confidentialité ; cette déclaration est un préalable à la mise en production, pas une exigence du produit.
