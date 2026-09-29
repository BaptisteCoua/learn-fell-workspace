# Guide de validation : Images dans les questions (003)

Ce guide prouve la feature de bout en bout. Le contrat est dans [contracts/api.md](contracts/api.md)
et les règles dans [data-model.md](data-model.md).

## Prérequis

- `back/` démarré, migré et seedé (voir `back/CLAUDE.md`), avec `queue:work` et `schedule:work`.
- `web/` en `pnpm dev` pour les parcours ; en build de production pour vérifier que le service
  worker ne met pas les images en cache.
- Trois comptes confirmés : un auteur, un autre inscrit, un administrateur
  (`./vendor/bin/sail artisan users:grant-admin <email>`).
- Des fichiers de test : une photo JPEG de téléphone avec position GPS et orientation EXIF à 90°,
  un PNG transparent, un WebP, un GIF animé, un SVG, un JPEG de 7 Mo, un PNG de 9 000 px de
  côté, un fichier texte renommé en `.jpg`.

## Tests automatisés

```bash
cd back && ./vendor/bin/sail artisan test functional/catalog functional/learning
```

```bash
cd web && pnpm test && pnpm lint && pnpm exec prettier --check .
```

Attendu : tout est vert. Les tests de visibilité couvrent la matrice complète de SC-004 (5 états
× 4 profils, plus l'image en attente).

## Scénarios manuels

### US1 : ajouter des images

1. Connecté comme auteur, ouvrez un brouillon, puis « Ajouter une question ».
2. Ajoutez la photo de téléphone : la progression s'affiche, puis l'aperçu, dans le bon sens.
3. Essayez d'enregistrer : c'est refusé, avec « Décrivez cette image. » sur l'image. Saisissez
   une description.
4. Ajoutez le PNG et le WebP. Tentez le GIF, le SVG, le fichier de 7 Mo et le faux `.jpg` : chacun
   est refusé avant l'envoi, avec le message des formats et des limites. Le PNG de 9 000 px est
   refusé par le back avec le message des dimensions.
5. Ajoutez une quatrième image : le bouton d'ajout se désactive.
6. Descendez la première image avec sa flèche, laissez le texte du recto vide, remplissez le
   verso et enregistrez : la question « image seule » est enregistrée. Rechargez la page : les
   images sont dans le nouvel ordre, avec leur description.
7. Remplacez une image, puis retirez-en une et enregistrez. Ouvrez l'ancienne adresse de chacune
   (`/api/question-images/{id}/960`) : `404`.
8. Retirez toutes les images d'une question sans texte : l'enregistrement est refusé avec
   « Ajoutez un texte ou une image au recto. ».
9. Sur un téléphone, touchez « Ajouter une image » : la galerie et l'appareil photo sont proposés.
10. Coupez le réseau : le bouton d'ajout est désactivé et le bandeau hors ligne s'affiche.

### US2 : voir les images

1. Publiez le sujet. Déconnecté, ouvrez-le : les images sont au-dessus du texte du recto, dans
   l'ordre choisi.
2. À 360 px de large, il n'y a pas de défilement horizontal. Touchez une image : elle s'ouvre en
   plein écran, et Échap ou « Fermer » ramène à la question.
3. Avec un lecteur d'écran, la description de chaque image est lue.
4. Dans les outils du navigateur, bloquez `/api/question-images/*` : la description s'affiche à
   la place de chaque image.
5. Connecté comme autre inscrit, apprenez le sujet et lancez une séance : l'image est visible
   avant le verso et les boutons de réponse restent accessibles. Sur un écran de 360 px, la
   page d'une carte charge moins de 500 Ko (variante de 480 px).
6. Ouvrez une image téléchargée avec un lecteur EXIF : aucune position, aucun appareil, aucune
   date de prise de vue.

### US3 : visibilité

1. Sur un brouillon avec une image, notez l'adresse de l'image. Ouvrez-la déconnecté, puis comme
   autre inscrit : `404`. Comme auteur, puis comme administrateur : l'image s'affiche.
2. Publiez : l'adresse répond à tous. Dépubliez : `404` pour le visiteur.
3. Republiez, puis retirez le sujet comme administrateur : `404` pour le visiteur, image visible
   par l'auteur dans « Mes sujets ». Rétablissez et republiez : l'image est de nouveau visible.
4. Supprimez la question, puis un sujet entier : `404` pour tous, administrateur compris, et
   plus aucun fichier sous `storage/app/private/question-images/{id}`.
5. Ajoutez une image à une nouvelle question sans l'enregistrer, puis fermez l'onglet. Son adresse
   donne `404` à l'autre inscrit. Après 24 heures (ou `$this->travel(25)->hours()` en test, puis
   `./vendor/bin/sail artisan model:prune`), la ligne et les fichiers ont disparu.

### Leitner

Un apprenant a une carte en boîte 3, due dans 4 jours. L'auteur ajoute une image à cette
question, la remplace, puis la retire : la carte reste en boîte 3 avec la même échéance.
