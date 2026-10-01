# Quickstart: valider l'import de questions

Contrat dans [contracts/api.md](contracts/api.md), règles dans [data-model.md](data-model.md).

## Prérequis

- Back démarré (`cd back && ./vendor/bin/sail up -d`), migrations passées, et un worker de file
  (`./vendor/bin/sail artisan queue:work`), sans lequel les questions n'arrivent pas aux apprenants.
- Web démarré (`cd web && pnpm dev`).
- Des fichiers de test exportés pour de vrai :
  - par Excel en français : CSV et XLSX ;
  - par LibreOffice : CSV et XLSX ;
  - par Google Sheets : CSV et XLSX ;
  - un export texte d'Anki et un export de Quizlet.

  Les tests automatisés utilisent des fixtures qui reproduisent ces sorties : encodage, BOM,
  séparateur, fins de ligne, chaînes partagées ou en ligne du XLSX. Les vrais exports servent au
  parcours manuel.

## Tests automatisés

```bash
cd back && ./vendor/bin/sail artisan test --compact functional/catalog functional/learning
cd web && pnpm test
```

| Exigences | Couche | Cas |
|---|---|---|
| FR-001 | `back/catalog` | auteur → 200 ; administrateur sur le sujet d'un autre, même retiré → 200 ; autre inscrit → 403/404 ; auteur d'un sujet retiré → `subject_retired` ; visiteur → 401 ; email non confirmé → 403 |
| FR-002, FR-009, US1-8 | `back/catalog` | PDF, image, XLSX corrompu → `import_unreadable` ; 5 Mo + 1 Ko ou 2 001 lignes → `import_too_large` ; `file` et `text` ensemble ou absents → 422 |
| FR-003, US2-6 | `back/catalog` | trois colonnes → deux premières gardées, notice `extra_columns_ignored` une fois |
| FR-004, US1-4, US1-5, SC-002 | `back/catalog` (unitaire du lecteur) | Excel FR (Windows-1252, `;`, CRLF) ; Google Sheets et LibreOffice (UTF-8, `,`, avec ou sans BOM) ; cellule entre guillemets avec `,`, `;` et retour à la ligne ; virgules dans les phrases avec `;` comme séparateur |
| FR-005, US3-1 à US3-4 | `back/catalog` | texte tabulé collé ; export Anki avec `#separator:tab` et `#html:true` ; export Quizlet ; cellule multiligne copiée de Google Sheets ; texte sans tabulation → `import_single_column` |
| FR-006 | `back/catalog` | XLSX à deux feuilles → première seulement, notice `first_sheet_only` ; formule → valeur ; nombre entier sans `.0` |
| FR-007, US1-6 | `back/catalog` | « Recto ; Verso », « QUESTION ; Réponse », « question ; reponse » → ignorée ; « Recto ; Lima » → importée |
| FR-008, US2-5 | `back/catalog` | lignes vides ignorées ; espaces des bords retirés ; cellule d'espaces = vide |
| FR-010 à FR-012, US2-1 à US2-4, US2-7 | `back/catalog` | numéros de ligne de la source (cellule multiligne comprise) ; `recto_empty`, `verso_empty`, `*_too_long` ; 450 + 80 → `question_limit_exceeded` avec `remaining: 50` ; `can_confirm` faux sur toute erreur ; source vide → `import_empty` |
| FR-013 | `back/catalog` | après un aperçu, le sujet est inchangé |
| FR-014, SC-003 | `back/catalog` | confirmation avec une erreur → `import_has_errors`, aucune question ; limite atteinte entre l'aperçu et la confirmation → rien ; droit perdu entre-temps → rien ; exception au milieu de l'import → transaction annulée, aucune question |
| FR-015 | `back/catalog` | sujet de 3 questions + 50 → positions 4 à 53 dans l'ordre de la source, les 3 premières intactes |
| FR-016, FR-027, SC-006 | `back/catalog` | une question importée se modifie, se déplace et se supprime par lomkit ; `recto_html` de l'aperçu identique à celui enregistré ; `<script>`, `<img onerror>`, `javascript:` → texte ou retirés ; `import_id` absent des réponses lomkit |
| FR-017 | `back/catalog` | même `import_id` deux fois → `201` puis `200`, mêmes questions une seule fois |
| FR-018 | `back/catalog`, `web/Authoring` | `imported` ; toast « 80 questions ajoutées » |
| FR-019, US4-1, US4-3 | `back/learning`, `web/Authoring` | `has_learners` vrai ou faux, auteur qui apprend son sujet compris ; 403 pour un autre inscrit ; avertissement affiché seulement si vrai |
| FR-020, US4-2 | `back/learning` | sujet appris par 2 inscrits, import de 30 → 30 cartes en boîte 1, à réviser aujourd'hui dans le fuseau de chacun ; cartes existantes inchangées |
| FR-021 | `back/catalog` | modèle : XLSX valide, en-tête et 2 exemples, textes des traductions ; le modèle réimporté donne 2 questions sans erreur |
| FR-024, FR-025, SC-007 | `back/catalog` (unitaire de `MarkdownCell`) | gras, italique, listes, code en ligne et délimité, liens ; `# Titre`, `> citation`, `![img](x)`, `<b>`, `---`, code indenté → texte tel quel ; jeu d'essai : `2 * 3 * 4 = 24`, `5*3*2`, `nom_de_variable`, `prix : 5 € *`, `_début sans fin`, `**`, `C:\dossier\*`, `a_b_c` → texte ; `\*` → `*` ; retour à la ligne → `<br>` |
| FR-026 | `back/catalog` | recto égal à une question du sujet (casse, accents, espaces, mise en forme différents) → `duplicate_in_subject` avec `position` ; deux lignes égales → `duplicate_in_source` sur la seconde ; `can_confirm` reste vrai |
| FR-002, FR-012, FR-022, FR-023 | `web/Authoring` | bouton « Importer des questions » sur l'éditeur, caché en lecture seule ; extension ou taille refusées avant l'envoi ; erreurs en `role="alert"` ; liens vers les lignes à corriger ; bouton désactivé tant que `can_confirm` est faux, et hors ligne ; `import_id` envoyé ; retour à l'éditeur après l'import |
| FR-021 | `web/Authoring` | lien du modèle vers `/api/question-import/template` |
| R12 | `web/ApiClient` | `upload(path, file, fields)` ajoute les champs au `FormData` |

