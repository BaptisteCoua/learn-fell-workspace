# Contrat : Rappels de révision (002)

Le back (`back/`) fournit l'API ; le web (`web/`) la consomme. Base :
`https://api.<domaine>/api` (en développement `http://localhost:8090/api`). Les conventions de la
001 s'appliquent : session Sanctum SPA, en-tête `X-XSRF-TOKEN`, erreurs métier
`{code, message}` venues des traductions françaises du back.

## 1. Ressources lomkit

Routes lomkit standard (`search`, `mutate`, `DELETE`, `details`, `actions/<action>`). Accès réservé
à un compte connecté à l'adresse confirmée ; périmètres dans [data-model.md](../data-model.md).

### reminder-settings

- **Champs** : `id`, `email_enabled`, `send_time` (`"HH:MM"`), `activated_at`,
  `proposal_seen_at`, `email_disabled_reason` (`null`, `unsubscribed` ou `bounced`),
  `next_reminder_at`. Comptage calculé : `devices_count`.
- **Lecture** : `search` renvoie une seule ligne, celle du compte connecté.
- **Modification** (`mutate`, opération `update`) : `email_enabled`, `send_time`.
  - `send_time` doit aller de `06:00` à `23:30`, minutes à `00` ou `30`, sinon 422
    `invalid_send_time` (FR-003).
  - Effets côté back :
    - `proposal_seen_at` est rempli s'il était nul (FR-002) ;
    - `activated_at` est rempli au premier canal actif ;
    - réactiver l'email remet `email_disabled_reason` à nul et `email_bounce_count` à 0, et
      incrémente `unsubscribe_version` ;
    - `next_reminder_at` est recalculé.
- **Création, suppression** : refusées (403). La ligne existe dès l'inscription.
- **Action `dismiss-proposal`** (standalone, sans champ) : « Plus tard », qui remplit
  `proposal_seen_at` et n'active rien (FR-002). Réponse 200.

### push-subscriptions

- **Champs** : `id`, `endpoint`, `device_label`, `last_delivered_at`, `created_at`.
  `public_key` et `auth_token` ne sont jamais renvoyés.
- **Lecture** : les appareils du compte connecté, les plus récents d'abord (FR-006).
- **Action `register-device`** (standalone) : enregistre l'appareil courant.
  - Champs : `endpoint` (URL `https`, 500 caractères au plus), `public_key`, `auth_token`,
    `content_encoding` (`aes128gcm` ou `aesgcm`), `device_label` (60 caractères au plus).
  - Un endpoint déjà connu est rattaché au compte connecté (research R11).
  - Effets : `activated_at` est rempli au premier canal actif, `proposal_seen_at` s'il était nul,
    et `next_reminder_at` est recalculé. Réponse 200.
- **Suppression** (`DELETE`, `resources: [id]`) : coupe les notifications de cet appareil
  (FR-006). Si c'était le dernier canal, `next_reminder_at` passe à nul.
- **Création et modification par `mutate`** : refusées. L'action `register-device` est le seul
  point d'entrée.

## 2. Désinscription (hors lomkit)

Sans session : le lien arrive par email (research R7).

| Méthode et chemin | Paramètres | Réponse | Exigences |
|---|---|---|---|
| `POST /reminders/unsubscribe/{id}` | `v`, `signature` en query ; corps `List-Unsubscribe=One-Click` ou vide | 204, email de rappel coupé (`email_disabled_reason = unsubscribed`), même s'il l'était déjà ; 403 `invalid_link` pour une signature invalide, un compte inconnu ou un `v` périmé, avec toujours la même réponse | FR-014, FR-015, FR-016 |

- Limite : 6 appels par minute et par adresse IP.
- Pas de CSRF pour un appel sans cookie (messagerie). La page web, elle, envoie le jeton comme
  pour tout autre appel.

## 3. Pages web

| Chemin | Rôle | Exigences |
|---|---|---|
| `/compte` | + section « Rappels » : email activé ou non, heure (select de 06:00 à 23:30 par 30 minutes), liste des appareils avec « Couper » sur chacun, bouton « Activer sur cet appareil » ou message d'installation iOS ou d'autorisation refusée, mention d'un email désactivé suite à des refus | FR-003 à FR-006, FR-018, US4 scénario 5 |
| dialogue après « Apprendre ce sujet » | cases « Notifications sur cet appareil » et « Email », non cochées ; heure à 19:00 ; boutons « Activer » et « Plus tard » | FR-002, US1 scénarios 1 à 3 |
| `/rappels/desinscription?id&v&signature` | publique ; appelle `POST /reminders/unsubscribe/{id}` au chargement ; affiche « Vous ne recevrez plus les rappels par email » avec un lien vers `/compte`, ou « Ce lien n'est pas valide » | FR-014, FR-016 |
| `/revisions/seance?sujets=1,2` | existante (001) ; lien des rappels, qui passe par `/connexion?redirect=…` sans session | FR-009, FR-010 |

## 4. Email de rappel

- **Expéditeur** : celui de la 001 (`CINQ`).
- **Objet** : « {n} carte à réviser aujourd'hui » ou « {n} cartes à réviser aujourd'hui ».
- **Corps** :
  - « Bonjour {nom affiché}, »
  - « Vous avez {n} cartes à réviser aujourd'hui. Quelques minutes suffisent. »
  - Bouton « Réviser maintenant », qui mène au lien de séance.
  - Pied : « Vous recevez cet email parce que vous avez activé les rappels de révision. » et le
    lien « Ne plus recevoir ces rappels », qui mène à la page de désinscription.
- **En-têtes** :
  - `List-Unsubscribe: <{APP_URL}/api/reminders/unsubscribe/{id}?v=…&signature=…>`
  - `List-Unsubscribe-Post: List-Unsubscribe=One-Click`

## 5. Notification push

Données chiffrées envoyées au service push de chaque appareil, lues par
`web/technical/Pwa/public/sw-push.js` :

```json
{
  "title": "CINQ",
  "body": "12 cartes à réviser aujourd'hui",
  "icon": "/pwa-192x192.png",
  "badge": "/pwa-64x64.png",
  "tag": "review-reminder",
  "data": { "url": "https://app.<domaine>/revisions/seance?sujets=1,2" }
}
```

- Options d'envoi : `TTL` de 14 400 secondes (4 heures), `urgency` `normal`.
- Un clic sur la notification met au premier plan un onglet de CINQ déjà ouvert et le dirige vers
  `data.url`, ou ouvre un nouvel onglet.

## 6. Codes d'erreur métier

| Code | Statut | Message (fr) |
|---|---|---|
| `invalid_send_time` | 422 | Choisissez une heure entre 6 h et 23 h 30, à l'heure pleine ou à la demi-heure. |
| `invalid_link` | 403 | Ce lien n'est pas valide. |
| `invalid_push_subscription` | 422 | Cet appareil n'a pas pu être enregistré pour les notifications. |
