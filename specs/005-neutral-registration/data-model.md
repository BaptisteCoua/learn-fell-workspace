# Data Model: Inscription neutre

Aucun changement de schéma : ni table, ni colonne, ni migration.

## Compte (`users`, couche `back/functional/users`)

Seul change le traitement d'une inscription sur une adresse existante (voir
[research.md, R3](research.md#r3-les-trois-cas-dans-une-action)) :

| État du compte à l'adresse | Données modifiées | Email |
|---|---|---|
| aucun | compte inactif créé (`display_name`, `email`, `password`, `timezone`) | lien de confirmation |
| en attente (`email_verified_at` nul) | aucune ; `created_at` inchangé, donc la suppression à 7 jours aussi | nouveau lien, au plus un par minute |
| confirmé, actif ou en cours de suppression | aucune | aucun |

## Lien de confirmation (existant)

Un nonce en cache par compte (`email-verification-nonce:{id}`, `EmailVerificationLink`) ; chaque
envoi le remplace, ce qui invalide le lien précédent.

## Limite d'envoi

Clé `registration-link:{id}` du limiteur de Laravel, 1 tentative par 60 secondes.
