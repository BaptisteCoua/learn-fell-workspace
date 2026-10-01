# Quickstart: valider l'inscription neutre

Contrat dans [contracts/api.md](contracts/api.md), règles dans [data-model.md](data-model.md).

## Tests automatisés

```bash
cd back && ./vendor/bin/sail artisan test --compact functional/users
cd web && pnpm test
```

| Exigences | Couche | Cas |
|---|---|---|
| FR-001, SC-001 | `users` | quatre inscriptions identiques sauf l'adresse (libre, confirmée, en cours de suppression, en attente) : même statut, même corps |
| FR-002 | `users` | adresse libre : compte inactif créé, lien envoyé, aucune session ouverte |
| FR-003, SC-003 | `users` | compte confirmé ou en cours de suppression : nom, mot de passe, fuseau et demande de suppression inchangés, aucune notification |
| FR-004 | `users` | compte en attente : nouveau lien, l'ancien refusé, nom, mot de passe, fuseau et `created_at` inchangés |
| FR-005 | `users` | deux inscriptions dans la minute sur un compte en attente : un seul email, même réponse |
| FR-006 | `users` | adresse avec d'autres majuscules : traitée comme la même |
| FR-007 | `users` | mêmes erreurs de champ pour une adresse libre et une adresse prise |
| FR-008, FR-009, FR-010 | `web/functional/Account` | plus de message d'adresse prise ; écran « Vérifiez vos emails » au conditionnel, avec liens de connexion et de mot de passe oublié |

## Parcours manuel à 360 px

1. S'inscrire avec une adresse neuve : écran « Vérifiez vos emails », lien dans Mailpit
   (http://localhost:8035), confirmation qui fonctionne.
2. S'inscrire avec l'adresse d'un compte seedé confirmé : même écran, rien dans Mailpit.
3. S'inscrire de nouveau avec l'adresse du parcours 1 avant de confirmer : même écran, un nouveau
   lien dans Mailpit, l'ancien affiche « lien expiré ».
