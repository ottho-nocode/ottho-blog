---
name: setup-ghost
description: Guide pas-à-pas pour mettre Ghost en place sur PikaPods, choisir où vivra le blog (URL PikaPods par défaut OU sous-domaine custom), créer les API keys et installer le MCP Ghost. Génère ghost-config.md à la racine du projet.
---

# /blog:setup-ghost — Installation de Ghost sur PikaPods

Tu vas accompagner l'utilisateur dans la mise en place de son instance Ghost. À la fin de cette commande, l'utilisateur aura un blog accessible **soit à son URL PikaPods par défaut** (ex. `wonderful-caribou.pikapod.net`, marche immédiatement, pas de DNS), **soit à `blog.<son-domaine-custom>`** (s'il possède un nom de domaine).

**Charge la skill `ghost-config`** pour les détails techniques (création compte PikaPods, configuration Custom Domain PikaPods, formats des API keys, dépannage SSL). Elle contient la procédure complète et toutes les valeurs à donner à l'utilisateur.

**Temps estimé** : ~15 min en URL PikaPods, ~25-30 min en sous-domaine custom (5-10 min de propagation DNS supplémentaires).

## ⚠️ Pourquoi PAS le subpath `<site>/blog` via Vercel rewrite

On a écarté l'option subpath (rewrites Vercel pointant `<site>/blog/*` vers PikaPods) parce que :

- Ghost a besoin que sa variable d'environnement `url` se termine par `/blog` pour générer correctement les liens canoniques, le sitemap et les liens internes du theme
- Le template Ghost de PikaPods **n'expose pas la variable `url` au niveau top-level** (seules `portal__url`, `sodoSearch__url`, `comments__url` sont éditables)
- PikaPods gère l'URL Ghost en interne via son système Custom Domain — tu ne peux pas l'override

Conséquence : un rewrite Vercel `/blog/*` → PikaPods produirait un Ghost qui croit toujours vivre à l'URL PikaPods, ce qui casse les `<link rel="canonical">`, les `<a href>` du theme, le sitemap et l'admin. **On ne propose donc pas cette option.**

## Préambule

Ouvre la commande par ce message :

> « On va mettre ton blog Ghost en ligne. Le plan :
>
> 1. Vérifier que tu as les pré-requis
> 2. Choisir où vivra ton blog (URL PikaPods par défaut OU sous-domaine custom)
> 3. Créer un compte PikaPods et lancer une instance Ghost
> 4. Brancher l'URL choisie (si scénario sous-domaine) ou rien (si scénario PikaPods URL)
> 5. Créer ton compte admin Ghost
> 6. Générer les 2 API keys et installer le MCP Ghost dans Claude Code
> 7. (Optionnel) Activer le mode privé tant que le blog est vide
>
> Compte ~15 min si tu pars sur l'URL PikaPods, ~25-30 min si tu mets un sous-domaine custom (à cause de la propagation DNS).
>
> Prêt ? On y va. »

## Protocole

Pose les questions **UNE PAR UNE**, en attendant la réponse avant de passer à la suivante. Ne jamais empiler plusieurs questions dans un seul message.

### Vérification pré-requis

Avant tout, valide les 2 pré-requis ci-dessous. Si l'un manque, **mets en pause** et indique à l'utilisateur ce qu'il doit faire avant de revenir.

#### Pré-requis 1 — Cours « Claude + Site web »

> « Première vérification : as-tu déjà fini le cours « Claude + Site web » jusqu'au chapitre déploiement Vercel ? Tu n'as pas besoin d'avoir un site parfait, mais le cadrage du brief et la mise en ligne d'un premier site sont nécessaires pour cette suite. »

Si non : pause la commande et redirige :

> « Pas de souci. Va d'abord faire au moins le chapitre 1 (cadrage) et le chapitre déploiement Vercel du cours « Claude + Site web ». Tu reviens ici dès que tu as un site en ligne. »

#### Pré-requis 2 — Site déployé sur Vercel

> « Deuxième vérification : as-tu un site déployé sur Vercel ? L'URL peut être :
>
> - Une URL Vercel par défaut (ex. `mon-projet.vercel.app`)
> - Un nom de domaine custom (ex. `monsite.com`)
>
> Quelle est ton URL de site actuelle ? »

Stocke la réponse comme `<SITE_URL>`. Si l'utilisateur n'a pas encore de site déployé : pause et redirige vers le cours « Claude + Site web ».

#### Pré-requis 3 — Techno du site (CRITIQUE — ça change l'architecture)

> « Troisième vérification : quelle techno fait tourner ton site sur Vercel ? Ça change radicalement l'architecture du blog.
>
> **A. HTML / CSS / JS pur** — site statique sans framework JS côté serveur. Pas de `next.config.js`, pas de `astro.config.mjs`, pas de Server Components. Ton repo contient principalement des `.html`, `.css`, `.js` (et éventuellement un `package.json` minimal pour des outils de build). C'est le cas par défaut du cours « Claude + Site web ».
>
> **B. Framework JS avec rendu serveur (Next.js, Astro, SvelteKit, Nuxt, Remix)** — site qui peut faire du SSR (server-side rendering) ou du SSG (static site generation), capable de consommer une API REST côté serveur pour fabriquer ses propres pages.
>
> Quelle techno utilises-tu ? »

Stocke la réponse comme `<TECHNO>` = `A` ou `B`.

> Note : ton blog Ghost vivra à une URL **distincte** de ton site (URL PikaPods OU sous-domaine custom OU subpath `/blog` selon ta techno). Le choix exact arrive à l'étape suivante.

---

### Choix du scénario d'hébergement

#### Si `<TECHNO>` = A (HTML / CSS / JS pur)

> « Tu as 2 options. Ton site statique ne peut pas consommer l'API Ghost côté serveur, donc le subpath `<site>/blog` via API headless est exclu — il faudrait migrer vers un framework JS avant. Le choix dépend uniquement de si tu as un nom de domaine custom :
>
> **A. URL PikaPods par défaut** — *recommandé si tu n'as pas de nom de domaine custom*
>
> Ton blog vivra à l'URL technique fournie par PikaPods (ex. `https://wonderful-caribou.pikapod.net`). Pas de DNS à configurer, ça marche immédiatement après le déploiement.
>
> ✓ Aucune config DNS, fonctionne en 5 min
> ✓ HTTPS auto (Let's Encrypt géré par PikaPods)
> ✓ Zéro coût additionnel
> ✗ URL un peu technique côté visiteurs (mais on peut la masquer en ajoutant un lien « Blog » dans la nav du site principal qui pointe vers cette URL)
>
> **B. Sous-domaine custom `blog.<domaine-custom>`** — *recommandé si tu possèdes un nom de domaine*
>
> Ton blog vivra à `https://blog.monsite.com`. On ajoute un CNAME chez ton registrar (OVH, Gandi, etc.) qui pointe `blog` vers PikaPods, et PikaPods s'occupe du HTTPS via son onglet **Custom Domain**.
>
> ✓ Branding propre (subdomain = légitimité perçue)
> ✓ SEO cohérent avec ton site principal
> ✗ Demande un nom de domaine custom (impossible avec `mon-projet.vercel.app`)
> ✗ DNS à propager (~10 min)
>
> Lequel tu choisis ? »

Stocke la réponse comme `<SCENARIO>` = `A` ou `B`.

#### Si `<TECHNO>` = B (Next.js / Astro / SvelteKit / Nuxt / Remix)

> « Tu as 3 options. Ton framework peut consommer l'API Ghost côté serveur (SSR/SSG), donc tu peux héberger le blog directement à `<ton-site>/blog` (option C) — c'est l'architecture la plus propre quand on a un framework JS.
>
> **A. URL PikaPods par défaut** — *option simple sans config*
>
> Ton blog vivra à l'URL technique fournie par PikaPods (ex. `https://wonderful-caribou.pikapod.net`). Pas de DNS, pas de code à écrire.
>
> ✓ 5 min de setup, marche immédiatement
> ✓ Ghost est servi tel quel par PikaPods
> ✗ URL technique côté visiteurs
>
> **B. Sous-domaine custom `blog.<domaine-custom>`** — *recommandé si tu as un domaine custom et que tu ne veux pas toucher au code de ton framework*
>
> Ton blog vivra à `https://blog.monsite.com`. CNAME → PikaPods, HTTPS auto.
>
> ✓ Branding propre
> ✓ Pas de code à écrire dans ton framework
> ✗ Demande un nom de domaine custom
> ✗ DNS à propager (~10 min)
>
> **C. Headless API — `<ton-site>/blog` rendu par ton framework** — *URL la plus propre, mais demande d'écrire du code*
>
> Ghost reste sur PikaPods (URL native), ton framework récupère les articles via la Content API et fabrique les pages `/blog` et `/blog/<slug>` lui-même. Tu ajoutes une lib (ex. `lib/ghost.ts`), des routes `app/blog/page.tsx` + `app/blog/[slug]/page.tsx` (Next.js) ou équivalent (Astro/SvelteKit), un sitemap, des canonical, du JSON-LD. La commande `/blog:integrate-headless` te scaffolde tout ça automatiquement.
>
> ✓ URL la plus propre (`<ton-site>/blog`), SEO unifié
> ✓ Tu maîtrises 100% du rendu (theme = ton site)
> ✓ Le plugin scaffolde tout via `/blog:integrate-headless` (Next.js 16 supporté en V1 ; Astro/SvelteKit en V1.5 avec adaptation manuelle depuis le code de démonstration fourni par le formateur)
> ✗ Demande d'écrire ~150 lignes de code dans ton framework (la commande s'en charge si tu es sur Next.js)
> ✗ Le theme Ghost (chapitre 2) ne sert plus à rien — tu rends avec le HTML/CSS de ton site
>
> Lequel tu choisis ? »

Stocke la réponse comme `<SCENARIO>` = `A`, `B` ou `C`.

Si `<SCENARIO>` = `A` :
- L'URL publique du blog sera `<PIKAPOD_URL>` (récupérée à l'étape 2)
- `<BLOG_URL>` sera renseignée à l'étape 2 dès qu'on connait l'URL technique

