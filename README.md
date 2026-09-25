# Agent Mail

Un agent qui travaille dans votre boîte mail **sous vos yeux** : il se connecte à
Gmail ou à Outlook, lit vos messages, apprend votre façon d'écrire, classe, marque
comme lu, archive et prépare vos brouillons — en temps réel, curseur à l'écran.

Un seul fichier (`index.html`), aucune dépendance, aucun build.

---

## Le mettre en ligne (5 minutes)

L'outil a besoin d'une adresse **https** : Google et Microsoft refusent `file://`
et refusent une origine non déclarée.

**Le plus rapide — Netlify Drop.** Ouvrez `app.netlify.com/drop`, glissez le dossier :
vous obtenez une adresse `https://…netlify.app` immédiatement.

**GitHub Pages.** Poussez ce dépôt, puis *Settings → Pages → Deploy from a branch →
main / root*. L'adresse est `https://<compte>.github.io/agent-mail/`.

```bash
git remote add origin git@github.com:<compte>/agent-mail.git
git push -u origin main
```

**Ensuite, déclarez cette adresse des deux côtés** (une fois pour toutes) :

| | Où | Quoi déclarer |
|---|---|---|
| Google | Cloud Console → Identifiants → ID client OAuth (Application Web) | *Origines JavaScript autorisées* : `https://votre-domaine` (sans chemin) |
| Microsoft | Entra ID → App registrations → Authentication | Plateforme **SPA**, URI de redirection : l'adresse complète, `https://votre-domaine/agent-mail/index.html` ou `https://votre-domaine/` selon l'endroit où le fichier est servi |

Reportez les deux identifiants obtenus dans `config.json`, à côté de `index.html` :

```json
{ "googleClientId": "…apps.googleusercontent.com", "microsoftClientId": "…" }
```

Rien à recompiler : rechargez la page, les boutons de connexion apparaissent.
L'adresse exacte à déclarer est rappelée dans l'écran « Réglages » de l'outil.

**Le moteur.** Pour que les apprenants n'aient aucune clé à saisir, déployez le
relais (`./relais/deployer.sh`) et reportez son adresse dans `config.json`. Sinon,
chacun saisit sa propre clé Anthropic ou OpenAI dans les réglages.

---

## 1 · Deux apparences, une seule application

L'écran d'accueil est en charte France IA (logo, Inter, bleu #4169E1, barres
tricolores). Dès la connexion, l'interface **prend l'apparence de la messagerie
branchée** :

| Messagerie | Apparence |
|---|---|
| Gmail | rail Google, pastille « Nouveau message », lignes sur une ligne, recherche arrondie, le message s'ouvre en pleine largeur |
| Outlook / Microsoft 365 | bandeau bleu Fluent, volet de lecture à droite, cartes à barre bleue pour les non-lus |


Par-dessus, la couche agent est volontairement d'un autre monde, façon curseurs
partagés d'un outil collaboratif :

- un **curseur nommé « Agent France IA »**, violet, qui se déplace d'un message à
  l'autre, avec un projecteur qui assombrit le reste de l'écran ;
- **votre propre curseur étiqueté « Vous »** pendant que l'agent travaille : on voit
  deux mains sur la même boîte ;
- de **petites cartes d'action qui s'ouvrent au curseur** : un menu « Classement »
  où l'option retenue s'allume, puis une carte « Étiqueté », « Marqué comme lu »,
  « Archivé », « Brouillon enregistré » à l'endroit exact où l'action a lieu ;
- une console sombre qui déroule le journal d'exécution, ligne par ligne.

## 2 · Ce que l'agent fait réellement

