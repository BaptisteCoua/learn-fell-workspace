# Contrat : Images dans les questions (003)

Le back (`back/`) fournit l'API ; le web (`web/`) la consomme. Base :
`https://api.<domaine>/api` (en développement `http://localhost:8090/api`). Les conventions de la
001 s'appliquent : session Sanctum SPA, en-tête `X-XSRF-TOKEN`, erreurs métier `{code, message}`
venues des traductions françaises du back, erreurs de validation `422` au format Laravel.

## 1. Envoi d'une image (hors lomkit)

`POST /question-images`, `multipart/form-data`, champ unique `file`.

- **Accès** : un compte connecté à l'adresse confirmée ; sinon `401` ou `403`. Limite de
  30 appels par minute et par compte (`429`).
- **Validation** (`422`, sur le champ `file`) :

| Règle | Message (fr) |
|---|---|
| fichier absent ou illisible | « Choisissez une image au format JPEG, PNG ou WebP. » |
| contenu autre que JPEG, PNG ou WebP, ou image animée | « Choisissez une image au format JPEG, PNG ou WebP. » |
| plus de 5 Mo | « L'image ne doit pas dépasser 5 Mo. » |
| plus de 8 000 px de côté | « L'image ne doit pas dépasser 8 000 pixels de côté. » |

- **Réponse `201`** : l'image en attente.

```json
{ "data": { "id": 812, "width": 1600, "height": 1067, "variant_widths": [480, 960, 1600] } }
```

- Le fichier est réorienté, réduit en variantes WebP et débarrassé de ses métadonnées avant
  la réponse (research R3). L'image n'est rattachée à aucune question.

## 2. Diffusion d'une image (hors lomkit)

`GET /question-images/{id}/{width}`, avec `width` ∈ `480`, `960`, `1600`.

- **Réponse `200`** : `image/webp`, `X-Content-Type-Options: nosniff`.
  - `Cache-Control: private, max-age=3600` si le sujet est publié ;
  - `private, no-store` sinon.
  - Une largeur non produite (petite image) renvoie la plus grande variante disponible.
- **Réponse `404`**, identique dans tous les cas : image inexistante, supprimée, ou non visible
  pour la personne (règles de [data-model.md](../data-model.md#règles-de-visibilité-fr-016)),
  ou largeur hors liste.
- Le web charge l'image avec `crossorigin="use-credentials"`. La réponse porte les en-têtes CORS
  de `api/*` (`Access-Control-Allow-Origin: <FRONTEND_URL>`,
  `Access-Control-Allow-Credentials: true`).

## 3. Ressources lomkit

### question-images (lecture seule)

- **Champs** : `id`, `question_id`, `alt`, `position`, `width`, `height`, `variant_widths`.
  `uploader_id` n'est jamais renvoyé.
- **Lecture** : uniquement comme relation incluse, `include: [{ relation: "images" }]` sur
  `questions`, ou `question.images` sur `card-progress`. Périmètres de
  `QuestionImageControl` ; les images en attente n'en sortent jamais.
- **`mutate`, `DELETE` et actions directs** : refusés (`403`). Les images changent uniquement
  par le `mutate` de leur question.

### questions (étendue)

- **Relation** : `images` (`HasMany` vers `question-images`), triée par `position`.
- **`mutate`, opérations `create` et `update`** : quand le recto ou ses images changent, le web
  envoie **la liste complète** des images.

```json
{
  "mutate": [{
    "operation": "update",
    "key": 311,
    "attributes": { "recto_html": "", "verso_html": "<p>Le Grand Duc</p>" },
    "relations": {
      "images": [
        { "operation": "update", "key": 812, "attributes": { "alt": "Hibou aux aigrettes, perché sur une branche, de face", "position": 0 } },
        { "operation": "update", "key": 790, "attributes": { "alt": "Le même hibou en vol, ailes déployées", "position": 1 } }
      ]
    }
  }]
}
```

- **Règles** (`422`) :
  - `relations.images` : 4 éléments au plus, sinon « Un recto porte au plus 4 images. » ;
  - `alt` : obligatoire, de 1 à 250 caractères, sinon « Décrivez cette image. » ou « La
    description ne doit pas dépasser 250 caractères. » ;
  - `position` : de 0 à 3, toutes distinctes ;
  - `recto_html` : de 1 à 5 000 caractères visibles, ou vide si au moins une image. Sinon,
    erreur métier `422` `recto_empty` : « Ajoutez un texte ou une image au recto. ».
- **Autorisation** : une image qui n'est ni en attente et envoyée par l'utilisateur courant, ni
  déjà rattachée à cette question, donne `403` et n'enregistre rien.
- **Effets** :
  - les images absentes de la liste sont supprimées définitivement ;
  - les autres prennent l'`alt` et la `position` reçus ;
  - la progression Leitner ne change pas.
- **Sans `relations.images`** (texte seul modifié, question sans image) : les images existantes
  sont conservées. La règle du recto tient alors compte des images déjà rattachées.

### card-progress (inclusion)

`include: [{ relation: "question.images" }]` est accepté par la séance. Les périmètres de
`question-images` s'appliquent.

## 4. Contenu mis en forme

Sans changement : `<img>` et toute autre balise hors de la liste de la 001 sont retirés à
l'enregistrement du recto et du verso (FR-015).

## 5. Web

Aucune nouvelle page.

| Surface | Couche | Changement |
|---|---|---|
| Éditeur de question (`/sujets/[id]/modifier`) | Authoring | champ d'images au-dessus du recto, aperçu des images dans `QuestionCard` |
| Lecture d'un sujet (`/sujets/[id]`) | Catalog | bloc d'images au-dessus du recto dans `QuestionItem` |
| Mode cartes (`/sujets/[id]/cartes`) | Catalog | bloc d'images sur la face recto |
| Séance (`/revisions/seance`) | Learning | bloc d'images au-dessus du recto dans `SessionCard` |
| Visionneuse | Catalog | `v-dialog` plein écran, fermeture par Échap, par un bouton ou par un toucher |

Les adresses d'image sont construites par le web : `${apiBaseUrl}/question-images/{id}/{width}`.