Si `<SCENARIO>` = `B` :
- Demande : « Quel sous-domaine custom veux-tu ? Par défaut je propose `blog.<DOMAIN>` — c'est l'usage standard. Tu peux aussi prendre `journal.<DOMAIN>` ou `articles.<DOMAIN>` si tu préfères. »
- Stocke `<SUBDOMAIN>` (ex. `blog.monsite.com`) et `<BLOG_URL>` = `https://<SUBDOMAIN>`

Si `<SCENARIO>` = `C` (headless API, `<TECHNO>` = B uniquement) :
- L'URL publique du blog sera `<SITE_URL>/blog` (Ghost reste à `<PIKAPOD_URL>` côté backend)
- Stocke `<BLOG_URL>` = `https://<SITE_URL>/blog` et `<GHOST_BACKEND_URL>` = `https://<PIKAPOD_URL>`
- ⚠️ **Important** : `/blog:theme` (chapitre 2) est inutile dans ce scénario — tu rendras les articles avec le HTML/CSS de ton site existant. Tu peux skipper `/blog:theme` et passer directement à `/blog:cocon`. La commande `/blog:integrate-headless` génère le code Next.js (V1) ou guide vers une intégration manuelle pour Astro/SvelteKit (V1.5).

---

### Étape 1 — Compte PikaPods

