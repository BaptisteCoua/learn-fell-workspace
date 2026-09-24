# CINQ Constitution

> Anciennement « Learn Fell ». Renommé en CINQ le 2026-09-24 (v1.0.1, sans changement de règle).

CINQ est un espace public d'apprentissage par la méthode Leitner. Le produit est un back
Laravel dans `back/` et un front Nuxt livré en PWA dans `web/`. Il n'a pas d'application native :
la PWA est son unique expérience mobile.

## Core Principles

### I. La spec d'abord

- Aucune fonctionnalité n'est développée sans une spec validée par le développeur.
- La spec, le plan et les tâches DOIVENT être commités et poussés dans le workspace à la fin de
  chaque étape du cycle : une spec qui n'existe que sur une machine n'existe pas pour l'équipe.
- Tout chemin écrit dans un plan, une tâche ou un outil DOIT commencer par son repo (`back/…`,
  `web/…`). Il n'y a pas de repo par défaut dans le workspace.
- Si le plan ou le développement révèle une spec floue, on corrige la spec avant de continuer ;
  on ne contourne pas le problème dans les tâches ou le code.

### II. Architecture en couches OSDD

- Le back DOIT être organisé en couches `xefi/laravel-osdd` : un domaine par couche, et chaque
  couche porte ses modèles, migrations, factories, tests et son service provider.
- Le web DOIT être organisé en couches `nuxt-osdd` : `technical/<Couche>` pour l'infrastructure,
  `functional/<Couche>` pour les domaines métier. Il n'a pas de dossier `app/` à la racine.
- La logique métier d'un domaine vit dans sa couche et nulle part ailleurs. Une couche ne dépend
  d'une autre qu'à travers ce que cette autre expose.

Raison : les domaines de CINQ (comptes, contenu, modération, révision) évoluent séparément ;
des couches isolées permettent de les faire grandir sans enchevêtrement.

### III. Paquets imposés et API par contrat

- Côté back, le CRUD DOIT passer par `lomkit/laravel-rest-api`, les autorisations par
  `lomkit/laravel-access-control`, le stockage des rôles et permissions par
  `spatie/laravel-permission`, et les factories par `xefi/faker-php-laravel`.
- L'accès se décide sur des permissions, jamais sur un nom de rôle codé en dur.
- Côté web, tout appel à l'API DOIT passer par les modèles `laravel-raom-nuxt`. Aucun `fetch`,
  `$fetch`, `useFetch` ou client HTTP direct vers un endpoint lomkit.
- Le back est le fournisseur : un endpoint est livré, testé et présent sur la branche principale
  du back avant que le web ne le consomme.

### IV. Tests obligatoires (NON NÉGOCIABLE)

- Chaque exigence fonctionnelle de la spec DOIT être couverte par au moins un test automatisé.
- Côté back : PHPUnit uniquement, lancé par `./vendor/bin/sail artisan test`. Côté web : Vitest
  avec `@nuxt/test-utils`, les tests vivant dans `<couche>/tests/`.
