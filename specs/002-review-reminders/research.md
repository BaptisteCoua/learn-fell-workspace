# Recherche : Rappels de révision (002)

Chaque décision indique ce qui a été choisi, pourquoi, et ce qui a été écarté. Les inconnues du
contexte technique du plan sont toutes tranchées ici.

## R1. Couche OSDD

- **Décision** : créer `back/functional/reminders` par `osdd:layer`, qui dépend de
  `functional/users` et de `functional/learning` ; créer `web/functional/Reminders`.
- **Pourquoi** : consentement, planification, envoi et désinscription forment un cycle de vie
  propre. Le mettre dans `learning` mélangerait la révision et la communication ; le mettre dans
  `users` ferait dépendre les comptes de la révision.
- **Écarté** : une couche technique `notifications` générique, alors qu'un seul usage existe
  (principe VII). Elle pourra être extraite quand un deuxième type de notification arrivera.

## R2. Notifications push

- **Décision** : Web Push standard avec clés VAPID, par
  `laravel-notification-channels/webpush` 13.x. Les clés sont générées par
  `sail artisan webpush:vapid` dans `back/.env`. La clé publique est aussi donnée au web par
  `NUXT_PUBLIC_VAPID_PUBLIC_KEY`. Chaque message part avec un TTL de 4 heures, une urgence
  `normal` et le tag `review-reminder`, qui remplace sur l'appareil un rappel précédent non lu.
- **Pourquoi** : c'est le seul mécanisme qui marche à la fois dans les navigateurs et dans la PWA
  installée, iOS compris depuis la version 16.4. Le paquet est compatible avec Laravel 13
  (`illuminate/*` ^13.13) et s'appuie sur `minishlink/web-push` ^11, la référence PHP.
- **Écarté** :
  - Firebase Cloud Messaging : service tiers, SDK web et compte Google, sans gain pour une PWA ;
  - OneSignal et les services équivalents : données des appareils chez un tiers (principe VI) ;
  - servir la clé publique par un endpoint de l'API : un appel de plus avant chaque abonnement.
    Le risque de décalage entre les deux fichiers `.env` est couvert par le guide de validation.

## R3. Qui reçoit la notification

- **Décision** : le destinataire (`Notifiable`) est le modèle `ReminderSetting`, et non `User`. Il
  porte le trait `HasPushSubscriptions` du paquet : les abonnements lui sont rattachés par la
  relation polymorphe du paquet. `routeNotificationForMail()` renvoie l'adresse de son
  utilisateur.
- **Pourquoi** : la couche `users` reste inchangée et ne connaît pas les rappels (principe II).
  Les réglages sont de toute façon lus à chaque envoi.
- **Écarté** : ajouter le trait sur `User`, ce qui ferait dépendre la couche `users` du paquet et
  de la configuration de la couche `reminders`.

## R4. Planification

- **Décision** : chaque ligne de `reminder_settings` porte `next_reminder_at` (UTC, indexée), qui
  est le prochain créneau local converti en UTC.
  - La commande `reminders:dispatch` est planifiée chaque minute, avec `withoutOverlapping()` et
    `onOneServer()`. Elle choisit les lignes où `next_reminder_at <= now()` et au moins un canal
    est actif. Pour chacune, elle place la tâche `SendReviewReminder` en file (`ShouldBeUnique` par
    compte), puis avance `next_reminder_at` au créneau suivant.
  - La tâche vérifie de nouveau l'éligibilité au moment de l'envoi (R5, R6), compte les cartes, les
    envoie et consigne le rappel dans `reminder_sends`.
  - `next_reminder_at` est recalculé quand l'heure, un canal ou le fuseau du compte changent
    (écoute de la mise à jour de `User` sur `timezone`).
- **Créneau suivant** (`NextReminderSlot`) : l'heure choisie aujourd'hui dans le fuseau du compte,
  si elle est encore à venir et qu'aucun rappel n'est déjà consigné pour cette date locale ; sinon
  la même heure le lendemain. La date et l'heure locales sont construites avec Carbon dans le
  fuseau. Une heure qui n'existe pas (passage à l'heure d'été) est avancée à l'heure suivante, et
  une heure ambiguë (passage à l'heure d'hiver) prend la première occurrence.
- **Pourquoi** : la commande de chaque minute lit un index, sans parcourir tous les comptes ni
  calculer les fuseaux en SQL. Avec le scheduler et la file Redis déjà en place, les rappels
  partent dans la minute (SC-002).
- **Unicité** : un index unique `(user_id, local_date)` sur `reminder_sends`. La tâche consigne le
  rappel avant d'envoyer, dans une transaction : un doublon échoue sur la contrainte et n'envoie
  rien (FR-007, edge case « heure changée le même jour »).
- **Écarté** :
  - calculer chaque minute l'heure locale de tous les comptes (coût linéaire) ;
  - une tâche retardée par compte (`delay`) : elle doit être annulée et replanifiée à chaque
    changement de réglage, et elle se perd si la file est vidée.

## R5. Éligibilité

`ReminderEligibility` renvoie les sujets concernés et le nombre de cartes, ou rien. Un rappel ne
part que si toutes les conditions suivantes sont réunies :

