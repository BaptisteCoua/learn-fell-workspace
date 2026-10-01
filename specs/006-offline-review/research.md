# Research: Révision hors ligne

Décisions techniques de la feature 006, avec leurs raisons et les alternatives écartées. Les
chemins renvoient à l'état du code au 2026-10-01.

## R1. Toute réponse passe par la file de l'appareil, en ligne comme hors ligne

**Decision**: le web calcule lui-même la nouvelle boîte et la date de retour avec les règles
Leitner (portage de `back/functional/learning/src/Domain/LeitnerSchedule.php` dans
`web/functional/Learning/app/utils/leitner.ts`), écrit la réponse dans la file de l'appareil, puis
vide la file vers le serveur si le réseau est là. Il n'y a plus de chemin « en ligne » séparé : la
séance ne relit plus la carte après l'action (`useReviewSession.ts:79-91`).

**Rationale**: une coupure en pleine séance ne doit rien changer (edge case) ; un seul chemin
l'assure. L'affichage ne dépend plus du réseau, ce qui rend SC-003 vérifiable : le web et le back
appliquent la même table de cas, testée des deux côtés (SC-005).

**Alternatives considered**: garder l'envoi direct en ligne et ne mettre en file qu'hors ligne
(deux chemins, et une réponse perdue si la coupure tombe pendant l'appel) ; laisser le serveur seul
calculer la boîte (impossible hors ligne).

## R2. Une action `answer` élargie, envoyée réponse par réponse

**Decision**: l'action lomkit existante `POST /api/card-progress/actions/answer` reçoit trois
champs de plus : `answer_id` (UUID créé par l'appareil), `answered_at` (date et heure ISO 8601 avec
décalage) et `due_on` (l'échéance que l'appareil voyait). Les trois sont facultatifs pour que le web
actuel continue de fonctionner entre la fusion du back et celle du web : à défaut, `answer_id` est
généré, `answered_at` vaut maintenant et `due_on` l'échéance courante de la carte. La file se vide
dans l'ordre de `answered_at`, une requête à la fois ; une réponse ne quitte la file qu'après un
`2xx`.

**Rationale**: la constitution (III) impose lomkit ; l'action existe déjà. L'envoi un par un rend
la reprise triviale (FR-016) : tout ce qui n'a pas été acquitté repart, et l'unicité de
`answer_id` rend le renvoi sans effet. Une semaine de révisions représente quelques dizaines de
requêtes, bien sous la minute de SC-002.

**Alternatives considered**: une action par lot (un seul aller-retour, mais un acquittement
partiel à gérer et une transaction longue sur plusieurs cartes) ; un endpoint de synchronisation
hors lomkit (contraire au principe III).

## R3. La première réponse compte : rejouer l'historique de la carte

**Decision**: chaque réponse enregistrée garde l'échéance à laquelle elle répondait (`due_on`) et
un statut `applied` ou `discarded`. À l'arrivée d'une réponse R sur une carte (verrou de ligne,
comme aujourd'hui) :

1. `answer_id` déjà connu → rien (FR-016).
2. Si des réponses `applied` de cette carte ont un `answered_at` postérieur à R, on revient à
   l'état d'avant la plus ancienne d'entre elles (`from_box` et `due_on` de celle-ci).
3. On rejoue, dans l'ordre de `answered_at`, R puis ces réponses postérieures. Chacune est
   `applied` si l'échéance de l'état courant est la sienne (`due_on`) et tombe au plus tard le jour
   de sa réponse dans le fuseau du compte ; sinon elle est `discarded`.
4. L'état final est écrit sur la carte (`box`, `next_review_on`, `last_answered_at`).

**Rationale**: FR-012 et FR-013 deviennent une seule règle : l'ordre de la carte est celui des
réponses données, et une échéance n'accepte qu'une réponse. Le scénario de la US3 se déroule
ainsi : la réponse de 12 h (boîte 2 → 3) est défaite, celle de 8 h envoie la carte en boîte 1 pour
le mercredi, et celle de 12 h, qui répondait à l'échéance du mardi, est `discarded`. Comparer
`due_on` évite qu'une réponse en double soit appliquée à l'échéance suivante parce qu'elle arrive
tard. Seules les réponses d'une même carte interagissent, d'où un rejeu borné à cette carte.

**Alternatives considered**: refuser la réponse arrivée en second (contredit FR-013 quand la plus
ancienne arrive en dernier) ; ordonner globalement toutes les réponses du compte (aucune règle ne
lie deux cartes entre elles).

## R4. Dates incohérentes et cartes disparues

**Decision**:

- `answered_at` dans le futur est ramené à l'heure de réception (FR-015), puis traité comme en R3.
- Une réponse dont l'échéance n'est pas encore atteinte à son `answered_at` (horloge en retard)
  est rejouée à l'heure de réception si sa carte est due à ce moment pour cette même échéance ;
  sinon elle est `discarded` (FR-015).
- Carte inexistante, d'un autre compte, ou de sujet non publié (brouillon, retiré, retenu) : la
  réponse est ignorée, sans ligne enregistrée, et l'action répond `200` comme pour une réponse
  appliquée (FR-014). Le code `card_not_due` (409) disparaît de l'action.

**Rationale**: aucune réponse ne doit afficher d'erreur (US3) ; répondre pareil pour une carte d'un
autre compte et une carte inexistante ne révèle rien. Une réponse ignorée ne s'attache à aucune
carte (la clé étrangère l'interdit) ; renvoyée, elle est de nouveau ignorée, donc l'idempotence
tient sans ligne.

**Alternatives considered**: garder le 409 pour les réponses sans `answer_id` (deux sémantiques
pour une même action) ; une tolérance de quelques minutes sur l'horloge (inutile : ramener à
maintenant donne le même résultat).

## R5. Le paquet hors ligne : une instruction `upcoming` et la liste des apprentissages

**Decision**: une instruction lomkit `upcoming` sur `card-progress`, sans champ, renvoie les cartes
du compte de sujets publiés dont `next_review_on` tombe au plus tard 7 jours après aujourd'hui dans
son fuseau, triées comme `due`. Le web la lit avec `question`, `question.images` (pour `alt`
seulement) et `subject`, page par page (limite 100). Il garde aussi le résultat de
`learnings/search` (sujets, titres, comptes par boîte). Le paquet est remplacé d'un bloc, dans une
transaction IndexedDB, et seulement quand la file est vide (FR-017), sinon il écraserait les boîtes
calculées localement. L'horizon de 7 jours est une constante de l'instruction.

**Rationale**: `due` ne voit qu'aujourd'hui ; une instruction sœur réutilise `DueCardsQuery` avec
une autre borne. Les comptes par boîte couvrent toutes les cartes d'un sujet, pas seulement celles
des 7 jours : ils viennent donc de `learnings`, ajustés localement à chaque réponse
(`from_box` −1, `to_box` +1).

**Alternatives considered**: une route `GET /api/learning/offline-pack` (hors lomkit, alors que
c'est une lecture de ressource) ; recompter les boîtes à partir des seules cartes gardées (faux
dès qu'un sujet a des cartes au-delà de 7 jours) ; un paramètre `days` (une option sans second
usage, principe VII).

## R6. Le fuseau du compte vient de `GET /api/user`

**Decision**: `CurrentUserController` (`back/functional/users`) ajoute `timezone` à sa réponse ;
`ISessionUser` (`web/technical/ApiClient/app/stores/useSessionStore.ts`) le porte, et le paquet en
garde une copie. Les jours se calculent par `Intl.DateTimeFormat('en-CA', { timeZone })` sur
l'heure de l'appareil (FR-007), en dates `YYYY-MM-DD` sans heure.

**Rationale**: le fuseau du compte n'est aujourd'hui connu que du back. Il est presque toujours
celui de l'appareil (envoyé à chaque connexion), mais un voyage hors ligne le décalerait.

**Alternatives considered**: le fuseau de l'appareil (FR-007 dit « fuseau du compte ») ; le
renvoyer dans chaque carte (répétitif).

## R7. Stockage : IndexedDB par `idb`

**Decision**: une base `cinq-offline-review` avec deux magasins : `pack` (une entrée : compte,
date de mise à jour, fuseau, apprentissages, cartes) et `answers` (clé `answer_id`, index
`answered_at`). Accès par `idb` 8 (dépendance directe, version stable la plus récente), tests par
`fake-indexeddb` 6 en dépendance de développement. Au premier enregistrement, l'app demande
`navigator.storage.persist()`. Si la base ne s'ouvre pas (navigation privée, quota, refus),
l'app passe à une file en mémoire et un paquet absent : la révision en ligne marche comme avant et
« Mes révisions » affiche l'indisponibilité (FR-019).

**Rationale**: seul IndexedDB survit à la fermeture et au redémarrage avec assez de place
(FR-008) ; `localStorage` est synchrone et limité à quelques Mo. `idb` est une fine surcouche à
promesses (≈ 1 ko), déjà présente de façon transitive via Workbox. La perte par le navigateur
(Safari, 7 jours sans visite) n'est pas détectable de façon fiable, puisque tout le stockage du
site part ensemble : aucun message n'est prévu pour ce cas.

**Alternatives considered**: Dexie (une couche de requêtes inutile pour deux magasins) ;
`idb-keyval` (pas d'index ni de transaction sur plusieurs clés) ; Workbox Background Sync (rejoue
des requêtes HTTP brutes dans le service worker, hors des modèles raom exigés par le principe III,
et indisponible sur Safari).

## R8. Ouvrir « Mes révisions » sans réseau

**Decision**:

- `routeRules` du layer `Learning` : `/revisions` et `/revisions/**` en `ssr: false`, avec
  `/revisions` prérendu (coquille SPA). Le précache ajoute `revisions/index.html`, et une règle de
  navigation placée avant la règle générale sert, pour `/revisions*`, le réseau puis cette coquille
  en repli (`NetworkOnly` + `precacheFallback`, comme `/hors-ligne`). Le routeur Vue résout
  ensuite `/revisions/seance` côté client.
- `useSessionStore.fetchUser()` distingue une erreur réseau (aucune réponse) d'un `401` : sur
  erreur réseau, `isUnreachable` passe à `true` et l'utilisateur reste inconnu. Le middleware
  `auth` laisse passer une page qui déclare `meta.availableOffline` quand `isUnreachable` est vrai ;
  la page s'appuie alors sur le paquet, qui appartient au dernier compte connecté.

**Rationale**: toute navigation est aujourd'hui rendue par le serveur et passe à `/hors-ligne`
sans réseau (`web/technical/Pwa/nuxt.config.ts:48-69`). Ces deux pages sont derrière la connexion
et sans enjeu de référencement ; les rendre côté client ne coûte qu'un affichage sans pré-rendu. Un
appareil ne garde qu'un compte à la fois (R9), donc le paquet présent est le bon hors ligne.

**Alternatives considered**: un `navigateFallback` global vers une coquille (toutes les pages
deviendraient des coquilles hors ligne, alors que seule la révision l'est) ; garder le rendu
serveur et mettre en cache le HTML rendu (un HTML personnel en cache, périmé dès la réponse
suivante). À vérifier en tâche : que le prérendu d'une route `ssr: false` produit bien la coquille
précachée ; à défaut, une page coquille dédiée prérendue joue le même rôle.

## R9. Un seul compte par appareil, effacé à la déconnexion

**Decision**: la couche `Learning` expose `useOfflineReview()` : `pendingCount`, `flush()` et
`discard()`. La couche `Account` l'utilise :

- `logOut()` (`useAccountMenu.ts:35-39`) tente `flush()` ; s'il reste des réponses, il ouvre
  `ConfirmDialog` avec « {n} réponses ne sont pas encore envoyées » et deux choix, « Attendre le
  réseau » et « Me déconnecter quand même » ; puis `discard()` et déconnexion (FR-018, FR-004).
- `requestAccountDeletion()` appelle `discard()` après la fermeture de session (edge case 004).

Un plugin client de `Learning` surveille l'utilisateur connecté : s'il diffère du propriétaire du
paquet, tout est effacé avant d'afficher quoi que ce soit (US4-4) ; s'il est le même, la file part
(session expirée puis reconnexion). Un `401` pendant l'envoi garde la file.

**Rationale**: la constitution (II) permet qu'une couche s'appuie sur ce qu'une autre expose ;
`Learning` réutilise déjà des composants de `Catalog`. Distinguer la déconnexion choisie (on
efface) de la session expirée (on garde) n'est possible que du côté de l'action de l'apprenant.

**Alternatives considered**: effacer dès que la session devient nulle (une session expirée
pendant la coupure perdrait les réponses, contre l'edge case) ; un hook Nuxt de déconnexion
(une indirection pour un seul abonné, principe VII).

## R10. Quand la file part et quand le paquet se met à jour

**Decision**: `flush()` est un appel unique partagé (une promesse en cours à la fois par onglet ;
deux onglets qui envoient la même réponse sont absorbés par `answer_id`). Il part :

- à l'ouverture en ligne de l'app (plugin client), et `useRevisions()` attend sa fin avant
  d'afficher (FR-010, US2-2) ;
- à l'événement `online` (`useConnectionStatus`, déjà présent dans `technical/Pwa`) ;
- juste après l'écriture de chaque réponse, si le réseau est là.

Le paquet se rafraîchit à l'ouverture en ligne, en fin de séance, après avoir appris ou arrêté un
sujet, et après chaque `flush()` qui vide la file (FR-003, FR-017). L'indication « Hors ligne —
cartes à jour du {date} » et le compteur « {n} réponses à envoyer » (FR-009) sont deux composants
de `Learning`, affichés dans « Mes révisions » et dans la séance.

**Rationale**: pas de synchronisation en arrière-plan, que Safari ne fournit pas et que la spec ne
demande pas (US2-2 prévoit l'envoi à la réouverture).

**Alternatives considered**: Background Sync API (voir R7) ; un intervalle de nouvelle tentative
(l'événement `online` suffit, et l'ouverture rattrape le reste).

## R11. Images hors ligne

**Decision**: une carte lue depuis le paquet n'affiche pas `QuestionImageGallery` : à la place,
pour chaque image, sa description (`alt`) et la mention « Image non disponible hors ligne »
(FR-002). Une séance commencée en ligne garde ses images. Aucun fichier d'image n'est mis en cache.

**Rationale**: choix du développeur ; `alt` est obligatoire depuis la 003.

## R12. Rappels

**Decision**: aucun changement dans `back/functional/reminders`. `ReminderEligibility` lit
`max(review_answers.answered_at)` (`ReminderEligibility.php:58`) : une réponse envoyée plus tard y
compte à sa date de réponse, ce que demande l'edge case. Une réponse `discarded` reste une
activité de révision et compte aussi.

**Alternatives considered**: ne compter que les réponses `applied` (une personne qui a révisé
serait relancée comme si elle avait décroché).
