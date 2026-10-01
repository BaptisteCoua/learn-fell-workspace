# Contract: Inscription neutre (back ↔ web)

## `POST /api/register` (couche `users`, remplace la route de Fortify)

Même URL, même corps que dans la feature 001 ; session Sanctum SPA (cookie et `X-XSRF-TOKEN`).

```json
{
  "display_name": "Camille Roux",
  "email": "camille@exemple.fr",
  "password": "…",
  "password_confirmation": "…",
  "timezone": "Europe/Paris"
}
```

| Statut | Corps | Cas |
|---|---|---|
| `201` | `{ "message": "Si cette adresse peut être utilisée, un lien de confirmation vient d'y être envoyé." }` | Toute saisie valide : adresse libre, prise par un compte confirmé, en cours de suppression ou en attente |
| `422` | erreurs de validation Laravel par champ (`display_name`, `email`, `password`, `timezone`) | Saisie invalide, quelle que soit l'adresse |

- Plus de `422 { code: "email_taken" }` ni `422 { code: "email_pending_verification" }`.
- Aucune session n'est ouverte, y compris pour un compte créé (comportement de la 001, qui
  déconnectait aussitôt).

## Emails

- Adresse libre : l'email de confirmation de la 001, inchangé.
- Compte en attente : le même email de confirmation, avec un nouveau lien ; au plus un par minute.
- Compte confirmé ou en cours de suppression : aucun email.