1. l'adresse du compte est confirmée (FR-019) ;
2. au moins un canal est actif : email activé ou au moins un appareil (FR-019) ;
3. le compte apprend au moins un sujet (FR-019) ;
4. au moins une carte est due aujourd'hui dans un sujet publié. La règle est celle de la
   `DueCardsInstruction` de la 001 : `next_review_on <= aujourd'hui` dans le fuseau du compte,
   sujet au statut `published`. Elle est extraite dans une requête réutilisable de `learning`
   (FR-008, SC-006) ;
5. l'espacement autorise un rappel aujourd'hui (R6).

- Le nombre annoncé et les sujets du lien sont ceux de cette requête au moment de l'envoi. La
  séance ouverte depuis le lien utilise la même règle, donc le même nombre (SC-006).

## R6. Espacement

- **Décision** : `ReminderSpacing` compte les jours d'inactivité : l'écart, en dates locales,
  entre aujourd'hui et la date de la dernière réponse en séance (`review_answers.answered_at`),
  ou la date d'activation si le compte n'a jamais répondu. L'écart minimal entre deux rappels en
  découle :

  | Jours d'inactivité | Écart minimal depuis le dernier rappel |
  |---|---|
  | 0 à 7 | 1 jour (chaque jour) |
  | 8 à 21 | 2 jours (un jour sur deux) |
  | 22 et plus | 7 jours (chaque semaine) |

  Un rappel part si aucun rappel n'a encore été envoyé, ou si l'écart en dates locales depuis le
  dernier rappel atteint ce minimum.
- **Pourquoi** : la règle se déduit de données déjà stockées. Une réponse la remet
  automatiquement à zéro (FR-013), sans compteur à tenir à jour. Les paliers sont des constantes
  d'une seule classe, faciles à ajuster après mesure (SC-007).
- **Tests** : chaque jour d'inactivité, du 1er au 60e, est vérifié dans un test unitaire, ainsi que
  la reprise après une réponse (SC-005).

## R7. Désinscription par email

- **Décision** : `UnsubscribeLink` produit une URL signée sans expiration
  (`URL::signedRoute`, relative puis réécrite vers le web, comme `EmailVerificationLink`). Elle
  porte l'identifiant du compte et un `v` égal au nombre de réactivations de l'email. Deux
  adresses en découlent :
  - dans le corps de l'email : `{FRONTEND_URL}/rappels/desinscription?id=…&v=…&signature=…`. La
    page web appelle l'API et affiche la confirmation ;
  - dans l'en-tête `List-Unsubscribe` : l'URL de l'API
    (`{APP_URL}/api/reminders/unsubscribe/{id}?v=…&signature=…`), accompagnée de
    `List-Unsubscribe-Post: List-Unsubscribe=One-Click` (RFC 8058). Les messageries l'appellent
    en `POST` sans cookie.
- **Route** : `POST api/reminders/unsubscribe/{id}` dans le groupe `api`, avec la limite
  `throttle:6,1`. Sans cookie de session ni en-tête `Origin` du web, Sanctum ne demande pas de
  jeton CSRF : l'appel d'une messagerie passe. L'appel de la page web, lui, envoie le jeton
  habituel.
- **Réponses** : la signature est vérifiée avant toute lecture en base. Une signature invalide, un
  compte inexistant ou un `v` périmé renvoient tous la même réponse `403 invalid_link` (FR-016).
  Un lien valide désactive l'email, note `email_disabled_reason = unsubscribed` et renvoie 204,
  même si l'email était déjà coupé.
- **Pourquoi `v`** : un lien reçu avant une réactivation ne doit pas couper les rappels
  réactivés depuis, par exemple dans un vieil email transféré.
- **Écarté** : un jeton aléatoire stocké par compte (une colonne et une rotation de plus pour le
  même résultat) ; un lien qui expire (les messageries appellent parfois l'en-tête des semaines
  plus tard).

## R8. Échecs d'envoi

- **Push** : le canal du paquet supprime l'abonnement quand le service push répond 404 ou 410
  (abonnement expiré ou révoqué, FR-017), sans lever d'erreur. Les autres appareils et l'email
  continuent. Un écouteur de `NotificationSent` met à jour `last_delivered_at` de l'appareil.
- **Email** : chaque canal part dans son propre appel, pour qu'un échec de l'un n'empêche pas
  l'autre. Le compteur `email_bounce_count` suit les refus :
  - un refus définitif, c'est-à-dire un code SMTP 5xx à l'envoi (`TransportException`), incrémente
    le compteur ;
  - un envoi accepté remet le compteur à zéro ;
  - au 3e refus consécutif, l'email est désactivé avec `email_disabled_reason = bounced` (FR-018),
    et la page Compte l'indique.
- **Limite connue** : un refus signalé plus tard par un message de non-remise (DSN) n'est pas
  capté sans webhook du fournisseur d'envoi. Il sera branché quand ce fournisseur sera choisi pour
  la production. L'interface `RecordsEmailBounce` est prévue pour cela.
