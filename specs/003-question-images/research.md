# Recherche : Images dans les questions (003)

Chaque décision indique ce qui a été choisi, pourquoi, et ce qui a été écarté. Les inconnues du
contexte technique du plan sont toutes tranchées ici.

## État des lieux

- **Back** : aucune gestion de fichier aujourd'hui (ni `Storage::`, ni `UploadedFile`, ni paquet
  d'image). Le disque par défaut est `local` (`storage/app/private`) ; le `compose.yaml` de Sail
  n'a ni S3 ni MinIO. Le runtime Sail 8.5 installe `php8.5-gd` et `php8.5-imagick`.
- **Contenu mis en forme** : `stevebauman/purify` avec `HTML.Allowed = p,br,strong,em,ul,ol,li,code,pre,a[href]` ;
  `<img>` est déjà supprimé à l'enregistrement. La règle `VisibleTextLength` impose 1 à 5 000
  caractères visibles au recto.
- **Visibilité** : purement par filtrage des requêtes (`QuestionControl`, `SubjectControl`) ; un
  contenu caché est absent de `search`, sans 404. Statuts de sujet : `draft`, `published`,
  `retired` ; dépublier ramène à `draft`.
- **Suppression** : aucune cascade en base (`restrictOnDelete`) ; les suppressions passent par
  des événements de modèle et des listeners (`SubjectDeleting` → `DeleteSubjectQuestions`,
  `QuestionDeleting` → `RemoveQuestionFromLearners`). Aucune `SoftDeletes`.
- **Web** : TipTap v3 sans extension image ; `laravel-raom-nuxt` 0.3.7 n'envoie que du JSON
  (pas de `FormData`, ni progression, ni annulation) mais sait envoyer les opérations de
  relation (`update`, `attach`, `detach`, `sync`) dans un `mutate`. Aucune image, aucun champ
  fichier dans l'application. Workbox sert `/api/*` en `NetworkOnly`.

## R1. Place dans les couches

