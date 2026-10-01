# Quickstart: valider la révision hors ligne

Contrat dans [contracts/api.md](contracts/api.md), règles dans [data-model.md](data-model.md).

## Tests automatisés

```bash
cd back && ./vendor/bin/sail artisan test --compact functional/learning functional/users functional/reminders
cd web && pnpm test
```

| Exigences | Couche | Cas |
|---|---|---|
| FR-001 | `back/learning` | `upcoming` : cartes à J, J+7 incluses, J+8 exclue, dans le fuseau du compte ; sujets brouillon, retirés, retenus exclus ; cartes d'un autre compte exclues |
| FR-011, SC-003 | `back/learning` | boîte 2, « sait » lundi, envoyée mercredi : boîte 3, échéance lundi + 4 jours ; `answered_at` gardé |
| FR-012 | `back/learning` | deux réponses d'une carte reçues dans le désordre : même état final que dans l'ordre |
| FR-013 | `back/learning` | US3 : 8 h « ne sait pas » reçue après 12 h « sait » : carte en boîte 1, réponse de 12 h `discarded` |
| FR-014 | `back/learning` | carte supprimée, d'un autre compte, sujet brouillon / retiré / retenu : `200`, aucune ligne, carte inchangée |
| FR-015 | `back/learning` | `answered_at` futur : compté maintenant ; horloge en retard : comptée maintenant si due, sinon `discarded` |
| FR-016, SC-002 | `back/learning` | même `answer_id` deux fois : une seule ligne, carte modifiée une fois |
| SC-005 | `back/learning`, `web/Learning` | la table des cas Leitner, des deux côtés |
| FR-007 | `back/users`, `web/Learning` | `GET /api/user` renvoie `timezone` ; le jour local suit le fuseau du compte, pas celui de l'appareil |
| edge case rappels | `back/reminders` | réponse envoyée plus tard : l'espacement part de son `answered_at` |
| FR-005, FR-006, FR-002 | `web/Learning` | hors ligne : « Mes révisions » depuis le paquet, avec la date de mise à jour ; séance dans l'ordre ; image remplacée par sa description |
| FR-008, FR-009 | `web/Learning` | réponse écrite dans IndexedDB (`fake-indexeddb`) avant tout envoi ; compteur affiché, puis retiré |
| FR-010, FR-016 | `web/Learning` | événement `online` : file envoyée dans l'ordre ; coupure au 3ᵉ envoi : reprise sans perte ni doublon |
| FR-003, FR-017 | `web/Learning` | paquet remis à jour à l'ouverture, en fin de séance, après apprendre / arrêter ; pas tant que la file n'est pas vide |
| FR-004, FR-018, SC-004 | `web/Account`, `web/Learning` | déconnexion avec réponses en attente : avertissement, puis base vide ; autre compte connecté : base vidée |
| FR-019 | `web/Learning` | IndexedDB refusé : révision en ligne inchangée, message dans « Mes révisions » |
| edge case session | `web/ApiClient` | erreur réseau sur `/user` : `isUnreachable`, pages `availableOffline` accessibles ; `401` : redirection vers la connexion |
| US4-5 | `web/Learning` | paquet de plus de 7 jours : seules ses cartes, message de reconnexion |

## Parcours manuel à 360 px, sur le build de production

Le service worker n'existe qu'en production :

```bash
cd web && pnpm build && PORT=3000 node --env-file=.env .output/server/index.mjs
```

1. Se connecter avec un compte seedé ayant des cartes dues, ouvrir « Mes révisions » en ligne.
2. Couper le réseau (DevTools → Network → Offline), recharger `/revisions` : la page s'affiche
   avec « Hors ligne — cartes à jour du … ».
3. Lancer une séance, répondre à 3 cartes : boîte et date affichées, compteur « 3 réponses à
   envoyer ». Une carte avec image montre sa description et « Image non disponible hors ligne ».
4. Fermer l'onglet, le rouvrir hors ligne : le compteur indique toujours 3.
5. Rétablir le réseau : le compteur disparaît en moins d'une minute ; sur un autre navigateur
   connecté au même compte, « Mes révisions » montre la progression.
6. Repasser hors ligne, répondre à une carte, puis se déconnecter : l'avertissement
   « 1 réponse n'est pas encore envoyée » s'affiche ; « Me déconnecter quand même » vide la base
   (DevTools → Application → IndexedDB).
7. Navigation privée : « Mes révisions » indique que la révision hors ligne est indisponible, la
   séance en ligne fonctionne.