- Les règles Leitner (boîte d'arrivée, intervalle, maintien en boîte 5, retour en boîte 1) et
  l'invisibilité des brouillons et sujets retirés pour les personnes non autorisées DOIVENT être
  couvertes à 100 % de leurs cas.
- Aucune merge request n'est fusionnée avec un test rouge ou une analyse statique en échec.

Raison : la promesse du produit est de faire revenir la bonne carte au bon moment ; une erreur
dans ce calcul est invisible pour l'utilisateur et ruine l'apprentissage.

### V. Accessibilité et mobile d'abord

- Les parcours principaux (consulter, s'inscrire, créer, publier, signaler, réviser) DOIVENT
  fonctionner à 360 px de large sans défilement horizontal.
- Le texte courant a un contraste d'au moins 4.5:1, le focus est visible, les cibles tactiles
  font au moins 44 px, et `prefers-reduced-motion` coupe les animations non essentielles.
- La typographie est fluide et exprimée en `rem` ; aucun texte ne descend sous 12 px ; le réglage
  de taille de texte du téléphone est respecté.
- L'interface est entièrement en français, en vouvoyant l'utilisateur. Aucun texte destiné à
  l'utilisateur n'est codé en dur : il vient des fichiers de traduction, côté back comme côté web.

### VI. Sécurité et données personnelles

- Le contenu mis en forme saisi par les utilisateurs DOIT être assaini à l'enregistrement ; rien
  de ce qu'un auteur écrit ne peut s'exécuter chez un lecteur.
- Aucun message ni aucune réponse de l'API ne révèle si une adresse email possède un compte.
- Un brouillon ou un sujet retiré n'est exposé qu'aux personnes autorisées ; pour toute autre
  personne, il est introuvable, que ce soit dans une liste, une recherche ou à son adresse directe.
- Les données personnelles suivent le RGPD : la suppression de compte DOIT être livrée avant
  l'ouverture au public.

### VII. Simplicité

- On ne construit que ce que la spec demande. Toute idée hors spec devient une nouvelle feature.
- Pas d'abstraction, de couche ou d'option de configuration sans un deuxième usage réel.
- On utilise la version stable la plus récente de chaque dépendance ; toute exception est
  justifiée dans le plan.

## Contraintes techniques

- **Back** (`back/`) : Laravel 13, PHP 8.5, PostgreSQL, Redis, exécutés par Laravel Sail (Docker).
  PHP n'est pas installé sur l'hôte : toute commande `php`, `composer`, `artisan` ou `npm` passe
  par `./vendor/bin/sail`. Formatage par Pint.
- **Web** (`web/`) : Nuxt 4, Vue 3, TypeScript 6, pnpm uniquement, Vuetify, `@nuxtjs/i18n`
  (français uniquement), `@vite-pwa/nuxt`, Pinia, `laravel-raom-nuxt`. Lint par ESLint et Prettier.
- **Direction visuelle** : celle de la maquette 001 validée (brutalisme, jaune `#FACC15` et bleu
  `#3B82F6` sur crème `#FFFBEB`, Archivo, Space Grotesk, JetBrains Mono). Le design system Xefi
  n'est pas utilisé pour ce produit.
- Chaque repo porte ses propres conventions dans son `CLAUDE.md`, à lire avant toute modification ;
  elles complètent cette constitution sans pouvoir la contredire.

## Flux de travail

- Chaque feature suit le cycle : spec, maquette si elle est utile, plan, tâches, développement,
  puis convergence jusqu'à ce que le code corresponde à la spec, au plan et aux tâches.
- Le développement se fait sur une branche de feature dans chaque repo concerné, créée à partir de
  la branche principale de ce repo avant la première ligne de code.
- Une merge request par repo, liées entre elles par le numéro de feature, fusionnées dans l'ordre
  des dépendances : le back d'abord, le web ensuite. Une merge request web reste ouverte tant que
  l'endpoint qu'elle consomme n'est pas fusionné et déployé côté back.
- Messages de commit à l'impératif, en anglais, sans aucune mention d'IA.
- Claude ne fusionne jamais une branche : le développeur s'en charge.
- Une feature n'est terminée que si aucun repo concerné n'a de travail non commité, de mauvaise
  branche ou de commit non poussé.

## Governance

- Cette constitution prime sur toute autre pratique du workspace et des repos.
- Tout amendement est proposé par écrit, validé par le développeur, daté et commité dans le
  workspace. Sa version suit le versionnement sémantique : MAJEURE pour un principe retiré ou
  redéfini de façon incompatible, MINEURE pour un principe ou une section ajouté ou nettement
  élargi, CORRECTIVE pour une clarification sans effet sur les règles.
- Chaque plan comporte une vérification de conformité (Constitution Check) avant la conception
  et après celle-ci. Tout écart y est justifié ; un écart non justifié bloque le plan.
- La relecture de chaque merge request vérifie le respect des principes, en particulier les
  principes III, IV et VI.

**Version**: 1.0.1 | **Ratified**: 2026-09-24 | **Last Amended**: 2026-09-24