- **Décision** : les images sont une partie de la question ; elles vivent dans la couche
  `back/functional/catalog` (modèle `QuestionImage`) et, côté web, dans `functional/Catalog`
  (modèle, affichage), `functional/Authoring` (saisie) et `functional/Learning` (séance, qui
  réutilise l'affichage de Catalog). L'envoi de fichier avec progression est un outil
  d'infrastructure placé dans `web/technical/ApiClient`, à côté de `useApiFetch`.
- **Pourquoi** : la visibilité, la suppression et les limites d'une image découlent entièrement
  de sa question et de son sujet, qui sont dans `catalog`. Une couche séparée devrait tout
  demander à `catalog`.
- **Écarté** : une couche technique `media` générique, sans deuxième usage (principe VII).

## R2. Stockage

- **Décision** : un disque Laravel privé, nommé par `catalog.images.disk`
  (`QUESTION_IMAGES_DISK`, `local` par défaut), sous `question-images/{id}/{largeur}.webp`.
  Aucun fichier n'est public : tout passe par la route de diffusion (R8).
- **Pourquoi** : le disque `local` est privé et suffit en développement et en test
  (`Storage::fake`). Le nom de disque en configuration permet de passer à un stockage objet
  sans toucher au code.
- **Écarté** :
  - le disque `public` et le lien `public/storage` : une adresse publique resterait lisible
    après la dépublication ou le retrait (FR-016) ;
  - S3 ou MinIO dès maintenant : aucun environnement de production n'est encore défini ; le
    paquet `league/flysystem-aws-s3-v3` s'ajoutera avec le déploiement.

## R3. Traitement des images

- **Décision** : `intervention/image` (dernière version stable, vérifiée par `composer show` à
  l'installation) avec le pilote Imagick, de façon synchrone à l'envoi :
  1. orientation corrigée d'après l'EXIF (FR-013, edge case « photo prise au téléphone ») ;
  2. trois variantes WebP, qualité 80, de 480, 960 et 1 600 px de large au plus, jamais
     agrandies ;
  3. toutes les métadonnées retirées ; le fichier d'origine n'est jamais conservé (FR-014,
     SC-005).
  Les dimensions de la plus grande variante sont enregistrées pour réserver la place à
  l'affichage.
- **Pourquoi** : le réencodage à partir des seuls pixels garantit qu'aucune métadonnée ne
  survit ; ne pas garder l'original supprime le risque au lieu de le filtrer. Les variantes
  tiennent l'objectif de SC-003 (une carte à 360 px charge une variante de 480 px, soit
  quelques dizaines de Ko). Le traitement synchrone d'une image de 5 Mo prend 1 à 2 secondes,
  dans le budget de SC-002, et l'image est affichable dès la réponse.
- **Écarté** :
  - Imagick en direct : API bas niveau et difficile à doubler en test pour un gain nul ;
  - GD : consomme toute la mémoire décodée d'une image de 8 000 × 8 000 px ;
  - un traitement en file d'attente : l'image ne serait pas affichable à la fin de l'envoi,
    et l'état « en cours de traitement » ajouterait un cycle de vie ;
  - `spatie/laravel-medialibrary` : un modèle polymorphe générique et ses conversions pour un
    seul usage (principe VII).

## R4. Contrôle des fichiers envoyés

- **Décision** :
  - côté back, les règles `mimes:jpg,jpeg,png,webp` (contenu réel, lu par `finfo`),
    `max:5120` et `dimensions:max_width=8000,max_height=8000`, appliquées avant tout décodage.
    Une image animée (plusieurs images dans le fichier) est refusée avec le même message que
    les formats non acceptés ;
  - côté web, un premier contrôle du type et de la taille avant l'envoi (FR-002 : « refusé avant
    d'être envoyé »), ainsi que du nombre d'images.
- **Pourquoi** : la règle `dimensions` lit l'en-tête sans décoder l'image, ce qui écarte les
  bombes de décompression. Le contrôle web évite d'envoyer 7 Mo pour rien sur mobile ; le
  contrôle back reste la seule garantie.
- **Écarté** : faire confiance à l'extension ou au type MIME envoyé par le navigateur.

## R5. Envoi du fichier

- **Décision** :
  - `POST /api/question-images`, en `multipart/form-data`, hors lomkit, déclaré dans
    `back/functional/catalog/routes/api.php` comme la désinscription de la 002. La route
    exige un compte connecté à l'adresse confirmée et est limitée à 30 envois par minute par
    compte. Elle crée une image **en attente** (sans question) qui appartient à son auteur
    et renvoie sa représentation ;
  - côté web, un composable `useUploadRequest` dans `technical/ApiClient` envoie le fichier par
    `XMLHttpRequest`, avec les cookies, l'en-tête `X-XSRF-TOKEN`, la progression et
    l'annulation.
- **Pourquoi** : raom et `$fetch` n'exposent ni la progression ni l'annulation d'un envoi
  (FR-007). La constitution interdit le client HTTP direct vers un endpoint **lomkit** ; cette
  route n'en est pas un, exactement comme les appels Fortify passés par `useApiFetch`.
  Envoyer l'image avant d'enregistrer la question permet d'afficher l'aperçu et la
  progression pendant que l'auteur écrit.
- **Écarté** :
  - une action lomkit : elle ne reçoit pas de fichier par raom ;
  - une contribution à raom pour l'envoi multipart : utile, mais bloquerait la feature ;
    elle reste possible ensuite ;
  - des données base64 dans le JSON du `mutate` : +33 % de volume, et aucune progression.

## R6. Rattachement à la question

- **Décision** : les images sont la relation `HasMany images` de `QuestionResource`. Le web
  envoie **toujours la liste complète** des images du recto avec le `mutate` de la question,
  une opération `update` par image (`key`, `alt`, `position`). Dans la même transaction, le
  back supprime les images de la question absentes de la liste (retrait, remplacement,
  FR-005, FR-017).
  - Une image n'est rattachable que si elle est en attente et envoyée par la personne qui
    enregistre, ou si elle appartient déjà à cette question (`QuestionImagePolicy::attach`).
    Sinon, 403.
  - La suppression des fichiers a lieu après la validation de la transaction.
- **Pourquoi** : une seule requête enregistre le texte et les images, ce qui garantit
  l'atomicité voulue par FR-004 et FR-008 (pas de question enregistrée avec une image sans
  description). `HasMany` de lomkit applique `update` à un modèle existant en posant la clé
  étrangère, et vérifie l'autorisation pour chaque image ; raom produit ces opérations à
  partir des modèles modifiés.
- **Écarté** :
  - enregistrer chaque image par sa propre ressource après la question : plusieurs requêtes,
    donc des états intermédiaires visibles, et aucune atomicité ;
  - `detach` au sens lomkit (clé étrangère remise à nul) : il laisserait une image orpheline.
