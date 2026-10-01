# Feature Specification: Inscription neutre

**Feature Branch**: `005-neutral-registration`

**Created**: 2026-10-01

**Status**: Draft

**Input**: User description: "Inscription neutre : l'inscription de CINQ ne doit plus révéler si une adresse email possède déjà un compte (principe VI de la constitution). Elle remplace le scénario 6 de la user story 2 de la feature 001 et étend FR-006 de la 001 à l'inscription. Toute inscription valide reçoit la même réponse et mène au même écran « Vérifiez vos emails ». Adresse libre : compte créé et lien envoyé. Adresse d'un compte confirmé, y compris en cours de suppression : rien n'est créé, modifié ni envoyé. Adresse d'un compte en attente de confirmation : nouveau lien envoyé, nom et mot de passe saisis ignorés. Les erreurs de saisie restent signalées. Hors périmètre : prévenir le titulaire d'une adresse prise, l'égalisation du temps de réponse, Google."

## Contexte

La constitution de CINQ (principe VI) interdit qu'un message ou une réponse de l'API révèle si une adresse email possède un compte. La connexion et la réinitialisation du mot de passe respectent déjà cette règle (feature 001, FR-006). L'inscription, elle, la contredit : la feature 001 (user story 2, scénario 6) refuse une adresse déjà utilisée avec un message qui le dit, et distingue même un compte actif d'un compte en attente de confirmation. N'importe qui peut ainsi vérifier si une personne est inscrite sur CINQ.

Cette feature rend l'inscription muette sur ce point : toute inscription correctement remplie reçoit la même réponse, quelle que soit l'adresse. Ce qui se passe ensuite dépend de l'adresse, mais se joue uniquement dans la boîte mail de son titulaire.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - S'inscrire sans apprendre si l'adresse est prise (Priority: P1)

Une personne remplit le formulaire d'inscription avec une adresse. Qu'un compte existe ou non à cette adresse, elle voit la même réponse et arrive sur le même écran « Vérifiez vos emails ». Rien à l'écran ne lui permet de savoir si l'adresse avait déjà un compte.

**Why this priority**: C'est la règle de confidentialité elle-même, exigée par la constitution avant l'ouverture au public.

**Independent Test**: Trois inscriptions identiques, sauf l'adresse (libre, prise par un compte confirmé, prise par un compte en attente), donnent exactement la même réponse et le même écran.

**Acceptance Scenarios**:

1. **Given** une adresse sans compte, **When** une personne s'inscrit avec un nom, cette adresse et un mot de passe valides, **Then** un compte inactif est créé, un lien de confirmation part à cette adresse, et l'écran « Vérifiez vos emails » s'affiche.
2. **Given** une adresse qui a un compte confirmé, **When** une personne s'inscrit avec cette adresse et des données valides, **Then** elle voit exactement la même réponse et le même écran qu'au scénario 1, et le compte existant n'est ni modifié ni prévenu.
3. **Given** une adresse dont le compte est en cours de suppression (feature 004), **When** une personne s'inscrit avec cette adresse, **Then** la réponse et l'écran sont ceux du scénario 1, et le compte, sa demande de suppression et sa date d'effacement ne changent pas.
4. **Given** une adresse écrite avec d'autres majuscules qu'à l'inscription d'origine, **When** une personne s'inscrit avec cette variante, **Then** elle est traitée comme la même adresse, sans réponse différente.
5. **Given** l'écran « Vérifiez vos emails » après une inscription, **When** la personne le lit, **Then** le texte reste vrai dans tous les cas : il annonce un lien « si cette adresse peut être utilisée » et rappelle de vérifier les indésirables, sans affirmer qu'un compte vient d'être créé.

---

### User Story 2 - Retrouver un compte resté en attente (Priority: P1)

Une personne s'est inscrite il y a quelques jours mais n'a jamais ouvert son lien de confirmation. Elle refait une inscription avec la même adresse. Elle reçoit un nouveau lien de confirmation pour son compte existant ; le nom et le mot de passe qu'elle vient de saisir sont ignorés.

**Why this priority**: C'est le cas réel le plus fréquent derrière une adresse « déjà prise » : sans lui, la personne qui a perdu son premier email resterait bloquée jusqu'à la suppression de son compte inactif.

**Independent Test**: Un compte en attente créé avec le nom « Camille » ; une nouvelle inscription à la même adresse avec le nom « Dominique » et un autre mot de passe ; un nouveau lien arrive, l'ancien ne fonctionne plus, et le compte confirmé s'appelle toujours « Camille » avec le mot de passe d'origine.

**Acceptance Scenarios**:

1. **Given** un compte en attente de confirmation, **When** une personne s'inscrit de nouveau avec la même adresse, **Then** un nouveau lien de confirmation part à cette adresse et remplace le précédent, qui ne fonctionne plus.
2. **Given** ce nouveau lien, **When** la personne l'ouvre, **Then** le compte est confirmé avec le nom et le mot de passe de la première inscription.
3. **Given** un compte en attente, **When** une nouvelle inscription arrive avec un autre nom et un autre mot de passe, **Then** ni le nom ni le mot de passe du compte ne changent.
4. **Given** un compte en attente créé il y a 6 jours, **When** une nouvelle inscription renvoie un lien, **Then** le compte reste soumis à la suppression des comptes non confirmés au bout de 7 jours après sa création (feature 001, FR-002) ; le renvoi ne prolonge pas ce délai.

---

### User Story 3 - Corriger une saisie (Priority: P2)

Les erreurs qui ne dépendent pas de l'existence d'un compte restent signalées sous les champs, comme aujourd'hui : nom trop court ou trop long, adresse mal formée, mot de passe de moins de 8 caractères, deux mots de passe différents.