- **Écarté** : relancer un canal en échec définitif ; les échecs temporaires (4xx, réseau) sont
  réessayés par la file, avec 3 essais.

## R9. Contenu

- **Email** : `MailMessage` dans le style des notifications de la 001, en français et au
  vouvoiement. Il contient :
  - le sujet « {n} cartes à réviser aujourd'hui » ;
  - la formule « Bonjour {nom affiché} » ;
  - un bouton « Réviser maintenant » ;
  - le lien « Ne plus recevoir ces rappels ».

  Les textes sont dans `back/functional/reminders/lang/fr/notifications.php`, avec leur pluriel.
- **Push** : titre « CINQ », texte « {n} cartes à réviser aujourd'hui », icône de la PWA, et dans
  `data.url` le lien de la séance. Rien d'autre, car la notification peut s'afficher sur un écran
  verrouillé (FR-011).
- **Lien de séance** : `{FRONTEND_URL}/revisions/seance?sujets=1,2`, déjà géré par la 001. Le
  middleware `auth` du web renvoie vers `/connexion?redirect=…`, puis vers la séance après la
  connexion (FR-010).

## R10. Service worker

- **Décision** : garder la stratégie `generateSW` de `@vite-pwa/nuxt` et ajouter
  `workbox.importScripts: ['sw-push.js']`. Le fichier `web/technical/Pwa/public/sw-push.js`
  écoute deux événements :
  - `push` : il affiche le titre, le texte, le tag et l'URL reçus ;
  - `notificationclick` : il met au premier plan un onglet de CINQ déjà ouvert et le dirige vers
    l'URL, ou en ouvre un nouveau.
- **Pourquoi** : la 001 a réglé la mise en cache et la mise à jour avec `generateSW`. Passer en
  `injectManifest` obligerait à réécrire tout le service worker pour deux écouteurs.
- **Développement** : le service worker n'existe qu'en build de production. Le guide de validation
  teste donc le push sur `pnpm build`, les autres écrans en `pnpm dev`.

## R11. Abonnement d'un appareil (web)

- **Décision** : `usePushDevice` réunit tout ce qui touche à l'abonnement d'un appareil.
  - Support : il vérifie que `serviceWorker`, `PushManager` et `Notification` existent. Sur iOS
    ou iPadOS hors PWA installée (`usePwaInstall` de la 001), il affiche le message d'installation
    (FR-005).
  - Activation : `Notification.requestPermission()` doit répondre `granted`, sinon il affiche le
    message d'autorisation et n'enregistre rien (FR-004). Il appelle ensuite
    `registration.pushManager.subscribe({ userVisibleOnly: true, applicationServerKey })`, puis
    l'action `register-device`.
  - Appareil courant : il est reconnu en comparant l'`endpoint` de l'abonnement du navigateur à
    ceux de la liste.
  - Coupure : couper l'appareil courant appelle `subscription.unsubscribe()` puis supprime la
    ligne. Couper un autre appareil supprime seulement la ligne ; le prochain envoi n'y part plus.
- **Nom de l'appareil** : il est déduit côté web du navigateur et du système
  (`navigator.userAgentData` quand il existe, sinon l'agent utilisateur), par exemple « Chrome sur
  Android ». Il fait 60 caractères au plus et n'est jamais saisi (hypothèse de la spec).
- **Même appareil, autre compte** : `register-device` s'appuie sur
  `updatePushSubscription($endpoint, …)`, qui rattache l'endpoint au compte connecté. Un
  navigateur ne reçoit donc jamais les rappels de deux comptes.

## R12. Proposition après le premier apprentissage

- **Décision** : la ligne `reminder_settings` est créée à l'inscription par un écouteur de la
  création de `User`, et une migration la crée pour les comptes existants. Après chaque « Apprendre ce
  sujet » réussi, `useLearningPanel` appelle `useReminderProposal().offer()`. Le dialogue ne
  s'ouvre que si `proposal_seen_at` est nul et qu'aucun canal n'est actif (FR-002).
  - « Activer » enregistre les canaux cochés et l'heure.
  - « Plus tard » appelle l'action `dismiss-proposal`.
  - Dans les deux cas, `proposal_seen_at` est rempli et le dialogue ne revient plus.
- **Pourquoi** : c'est le back qui garde en mémoire que la proposition a été vue, donc tous les
  appareils du compte le savent.
- **Écarté** : garder cette information dans le `localStorage` (la proposition reviendrait sur
  chaque appareil).

## R13. Suivi et rétention

- `reminder_sends` garde la date locale, le nombre de cartes, les canaux effectivement servis et
  l'heure d'envoi (FR-020). Il sert à l'unicité quotidienne, à l'espacement et à la mesure de
  SC-007 : la part des rappels suivis d'une réponse le même jour local se calcule par une requête
  croisée avec `review_answers`.
- `ReminderSend` est `Prunable` au-delà de 90 jours, car seul le dernier rappel sert à
  l'espacement. `model:prune` est déjà planifié chaque jour.
- La suppression d'un compte supprime ses réglages, ses appareils et son journal par un écouteur
  de `User` `deleting`, sans cascade en base (convention de la 001).
