# Contrat d'API : Révision hors ligne (006)

Complète le contrat de la 001 ([specs/001-learning-content/contracts/api.md](../../001-learning-content/contracts/api.md)).
Règles de rejeu dans [data-model.md](../data-model.md).

## `GET /api/user` (modifié)

Ajoute `timezone` (nom IANA du fuseau du compte) :

```json
{ "id": 7, "display_name": "Camille", "email": "camille@exemple.fr", "permissions": [], "timezone": "Europe/Paris" }
```

## `card-progress` : instruction `upcoming` (nouvelle)

`POST /api/card-progress/search`

```json
{ "search": {
    "instructions": [{ "name": "upcoming" }],
    "includes": [{ "relation": "question" }, { "relation": "question.images" }, { "relation": "subject" }],
    "limit": 100, "page": 1 } }
```

- Cartes de l'utilisateur connecté, de sujets publiés, dont `next_review_on` tombe au plus tard
  aujourd'hui + 7 jours dans le fuseau du compte (FR-001).
- Tri : `next_review_on`, `subject_id`, `position` de la question, comme `due`.
- Mêmes champs et relations que `due`. Le web n'utilise que `alt` des images (FR-002).
- Pagination lomkit habituelle ; le web lit toutes les pages.

## `card-progress` : action `answer` (modifiée)

`POST /api/card-progress/actions/answer`

```json
{ "fields": [
    { "name": "answer_id", "value": "6f1c2b3e-8d1a-4c7e-9a52-0d4f0b7c1e21" },
    { "name": "card_progress_id", "value": 42 },
    { "name": "known", "value": true },
    { "name": "answered_at", "value": "2026-10-05T08:12:31+02:00" },
    { "name": "due_on", "value": "2026-10-05" } ] }
```

| Champ | Règle |
|---|---|
| `answer_id` | facultatif, UUID ; généré par le serveur s'il manque |
| `card_progress_id` | requis, entier |
| `known` | requis, booléen |
| `answered_at` | facultatif, date ISO 8601 ; maintenant s'il manque, ramené à maintenant s'il est dans le futur |
| `due_on` | facultatif, date `Y-m-d` ; échéance courante de la carte s'il manque |

Réponses :

- `200` dans tous les cas métier : réponse appliquée, écartée (première réponse déjà là, horloge
  incohérente), déjà reçue (`answer_id` connu), ou ignorée (carte inexistante, d'un autre compte,
  de sujet non publié). Le corps reste celui d'une action lomkit et n'est pas lu par le web, qui
  remet son paquet à jour après l'envoi (FR-017).
- `422` si un champ est mal formé (UUID, date, booléen). Le web ne produit pas ce cas ; une
  réponse en `422` est retirée de la file pour ne pas la bloquer.
- `401` si la session est fermée : la réponse reste dans la file.
- Le code `card_not_due` (409) est retiré.

## Fichiers servis par le PWA (web)

| Chemin | Rôle |
|---|---|
| `/revisions` (précaché, coquille SPA) | sert `/revisions` et `/revisions/seance` sans réseau |
| `/hors-ligne` | inchangé ; mentionne que la révision hors ligne est possible après une première ouverture en ligne |