**Why this priority**: Ces messages aident la personne sans rien révéler ; il ne faut pas les perdre en rendant la réponse neutre.

**Independent Test**: Une inscription avec un mot de passe de 5 caractères affiche l'erreur sous le champ, que l'adresse ait un compte ou non, et le message est le même dans les deux cas.

**Acceptance Scenarios**:

1. **Given** un formulaire avec deux mots de passe différents, **When** la personne l'envoie, **Then** l'erreur s'affiche sous la confirmation du mot de passe, sans rien envoyer.
2. **Given** une saisie invalide (nom, adresse ou mot de passe), **When** l'adresse a un compte et quand elle n'en a pas, **Then** les erreurs affichées sont les mêmes dans les deux cas.

---

### Edge Cases

- **Inscriptions répétées sur un compte en attente** : chaque inscription valide renvoie un lien, dans la limite d'un envoi par minute et par adresse ; au-delà, la réponse reste la même et aucun email supplémentaire ne part.
- **Inscriptions répétées sur une adresse libre** : la deuxième trouve un compte en attente et suit la user story 2 ; il n'y a jamais deux comptes pour une même adresse.
- **Adresse d'un compte effacé (feature 004)** : elle est libre ; l'inscription suit le scénario 1 de la user story 1.
- **Échec d'envoi de l'email** : la réponse à l'écran ne change pas ; la personne peut redemander un lien depuis l'écran « Vérifiez vos emails ».
- **Personne qui croit s'inscrire mais a déjà un compte confirmé** : elle ne reçoit rien ; l'écran « Vérifiez vos emails » l'invite, si rien n'arrive, à se connecter ou à réinitialiser son mot de passe.
- **Fuseau horaire envoyé à l'inscription** : il n'est enregistré que pour un compte créé ; un compte existant garde le sien.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Toute inscription dont la saisie est valide DOIT recevoir la même réponse, et mener au même écran, que l'adresse soit libre, prise par un compte confirmé, prise par un compte en cours de suppression ou prise par un compte en attente de confirmation.
- **FR-002**: Pour une adresse libre, le système DOIT créer un compte inactif et envoyer un lien de confirmation, comme le prévoit la feature 001 (FR-001, FR-002).
- **FR-003**: Pour une adresse d'un compte confirmé, y compris en cours de suppression, le système NE DOIT créer, modifier ni envoyer quoi que ce soit.
- **FR-004**: Pour une adresse d'un compte en attente de confirmation, le système DOIT envoyer un nouveau lien de confirmation qui remplace le précédent, et NE DOIT modifier ni le nom, ni le mot de passe, ni le fuseau horaire, ni la date de création du compte.
- **FR-005**: Un même compte en attente NE DOIT pas recevoir plus d'un lien par minute du fait d'inscriptions répétées ; les inscriptions en trop reçoivent la même réponse, sans envoi.
- **FR-006**: Les adresses DOIVENT être comparées sans tenir compte des majuscules, comme à la connexion.
- **FR-007**: Les erreurs de saisie (nom, adresse, mot de passe et sa confirmation) DOIVENT rester signalées champ par champ, et DOIVENT être les mêmes que l'adresse ait un compte ou non.
- **FR-008**: L'écran affiché après une inscription DOIT rester vrai dans tous les cas : annoncer un lien « si cette adresse peut être utilisée », rappeler de vérifier les indésirables, proposer de renvoyer le lien, et, si rien n'arrive, de se connecter ou de réinitialiser son mot de passe.
- **FR-009**: Le formulaire d'inscription NE DOIT plus afficher de message indiquant qu'une adresse a déjà un compte ou qu'un compte attend sa confirmation.
- **FR-010**: Les textes de la réponse et de l'écran DOIVENT être en français, en vouvoyant la personne.

### Key Entities

- **Compte** (existant) : aucun changement de données ; seul le traitement d'une inscription sur une adresse déjà prise change.
- **Lien de confirmation** (existant, feature 001) : valable 24 heures, à usage unique ; un nouveau lien rend le précédent inutilisable.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Pour une même saisie valide, les réponses du système sont identiques, mot pour mot et code pour code, pour une adresse libre, prise par un compte confirmé, prise par un compte en cours de suppression et prise par un compte en attente. Ceci est vérifié par des tests qui comparent les quatre réponses.
- **SC-002**: Une personne dont le compte est resté en attente le confirme en moins de 2 minutes après une nouvelle inscription avec la même adresse.
- **SC-003**: 0 compte confirmé n'est modifié par une inscription sur son adresse, et 0 email ne lui est envoyé. Ceci est vérifié par des tests qui couvrent FR-003.
- **SC-004**: Aucun texte de l'écran d'inscription ni de l'écran « Vérifiez vos emails » ne permet de déduire qu'une adresse a un compte.

## Assumptions

- **Remplace une décision de la feature 001** : le scénario 6 de sa user story 2 (refus avec un message) est abandonné ; FR-006 de la 001 s'étend désormais à l'inscription.
- **Cohérent avec la feature 004** : une adresse en cours de suppression répond comme toute adresse prise, ce que la 004 exigeait déjà ; la note de la 004 qui signalait l'écart est résolue.
- **Aucun email au titulaire d'une adresse prise** : choix du développeur. Une personne qui a oublié qu'elle avait un compte est guidée par l'écran « Vérifiez vos emails » (connexion, mot de passe oublié).
- **Limite d'un lien par minute** : même rythme que le bouton « Renvoyer le lien » existant ; une valeur par défaut raisonnable contre l'envoi massif d'emails vers une même boîte.
- **Hors périmètre** : l'égalisation du temps de réponse entre les cas (une mesure fine du délai pourrait encore distinguer une création de compte) ; la connexion avec Google ; prévenir le titulaire d'une adresse prise.