- **Amendement du 2026-09-29, pendant l'implémentation** : raom 0.3.7 n'envoie pas une liste de
  relation vide (`buildRelationPayload` renvoie `undefined` quand aucune opération n'est en
  file). Le retrait de la dernière image d'un recto qui a du texte serait donc perdu.
  - Le web envoie donc aussi un `detach` pour chaque image retirée, et le back traite un
    `detach` sur `images` comme une suppression définitive dans la même transaction.
  - `authorizeToDetach` n'accepte que les images de cette question.
  - La règle « absente d'une liste non vide = supprimée » reste en place.

## R7. Recto sans texte

- **Décision** : la règle du recto devient « 1 à 5 000 caractères visibles, ou vide si la liste
  d'images du payload en contient au moins une » ; au plus 4 images, positions 0 à 3 toutes
  distinctes ; `alt` de 1 à 250 caractères, rogné, sans balise. Un recto vide et sans image est
  refusé avec le code `recto_empty` (FR-008). Toutes les règles sont évaluées avant toute
  écriture.
- **Pourquoi** : la liste complète envoyée avec la question (R6) permet de tout valider sur le
  seul payload.
- **Écarté** : valider après l'écriture puis annuler, plus fragile et moins lisible.

## R8. Diffusion des images

- **Décision** :
  - `GET /api/question-images/{id}/{largeur}` (largeur 480, 960 ou 1 600), servi par un
    contrôleur de la couche `catalog`. Il décide avec les mêmes règles que la lecture des
    questions (FR-016) :
    - image rattachée : visible si sa question l'est pour la personne, c'est-à-dire sujet
      publié pour tous, et brouillon ou sujet retiré pour l'auteur et les modérateurs ;
    - image en attente : visible de son seul auteur.
  - Toute autre situation répond `404`, exactement comme une image qui n'existe pas.
  - En-têtes :
    - `Cache-Control: private, max-age=3600` pour une image de sujet publié ;
    - `private, no-store` sinon ;
    - `X-Content-Type-Options: nosniff`, `Content-Type: image/webp`.
  - Le web pose `crossorigin="use-credentials"` sur chaque `<img>`. Le navigateur envoie
    alors l'en-tête `Origin`, que Sanctum exige pour reconnaître la session, ainsi que les
    cookies. La réponse passe par la configuration CORS existante (`api/*`,
    `supports_credentials`).
- **Pourquoi** :
  - une adresse stable, contrôlée à chaque requête, rend une image introuvable dès que son
    sujet ne l'est plus, sans délai d'expiration ;
  - le front et l'API sont sur le même site (`localhost`, puis `<domaine>` et
    `api.<domaine>`), donc le cookie `SameSite=Lax` accompagne les images ;
  - `private` interdit à un cache partagé de garder une image après le retrait du sujet.
- **Écarté** :
  - des URL signées temporaires : une adresse copiée resterait lisible jusqu'à son expiration,
    après une dépublication ou un retrait ;
  - un cache public ou un CDN : même défaut ;
  - des images en base64 dans la réponse de l'API : charge inutile de chaque `search`.

## R9. Suppression et expiration

- **Décision** :
  - `QuestionImage` déclare `deleted` → `QuestionImageDeleted` dans `$dispatchesEvents`. Le
    listener `DeleteQuestionImageFiles` efface le dossier de l'image après la validation de
    la transaction ;
  - `QuestionDeleting` reçoit un nouveau listener de `catalog`, `DeleteQuestionImages`, qui
    supprime chaque image comme un modèle. La suppression d'un sujet passe déjà par celle de
    ses questions ;
  - les images en attente sont `Prunable` (`question_id` nul et plus de 24 heures). Le modèle
    est élagué toutes les heures par `model:prune --model=…QuestionImage`, qui déclenche les
    événements et donc la suppression des fichiers (FR-018).
- **Pourquoi** : c'est le schéma de suppression de la 001 (listeners, pas de cascade en base).
  L'élagage quotidien global laisserait une image jusqu'à 48 heures.
- **Écarté** :
  - une cascade en base, qui ne supprime pas les fichiers ;
  - une commande de purge dédiée : `Prunable` la remplace.

## R10. Contenu mis en forme

