# Research: Inscription neutre

Décisions techniques de la feature 005, avec leurs raisons et les alternatives écartées. Les
chemins renvoient à l'état du code au 2026-10-01.

## R1. Une route d'inscription propre à la couche `users`

**Decision**: retirer `Features::registration()` de `back/functional/users/config/fortify.php` et
servir `POST /api/register` par un contrôleur de la couche `users`
(`RegisterController`, dans `back/functional/users/routes/account.php`, middleware `web` et
préfixe `/api` comme les autres routes de compte). L'URL et le corps de la requête ne changent pas.

**Rationale**: le contrôleur de Fortify (`RegisteredUserController::store`) connecte le compte que
renvoie `CreatesNewUsers::create()`, puis déclenche `Registered`. Il faudrait donc, pour une adresse
prise, soit lever une erreur (ce qui révèle le compte, l'état actuel), soit renvoyer le compte
existant, et Fortify ouvrirait alors une session sur un compte que la personne ne possède peut-être
pas, avant que `RegisteredResponse` ne la referme. Un contrôleur propre ne connecte personne.

**Alternatives considered**: renvoyer le compte existant à Fortify et compter sur
`RegisteredResponse` pour déconnecter (une session authentifiée, même brève, sur le compte d'un
tiers) ; un middleware qui intercepte `/register` avant Fortify (deux mécanismes pour une route).

## R2. Une seule réponse

**Decision**: toute inscription valide répond `201 { "message": "…" }`, avec un texte neutre :
« Si cette adresse peut être utilisée, un lien de confirmation vient d'y être envoyé. » Aucune
session n'est ouverte ni modifiée dans aucun cas.

**Rationale**: FR-001 et SC-001 demandent des réponses identiques, mot pour mot et code pour code.
L'actuel « Compte créé » serait faux pour une adresse prise.

**Alternatives considered**: `202 Accepted` (changerait le contrat de la 001 pour le web sans gain
pour la personne).

## R3. Les trois cas, dans une action

**Decision**: une action `RegisterAccount` remplace `CreateNewUser` :

| Adresse | Effet |
|---|---|
| libre | création du compte inactif, puis `event(new Registered($user))`, qui envoie le lien comme aujourd'hui |
| compte confirmé (actif ou en cours de suppression) | rien |
| compte en attente | `sendEmailVerificationNotification()` sur le compte existant, dans la limite de R4 ; aucune autre donnée n'est modifiée |

Les adresses sont comparées en minuscules, comme le fait déjà le mutateur `email` du modèle
`User` (FR-006).

**Rationale**: `EmailVerificationLink` remplace le nonce à chaque envoi, donc un nouveau lien rend
le précédent inutilisable (FR-004), sans code nouveau. `Registered` garde le même chemin d'envoi
pour un compte créé.

**Alternatives considered**: mettre à jour le compte en attente avec la nouvelle saisie (écarté
par le développeur : un tiers pourrait écraser un compte en attente).

## R4. Un lien par minute et par compte en attente

**Decision**: `RateLimiter::attempt('registration-link:'.$user->getKey(), 1, …, 60)` autour du
renvoi ; au-delà, l'inscription répond comme les autres, sans envoi (FR-005).

**Rationale**: même rythme que le bouton « Renvoyer le lien » (`throttle:6,1` par adresse IP sur
`email/verification-notification`), mais compté par compte, puisque c'est la boîte mail du
titulaire qu'il faut protéger, quelle que soit l'adresse IP de l'expéditeur.

**Alternatives considered**: un `throttle` sur la route, par adresse IP (n'empêche pas d'inonder
une boîte depuis plusieurs adresses IP, et un 429 ne concerne pas que les adresses prises).

## R5. Des erreurs de saisie identiques

**Decision**: les règles de validation restent celles de `CreateNewUser` (nom de 2 à 60
caractères, adresse valide de 255 caractères au plus, mot de passe de 8 caractères au moins et
confirmé, fuseau valide), sans règle `unique`. Elles s'appliquent avant toute lecture de la base,
donc leur réponse ne dépend pas de l'existence d'un compte (FR-007).

**Rationale**: une règle `unique` réintroduirait la fuite sous forme d'erreur de champ.

## R6. Le web

**Decision**: `useRegisterForm` perd l'état `emailConflict` et ses deux messages ; la page
`inscription/index.vue` perd le bloc correspondant. L'écran « Vérifiez vos emails »
(`inscription/confirmation.vue`) devient vrai dans tous les cas : « Si cette adresse peut être
utilisée, vous allez recevoir un lien de confirmation », et ajoute, si rien n'arrive, les liens
« Se connecter » et « Mot de passe oublié ». L'appel reste `POST /register` par `useApiFetch`
(`useAuth.register`), comme dans la 001.

**Rationale**: FR-008 et FR-009 ; la personne qui avait oublié son compte est guidée vers la
connexion sans que rien ne lui dise que le compte existe.

## R7. Ce qui disparaît

**Decision**: supprimer `CreateNewUser`, `RegisteredResponse`, leurs liaisons dans
`FortifyServiceProvider` et les codes `email_taken` et `email_pending_verification` de
`back/technical/osdd/lang/fr/errors.php`.

**Rationale**: ils ne servent plus qu'à l'inscription de Fortify, retirée (principe VII).