| Mission | Ce qui se passe vraiment |
|---|---|
| **Me connaître** | À la première connexion, l'agent lit les 12 derniers messages envoyés et 8 messages reçus, en déduit une fiche (identité, activité, ton, formules d'ouverture et de clôture, signature, interlocuteurs, sujets) et l'injecte ensuite dans chaque consigne donnée au moteur. La fiche reste dans le navigateur, s'affiche et se corrige à la main. |
| **Trier la boîte** | Chaque non-lu est lu par Claude, classé en six catégories, étiqueté dans la messagerie (libellé Gmail `Agent/…` ou catégorie Outlook), puis marqué comme lu s'il n'appelle aucune action. |
| **Résumé du jour** | Lecture des 16 derniers messages, synthèse en quatre parties rédigée en streaming. |
| **Rédiger les réponses** | Pour chaque message qui attend une réponse : brouillon écrit sous les yeux de l'apprenant, puis **enregistré comme vrai brouillon** (Gmail `drafts.create`, Graph `createReply`). |
| **Ranger les diffusions** | Marquage lu + archivage, après confirmation avec la liste des messages concernés. |

Actions unitaires depuis le volet de lecture : résumer, demander une réponse,
marquer lu, archiver, copier le brouillon, ouvrir le message dans la messagerie.

**Garde-fous inscrits dans le code** — aucun envoi (brouillons seulement), aucune
suppression (archivage réversible), confirmation obligatoire avant tout archivage
en série, touche Échap pour tout arrêter, aucun stockage hors du navigateur.

## 3 · Le moteur : Claude ou ChatGPT, et qui paie

L'agent peut tourner sur **Claude** (API Anthropic) ou sur **ChatGPT** (API OpenAI) :
le moteur se choisit dans les réglages, et le reste de l'outil ne change pas.

**Un abonnement ChatGPT Plus ou Claude Pro ne donne pas accès à l'API.** Ce sont deux
produits facturés séparément, et il n'existe pas de « connecteur » permettant à une
application web tierce de consommer l'abonnement de son utilisateur. Une application
qui appelle un modèle paie donc des jetons, dans tous les cas. Trois façons de s'en
sortir, par ordre de simplicité pour l'apprenant :

| | Ce que fait l'apprenant | Qui paie | État |
|---|---|---|---|
| **Relais France IA** | rien | France IA, quelques centimes par boîte | codé, à déployer |
| **Sa propre clé API** | crée une clé, la colle une fois | lui, à l'usage | codé |
| **Connecteur MCP** | ajoute un connecteur dans son ChatGPT ou son Claude | son abonnement, sans surcoût | à construire |

