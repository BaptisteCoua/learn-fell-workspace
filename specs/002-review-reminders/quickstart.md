# Guide de validation : Rappels de révision (002)

Ce guide prouve, de bout en bout, que la feature fonctionne. Il ne décrit pas l'implémentation :
voir [plan.md](plan.md), [data-model.md](data-model.md) et [contracts/api.md](contracts/api.md).

## Prérequis

- L'environnement de la 001 : Docker pour Sail, Node.js et pnpm.
- Les deux repos sur la branche `002-review-reminders`.
- Pour le push : Chrome ou Firefox sur ordinateur ; en option un téléphone Android, ou un iPhone
  sous iOS 16.4 ou plus avec la PWA installée. Il faut alors une URL `https` (tunnel), car le push
  exige un contexte sécurisé hors `localhost`.

## Démarrer

```bash
# back
cd back
./vendor/bin/sail composer install
./vendor/bin/sail artisan webpush:vapid          # écrit VAPID_PUBLIC_KEY et VAPID_PRIVATE_KEY dans .env
./vendor/bin/sail artisan migrate
./vendor/bin/sail artisan queue:work             # envoi des rappels
./vendor/bin/sail artisan schedule:work          # reminders:dispatch chaque minute

# web : même clé publique que le back
cd ../web
echo "NUXT_PUBLIC_VAPID_PUBLIC_KEY=$(grep VAPID_PUBLIC_KEY ../back/.env | cut -d= -f2)" >> .env
pnpm dev                                         # écrans, sur http://localhost:3000

# push : le service worker n'existe qu'en build de production
pnpm build && PORT=3000 node .output/server/index.mjs
```

Pour ne pas attendre l'heure réelle, la commande accepte une horloge :
`./vendor/bin/sail artisan reminders:dispatch --now="2026-09-26 19:00"`.

## Tests automatisés

```bash
cd back && ./vendor/bin/sail artisan test --compact functional/reminders
cd back && ./vendor/bin/sail artisan test && ./vendor/bin/sail bin pint --test
cd web && pnpm test && pnpm lint && pnpm exec prettier --check .
```

**Attendu** : tout est vert, avec au moins les cas suivants.

- `NextReminderSlot` :
  - heure à venir, passée, ou déjà servie aujourd'hui ;
  - passage à l'heure d'été : 02:30 à Paris le dernier dimanche de mars ;
  - passage à l'heure d'hiver ;
  - changement de fuseau vers l'est et vers l'ouest.
- `ReminderSpacing` :
  - chaque jour d'inactivité du 1er au 60e ;
  - reprise du rythme quotidien après une réponse (SC-005).
- `ReminderEligibility` :
  - chaque cas de FR-008 et FR-019 : adresse non confirmée, aucun canal, aucun apprentissage,
    aucune carte due, cartes seulement dans des sujets dépubliés ou retirés ;
  - le nombre annoncé est égal à celui de la `DueCardsInstruction` (SC-006).
- Unicité : deux tâches pour le même compte et le même jour n'envoient qu'un rappel ; l'heure
  changée pour plus tard le même jour n'en envoie pas un second (SC-003).
- Canaux :
  - `Notification::fake()` pour l'email ;
  - client HTTP simulé pour le push : 201 → `last_delivered_at` mis à jour ; 410 → appareil
    supprimé, email tout de même envoyé ;
  - trois refus SMTP 5xx → email désactivé (`bounced`).
- Désinscription : lien valide → 204, idempotent ; signature altérée, compte inconnu ou `v`
  périmé → même réponse 403 ; appel sans cookie accepté (RFC 8058) ; en-têtes
  `List-Unsubscribe` présents dans l'email.
- Accès : un compte ne voit ni ne modifie les réglages ou les appareils d'un autre (404).
- Web : proposition affichée une seule fois ; refus de l'autorisation ; message iOS hors PWA ;
  appareil courant reconnu ; page de désinscription valide et invalide.

## Scénarios manuels

Chaque scénario renvoie aux critères d'acceptation de [spec.md](spec.md).

1. **Proposition** (US1, scénarios 1 à 3) :
   - avec un compte neuf, apprendre un sujet : le dialogue s'ouvre, cases décochées et heure à
     19:00 ;
   - « Plus tard », puis apprendre un deuxième sujet : plus de dialogue ;
   - avec un autre compte, cocher l'email et activer : une confirmation s'affiche.
2. **Section « Rappels »** (US1, scénarios 4, 6 et 7) :
   - dans `/compte`, passer l'heure à 08:00 ;
   - activer les notifications (build de production), accepter l'autorisation : l'appareil
     « Chrome sur Linux » apparaît ;
   - recommencer dans Firefox : deux appareils sont listés ;
   - refuser l'autorisation dans un troisième profil : le message d'aide s'affiche et rien n'est
     enregistré.
3. **iOS** (US1, scénario 5) : sur iPhone, dans Safari sans installation, le message renvoie vers
   les instructions d'installation.
4. **Rappel du jour** (US2) :
   - avoir des cartes dues sur 2 sujets et lancer `reminders:dispatch --now` à l'heure réglée ;
   - Mailpit (http://localhost:8035) reçoit l'email « N cartes à réviser aujourd'hui » ;
   - chaque navigateur abonné affiche la notification ;
   - le clic ouvre `/revisions/seance?sujets=…` avec N cartes ;
   - relancer la commande : rien de plus.
5. **Sans carte due** (US2, scénarios 3, 4 et 6) : réviser tout, ou dépublier les sujets, ou
   arrêter tous les apprentissages, puis lancer la commande : rien ne part.
6. **Connexion depuis le lien** (US2, scénario 7) : se déconnecter, ouvrir le lien de l'email, se
   connecter : on arrive sur la séance.
7. **Espacement** (US3) : avec `--now`, simuler 10 jours sans réponse (un jour sur deux), puis 30
   jours (une fois par semaine) ; répondre à une carte, puis lancer la commande le lendemain : le
   rappel repart.
8. **Désinscription** (US4) :
   - cliquer « Ne plus recevoir ces rappels » dans Mailpit, sans session : la confirmation
     s'affiche ;
   - `/compte` montre l'email désactivé, les notifications continuent ;
   - modifier un caractère de la signature : « Ce lien n'est pas valide » ;
   - `curl -X POST -d 'List-Unsubscribe=One-Click' '<URL de List-Unsubscribe>'` → 204.
9. **Appareil révoqué** (US4, scénario 6) : retirer l'autorisation dans les réglages du
   navigateur, puis lancer la commande : l'appareil disparaît de la liste et l'email part quand
   même.
10. **Mise en page** : proposition, section « Rappels » et page de désinscription de 360 à 1440 px,
    sans défilement horizontal ni libellé tronqué ; axe sans violation (script `a11y.cjs` de la
    001).