## Parcours manuels à 360 px

1. **Fichier** : depuis un brouillon de 3 questions, « Importer des questions », « Fichier »,
   envoyer l'export XLSX d'Excel → aperçu de 50 questions, accents justes → « Importer 50
   questions » → retour à l'éditeur, 53 questions, toast.
2. **CSV Excel FR** : même parcours avec le CSV d'Excel en français (Windows-1252, `;`) → accents
   et colonnes justes.
3. **Coller** : copier 20 lignes de deux colonnes dans Google Sheets, dont une cellule multiligne
   → « Coller » → aperçu sans défilement horizontal → importer.
4. **Erreurs** : fichier avec un verso vide à la ligne 12 et un doublon → le résumé en tête
   renvoie à la ligne 12, le bouton est désactivé, le doublon est signalé sans bloquer ; corriger
   le fichier, le renvoyer → import.
5. **Markdown** : cellule `Le passé de *go* est **went**`, puis `2 * 3 * 4 = 24`, puis `# Titre`
   → rendu attendu dans l'aperçu, identique après l'import dans l'éditeur et en lecture publique.
6. **Sujet appris** : un sujet publié appris par un second compte → avertissement dans l'aperçu
   → importer 5 questions → le second compte les voit en boîte 1 dans « Mes révisions » le jour
   même.
7. **Coupure** : passer hors ligne sur l'aperçu → bouton désactivé avec explication ; couper le
   réseau pendant la confirmation, puis réessayer → les questions n'apparaissent qu'une fois.
8. **Modèle** : télécharger le modèle, l'ouvrir dans LibreOffice et dans Google Sheets, ajouter une
   ligne, l'importer.
