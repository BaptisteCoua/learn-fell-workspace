# Data Model: Révision hors ligne

Décisions dans [research.md](research.md), contrat dans [contracts/api.md](contracts/api.md).

## Back : `review_answers` (couche `back/functional/learning`)

Table existante (feature 001). Trois colonnes de plus :

| Colonne | Type | Règle |
|---|---|---|
| `answer_id` | uuid, unique, non nul | identifiant créé par l'appareil ; rend le renvoi sans effet (FR-016) |
| `due_on` | date, non nul | l'échéance à laquelle la réponse répondait (FR-013) |
| `status` | string(16), non nul | `applied` ou `discarded` ; enum PHP `AnswerStatus`, pas d'enum en base |

Colonnes existantes dont le sens change :

- `answered_at` : l'heure où la réponse a été donnée sur l'appareil, ramenée à l'heure de
  réception si elle est dans le futur (FR-011, FR-015). `created_at` reste l'heure de réception.
- `from_box`, `to_box` : calculés au rejeu ; pour une réponse `discarded`, `to_box` = `from_box`.

Index : `(card_progress_id, answered_at)` pour le rejeu.

Migration des lignes existantes : `answer_id` reçoit un UUID par ligne, `due_on` la date de
`answered_at` dans le fuseau du compte, `status` = `applied`, puis les trois colonnes passent non
nulles. Aucune valeur par défaut en base.

### Règles du rejeu (R3, R4)

Entrée : une réponse R (`answer_id`, `card_progress_id`, `known`, `answered_at`, `due_on`) d'un
compte de fuseau `tz`. `jour(t)` = la date de `t` dans `tz`.

1. `answer_id` existe → aucun effet.
2. Carte absente, d'un autre compte, ou de sujet non publié → aucun effet, aucune ligne.
3. `answered_at` > maintenant → `answered_at` = maintenant.
4. `L` = réponses `applied` de la carte avec `answered_at` > R.`answered_at`, triées.
   État de départ `S` = (`box`, `next_review_on`) de la carte si `L` est vide, sinon
   (`from_box`, `due_on`) du premier élément de `L`.
5. Pour chaque réponse X de `[R] + L`, dans l'ordre de `answered_at` :
   - si `S.next_review_on` = X.`due_on` et `S.next_review_on` ≤ `jour(X.answered_at)` →
     `applied`, `from_box` = `S.box`, `to_box` = `arrivalBox(S.box, known)`, et
     `S` = (`to_box`, `jour(X.answered_at)` + intervalle de `to_box`) ;
   - sinon, si X est R, que `S.next_review_on` = R.`due_on` et que
     `S.next_review_on` > `jour(R.answered_at)` (horloge en retard) → R est remise en fin de
     rejeu avec `answered_at` = maintenant, et y est `applied` si la condition ci-dessus tient
     alors, `discarded` sinon ;
   - sinon → `discarded`.
6. La carte prend `S` ; `last_answered_at` = `answered_at` de la dernière réponse `applied`.

Toute l'opération se fait dans une transaction, la carte verrouillée (`lockForUpdate`).

### Table des cas Leitner (inchangée, partagée par les tests back et web)

| Boîte | Réponse | Boîte d'arrivée | Prochaine révision |
|---|---|---|---|
| 1 à 4 | sait | boîte + 1 | jour de réponse + 2, 4, 8, 16 jours |
| 5 | sait | 5 | jour de réponse + 16 jours |
| 1 à 5 | ne sait pas | 1 | jour de réponse + 1 jour |

## Back : `GET /api/user` (couche `back/functional/users`)

Ajoute `timezone` à la réponse. Aucun changement de schéma.

## Web : base IndexedDB `cinq-offline-review` (couche `web/functional/Learning`)

### Magasin `pack` (une seule entrée, clé `current`)

| Champ | Type | Source |
|---|---|---|
| `user_id` | number | `ISessionUser.id` ; propriétaire de tout le contenu de la base |
| `timezone` | string | `ISessionUser.timezone` |
| `updated_at` | string ISO | heure de l'appareil à la mise à jour |
| `learnings` | tableau | `learnings/search` : `id`, `subject_id`, titre du sujet, `box_1_count` à `box_5_count` |
| `cards` | tableau | instruction `upcoming` : `id`, `subject_id`, `question_id`, `position`, `box`, `next_review_on`, `recto_html`, `verso_html`, `image_alts[]` |

Règles :

- Remplacé d'un bloc, seulement quand le magasin `answers` est vide (FR-017).
- À chaque réponse locale, la carte prend sa nouvelle boîte et sa nouvelle échéance, et le
  `learning` de son sujet passe `box_{from}` −1, `box_{to}` +1.
- Cartes à réviser un jour `J` : `next_review_on` ≤ `J`, triées par `next_review_on`,
  `subject_id`, `position` (FR-006). Hors ligne, `J` = `jour(maintenant)` dans `timezone` ; si
  `J` dépasse `updated_at` + 7 jours, le message de reconnexion s'affiche (US4-5).

### Magasin `answers` (clé `answer_id`, index `answered_at`)

| Champ | Type | Règle |
|---|---|---|
| `answer_id` | string | `crypto.randomUUID()` |
| `card_progress_id` | number | |
| `known` | boolean | |
| `answered_at` | string ISO avec décalage | heure de l'appareil au moment de la réponse |
| `due_on` | string `YYYY-MM-DD` | `next_review_on` de la carte au moment de la réponse |

Cycle de vie : écrit dès la réponse (FR-008) → envoyé dans l'ordre de `answered_at` → supprimé
après un `2xx` du serveur. Un `401` ou une erreur réseau arrête l'envoi et garde la file.

### Effacement

`discard()` supprime la base entière. Il est appelé à la déconnexion choisie, à la demande de
suppression de compte, et à la connexion d'un compte autre que `pack.user_id` (FR-004).

### Stockage indisponible

Si la base ne s'ouvre pas, la file vit en mémoire le temps de l'onglet et il n'y a pas de
paquet ; « Mes révisions » l'indique (FR-019).