> « On commence. As-tu déjà un compte PikaPods ? »

**Si non** :

> « Va sur https://www.pikapods.com et clique sur **Sign up**. Crée ton compte avec un email et un mot de passe fort (utilise un gestionnaire de mots de passe). PikaPods offre des crédits gratuits à l'inscription, suffisants pour valider l'installation.
>
> Dis-moi quand c'est fait. »

Attends la confirmation avant de passer à l'étape 2.

**Si oui** : passe directement à l'étape 2.

---

### Étape 2 — Lancer l'instance Ghost

> « Une fois connecté à PikaPods :
>
> 1. **Catalog** → tape « Ghost » dans la recherche → clique sur la fiche Ghost
> 2. Bouton **Run pod**
> 3. **Taille** : **S (256 MB RAM)** — largement suffisant pour démarrer
> 4. **Région** : **Europe** (Allemagne ou Pays-Bas) — meilleure latence + RGPD direct
> 5. Clique **Deploy**
>
> Le pod démarre en 2-3 min. Tu obtiens une URL technique du type `https://wonderful-caribou.pikapod.net`.
>
> Donne-moi cette URL technique dès que ton instance est en ligne. »

Stocke la réponse comme `<PIKAPOD_URL>` (ex. `wonderful-caribou.pikapod.net`).

