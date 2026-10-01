# Research: Connexion avec Google

Décisions techniques de la feature 007, avec leurs raisons et les alternatives écartées. Les
chemins renvoient à l'état du code au 2026-10-01.

## R1. `laravel/socialite` pour parler à Google

**Decision**: ajouter `laravel/socialite` (version stable la plus récente, 5.x) à la couche
`back/functional/users`, avec le pilote `google`, les portées `openid email profile` et PKCE
activé. Les identifiants vivent dans la configuration de la couche (`services.google` surchargé dans
le `register()` de `UsersServiceProvider`), lus depuis `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET` et
`GOOGLE_REDIRECT_URI`.

**Rationale**: paquet officiel de Laravel, maintenu, qui gère l'échange du code, le `state` en
session et la lecture du profil (`id`, `email`, `name`, et `email_verified` dans les attributs
bruts). Aucune bibliothèque OAuth n'est installée aujourd'hui (`back/composer.json`). `back/AGENTS.md`
demande l'accord du développeur pour toute nouvelle dépendance : la validation de ce plan vaut cet
accord.

**Alternatives considered**: écrire le flux OpenID Connect à la main (vérification de jeton,
rotation des clés de Google : du code de sécurité sans second usage, principe VII) ;
`google/apiclient` (lourd, pensé pour les API Google, pas pour la connexion) ; Google Identity
Services côté web avec envoi du jeton d'identité au back (le web n'appelle l'API que par
`laravel-raom-nuxt` ou `useApiFetch`, et ce bouton dépend d'un script Google sur toutes les pages).

## R2. Un flux par redirection, porté par le back

**Decision**: le bouton du web est un lien vers
`GET /api/auth/google/redirect?intention=…&redirect=…&timezone=…`. Le back garde l'intention, la
page de retour et le fuseau en session, puis redirige vers Google. Google revient sur
`GET /api/auth/google/callback`. Le back y connecte, lie ou prépare le compte, puis redirige vers
une page du web, `/connexion-google`, avec un code de résultat. Les deux routes sont dans
`back/functional/users/routes/account.php` (middleware `web`, préfixe `/api`), comme les autres routes
de compte. Le contrôleur intercepte toute exception et répond par une redirection, jamais par du
JSON (`bootstrap/app.php` rend du JSON pour `api/*`).

**Rationale**: le cookie de session est posé par le back sur le domaine `localhost`, qui couvre
`localhost:3000` (`SESSION_DOMAIN`), comme pour la confirmation d'email (`VerifyEmailController`, qui
connecte déjà le compte hors de Fortify). Le secret client ne quitte jamais le serveur. Le `state`
de Socialite, lié à la session du navigateur qui a commencé le parcours, et PKCE refusent un retour
rejoué, ouvert sur un autre appareil ou modifié (FR-013).

**Alternatives considered**: retour de Google sur une page du web qui transmet le code au back (le
code circule dans le JavaScript, et le `state` doit être revérifié des deux côtés) ; fenêtre
surgissante (bloquée par les navigateurs mobiles, et inutilisable dans la PWA installée).

## R3. Le lien Google est une colonne de `users`

**Decision**: `users` gagne `google_id` (string, unique, nul) et `google_linked_at` (timestamp,
nul). `password` devient nul. Aucun jeton d'accès Google n'est gardé.

**Rationale**: un compte a au plus un lien et un lien un seul compte (FR-004) : une colonne unique
suffit. Elle disparaît avec la ligne du compte à l'effacement (FR-011), sans écouteur de plus. Le
jeton d'accès ne sert qu'à lire le profil pendant le retour : le garder serait une donnée
personnelle sans usage (FR-012, principe VI).

**Alternatives considered**: une table `social_identities` (prévue pour plusieurs fournisseurs, ce que
la spec exclut, principe VII) ; garder le jeton pour rafraîchir le profil (aucun besoin).

## R4. Ce que fait le retour de Google

**Decision**: une action `SignInWithGoogle` traite le profil reçu, dans cet ordre :

| Cas | Effet | Résultat |
|---|---|---|
| adresse non vérifiée par Google | rien | `adresse-non-verifiee` |
| `google_id` connu | connexion de ce compte | `connecte` |
| adresse d'un compte confirmé non lié | lien, puis connexion (FR-007) | `lie` |
| adresse d'un compte confirmé déjà lié à un autre `google_id` | rien | `autre-compte-google` |
| adresse d'un compte en attente | confirmation, mot de passe retiré, lien, connexion (FR-008) | `lie` |
| aucune adresse | profil gardé en session 15 minutes, aucun compte (FR-003) | `nom` |
| refus chez Google, `state` invalide, erreur de Google | rien | `annule`, `echec` |

Une connexion réussie passe par ce que fait la connexion par email : fuseau mis à jour, demande de
suppression annulée (feature 004, FR-014), session régénérée. Les méthodes privées
`rememberTimezone` et `cancelPendingDeletion` de `AuthenticateUser` deviennent une action partagée,
`CompleteSignIn`, appelée par les deux connexions. Le résultat devient `suppression-annulee` quand
une demande est annulée. Le verrouillage après 5 échecs de mot de passe ne bloque pas Google, qui
prouve à lui seul la possession de l'adresse.

**Rationale**: FR-002 à FR-008 et l'edge case « compte déjà lié » deviennent une seule table de
décision, testable cas par cas. Les adresses sont comparées en minuscules, comme le fait déjà le
mutateur `email` de `User`.

**Alternatives considered**: créer le compte dès le retour avec le nom Google (écarté par le
développeur, FR-003) ; refuser l'adresse d'un compte existant (écarté par le développeur, FR-007).

## R5. Le nom affiché avant la création

