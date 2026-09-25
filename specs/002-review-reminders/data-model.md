# Modèle de données : Rappels de révision (002)

Base PostgreSQL du back. Les tables appartiennent à la couche `functional/reminders`, et les
migrations vivent dans `back/functional/reminders/database/migrations/`. Comme en 001, aucune clé
étrangère n'est en cascade : les suppressions liées sont faites par l'application (research R13).

## reminder_settings

Une ligne par compte, créée à l'inscription ; une migration la crée pour les comptes existants
(research R12).

| Colonne | Type | Règles |
|---|---|---|
| id | bigint PK | |
| user_id | bigint FK users | unique |
| email_enabled | boolean | défaut `false` (FR-001) |
| send_time | time | défaut `19:00` ; de `06:00` à `23:30`, minutes à `00` ou `30` (FR-003) |
| activated_at | timestamp null | première fois qu'un canal devient actif ; base de l'espacement sans réponse (FR-012) |
| proposal_seen_at | timestamp null | rempli par « Activer » ou « Plus tard » dans la proposition (FR-002) |
| email_disabled_reason | varchar(20) null | `unsubscribed` ou `bounced` ; remis à nul quand l'utilisateur réactive l'email |
| email_bounce_count | smallint | défaut 0 ; refus définitifs consécutifs (FR-018) |
| unsubscribe_version | integer | défaut 0 ; +1 à chaque réactivation de l'email, ce qui périme les anciens liens (research R7) |
| next_reminder_at | timestamp null | prochain créneau en UTC ; nul si aucun canal n'est actif ; indexé (research R4) |
| created_at, updated_at | timestamps | |

- **Canal push actif** : le compte a au moins une ligne dans `push_subscriptions`. Il n'y a pas de
  booléen : l'appareil est le réglage (FR-006).
- **Destinataire des notifications** : le modèle `ReminderSetting` est `Notifiable` et porte
  `HasPushSubscriptions` (research R3).
- **Recalcul de `next_reminder_at`** quand l'un des éléments suivants change : `send_time`,
  `email_enabled`, le premier ou le dernier appareil, le fuseau du compte. Il est aussi avancé à
  chaque passage de la commande (research R4).

## push_subscriptions

Table du paquet `laravel-notification-channels/webpush`, publiée dans la couche, avec deux colonnes
de plus. Le modèle `Functional\Reminders\Models\PushSubscription` étend celui du paquet et il est
déclaré dans `webpush.model`.

| Colonne | Type | Règles |
|---|---|---|
| id | bigint PK | |
| subscribable_type, subscribable_id | morph | le `ReminderSetting` du compte |
| endpoint | varchar(500) | unique ; URL du service push de l'appareil |
| public_key | varchar null | clé `p256dh` |
| auth_token | varchar null | clé `auth` |
| content_encoding | varchar null | `aes128gcm` par défaut |
| device_label | varchar(60) | « Chrome sur Android », déduit côté web (research R11) |
| last_delivered_at | timestamp null | dernière réception acceptée par le service push |
| created_at, updated_at | timestamps | `created_at` = date d'enregistrement |

- Supprimé par l'utilisateur (FR-006) ou par le canal au premier 404 ou 410 (FR-017).
- Un `endpoint` déjà connu est rattaché au compte qui l'enregistre (research R11).

## reminder_sends

Le journal des rappels (FR-020).

| Colonne | Type | Règles |
|---|---|---|
| id | bigint PK | |
| user_id | bigint FK users | |
| local_date | date | date du rappel dans le fuseau du compte au moment de l'envoi |
| cards_count | integer | nombre annoncé, supérieur ou égal à 1 |
| subject_ids | jsonb | sujets du lien de séance |
| channels | jsonb | canaux servis : `["mail"]`, `["webpush"]` ou `["mail","webpush"]` |
| sent_at | timestamp | |

- **Unique** : `(user_id, local_date)`, soit au plus un rappel par jour et par compte (FR-007).
- **Prunable** : au-delà de 90 jours (research R13).
- La ligne est écrite avant l'envoi. Si aucun canal n'est finalement servi (tous en échec
  définitif), elle reste et compte comme le rappel du jour : on ne relance pas le même jour.

## Règles du domaine

Trois classes pures dans `back/functional/reminders/src/Domain/`, couvertes à 100 % par des tests
unitaires.

### NextReminderSlot

Entrée : `send_time`, fuseau du compte, instant présent, dernière `local_date` consignée.

- Si `send_time` est encore à venir aujourd'hui dans le fuseau et qu'aucun rappel n'est consigné
  pour aujourd'hui : aujourd'hui à `send_time`.
- Sinon : demain à `send_time`.
- Une heure locale inexistante (passage à l'heure d'été) est avancée à l'heure suivante. Une heure
  ambiguë (passage à l'heure d'hiver) prend la première occurrence.
- Le résultat est rendu en UTC.

### ReminderSpacing

Entrée : aujourd'hui (date locale), date locale de la dernière réponse ou, à défaut, de
`activated_at`, date locale du dernier rappel.

| Jours d'inactivité | Écart minimal depuis le dernier rappel |
|---|---|
| 0 à 7 | 1 jour |
| 8 à 21 | 2 jours |
| 22 et plus | 7 jours |

Autorisé si aucun rappel n'a encore été envoyé, ou si `aujourd'hui − dernier rappel` atteint
l'écart minimal (FR-012, FR-013).

### ReminderEligibility

Renvoie `{cards_count, subject_ids}` ou `null` si l'une des conditions de research R5 manque :

1. adresse confirmée ;
2. au moins un canal actif ;
3. au moins un apprentissage ;
4. au moins une carte due aujourd'hui dans un sujet publié ;
5. espacement autorisé.

## États d'un canal email

```text
désactivé (défaut) ──activer──▶ activé ──lien de désinscription──▶ désactivé (unsubscribed)
      ▲                           │  └──3 refus consécutifs──────▶ désactivé (bounced)
      └────────désactiver─────────┘
désactivé (unsubscribed | bounced) ──réactiver──▶ activé   (unsubscribe_version +1, compteur à 0)
```

## Accès (`lomkit/laravel-access-control`)

| Ressource | Périmètre | Lecture | Écriture |
|---|---|---|---|
| reminder-settings | ligne du compte connecté | propriétaire | modification par le propriétaire ; ni création ni suppression par l'API |
| push-subscriptions | abonnements du `ReminderSetting` du compte connecté | propriétaire | création par l'action `register-device`, suppression par le propriétaire |

Hors périmètre, une ligne est absente des résultats et répond 404 en accès direct, comme en 001.
Toutes les routes demandent une adresse confirmée (middleware `verified`).
