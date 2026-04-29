---
name: setup-ghost
description: Guide pas-à-pas pour mettre Ghost en place sur PikaPods, brancher le blog au site existant (subpath /blog par défaut, ou sous-domaine custom), créer les API keys et installer le MCP Ghost. Génère ghost-config.md à la racine du projet.
---

# /blog:setup-ghost — Installation de Ghost sur PikaPods

Tu vas accompagner l'utilisateur dans la mise en place de son instance Ghost. À la fin de cette commande, l'utilisateur aura un blog accessible **soit à `<son-site>/blog`** (par défaut, marche avec n'importe quelle URL Vercel y compris `<projet>.vercel.app`), **soit à `blog.<son-domaine-custom>`** (alternative pour qui possède un nom de domaine).

**Charge la skill `ghost-config`** pour les détails techniques (URLs PikaPods, configuration DNS, formats des API keys, dépannage SSL, rewrites Vercel). Elle contient la procédure complète et toutes les valeurs à donner à l'utilisateur.

**Temps estimé** : ~20-30 min selon le scénario choisi (le subpath via Vercel rewrite est plus rapide ; le sous-domaine custom demande 5-10 min de propagation DNS).

## Préambule

Ouvre la commande par ce message :

> « On va mettre ton blog Ghost en ligne. Le plan :
>
> 1. Vérifier que tu as bien les pré-requis
> 2. Choisir où vivra ton blog (subpath `/blog` ou sous-domaine custom)
> 3. Créer un compte PikaPods et lancer une instance Ghost
> 4. Brancher le blog à ton site existant
> 5. Créer ton compte admin Ghost
> 6. Générer les 2 API keys et installer le MCP Ghost dans Claude Code
> 7. (Optionnel) Activer le mode privé tant que le blog est vide
>
> Compte ~20-30 min en tout. Tu auras besoin d'ouvrir ton navigateur pour PikaPods. Si tu choisis le sous-domaine custom, prévois 5-10 min de plus pour la propagation DNS.
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

> « Deuxième vérification : as-tu un site déployé sur Vercel auquel tu as accès via le CLI Vercel et le code source ? L'URL peut être :
>
> - Une URL Vercel par défaut (ex. `mon-projet.vercel.app`) — c'est le cas par défaut, ça marche très bien
> - Un nom de domaine custom (ex. `monsite.com`)
>
> Quelle est ton URL de site actuelle ? »

Stocke la réponse comme `<SITE_URL>` (ex. `mon-projet.vercel.app` ou `monsite.com`). Si l'utilisateur n'a pas encore de site déployé : pause et redirige vers le cours « Claude + Site web ».

---

### Choix du scénario

> « Maintenant, deux options pour héberger ton blog. Le choix dépend uniquement de ton URL :
>
> **A. Subpath `<son-site>/blog`** — *recommandé par défaut, marche pour tout le monde*
>
> Ton blog vivra à `https://<SITE_URL>/blog`. C'est-à-dire qu'on ajoute un *rewrite* dans `vercel.json` qui dit à Vercel : « si quelqu'un demande `/blog/*`, va chercher la réponse sur l'instance Ghost ». Le visiteur ne voit qu'une seule URL (la tienne), Ghost est invisible côté infrastructure.
>
> ✓ Marche avec `mon-projet.vercel.app/blog` ou `monsite.com/blog`
> ✓ Pas de DNS à configurer
> ✓ SEO unifié (un seul domaine = autorité topique unique)
> ✗ Demande d'éditer `vercel.json` (1 fichier dans le repo de ton site)
>
> **B. Sous-domaine custom `blog.<domaine-custom>`** — *alternative pour qui a un nom de domaine*
>
> Ton blog vivra à `https://blog.monsite.com`. On ajoute un CNAME chez ton registrar (OVH, Gandi, etc.) qui pointe `blog` vers PikaPods, et PikaPods s'occupe du HTTPS.
>
> ✓ Branding propre (subdomain = legitimité perçue)
> ✗ Demande un nom de domaine custom (impossible avec `mon-projet.vercel.app`)
> ✗ DNS à propager (~10 min)
>
> Lequel tu choisis ? »

Stocke la réponse comme `<SCENARIO>` = `A` ou `B`.

Si `<SCENARIO>` = `A` :
- L'URL publique du blog sera `https://<SITE_URL>/blog`
- Stocke `<BLOG_URL>` = `https://<SITE_URL>/blog`

Si `<SCENARIO>` = `B` :
- Demande : « Quel sous-domaine custom veux-tu ? Par défaut je propose `blog.<DOMAIN>` — c'est l'usage standard. Tu peux aussi prendre `journal.<DOMAIN>` ou `articles.<DOMAIN>` si tu préfères. »
- Stocke `<SUBDOMAIN>` (ex. `blog.monsite.com`) et `<BLOG_URL>` = `https://<SUBDOMAIN>`

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

---

### Étape 3 — Brancher le blog au site

C'est ici que les 2 scénarios divergent.

#### Si `<SCENARIO>` = A (subpath `/blog`)

> « On va ajouter un *rewrite* dans le `vercel.json` de ton site pour que `/blog/*` aille chercher la réponse sur PikaPods.
>
> Ouvre le fichier `vercel.json` à la racine de ton projet de site (s'il n'existe pas, on le crée). Ajoute (ou complète) la section `rewrites` :
>
> ```json
> {
>   "rewrites": [
>     {
>       "source": "/blog",
>       "destination": "https://<PIKAPOD_URL>/blog"
>     },
>     {
>       "source": "/blog/:path*",
>       "destination": "https://<PIKAPOD_URL>/blog/:path*"
>     }
>   ]
> }
> ```
>
> Si ton `vercel.json` contient déjà d'autres clés (`headers`, `redirects`, etc.), garde-les et ajoute juste `rewrites` à côté.
>
> Sauvegarde, commit le fichier, pousse sur la branche déployée :
>
> ```bash
> git add vercel.json
> git commit -m "feat: rewrite /blog/* to ghost"
> git push
> ```
>
> Vercel redéploie automatiquement en ~30 secondes. Confirme-moi quand c'est fait. »

Attends la confirmation. Note : à ce stade, `https://<SITE_URL>/blog` ne répondra pas encore correctement parce que Ghost n'est pas encore configuré pour servir à un subpath. On règle ça à l'étape 4.

#### Si `<SCENARIO>` = B (sous-domaine custom)

> « Ouvre l'interface de ton registrar (OVH, Gandi, Namecheap, Cloudflare DNS, Infomaniak) et va dans la zone DNS de `<DOMAIN>`. Crée un enregistrement **CNAME** :
>
> - **Nom** : le préfixe que tu as choisi (ex. `blog`)
> - **Cible** : `<PIKAPOD_URL>`
> - **TTL** : 3600 (1 h) ou la valeur par défaut
>
> Sauvegarde. La propagation prend 10 min à 1 h.
>
> Pour vérifier la propagation :
>
> 1. En ligne de commande :
>    ```bash
>    dig +short <SUBDOMAIN>
>    ```
>    Tu dois voir `<PIKAPOD_URL>` apparaître.
>
> 2. Sans terminal : va sur https://dnschecker.org, entre `<SUBDOMAIN>` et choisis « CNAME » dans le menu déroulant. Tu dois voir l'enregistrement résolu sur les serveurs du monde entier.
>
> Confirme-moi quand le CNAME est créé et propagé. »

Attends la confirmation avant de passer à l'étape 4.

---

### Étape 4 — Configurer Ghost à la bonne URL

#### Si `<SCENARIO>` = A (subpath)

> « On dit maintenant à Ghost qu'il vit à `<BLOG_URL>` (et plus à son URL technique PikaPods).
>
> 1. PikaPods dashboard → ton pod Ghost → onglet **Environment** (ou **Settings** selon la version de l'interface)
> 2. Cherche la variable d'environnement **`url`** (ou **`SITE_URL`** selon les images PikaPods)
> 3. Mets la valeur : `<BLOG_URL>` (avec slash final OK : `https://<SITE_URL>/blog/`)
> 4. Sauvegarde et redémarre le pod (PikaPods → Restart)
>
> Le redémarrage prend 30 sec. Tu peux maintenant tester :
>
> ```bash
> curl -I https://<SITE_URL>/blog
> ```
>
> Tu dois recevoir un `200 OK` avec du HTML Ghost. Si tu vois une erreur `502` ou un timeout, c'est probablement que :
> - Le rewrite Vercel n'est pas encore propagé (attends 1-2 min)
> - L'instance Ghost n'a pas redémarré avec la nouvelle URL (vérifie côté PikaPods)
>
> Confirme-moi quand `https://<SITE_URL>/blog` répond avec l'écran Ghost. »

#### Si `<SCENARIO>` = B (sous-domaine)

> « On dit maintenant à PikaPods que `<SUBDOMAIN>` est bien le tien :
>
> 1. PikaPods dashboard → ton pod Ghost → onglet **Domains** (ou **Settings**)
> 2. **Add domain** → entre `<SUBDOMAIN>` → valide
> 3. PikaPods provisionne automatiquement un certificat HTTPS via Let's Encrypt (~2 min)
>
> Test : ouvre `https://<SUBDOMAIN>` dans un onglet privé. Tu dois voir l'écran d'accueil Ghost « Welcome ».
>
> ⚠️ Si tu vois une erreur SSL ou un timeout, attends 5-10 min de plus — Let's Encrypt peut être lent à la première génération. Si ça persiste au-delà de 15 min, vérifie que ton CNAME pointe bien sur `<PIKAPOD_URL>`.
>
> Confirme-moi quand `https://<SUBDOMAIN>` répond avec l'écran Ghost. »

Attends la confirmation.

---

### Étape 5 — Premier login Ghost admin

> « On crée ton compte admin :
>
> 1. URL admin : `<BLOG_URL>/ghost`  (ex. `https://<SITE_URL>/blog/ghost` ou `https://<SUBDOMAIN>/ghost`)
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

### Étape 6 — Custom Integration (les 2 API keys)

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
> - **API URL** — c'est `<BLOG_URL>` (le subpath ou le sous-domaine selon ton scénario)
>
> ⚠️ **Sécurité critique** : ces 2 clés sont des secrets. Tu ne les commites JAMAIS dans git. L'Admin API Key permet de tout faire sur ton blog, y compris supprimer des articles.
>
> Stocke-les temporairement pour la prochaine étape (clipboard, ou note locale dans un fichier non commité). On va les passer à la config MCP juste après — pas besoin de les coller dans le chat ici, je n'ai pas besoin de les voir. »

Attends la confirmation que l'intégration est créée et que l'utilisateur a les 3 valeurs sous la main.

---

### Étape 7 — Installer le MCP Ghost dans Claude Code

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
> Le serveur MCP va te demander 3 valeurs — c'est ici que tu colles les clés de l'étape 6 :
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

### Étape 8 — Mode privé (optionnel)

> « Dernière étape, optionnelle mais recommandée : tant que ton blog est vide ou en chantier, tu ne veux pas que Google indexe les 3 articles de démo Ghost. Veux-tu activer le mode privé pour l'instant ? »

**Si oui** :

> « Va dans Ghost admin → **Settings → General → Make this site private** → active le toggle **Private**. Ghost te génère un mot de passe partagé que tu peux donner aux personnes qui veulent voir le site en preview.
>
> ⚠️ Important : désactive ce mode dès que tu publies ton premier vrai article — sinon Google ne pourra pas l'indexer. On en reparlera dans le chapitre Production.
>
> Confirme quand c'est activé. »

Attends la confirmation, et stocke `<PRIVATE_MODE>` = `oui`.

**Si non** :

> « OK, on laisse le blog public. Note que les 3 articles de démo Ghost sont toujours là — on les supprimera dans la commande `/blog:write` quand tu publieras ton premier vrai article. »

Stocke `<PRIVATE_MODE>` = `non`.

---

## Génération de `ghost-config.md`

Une fois les 8 étapes validées, **génère** un fichier `ghost-config.md` à la racine du projet de l'étudiant (le dossier courant). **Aucun secret ne doit y figurer** — uniquement les URLs et l'état de la config.

Structure (adapte selon `<SCENARIO>`) :

```markdown
# Ghost — Configuration

Date setup : <YYYY-MM-DD>

## Scénario d'hébergement

- **Type** : <subpath /blog via Vercel rewrite | sous-domaine custom>
- **Site web hôte** : <SITE_URL>
- **URL publique du blog** : <BLOG_URL>
- **Admin** : <BLOG_URL>/ghost
- **Instance backend (PikaPods)** : <PIKAPOD_URL>

## Mode privé

- Activé : <oui / non>

## API

- **Admin API Key** : *(stockée dans la config MCP, pas ici)*
- **Content API Key** : *(idem)*

## MCP Ghost

- Installé dans Claude Code : oui
- Test ping : OK

## Vercel rewrites *(scénario A uniquement)*

Ajoutés dans `vercel.json` :

- `/blog` → `<PIKAPOD_URL>/blog`
- `/blog/:path*` → `<PIKAPOD_URL>/blog/:path*`

Si tu modifies ces rewrites, redéploie le site sur Vercel pour qu'ils prennent effet.

## Webhooks (avancé, optionnel)

- Aucun pour l'instant — à configurer au chapitre Production si tu utilises un cache ISR.

---

Setup généré par `/blog:setup-ghost`.
```

Remplace `<YYYY-MM-DD>` par la date du jour. Pour la section « Vercel rewrites », **ne l'inclus que si `<SCENARIO>` = A** ; sinon supprime cette section du fichier.

## Vérifications finales

Avant d'écrire le fichier `ghost-config.md` et de clôturer, valide la checklist suivante avec l'utilisateur :

- [ ] `<BLOG_URL>` répond avec l'écran d'accueil Ghost (ou la page publique si mode privé désactivé)
- [ ] Compte admin Ghost créé, config initiale faite (title, description, timezone, langue)
- [ ] 2 API keys (Admin + Content) générées via Settings → Integrations → Claude Code
- [ ] Les clés sont stockées **uniquement dans la config MCP** — jamais dans le code, jamais dans git
- [ ] MCP Ghost installé et `mcp ping ghost` répond OK
- [ ] Test fonctionnel passé : Claude Code peut lister les articles Ghost
- [ ] Si `<SCENARIO>` = A : `vercel.json` mis à jour, commité, redéployé
- [ ] `ghost-config.md` écrit à la racine du projet (sans secrets)

Si un critère n'est pas rempli, **retourne sur l'étape correspondante** avant d'écrire le fichier.

## Clôture

Une fois le fichier écrit :

> « Ghost est en ligne sur `<BLOG_URL>` et connecté à Claude Code via le MCP. C'est ta plateforme de publication.
>
> Récap de ce qu'on a fait :
> - Instance Ghost lancée sur PikaPods (région Europe, RGPD-compliant)
> - Blog accessible via <subpath /blog | sous-domaine custom>, HTTPS auto
> - Compte admin créé, config initiale faite
> - 2 API keys générées et configurées dans le MCP Ghost
> - `ghost-config.md` écrit à la racine du projet pour mémoire
>
> Prochaine étape : `/blog:theme` pour générer un theme Ghost qui matche la charte graphique de ton site existant. »
