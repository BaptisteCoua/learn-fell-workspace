# Contrat d'API : Connexion avec Google (007)

Complète le contrat de la 001 ([specs/001-learning-content/contracts/api.md](../../001-learning-content/contracts/api.md)).
Toutes les routes sont des routes de compte : middleware `web`, préfixe `/api`, hors lomkit.

## `GET /api/auth/google/redirect` (nouvelle)

Navigation de premier niveau depuis le bouton du web, jamais un appel `fetch`.

| Paramètre | Règle |
|---|---|
| `intention` | `connexion` (par défaut) ou `suppression` |
| `redirect` | chemin interne du web ; sinon `/categories` |
| `timezone` | fuseau de l'appareil, facultatif |
| `keep_published_subjects` | `1`, `0` ou absent ; avec `intention=suppression` seulement |

Réponse : `302` vers Google (portées `openid email profile`, `state` et PKCE). Limite :
`throttle:10,1` par adresse IP. `intention=suppression` sans session connectée renvoie vers
`/connexion`.

## `GET /api/auth/google/callback` (nouvelle)

Adresse de retour déclarée chez Google (`GOOGLE_REDIRECT_URI`). Répond toujours par une redirection
vers le web (`FRONTEND_URL`), jamais par du JSON :

| Destination | Quand |
|---|---|
| `/connexion-google?resultat=connecte&redirect=…` | compte lié connu (FR-005) |
| `/connexion-google?resultat=lie&redirect=…` | compte existant lié à l'instant (FR-007, FR-008) |
| `/connexion-google?resultat=suppression-annulee&redirect=…` | connexion qui annule une demande de suppression |
| `/connexion-google?resultat=nom&redirect=…` | aucune adresse : nom affiché à choisir (FR-003) |
| `/connexion-google?resultat=adresse-non-verifiee` | FR-006 |
| `/connexion-google?resultat=autre-compte-google` | adresse liée à un autre compte Google |
| `/connexion-google?resultat=annule` | refus ou fermeture chez Google |
| `/connexion-google?resultat=echec` | `state` invalide, retour rejoué, Google indisponible (FR-013) |
| `/compte-supprime?le=AAAA-MM-JJ` | `intention=suppression` confirmée (FR-010) |
| `/supprimer-mon-compte?google=autre-compte-google` ou `?google=annule` | suppression non confirmée |

Le `redirect` renvoyé est celui gardé en session au départ.

## `GET /api/auth/google/pending` (nouvelle)

Profil gardé en session après le résultat `nom`.

- `200 { "name": "Camille Roux", "email": "camille@gmail.com" }` (`name` vide s'il fait moins de 2
  caractères).
- `404 { "code": "google_profile_expired", "message": "…" }` sans profil ou après 15 minutes.

## `POST /api/auth/google/register` (nouvelle)

```json
{ "display_name": "Camille", "timezone": "Europe/Paris" }
```

- `display_name` : requis, 2 à 60 caractères (règles de la 001).
- `timezone` : facultatif, fuseau valide ; `Europe/Paris` par défaut.
- `201 { "message": "…" }` : compte confirmé créé, lié, connecté ; profil en attente retiré.
- `422` avec les erreurs de champ.
- `404 google_profile_expired`.
- Si l'adresse a obtenu un compte entre-temps : ce compte est lié et connecté, comme au retour de
  Google (aucun second compte, SC-002).

## `GET /api/account/deletion` (modifiée)

Ajoute `confirmations`, la liste `["password"]`, `["google"]` ou `["password", "google"]`.

## `POST /api/account/deletion` (modifiée)

Inchangée pour un compte qui a un mot de passe. Un compte sans mot de passe y répond
`422 { "code": "password_not_set" }` : il confirme par Google (`GET /api/auth/google/redirect?intention=suppression`).

## `POST /api/login` (comportement précisé)

Un compte sans mot de passe répond comme un mot de passe faux : `invalid_credentials`, même compteur
d'échecs (FR-014).

## Codes d'erreur ajoutés

| Code | HTTP | Exigence |
|---|---|---|
| `google_profile_expired` | 404 | FR-003 |
| `password_not_set` | 422 | FR-010 |
