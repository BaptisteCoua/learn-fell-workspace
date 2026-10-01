# Data Model: Connexion avec Google

Décisions dans [research.md](research.md), contrat dans [contracts/api.md](contracts/api.md).

## Back : `users` (couche `back/functional/users`)

| Colonne | Type | Règle |
|---|---|---|
| `password` | string, **désormais nul** | nul pour un compte créé avec Google, ou en attente confirmé par Google (FR-008) |
| `google_id` | string(255), unique, nul | identifiant stable du compte Google (`sub`) ; le lien suit lui, pas l'adresse (FR-004) |
| `google_linked_at` | timestamp, nul | date du lien |

Aucune valeur par défaut en base. Aucun jeton Google n'est enregistré (FR-012). Les deux colonnes
disparaissent avec la ligne à l'effacement (FR-011). `google_id` est ajouté à `#[Hidden]`.

## Back : profil Google en attente (session)

Gardé dans la session du navigateur qui a fait le parcours, sous la clé `users.google_profile`,
quand l'adresse n'a pas de compte (R4, cas `nom`) :

| Champ | Source |
|---|---|
| `google_id` | `sub` du profil Google |
| `email` | adresse vérifiée, en minuscules |
| `name` | nom Google, coupé à 60 caractères |
| `expires_at` | réception + 15 minutes |

Il est retiré à la création du compte, à la connexion d'un compte et à son expiration.

## Back : parcours en session

Clé `users.google_flow`, posée par `GET /api/auth/google/redirect`, lue et retirée au retour :

| Champ | Règle |
|---|---|
| `intention` | `connexion` ou `suppression` |
| `redirect` | chemin interne du web (commence par `/`, pas par `//`), sinon `/categories` |
| `timezone` | fuseau valide (`timezone:all_with_bc`), sinon ignoré |
| `keep_published_subjects` | pour `suppression` seulement : `true`, `false` ou nul, comme dans la 004 |

## Règles du retour de Google

Table de décision complète dans [research.md](research.md), R4. Résumé de l'ordre :

1. `state` ou PKCE invalide, refus, erreur → `annule` ou `echec`.
2. `intention = suppression` → R7 : compte connecté et `google_id` égal au sien, sinon
   `autre-compte-google` ; demande de suppression enregistrée.
3. `email_verified` faux → `adresse-non-verifiee`.
4. `google_id` connu → connexion.
5. Adresse d'un compte : déjà lié à un autre `google_id` → `autre-compte-google` ; confirmé → lien ;
   en attente → `email_verified_at` = maintenant, `password` = nul, lien. Puis connexion.
6. Aucune adresse → profil en attente, résultat `nom`.

Une connexion (4, 5, création) appelle `CompleteSignIn` : fuseau, annulation d'une demande de
suppression, `session()->regenerate()`, retrait du profil en attente.

## Moyens de confirmer une suppression

| Compte | `confirmations` |
|---|---|
| mot de passe, pas de Google | `["password"]` |
| Google, pas de mot de passe | `["google"]` |
| les deux | `["password", "google"]` |

## Web : aucune donnée nouvelle

La page `/connexion-google` ne garde rien : le profil en attente reste sur le serveur. Le fuseau et
la page demandée partent dans l'URL du bouton.
