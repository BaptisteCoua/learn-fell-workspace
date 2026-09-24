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
./vendor/bin/sail up -d                 # API sur http://localhost:8090, Mailpit sur http://localhost:8025
./vendor/bin/sail artisan migrate --seed
./vendor/bin/sail artisan users:grant-admin admin@cinq.test

# web
cd ../web
pnpm install
pnpm dev                                # http://localhost:3000
```

## Tests automatisés

```bash
cd back && ./vendor/bin/sail artisan test      # PHPUnit, toutes les couches
cd back && ./vendor/bin/sail bin pint --test
cd web && pnpm test && pnpm lint
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