La troisième voie est la seule réellement sans surcoût, parce qu'elle **inverse le
sens** : ce n'est plus l'outil qui appelle le modèle, c'est le ChatGPT (ou le Claude)
de l'apprenant qui appelle un serveur MCP exposant sa boîte mail. L'inférence est
alors payée par son abonnement. En contrepartie, la conversation se déroule dans
ChatGPT, pas dans cette interface — sauf à faire de cette page un écran de suivi
branché sur le même serveur, ce qui suppose un vrai service hébergé (OAuth des
messageries côté serveur, sessions, flux d'événements). C'est un projet en soi,
à décider séparément.

## 3 bis · L'accès au moteur : l'apprenant ne saisit rien

Un apprenant ne créera pas de clé API. L'outil sait donc fonctionner de deux façons :

1. **Relais France IA (recommandé)** — un Cloudflare Worker détient *votre* clé
   Anthropic, vérifie un code d'accès, applique des quotas et transmet le flux.
   L'apprenant ouvre le lien et lance une mission : rien d'autre. Déploiement en
   trois commandes, voir `relais/README.md`, puis en tête de `index.html` :

   ```bash
   ./relais/deployer.sh
   ```

   Le script demande l'adresse du site, la clé (elle n'apparaît pas à l'écran et
   n'est écrite nulle part sur le disque) et le code d'accès, puis met le relais en
   ligne. Il ne reste qu'à recopier l'adresse obtenue dans `config.json` :

   ```json
   { "relaisUrl": "https://agent-mail-relais.<compte>.workers.dev", "relaisCode": "FIA-OCT26" }
   ```

   Vous payez la consommation : comptez quelques dizaines de centimes par boîte
   traitée avec `claude-opus-5`, quelques centimes avec `claude-haiku-4-5`
   (modèle réglable dans l'outil). Les quotas par poste et par jour sont dans
   `wrangler.toml` ; changer le code coupe l'accès aux anciennes copies.

2. **Clé personnelle** — repliée sous « Utiliser ma propre clé Anthropic » sur
   l'écran d'accueil. L'appel part alors directement du navigateur vers Anthropic,
   sans relais. C'est la voie des apprenants avancés, et votre voie pour tester
   avant d'avoir déployé le relais.

Sans relais configuré, l'écran d'accueil l'annonce et ouvre la section clé : aucun
état trompeur.

> Il n'existe pas de « connexion avec son compte Claude » pour une application web
> tierce : l'API s'authentifie par clé. Le relais est donc le seul moyen d'offrir
> l'outil sans faire créer de clé à chacun.

## 3 ter · La connexion

L'écran d'entrée n'a pas de boutons maison : ce sont les composants officiels des
fournisseurs.

**Google** — le bouton est *rendu par Google* (Google Identity Services,
`google.accounts.id.renderButton`), en français, forme pilule, plus l'invite
« One Tap » quand aucune session n'est ouverte. La connexion se fait en deux temps,
comme Google le prescrit :

1. **Identité** — le bouton renvoie un jeton d'identité signé (OpenID Connect) :
   nom, adresse, photo. L'outil sait qui est là, et l'affiche.
2. **Autorisation** — une seconde demande, explicite, ouvre l'accès à la boîte
   (`gmail.modify`). Au retour d'un utilisateur déjà consentant, elle est silencieuse.

**Microsoft** — bouton aux proportions et couleurs de la charte Microsoft
(41 px, logo 21 px, Segoe UI 15 px, `#5E5E5E` sur fond blanc bordé `#8C8C8C`),
puis OAuth 2.0 + PKCE. L'identité vient du `id_token` renvoyé avec l'accès ;
le jeton de rafraîchissement reste en `sessionStorage`, le temps de l'onglet.

**Apple** — pas de bouton, et ce n'est pas un oubli. « Se connecter avec Apple »
est une brique d'identité : elle renvoie un nom et une adresse, jamais l'accès à la
boîte. iCloud Mail n'expose aucune API — seulement l'IMAP, avec un mot de passe
d'application créé à la main. Il faudrait en plus un compte développeur Apple et un
secret signé côté serveur, pour un bouton qui n'ouvrirait aucune boîte.
Même situation pour Free, OVH ou un serveur d'entreprise : IMAP seulement, protocole
qu'un navigateur ne sait pas parler. Les couvrir suppose un pont IMAP hébergé —
chantier séparé.

**Session** — l'outil retient le nom, l'adresse et la photo du dernier compte
connecté (`localStorage`), **jamais les jetons**. Au retour, l'écran affiche
« Continuer » avec la photo, et la reconnexion à la boîte se fait sans écran
intermédiaire quand le fournisseur l'autorise. « Utiliser un autre compte » efface
la session et redonne les boutons.

## 4 · Configuration des messageries

Deux identifiants OAuth publics à renseigner une fois — dans `config.json`, déposé
à côté de `index.html` (le plus simple : aucun code à toucher), dans l'objet `CONFIG`
en tête de script, ou dans « Réglages » poste par poste. L'ordre de priorité est :
réglages de l'utilisateur, puis `config.json`, puis `CONFIG`. Google et Microsoft refusent `file://` : hébergez la page
à une adresse https **stable**, par exemple `https://outils.franceia.com/agent-mail/`
(GitHub Pages, Netlify, Vercel, votre hébergement). C'est cette adresse exacte que
vous déclarerez des deux côtés ; elle est rappelée dans « Réglages ».
La boîte d'entraînement fonctionne même en double-cliquant le fichier.

### Outlook / Microsoft 365 — le plus simple

1. `portal.azure.com` → **Microsoft Entra ID** → *App registrations* → *New registration*.
2. Comptes pris en charge : **annuaire quelconque + comptes Microsoft personnels**.
3. URI de redirection : plateforme **Single-page application (SPA)**, l'adresse exacte de la page.
4. *API permissions* → Microsoft Graph, permissions déléguées `Mail.ReadWrite` et `User.Read`.
5. Copier l'**Application (client) ID** dans `microsoftClientId`.

Aucune vérification préalable. Certains tenants d'entreprise verrouillent le
consentement utilisateur et réclameront l'accord de leur administrateur.

### Gmail — un obstacle à connaître

1. `console.cloud.google.com` → nouveau projet → activer l'**API Gmail**.
2. Écran de consentement OAuth : type **Externe**.
3. *Identifiants* → **ID client OAuth** → **Application Web** → *Origines JavaScript
   autorisées* = l'origine de la page (schéma + domaine, sans chemin).
4. Copier l'ID dans `googleClientId`.

Lire une boîte Gmail exige `gmail.modify`, périmètre **restreint** par Google :

| Voie | Portée | Coût |
|---|---|---|
| Application en **mode Test** | 100 utilisateurs, ajoutés un par un | gratuit, immédiat |
| Application **publiée** | illimité | vérification Google + évaluation CASA : semaines, prestataire payant |
| **Chaque apprenant crée son ID client** et le colle dans Réglages | illimité | 10 minutes par personne |

Combinaison réaliste pour distribuer largement tout de suite : **Outlook branché
d'emblée**, **boîte d'entraînement pour la démonstration**, procédure Gmail en dix
minutes pour ceux qui y tiennent — et vérification Google engagée en parallèle si
l'outil s'installe dans la durée.

## 5 · Ce qui circule, et où

- Le navigateur parle **directement** à Google ou Microsoft (lecture, étiquettes,
  brouillons). Pour l'analyse, il parle à Anthropic — directement avec une clé
  personnelle, ou via votre relais, qui ne conserve rien.
- Le jeton d'accès à la messagerie reste **en mémoire vive** : il disparaît à la
  fermeture de l'onglet. Une clé personnelle est dans le `localStorage` ;
  « Réglages → Effacer mes données » la retire.
- Le contenu des messages traités est transmis à Anthropic pour être analysé.
  Le bouton « Comment ça marche » le dit aux apprenants, en une page.
- Sur un poste partagé : « Quitter », puis « Effacer mes données ».

## 6 · Structure du fichier (pour la reprise)

Neuf sections numérotées dans le `<script>` : 1 configuration · 2 boîte
d'entraînement · 3 fournisseurs de messagerie (interface commune `connecter /
profil / lister / corps / marquerLu / archiver / etiqueter / creerBrouillon`) ·
4 moteur Claude (relais ou clé, SSE, réflexion diffusée) · 5 couche agent ·
6 missions · 7 affichage · 8 modales · 9 connexion et démarrage.

Ajouter un fournisseur (IMAP via un relais, par exemple) revient à écrire un objet
de la section 3 et à l'inscrire dans `FOURNISSEURS` ; missions et affichage ne
changent pas. Ajouter une apparence : un bloc de variables
`:root[data-skin=…]` et une entrée dans `SKINS`.

## 7 · Limites connues

- 25 messages chargés par session, boîte de réception seule.
- Les pièces jointes ne sont ni lues ni jointes aux brouillons.
- Le fil de discussion n'est pas reconstitué : l'agent lit le dernier message.
- Les corps HTML sont convertis en texte brut avant analyse.
- Outlook : l'étiquetage crée des catégories `Agent — …` ; leur couleur se règle dans Outlook.
- Un jeton Google expire au bout d'une heure ; il suffit de se reconnecter.
- `index-v1-charte.html.bak` conserve la première version, entièrement en charte France IA.