Si `<SCENARIO>` = `A` :
- Stocke `<BLOG_URL>` = `https://<PIKAPOD_URL>`
- Passe directement à l'étape 4 (étape 3 = config sous-domaine, inutile ici)

Si `<SCENARIO>` = `B` :
- Continue à l'étape 3 (DNS + Custom Domain)

Si `<SCENARIO>` = `C` (headless API) :
- Stocke `<GHOST_BACKEND_URL>` = `https://<PIKAPOD_URL>` (l'URL Ghost technique, jamais visible des visiteurs)
- `<BLOG_URL>` reste `https://<SITE_URL>/blog` (à rendre par le framework de l'élève — pas par cette commande)
- Passe directement à l'étape 4 (étape 3 inutile, on garde l'URL PikaPods côté backend)

---

### Étape 3 — Brancher le sous-domaine custom (Scénario B uniquement)

**À sauter intégralement si `<SCENARIO>` = A.**

> « On va dire à PikaPods et à ton registrar que `<SUBDOMAIN>` doit pointer sur ton instance Ghost.
>
> **Côté registrar (OVH, Gandi, Namecheap, Cloudflare DNS, Infomaniak)** :
>
> 1. Ouvre la zone DNS de `<DOMAIN>`
> 2. Crée un enregistrement **CNAME** :
>    - **Nom** : le préfixe que tu as choisi (ex. `blog`)
>    - **Cible** : `<PIKAPOD_URL>`
>    - **TTL** : 3600 (1 h) ou la valeur par défaut
> 3. Sauvegarde
>
> La propagation prend 10 min à 1 h. Pour vérifier :
>
> ```bash
> dig +short <SUBDOMAIN>
> ```
>
> Tu dois voir `<PIKAPOD_URL>` apparaître. Tu peux aussi utiliser https://dnschecker.org en y entrant `<SUBDOMAIN>` (type CNAME).
>
> **Côté PikaPods** (une fois la propagation faite) :
>
> 1. PikaPods dashboard → ton pod Ghost → onglet **Custom Domain** (ou **Domains**)
> 2. **Add domain** → entre `<SUBDOMAIN>` → valide
> 3. PikaPods provisionne automatiquement un certificat HTTPS via Let's Encrypt (~2 min)
>
> Test : ouvre `https://<SUBDOMAIN>` dans un onglet privé. Tu dois voir l'écran d'accueil Ghost « Welcome ».
>
> ⚠️ Si tu vois une erreur SSL ou un timeout, attends 5-10 min de plus — Let's Encrypt peut être lent à la première génération. Si ça persiste au-delà de 15 min, vérifie que ton CNAME pointe bien sur `<PIKAPOD_URL>`.
>
> Confirme-moi quand `https://<SUBDOMAIN>` répond avec l'écran Ghost. »

Attends la confirmation avant de passer à l'étape 4.

---

### Étape 4 — Premier login Ghost admin

> « On crée ton compte admin :
>
> 1. URL admin : `<BLOG_URL>/ghost`
> 2. Crée le compte **owner** :
>    - Email : ton email pro
>    - Nom : prénom + nom
>    - Mot de passe : fort (12+ caractères, mélange casse + chiffres + symboles)
>
> Une fois connecté, fais tout de suite la config initiale (Settings → General) :
>
> - **Site title** : nom du blog (ex. « Mon Blog »)
> - **Site description** : 1 phrase, 150 caractères max — c'est ta meta description par défaut
> - **Timezone** : `Europe/Paris`
> - **Publication language** : `fr`
> - **Navigation** : 3-4 entrées max pour démarrer (Accueil, À propos, Contact)
>
> Confirme-moi quand le compte admin est créé et que la config initiale est faite. »

Attends la confirmation.

---

### Étape 5 — Custom Integration (les 2 API keys)

C'est l'étape qui débloque l'usage de Ghost depuis Claude Code.

> « On crée maintenant l'intégration qui va te donner les 2 API keys nécessaires pour brancher Ghost à Claude Code :
>
> 1. Ghost admin → **Settings → Integrations** (icône de prise dans la barre latérale)
> 2. Section **Custom integrations** → bouton **Add custom integration**
> 3. **Nom** : `Claude Code`
> 4. Valide
>
> Tu vois maintenant 3 valeurs critiques :
>
> - **Content API Key** — 24 caractères hexadécimaux (ex. `2b1c4d5e6f7a8b9c0d1e2f3a`). C'est la clé **publique en lecture seule** — utilisée pour afficher les articles côté front.
> - **Admin API Key** — format `id:secret` (ex. `5a1b...:6c2d...`, où `id` = 24 hex et `secret` = 64 hex). C'est la clé **privée en lecture/écriture** — utilisée pour publier des articles depuis Claude Code.
> - **API URL** — c'est `<BLOG_URL>`
>
> ⚠️ **Sécurité critique** : ces 2 clés sont des secrets. Tu ne les commites JAMAIS dans git. L'Admin API Key permet de tout faire sur ton blog, y compris supprimer des articles.
>
> Stocke-les temporairement pour la prochaine étape (clipboard, ou note locale dans un fichier non commité). On va les passer à la config MCP juste après — pas besoin de les coller dans le chat ici, je n'ai pas besoin de les voir. »

Attends la confirmation que l'intégration est créée et que l'utilisateur a les 3 valeurs sous la main.

---

### Étape 6 — Installer le MCP Ghost dans Claude Code

> « On branche maintenant Claude Code à ton instance Ghost via le MCP Ghost. Une fois installé, tu pourras me demander « liste mes 5 derniers articles » ou « publie cet article » et je pourrai le faire directement.
>
> Pour l'install, lance cette commande dans Claude Code :
>
> ```
> /mcp install ghost
> ```
>
> *(Note : ton formateur te fournira le repo MCP Ghost officiel ou un fork compatible avec l'Admin API + la Content API si la commande ne marche pas du premier coup.)*
>
> Le serveur MCP va te demander 3 valeurs — c'est ici que tu colles les clés de l'étape 5 :
>
> 1. **URL Ghost** : `<BLOG_URL>`
> 2. **Admin API Key** : la clé `id:secret`
> 3. **Content API Key** : la clé hex
>
> Une fois la config validée, teste avec :
>
> ```
> /mcp ping ghost
> ```
>
> Tu dois recevoir une réponse `OK`. Si tu obtiens une erreur 401, tes clés ne sont pas correctes — retourne dans Settings → Integrations dans Ghost admin pour les régénérer et recommence.
>
> Test fonctionnel : demande-moi « liste mes 5 derniers articles Ghost ». Je dois pouvoir te répondre avec les articles de démo Ghost (« Coming soon », « Start here », etc.).
>
> Confirme-moi quand le MCP répond et que tu vois bien tes articles. »

Attends la confirmation.

---

### Étape 7 — Mode privé (optionnel)

> « Dernière étape, optionnelle mais recommandée : tant que ton blog est vide ou en chantier, tu ne veux pas que Google indexe les 3 articles de démo Ghost. Veux-tu activer le mode privé pour l'instant ? »

**Si oui** :

> « Va dans Ghost admin → **Settings → General → Make this site private** → active le toggle **Private**. Ghost te génère un mot de passe partagé que tu peux donner aux personnes qui veulent voir le site en preview.
>
> ⚠️ Important : désactive ce mode dès que tu publies ton premier vrai article — sinon Google ne pourra pas l'indexer. On en reparlera dans le chapitre Production.
>
> Confirme quand c'est activé. »

Attends la confirmation, et stocke `<PRIVATE_MODE>` = `oui`.

**Si non** :

> « OK, on laisse le blog public. Note que les 3 articles de démo Ghost sont toujours là — on les supprimera dans la commande `/blog:article` quand tu publieras ton premier vrai article. »

Stocke `<PRIVATE_MODE>` = `non`.

---

## Génération de `ghost-config.md`

Une fois les 7 étapes validées, **génère** un fichier `ghost-config.md` à la racine du projet de l'étudiant (le dossier courant). **Aucun secret ne doit y figurer** — uniquement les URLs et l'état de la config.

Structure (adapte selon `<SCENARIO>`) :

```markdown
# Ghost — Configuration

Date setup : <YYYY-MM-DD>

## Stack du site hôte

- **Techno** : <HTML/CSS/JS pur | Next.js | Astro | SvelteKit | Nuxt | Remix>
- **Site web (Vercel)** : <SITE_URL>

## Scénario d'hébergement du blog

- **Type** : <URL PikaPods par défaut | sous-domaine custom | headless API rendu par le framework>
- **URL publique du blog** : <BLOG_URL>
- **Admin Ghost** : <GHOST_BACKEND_URL ou BLOG_URL>/ghost
- **Instance backend (PikaPods)** : <PIKAPOD_URL>

## Mode privé

- Activé : <oui / non>

## API

- **Admin API Key** : *(stockée dans la config MCP, pas ici)*
- **Content API Key** : *(idem)*

## MCP Ghost

- Installé dans Claude Code : oui
- Test ping : OK

## Lien depuis le site Vercel

À ajouter dans la nav de ton site Vercel (à la fin du chapitre 2 — theme) :

```html
<a href="<BLOG_URL>" target="_blank" rel="noopener">Blog</a>
```

## Headless API *(scénario C uniquement)*

- **Ghost backend (jamais visible)** : <GHOST_BACKEND_URL>
- **URL publique rendue par le framework** : <SITE_URL>/blog
- **Lib à écrire** : `lib/ghost.ts` (fetch Content API — généré automatiquement par `/blog:integrate-headless` en Next.js)
- **Routes à créer** : `/blog` (liste) + `/blog/<slug>` (détail) — selon ton framework
- **Sitemap** : merger les routes statiques + les slugs Ghost (`getAllPosts()`)
- **Revalidation** : webhook Ghost → endpoint sur ton site qui appelle `revalidateTag('blog')` ou équivalent
- **Theme Ghost (chapitre 2)** : skippable — non utilisé en scénario C

## Webhooks (avancé, optionnel)

- Aucun pour l'instant — à configurer au chapitre Production si tu utilises un cache ISR Next/Astro qui consomme l'API Ghost.

---

Setup généré par `/blog:setup-ghost`.
```

Remplace `<YYYY-MM-DD>` par la date du jour.

## Vérifications finales

Avant d'écrire le fichier `ghost-config.md` et de clôturer, valide la checklist suivante avec l'utilisateur :

- [ ] `<BLOG_URL>` répond avec l'écran d'accueil Ghost (ou la page publique si mode privé désactivé)
- [ ] Compte admin Ghost créé, config initiale faite (title, description, timezone, langue)
- [ ] 2 API keys (Admin + Content) générées via Settings → Integrations → Claude Code
- [ ] Les clés sont stockées **uniquement dans la config MCP** — jamais dans le code, jamais dans git
- [ ] MCP Ghost installé et `mcp ping ghost` répond OK
- [ ] Test fonctionnel passé : Claude Code peut lister les articles Ghost
- [ ] Si `<SCENARIO>` = B : `dig +short <SUBDOMAIN>` retourne `<PIKAPOD_URL>` et le Custom Domain est validé côté PikaPods
- [ ] `ghost-config.md` écrit à la racine du projet (sans secrets)

Si un critère n'est pas rempli, **retourne sur l'étape correspondante** avant d'écrire le fichier.

## Clôture

Une fois le fichier écrit :

> « Ghost est en ligne sur `<BLOG_URL>` et connecté à Claude Code via le MCP. C'est ta plateforme de publication.
>
> Récap de ce qu'on a fait :
> - Instance Ghost lancée sur PikaPods (région Europe, RGPD-compliant)
> - Blog accessible via <URL PikaPods | sous-domaine custom>, HTTPS auto
> - Compte admin créé, config initiale faite
> - 2 API keys générées et configurées dans le MCP Ghost
> - `ghost-config.md` écrit à la racine du projet pour mémoire
>
> Prochaine étape : `/blog:theme` pour générer un theme Ghost qui matche la charte graphique de ton site existant. »
