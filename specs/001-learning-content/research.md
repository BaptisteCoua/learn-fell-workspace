# Recherche : Comptes, contenu et révision Leitner (001)

Chaque décision indique ce qui a été choisi, pourquoi, et ce qui a été écarté. Les inconnues du
contexte technique du plan sont toutes tranchées ici.

## R1. Architecture OSDD du back

- **Décision** : initialiser `xefi/laravel-osdd` avec `osdd:start`, puis créer une couche par
  domaine : `functional/users` (fournie par `osdd:start`, étendue aux comptes), `functional/catalog`,
  `functional/moderation`, `functional/learning`, et garder `technical/osdd`.
- **Pourquoi** : le repo est encore au squelette Laravel (`app/`, `database/`, `config/`). Le paquet
  range ses couches dans `functional/` et `technical/` (valeur par défaut de
  `vendor/xefi/laravel-osdd/config/osdd.php`), pas dans `layers/` comme l'écrit `back/CLAUDE.md` :
  la doc sera corrigée dans la première tâche.
- **Écarté** : une couche unique « content » regroupant catalogue et modération (deux cycles de vie
  distincts) ; une couche technique « sanitizer » pour un seul usage (principe VII).

## R2. Authentification

- **Décision** : `laravel/sanctum` en mode SPA (cookie de session `httpOnly` + protection CSRF) et
  `laravel/fortify` en mode headless (`views => false`) pour l'inscription, la connexion, la
  déconnexion, la vérification d'email et la réinitialisation du mot de passe. Pilote de session
  `database`.
- **Pourquoi** : le web et l'API partagent le même site (`app.` et `api.` d'un même domaine en
  production, `localhost` en développement), ce qui permet le cookie de première partie. Aucun jeton
  n'est stocké dans le navigateur, ce qui réduit l'impact d'une faille XSS dans une PWA. Fortify
  fournit déjà les flux exigés (FR-001 à FR-006) ; le pilote `database` permet de fermer les autres
  sessions à la réinitialisation (FR-005).
- **Réglages** : limiteur `login` à 5 tentatives par email, verrou de 15 minutes (FR-004) ;
  `auth.verification.expire = 1440` (lien de 24 h, FR-002) ; `auth.passwords.users.expire = 60`
  (FR-005) ; connexion refusée tant que `email_verified_at` est nul, avec un code d'erreur
  `email_not_verified` renvoyé seulement après un mot de passe correct (FR-006 respecté) ;
  « se souvenir » activé par défaut pour 30 jours.
- **Écarté** : `tymon/jwt-auth` (jetons à stocker côté client, rafraîchissement à gérer, aucun
  bénéfice pour un client de première partie) ; des contrôleurs d'authentification écrits à la main
  (Fortify couvre le besoin et est maintenu par Laravel).

## R3. Comptes jamais confirmés

- **Décision** : `User` implémente `Prunable` : `prunable()` cible les comptes sans
  `email_verified_at` créés il y a plus de 7 jours ; `model:prune` est planifié chaque jour.
- **Pourquoi** : conventions Xefi (rétention par `Prunable`, jamais `MassPrunable`) ; la suppression
  passe par les événements du modèle, donc par le nettoyage applicatif des données liées.

## R4. Contenu mis en forme

- **Décision** : l'éditeur web est TipTap (`@tiptap/vue-3`, `@tiptap/starter-kit`,
  `@tiptap/extension-link`) limité au gras, à l'italique, aux listes, au code en ligne et en bloc,
  et aux liens. Il produit du HTML. Le back l'assainit à l'enregistrement avec
  `stevebauman/purify` (HTMLPurifier) et une liste blanche identique :
  `p, br, strong, em, ul, ol, li, code, pre, a[href]`, liens `http(s)` et `mailto` uniquement,
  `rel="noopener nofollow ugc"` ajouté. La longueur maximale (5 000) porte sur le texte visible.
- **Pourquoi** : FR-015 et le principe VI demandent un assainissement au stockage ; HTMLPurifier est
  la référence PHP pour ce besoin. TipTap est l'éditeur Vue 3 le plus répandu et se limite
  proprement à une liste d'extensions.
- **Hors liste Xefi** : TipTap et Purify ne figurent pas dans les paquets approuvés ; ils sont
  signalés dans la merge request (voir Complexity Tracking du plan).
- **Écarté** : Markdown brut (les auteurs verraient la syntaxe ; la maquette montre une barre de mise
  en forme) ; un éditeur `contenteditable` maison (sécurité et accessibilité coûteuses).

## R5. Recherche

- **Décision** : colonne `subjects.search_document` (titre + description + tags, en minuscules et
  sans accents), mise à jour par l'application à chaque enregistrement du sujet ou de ses tags,
  indexée en GIN `pg_trgm`. La recherche passe par une instruction lomkit `search` sur la ressource
  `subjects`, qui normalise la requête de la même façon et filtre par `ILIKE`.