- **Décision** : l'allow-list de Purify ne change pas, donc `<img>` reste supprimé du recto et
  du verso (FR-015). TipTap ne reçoit pas d'extension image. Un test vérifie qu'une balise
  `<img src="https://…">` saisie est retirée.
- **Pourquoi** : les images s'affichent en bloc au-dessus du texte (FR-010) ; elles ne sont
  jamais dans le HTML.

## R11. Affichage web

- **Décision** : un composant `QuestionImageGallery` dans `functional/Catalog`, utilisé par la
  lecture du sujet, le mode cartes, la séance (`functional/Learning`) et l'aperçu de
  l'éditeur.
  - Le bloc est placé au-dessus du texte du recto : 1 image en pleine largeur, 2 à 4 en grille
    de deux colonnes. `srcset` sur 480w, 960w et 1600w, `sizes` selon la grille ; `width` et
    `height` réservent la place.
  - `loading="lazy"`, sauf en séance où la carte courante est chargée tout de suite.
  - Un clic ou un toucher ouvre `QuestionImageViewer`, un `v-dialog` plein écran qui se ferme
    par la touche Échap, le bouton « Fermer » ou un toucher.
  - En cas d'échec de chargement, la description s'affiche dans un cadre à la place de
    l'image (FR-012).
  - Les URL sont construites à partir de `apiBaseUrl`, de l'`id` et de la largeur.
- **Pourquoi** : un seul composant garantit le même rendu partout où le recto s'affiche
  (FR-009). Les variantes étant fixes, l'API n'a pas à renvoyer d'URL.
- **Écarté** : une bibliothèque de visionneuse, alors que `v-dialog` couvre le besoin.

## R12. Saisie web

- **Décision** : un composant `QuestionImagesField` au-dessus de l'éditeur du recto dans
  `QuestionForm`, avec le composable `useQuestionImages`.
  - Un bouton « Ajouter une image » ouvre `<input type="file" accept="image/jpeg,image/png,image/webp">`
    sans attribut `capture`, pour que les téléphones proposent à la fois la galerie et
    l'appareil photo (FR-006).
  - Chaque image affiche un aperçu, un champ de description (compteur de 250 caractères),
    des flèches pour monter et descendre, « Remplacer » et « Retirer ».
  - Pendant l'envoi, l'image affiche une barre de progression et un bouton « Annuler ».
  - `useQuestionDraft` bloque l'enregistrement tant qu'un envoi est en cours ou qu'une
    description manque (FR-004, FR-007), et considère la question comme non vide dès qu'elle
    a du texte ou une image (FR-008).
  - L'ajout est désactivé hors ligne, grâce à `useConnectionStatus().isOffline` (edge case
    « envoi interrompu »).
- **Pourquoi** : les flèches reprennent le réordonnancement des questions de la 001. Elles
  sont accessibles au clavier et fonctionnent à 360 px, sans bibliothèque de glisser-déposer.
- **Écarté** : le glisser-déposer (`vuedraggable`), dépendance de plus et peu accessible.

## R13. Leitner

- **Décision** : rien à changer. La mise à jour d'une question n'a aucun listener côté
  `learning`, donc la boîte et l'échéance sont conservées (FR-019). Un test le vérifie pour
  l'ajout, le remplacement et le retrait d'une image.

## R14. Tests

- **Back** (PHPUnit, `Storage::fake`, fichiers `UploadedFile::fake()->image()` et de vrais
  fichiers de test pour l'EXIF et l'orientation) :
  - envoi : formats, taille, dimensions, fichier dont l'extension ment, image animée ;
  - rattachement : limite de 4, description obligatoire, recto vide, image d'un autre compte
    refusée ;
  - réordonnancement, remplacement et retrait avec suppression des fichiers ;
  - diffusion : matrice statut × profil de SC-004, image en attente, 404 identique ;
  - métadonnées absentes et orientation corrigée ;
  - élagage à 24 heures ;
  - suppression en cascade par question et par sujet ;
  - `<img>` retiré du HTML ;
  - Leitner inchangé.
- **Web** (Vitest, `@nuxt/test-utils`) :
  - `QuestionImagesField` : refus avant envoi, progression, annulation, description
    obligatoire, flèches ;
  - `useQuestionDraft` : payload complet des images, enregistrement bloqué ;
  - `QuestionImageGallery` : ordre, `srcset`, `crossorigin`, repli sur la description,
    visionneuse et touche Échap ;
  - `SessionCard` et `QuestionItem` : images au recto.
