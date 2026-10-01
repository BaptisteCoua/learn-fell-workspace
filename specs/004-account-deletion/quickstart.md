# Quickstart: valider la suppression de compte

Guide de validation de bout en bout. Le contrat est dans [contracts/api.md](contracts/api.md),
les règles dans [data-model.md](data-model.md).

## Prérequis

```bash
cd back && ./vendor/bin/sail up -d && ./vendor/bin/sail artisan migrate && ./vendor/bin/sail artisan osdd:seed
cd back && ./vendor/bin/sail artisan queue:work
cd web && pnpm dev
```

Mailpit sur http://localhost:8035 reçoit l'email de confirmation. Pour simuler la fin du délai
sans attendre 30 jours, avancer `deletion_requested_at` du compte de 31 jours en base, puis :

```bash
cd back && ./vendor/bin/sail artisan model:prune --model="Functional\Users\Models\User"
```

## Tests automatisés

```bash
cd back && ./vendor/bin/sail artisan test --compact functional/users functional/catalog functional/learning functional/moderation functional/reminders
cd web && pnpm test
```

Cas qui doivent exister (un par exigence, principe IV) :

| Exigences | Couche | Cas |
|---|---|---|
| FR-002, FR-005 | `users` | `GET account/deletion` : date à J+30 ; `last_admin` pour le seul détenteur d'une permission d'administration ; accepté s'il en reste un autre actif ; refusé de nouveau pour le second pendant la demande du premier |
| FR-003, FR-004 | `users` | mot de passe faux sans effet ; `subject_choice_required` avec un sujet publié ; choix ignoré sans sujet publié |
| FR-006, FR-013 | `users` | toutes les sessions du compte supprimées ; requête suivante en 401 ; réponse avec `erase_on` |
| FR-007 | `reminders` | aucun rappel pour un compte en suppression ; reprise au créneau normal après annulation |
| FR-008, FR-009 | `catalog`, `learning` | brouillons, dépubliés et retirés introuvables ; « Tout effacer » : sujets `withheld`, absents du catalogue, de la recherche, de l'adresse directe, des séances et des rappels d'autrui ; visibles pour `subjects.moderate` |
| FR-010, FR-011 | `catalog`, `moderation` | « Laisser » : sujets apprenables ; `author.display_name`, `reporter.display_name` et `admin.display_name` nuls pendant le délai |
| FR-012 | `users` | notification envoyée avec la date et le nom |
| FR-014, FR-015 | `users`, `catalog`, `reminders` | connexion qui annule (`deletion_cancelled`), y compris après réinitialisation du mot de passe ; sujets `withheld` repassés `published` avec le même `published_at` ; nom, progression, réglages, appareils et droits rétablis |
| FR-016, FR-022 | `users` | élagage à J+30 et pas à J+29 ; un écouteur qui échoue laisse le compte entier |
| FR-017 à FR-021, FR-023 | toutes | effacement complet ; « Tout effacer » : sujets, images, fichiers, progression d'autrui et signalements supprimés, décisions conservées avec le titre ; « Laisser » : sujets publiés avec `author_id` nul, images avec `uploader_id` nul, progression d'autrui intacte, autres sujets supprimés ; `reporter_id` et `admin_id` nuls ; réinscription possible avec l'adresse |
| FR-024 | `users` | connexion avec un mauvais mot de passe : même réponse qu'un compte actif ; inscription : même réponse qu'une adresse prise ; après effacement : mêmes réponses qu'une adresse jamais utilisée |
| FR-001, FR-025 | `web/functional/Account` | entrée « Supprimer mon compte » ; écran, choix, confirmation et message d'annulation en français |
| FR-010, FR-011 | `web/functional/Catalog`, `web/functional/Moderation` | « Auteur supprimé » et « Compte supprimé » quand la relation ou le nom est nul |

## Parcours manuels à 360 px

1. **Sans sujet publié** : Compte → « Supprimer mon compte » ; pas de choix sur les sujets ;
   mauvais mot de passe refusé ; bon mot de passe → page de confirmation avec la date, email dans
   Mailpit, un second navigateur connecté au même compte est déconnecté.
2. **Avec « Laisser »** : un sujet publié appris par un second compte ; demande avec « Laisser » ;
   le second compte voit le sujet signé « Auteur supprimé » et le révise.
3. **Avec « Tout effacer »** : même préparation ; le sujet disparaît pour le second compte, de son
   catalogue et de ses révisions ; un admin le voit « Retenu » dans la modération.
4. **Annulation** : se reconnecter avec le compte du parcours 3 ; message d'annulation ; le sujet
   revient pour le second compte avec sa progression.
5. **Effacement** : refaire le parcours 3, avancer la date, lancer l'élagage ; le second compte
   n'a plus le sujet ; l'adresse permet une nouvelle inscription.
6. **Dernier admin** : avec un seul compte qui a reçu `users:grant-admin`, l'écran explique que la
   suppression est impossible.
