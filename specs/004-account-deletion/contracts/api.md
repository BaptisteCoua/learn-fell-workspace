# Contract: Suppression de compte (back ↔ web)

Toutes les routes sont sous `/api`, en session Sanctum SPA (cookie et en-tête `X-XSRF-TOKEN`),
comme les routes de compte de la 001. Les erreurs métier suivent le format existant de
`BusinessRuleException` : `422 { "code": "<code>", "message": "<texte traduit>" }`.

## `GET /api/account/deletion` (couche `users`)

Lecture de l'état avant la confirmation. Compte connecté obligatoire (401 sinon).

```json
200
{
  "can_request": true,
  "blocked_reason": null,
  "erase_on": "2026-10-31"
}
```

- `blocked_reason` : `"last_admin"` quand `can_request` est faux (FR-005).
- `erase_on` : date (jour, fuseau du compte) qu'aurait l'effacement si la demande était faite
  maintenant.

## `GET /api/learning/authored-subjects-summary` (couche `learning`)

Nombres affichés avant le choix sur les sujets (FR-004). Compte connecté obligatoire.

```json
200
{ "published_subjects_count": 2, "learners_count": 37 }
```

- `learners_count` : comptes distincts, autres que l'auteur, qui apprennent au moins un de ses
  sujets publiés.
- Le web ne propose le choix sur les sujets que si `published_subjects_count > 0`.

## `POST /api/account/deletion` (couche `users`)

```json
{ "password": "…", "keep_published_subjects": true }
```

- `keep_published_subjects` : obligatoire si le compte a au moins un sujet publié, ignoré sinon.

Réponses :

| Statut | Corps | Cas |
|---|---|---|
| `200` | `{ "erase_on": "2026-10-31" }` | Demande enregistrée ; toutes les sessions du compte sont fermées, y compris celle de la requête |
| `422` | `errors.password` (format de validation Laravel) | Mot de passe incorrect (FR-003) |
| `422` | `code: "subject_choice_required"` | Sujets publiés sans choix (FR-004) |
| `422` | `code: "last_admin"` | Dernier compte d'administration (FR-005) |
| `401` | | Non connecté |

Après un `200`, le web vide le store de session et affiche la page de confirmation avec
`erase_on` (FR-013). Toute requête suivante du même navigateur reçoit 401.

## `POST /api/login` (Fortify, modifié)

Inchangé pour les erreurs (`invalid_credentials`, `locked`, `email_not_verified`) : un compte en
suppression répond exactement comme un compte actif (FR-024).

```json
200
{ "deletion_cancelled": true }
```

- `deletion_cancelled` vaut `true` seulement quand la connexion vient d'annuler une demande
  (FR-014) ; il est absent sinon. Le web affiche alors le message d'annulation.

## Ressources lomkit existantes, sortie modifiée

`author` (`subjects`), `reporter` (`reports`) et `admin` (`moderation-decisions`), exposés par
`PublicUserResource` :

| Situation | Valeur |
|---|---|
| Compte actif | `{ "id": 12, "display_name": "Inès Martin" }` |
| Compte en suppression | `{ "id": 12, "display_name": null }` |
| Compte effacé | `null` |

Le web affiche « Auteur supprimé » (sujets) ou « Compte supprimé » (modération) quand la relation
ou son `display_name` est nul.

`subjects.status` peut valoir `"withheld"`, visible seulement des comptes qui ont
`subjects.moderate` ; le web l'affiche « Retenu (suppression de compte en cours) » dans la file de
modération.

## Email : demande de suppression (couche `users`)

- **Objet** : « Votre compte CINQ sera supprimé le 31 octobre 2026 »
- **Corps** : salutation avec le nom affiché ; date d'effacement ; ce qui sera effacé et ce qui
  sera conservé selon le choix ; « Vous avez changé d'avis ? Il vous suffit de vous reconnecter
  avant cette date. » ; bouton « Me reconnecter » vers `/connexion`.
- Aucun lien qui annule ou confirme sans connexion.