**Decision**: le résultat `nom` mène la page `/connexion-google` à un formulaire de nom affiché.
`GET /api/auth/google/pending` lui renvoie `{ name, email }` du profil gardé en session. Le nom est
coupé à 60 caractères, et vide s'il en a moins de 2. `POST /api/auth/google/register`
(`display_name`, `timezone`) crée le compte confirmé, le lie et le connecte. Les règles du nom sont
celles de `RegisterAccount` (2 à 60 caractères). Passé 15 minutes, ou sans profil en session, les deux
routes répondent `404 google_profile_expired` et le web propose de recommencer.

**Rationale**: le profil reste côté serveur, lié à la session qui a prouvé la possession de l'adresse
(FR-014) : aucune autre session ne peut lire cette adresse ni créer le compte.

**Alternatives considered**: transmettre le profil au web dans l'URL de retour (adresse et nom dans
l'historique du navigateur) ; un jeton signé à usage unique (la session fait déjà ce travail).

## R6. Une seule page de retour sur le web

**Decision**: une page `web/functional/Account/app/pages/connexion-google.vue`, sans middleware,
lit `?resultat=` :

- `connecte`, `lie`, `suppression-annulee` : relit le compte (`fetchUser`), affiche le message (« Vous
  pourrez désormais vous connecter avec Google », « Votre demande de suppression est annulée »),
  puis navigue vers la page demandée, gardée par le back et validée comme dans `useLoginForm`
  (chemin interne seulement).
- `nom` : formulaire du nom affiché (R5).
- `annule`, `echec`, `adresse-non-verifiee`, `autre-compte-google` : message et deux boutons,
  « Réessayer avec Google » et « Utiliser mon email » (FR-015).

Le bouton `GoogleButton.vue` remplace `GoogleSoonButton.vue` dans `connexion.vue` et
`inscription/index.vue`. C'est un lien vers l'URL de R2, construite avec `apiBaseUrl`, la page
demandée et le fuseau de l'appareil. Il est désactivé hors ligne, avec une explication
(`useConnectionStatus`, edge case).

**Rationale**: les données de révision hors ligne du compte précédent sont déjà effacées à
l'ouverture par le plugin de la 006, qui compare le compte connecté au propriétaire du paquet après
chaque chargement complet de page : le retour de Google en est un.

**Alternatives considered**: renvoyer directement sur la page demandée avec un paramètre de message
(chaque page devrait savoir l'interpréter) ; une page par résultat (trois pages pour un parcours).

## R7. Supprimer un compte sans mot de passe

**Decision**:

- `GET /api/account/deletion` renvoie en plus `confirmations`, la liste des moyens possibles :
  `password` si le compte a un mot de passe, `google` s'il est lié.
- Pour `google`, le bouton du web suit le parcours de R2 avec `intention=suppression` et
  `keep_published_subjects`. Au retour, le back vérifie que la session est celle d'un compte connecté
  et que le `google_id` reçu est le sien. Il appelle alors `RequestAccountDeletion` sans mot de passe,
  ferme les sessions comme `AccountDeletionController::store`, et redirige vers
  `/compte-supprime?le=<date>`.
- Un autre compte Google ou une annulation renvoie sur `/supprimer-mon-compte?google=…` sans rien
  changer (FR-010, US4-3).
- `RequestAccountDeletion` reçoit la preuve à vérifier (mot de passe ou identité Google) au lieu d'un
  mot de passe seul. `POST /api/account/deletion` sans mot de passe sur un compte sans mot de passe
  répond `422 password_not_set`.
- La page `/compte-supprime` efface les données de révision de l'appareil
  (`useOfflineReview().discard()`), puisque la demande n'est pas passée par `useAuth`.

**Rationale**: une confirmation doit prouver que la personne présente est bien la titulaire, comme le
fait le mot de passe dans la 004. Une nouvelle identification chez Google le prouve.

**Alternatives considered**: confirmer en tapant son adresse email (ne prouve rien pour une session
oubliée ouverte) ; un code envoyé par email (un parcours de plus pour un cas rare).

## R8. Mot de passe nul

**Decision**: une migration rend `users.password` nul. `AuthenticateUser` traite un compte sans mot de
passe comme un mot de passe faux : même message `invalid_credentials`, même compteur, rien qui révèle
que le compte existe ou qu'il est lié à Google (FR-014). « Mot de passe oublié » et
`ResetUserPassword` fonctionnent tels quels, puisqu'ils ne lisent pas l'ancien mot de passe :
ils donnent un mot de passe au compte (edge case). `UserFactory` gagne un état `withGoogle()`, sans
mot de passe.

## R9. PWA installée

**Decision**: le lien du bouton s'ouvre dans la même fenêtre (`target` par défaut). Sur Android,
une URL hors de la portée de l'application s'ouvre dans un onglet intégré, et le retour sur
`/connexion-google`, dans la portée, rouvre l'application. Sur iOS, le comportement d'une PWA
installée qui quitte son origine est à vérifier sur un appareil : si le cookie de session ne revient
pas dans l'application, la page `/connexion-google` indique de rouvrir CINQ, et la connexion par
email reste possible. Ce point est noté dans [quickstart.md](quickstart.md).

**Alternatives considered**: déclarer l'origine de l'API dans `scope` (une portée ne peut pas
couvrir une autre origine) ; une fenêtre surgissante (R2).

## R10. Tests sans Google

**Decision**: côté back, le pilote `google` de Socialite est remplacé dans les tests par un double
qui renvoie un profil donné (`Socialite::fake()` si la version installée le fournit, sinon
`Socialite::shouldReceive('driver->user')`). Un `state` manquant ou différent produit
`InvalidStateException`, ce qui teste FR-013. Côté web, la page `/connexion-google` est montée avec
chaque `resultat`, et les appels à `pending` et `register` passent par le `stubAccountApi`
existant.