- **Pourquoi** : FR-025 demande une recherche insensible à la casse et aux accents, sur des
  sous-chaînes, à partir de 2 caractères ; SC-005 vise 10 000 sujets en moins de 2 secondes, ce que
  couvre un index trigramme. Normaliser dans l'application évite `unaccent()` (non `IMMUTABLE`,
  donc inutilisable dans une colonne générée).
- **Écarté** : Laravel Scout avec Meilisearch (un service de plus pour un volume modeste) ; la
  recherche plein texte `tsvector` (ne gère pas les fragments de mots comme « irreg »).

## R6. Planification Leitner

- **Décision** : une classe de domaine pure `LeitnerSchedule` dans `functional/learning` calcule la
  boîte d'arrivée et la date de prochaine révision : boîtes 1 à 5, intervalles 1, 2, 4, 8, 16 jours
  selon la boîte d'arrivée ; « je savais » : boîte + 1, plafonnée à 5 ; « je ne savais pas » :
  boîte 1, lendemain.
- **Échéance** : `card_progress.next_review_on` est une date (sans heure), calculée dans le fuseau de
  l'utilisateur (`users.timezone`, fuseau IANA envoyé par le navigateur à l'inscription et à la
  connexion). Une carte est due si `next_review_on <= aujourd'hui` dans ce fuseau.
- **Une seule réponse par présentation** : la réponse n'est acceptée que si la carte est due ; après
  la réponse, sa date passe dans le futur, donc une deuxième réponse (double clic, requête rejouée)
  est refusée par le serveur (409). Le client désactive aussi les boutons après le premier clic.
- **Pourquoi** : FR-042, FR-046 et SC-010 ; une classe pure se teste exhaustivement (principe IV).

## R7. Surface d'API

- **Décision** : `lomkit/laravel-rest-api` pour toutes les ressources et leurs actions (publier,
  dépublier, retirer, rétablir, réordonner, apprendre, arrêter d'apprendre, répondre, ignorer les
  signalements). Les seuls contrôleurs classiques sont ceux de Fortify et Sanctum. Détail dans
  [contracts/api.md](contracts/api.md).
- **Accès** : `lomkit/laravel-access-control`, une `Control` par modèle avec ses périmètres ; les
  visiteurs non connectés passent par le périmètre « public » (sujets publiés uniquement).
  Permissions `spatie/laravel-permission` : `categories.manage`, `subjects.moderate`,
  `reports.review`, `moderation.history.view`. Rôles `member` et `admin` insérés par migration
  (données de référence).
- **Premier administrateur** : commande Artisan réutilisable `users:grant-admin {email}`.

## R8. Répercussion des changements de contenu sur la révision

- **Décision** : événements de modèle écoutés par des listeners (pas d'observers, pas de cascade en
  base) : question créée → une progression en boîte 1 pour chaque apprenant du sujet (job en file) ;
  question supprimée → suppression de ses progressions ; sujet supprimé → suppression des
  apprentissages et progressions. Un sujet dépublié ou retiré n'est pas modifié : l'instruction
  `due` exclut les sujets non publiés, ce qui met les cartes en pause sans perdre leur état.
- **Pourquoi** : FR-051 ; conventions Xefi (`no-cascade-delete`, `no-observers`).

## R9. Web : rendu, client API et PWA

- **Rendu** : SSR Nuxt conservé pour les pages publiques (catalogue et sujets indexables). Le plugin
  `laravel-raom` transmet l'en-tête `cookie` de la requête pendant le rendu serveur.
- **Client API** : un plugin dans `technical/ApiClient` fournit `$laravelRaom.fetch`
  (`baseURL` depuis `runtimeConfig.public.apiBaseUrl`, `credentials: 'include'`, en-tête
  `X-XSRF-TOKEN` lu dans le cookie `XSRF-TOKEN`, appel préalable à `/sanctum/csrf-cookie`). Les
  appels d'authentification Fortify, hors lomkit, passent par le même `fetch`.
- **PWA** : `registerType` passe de `autoUpdate` à `prompt`, pour proposer « Mettre à jour » ou
  « Plus tard » (FR-040). Workbox : `NetworkOnly` pour l'API, page de repli hors ligne (FR-039).
  Invitation d'installation via `beforeinstallprompt`, instructions dédiées sur iOS (FR-037).
- **Style** : Vuetify reste la bibliothèque de composants, avec un thème CINQ (angles à 0,
  élévation 0, couleurs et polices de la maquette). Règles globales : `box-sizing: border-box` sur
  liens, boutons et champs ; typographie en `rem` avec `clamp()` sur mobile, 12 px minimum ;
  soulignement d'onglet actif réservé sur tous les onglets ; boutons côte à côte en `flex: 1 1 0`.

## R10. Environnement de développement

- **Décision** : ajouter le service `mailpit` au `compose.yaml` de Sail pour lire les emails de
  confirmation et de réinitialisation ; corriger `.env.example` (PostgreSQL au lieu de SQLite,
  `APP_LOCALE=fr`, variables `SANCTUM_STATEFUL_DOMAINS` et `SESSION_DOMAIN`).
