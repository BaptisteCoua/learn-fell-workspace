# Guide de validation : Comptes, contenu et révision Leitner (001)

Ce guide prouve, de bout en bout, que la feature fonctionne. Il ne décrit pas l'implémentation :
voir [plan.md](plan.md), [data-model.md](data-model.md) et [contracts/api.md](contracts/api.md).

## Prérequis

- Docker, pour Laravel Sail (PHP n'est pas installé sur l'hôte).
- Node.js et pnpm pour le web.
- Les deux repos sur la branche `001-learning-content`.

## Démarrer

```bash
# back
cd back
./vendor/bin/sail up -d                 # API sur http://localhost:8090, Mailpit sur http://localhost:8035
./vendor/bin/sail artisan migrate
./vendor/bin/sail artisan osdd:seed     # catégories, auteurs et sujets de démonstration
./vendor/bin/sail artisan queue:work    # une question ajoutée rejoint les apprenants par une tâche en file
./vendor/bin/sail artisan users:grant-admin <email d'un compte confirmé>

# web
cd ../web
pnpm install
pnpm dev                                # http://localhost:3000 (l'API n'accepte que cette origine)

# PWA : le service worker n'existe qu'en build de production
pnpm build && PORT=3000 node .output/server/index.mjs
```

## Tests automatisés

```bash
cd back && ./vendor/bin/sail artisan test      # PHPUnit, toutes les couches
cd back && ./vendor/bin/sail bin pint --test
cd web && pnpm test && pnpm lint && pnpm exec prettier --check .
```

**Attendu** : tout est vert. Les tests de `LeitnerSchedule` couvrent les 10 transitions du tableau
de [data-model.md](data-model.md) (SC-010) ; les tests d'accès couvrent liste, recherche et adresse
directe pour un brouillon et un sujet retiré, en visiteur, en inscrit et en auteur (SC-006).

## Scénarios manuels

Chaque scénario renvoie aux critères d'acceptation de [spec.md](spec.md). Les écrans de référence
sont ceux de la [maquette validée](https://claude.ai/artifact/DxQK5ap9dAK6UYGuZURLsP).

1. **Compte** (story 2) : s'inscrire avec deux mots de passe différents (refus), puis identiques ;
   vérifier dans Mailpit l'email de confirmation ; tenter de se connecter avant de cliquer (refus
   avec renvoi du lien) ; cliquer, être connecté ; se déconnecter ; rater 5 connexions (verrou de
   15 min) ; réinitialiser le mot de passe (lien de 60 min, autres sessions fermées).
2. **Publier** (story 3) : créer un brouillon, tenter de le publier sans question (refus), ajouter
   3 questions avec gras, liste et bloc de code, les réordonner, publier ; vérifier qu'un visiteur
   le voit, dépublier, vérifier qu'il ne le voit plus (page « contenu introuvable » en accès direct).
3. **Consulter** (story 1) : en visiteur, parcourir une catégorie, chercher « irreguliers » et
   trouver « Verbes irréguliers anglais », chercher « r » (message « au moins 2 caractères »).
4. **Catégories** (story 4) : en admin, créer « Histoire » alors qu'elle existe (refus), renommer,
   réordonner, tenter de supprimer une catégorie non vide (refus avec le nombre de sujets).
5. **Modérer** (story 5) : en inscrit, signaler un sujet, tenter un second signalement (refus) ; en
   admin, retirer le sujet sans motif (refus), puis avec motif ; vérifier le statut et le motif côté
   auteur, l'impossibilité de republier, l'entrée dans l'historique ; rétablir.
6. **Réviser** (story 6) : apprendre un sujet de 5 questions ; ouvrir « Mes révisions », cocher le
   sujet, lancer la séance ; répondre 3 « je savais » et 2 « je ne savais pas » ; vérifier le bilan,
   puis les boîtes (3 cartes en boîte 2 dues dans 2 jours, 2 en boîte 1 dues demain) ; double-cliquer
   sur une réponse (une seule prise en compte) ; arrêter d'apprendre (progression supprimée).
7. **Mobile et PWA** (FR-036 à FR-040, FR-052, SC-008) : à 360 px de large, refaire les scénarios
   1, 3 et 6 sans défilement horizontal ; installer l'application depuis Chrome Android ; couper le
   réseau (écran « Vous êtes hors ligne ») ; publier une nouvelle version (invitation à mettre à
   jour) ; agrandir la taille de texte du téléphone (les textes suivent).

## Résultats de la validation du 2026-09-25

Tests automatisés : back 139 tests verts et Pint propre, web 70 tests verts, lint et Prettier
propres. Chaque scénario a ensuite été déroulé dans Chromium piloté par Playwright, avec Mailpit
pour les emails et des comptes de test supprimés ensuite.

| Scénario | Résultat | Écart ou remarque |
|---|---|---|
| 1. Compte | Conforme | Le verrou de 5 échecs est prouvé par les tests (`LoginTest`) et affiché par le web (test `LoginPage`), mais n'a pas été rejoué à la main. |
| 2. Publier | Conforme | Création, refus sans question, 3 questions dont un bloc de code, réordonnancement conservé au rechargement, publication, vue visiteur, dépublication, dernière question protégée. |
| 3. Consulter | Conforme | Les données de démonstration sont en faux latin : « Verbes irréguliers anglais » n'existe pas, la recherche a été vérifiée sur « re » (8 sujets) et un mot absent. |
| 4. Catégories | Conforme | Création, doublon refusé sans tenir compte de la casse ni des accents, renommage, réordonnancement, refus de suppression avec le nombre de sujets. |
| 5. Modérer | Conforme | Signalement, second refusé, retrait refusé sans motif puis accepté, statut et motif côté auteur, republication impossible, historique, rétablissement. |
| 6. Réviser | Conforme | 3 « je savais » et 2 « je ne savais pas » : 3 cartes en boîte 2 dues dans 2 jours, 2 en boîte 1 dues demain ; un double clic compte une fois ; bilan et arrêt d'apprendre vérifiés. |
| 7. Mobile et PWA | Conforme, avec une réserve | Aucun défilement horizontal ni bouton tronqué de 360 à 1440 px, y compris texte à 200 % à 360 px ; axe-core sans violation WCAG A/AA ; cibles de 44 px. Manifeste et icônes servis, écran « Vous êtes hors ligne » sans réseau puis retour à la page demandée, invitation « Mettre à jour » entre deux builds, instructions iPhone. **Réserve** : l'invitation d'installation Chrome Android (`beforeinstallprompt`) ne se déclenche pas dans un navigateur piloté ; elle est couverte par un test unitaire et reste à confirmer sur un vrai téléphone. |
