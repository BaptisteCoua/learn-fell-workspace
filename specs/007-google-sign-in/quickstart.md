# Quickstart: valider la connexion avec Google

Contrat dans [contracts/api.md](contracts/api.md), règles dans [data-model.md](data-model.md).

## Prérequis

Un client OAuth « Application Web » dans Google Cloud Console, avec l'origine `http://localhost:3000`
et l'URI de redirection `http://localhost:8090/api/auth/google/callback`. Dans `back/.env` :

```bash
GOOGLE_CLIENT_ID=…
GOOGLE_CLIENT_SECRET=…
GOOGLE_REDIRECT_URI=http://localhost:8090/api/auth/google/callback
```

Les tests automatisés n'en ont pas besoin : Google y est remplacé par un double (research R10).

## Tests automatisés

```bash
cd back && ./vendor/bin/sail artisan test --compact functional/users
cd web && pnpm test
```

| Exigences | Couche | Cas |
|---|---|---|
| FR-002, US1-1, US1-3 | `back/users` | adresse inconnue → résultat `nom`, aucun compte ; `register` crée un compte confirmé, lié, connecté, au fuseau envoyé, sans email de confirmation |
| FR-003, US1-2, US1-6 | `back/users`, `web/Account` | nom prérempli, coupé à 60, vide sous 2 ; règles de la 001 ; profil expiré après 15 min → `404` ; quitter l'écran ne crée rien |
| FR-004, US2-4 | `back/users` | même `google_id` avec une autre adresse → même compte, adresse et nom inchangés ; adresse liée à un autre `google_id` → `autre-compte-google` |
| FR-005, US2-1 à US2-3 | `back/users` | connexion : fuseau mis à jour, session régénérée, demande de suppression annulée → `suppression-annulee` ; `redirect` interne seulement |
| FR-006 | `back/users` | `email_verified` faux → aucun compte, aucune session |
| FR-007, US3-1, SC-002 | `back/users` | compte confirmé par email → lié et connecté, toujours un seul compte pour l'adresse, dans tous les ordres email / Google |
| FR-008, US3-3 | `back/users` | compte en attente → confirmé, mot de passe nul, lié ; l'ancien mot de passe ne connecte plus |
| FR-009, US3-2 | `back/users` | compte lié qui a un mot de passe → connexion par email toujours possible |
| FR-010, US4, SC-003 | `back/users`, `web/Account` | `confirmations` selon le compte ; suppression par Google avec le bon compte → demande enregistrée, sessions fermées ; autre compte ou annulation → rien ; `POST` sans mot de passe → `password_not_set` |
| FR-011 | `back/users` | effacement du compte → plus de `google_id` en base |
| FR-012 | `back/users` | portées `openid email profile` ; aucun jeton gardé |
| FR-013, SC-004 | `back/users` | `state` absent, différent ou rejoué → `echec`, aucune session |
| FR-014 | `back/users` | `/api/login` sur un compte sans mot de passe → `invalid_credentials` identique à un mot de passe faux |
| FR-001, FR-015, FR-016 | `web/Account` | bouton actif avec l'URL attendue (intention, page demandée, fuseau) ; désactivé hors ligne ; chaque `resultat` d'échec affiche son message et « Réessayer avec Google » / « Utiliser mon email » |
| edge case 006 | `web/Account` | `/compte-supprime` efface les données de révision de l'appareil |

## Parcours manuels à 360 px

1. **Inscription** : « Continuer avec Google » depuis `/inscription` avec un compte Google inconnu →
   écran du nom prérempli → valider → connecté sur `/categories`, aucun email dans Mailpit.
2. **Connexion** : se déconnecter, « Continuer avec Google » depuis `/connexion?redirect=/revisions`
   → retour sur `/revisions`.
3. **Compte existant** : compte seedé créé par email avec l'adresse du compte Google de test →
   « Continuer avec Google » → message « Vous pourrez désormais vous connecter avec Google », mêmes
   sujets ; la connexion par mot de passe fonctionne toujours.
4. **Annulation** : fermer l'écran de Google → message, boutons « Réessayer » et « Utiliser mon email ».
5. **Suppression** : avec le compte du parcours 1, « Supprimer mon compte » → « Confirmer avec Google »
   → `/compte-supprime` avec la date ; reconnexion avec Google → demande annulée.
6. **PWA installée** : sur Android, puis sur iOS, refaire le parcours 2 depuis l'application installée
   et noter si l'on revient connecté dans l'application (research R9).
