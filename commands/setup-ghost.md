---
name: setup-ghost
description: Guide pas-à-pas pour mettre Ghost en place sur PikaPods, brancher le sous-domaine, créer les API keys et installer le MCP Ghost. Génère ghost-config.md à la racine du projet.
---

# /blog:setup-ghost — Installation de Ghost sur PikaPods

Tu vas accompagner l'utilisateur dans la mise en place de son instance Ghost. À la fin de cette commande, l'utilisateur aura un blog hébergé sur `blog.tonsite.com`, connecté à Claude Code via le MCP Ghost.

**Charge la skill `ghost-config`** pour les détails techniques (URLs PikaPods, configuration DNS, formats des API keys, dépannage SSL). Elle contient la procédure complète et toutes les valeurs à donner à l'utilisateur.

**Temps estimé** : ~30 min (dont 10 min d'attente pour la propagation DNS).

## Préambule

Ouvre la commande par ce message :

> « On va mettre ton blog Ghost en ligne. Le plan :
>
> 1. Vérifier que tu as bien les pré-requis
> 2. Créer un compte PikaPods et lancer une instance Ghost
> 3. Brancher ton sous-domaine `blog.tonsite.com`
> 4. Créer ton compte admin Ghost
> 5. Générer les 2 API keys et installer le MCP Ghost dans Claude Code
> 6. (Optionnel) Activer le mode privé tant que le blog est vide
>
> Compte ~30 min en tout, dont 5 à 10 min d'attente pour la propagation DNS. Tu auras besoin d'ouvrir ton navigateur pour PikaPods et l'interface de ton registrar de domaine.
>
> Prêt ? On y va. »

## Protocole

Pose les questions **UNE PAR UNE**, en attendant la réponse avant de passer à la suivante. Ne jamais empiler plusieurs questions dans un seul message.

### Vérification pré-requis

Avant tout, valide les 3 pré-requis ci-dessous. Si l'un manque, **mets en pause** et indique à l'utilisateur ce qu'il doit faire avant de revenir.

#### Pré-requis 1 — Cours « Claude + Site web »

> « Première vérification : as-tu déjà fini le cours « Claude + Site web » jusqu'au chapitre déploiement Vercel ? Tu n'as pas besoin d'avoir un site parfait, mais le cadrage du brief et la mise en ligne d'un premier site sont nécessaires pour cette suite. »

Si non : pause la commande et redirige :

> « Pas de souci. Va d'abord faire au moins le chapitre 1 (cadrage) et le chapitre déploiement Vercel du cours « Claude + Site web ». Tu reviens ici dès que tu as un site en ligne avec un nom de domaine. »

#### Pré-requis 2 — Nom de domaine actif

> « Deuxième vérification : as-tu un nom de domaine actif que tu contrôles ? (ex. `monsite.com`, `prenom-nom.fr`, etc.) »

Si non : pause et redirige :

> « Il te faut un domaine pour brancher ton blog dessus. Achète-en un chez OVH, Gandi, Namecheap ou Cloudflare DNS (~10 €/an pour un .com), puis reviens. »

#### Pré-requis 3 — Accès au registrar

> « Troisième vérification : as-tu accès à l'interface de ton registrar (OVH, Gandi, Namecheap, Cloudflare DNS, Infomaniak) pour pouvoir ajouter un enregistrement DNS de type CNAME ? »

Si non : pause et redirige :

> « Récupère les identifiants de ton registrar avant de continuer. On aura besoin d'ajouter un CNAME pour pointer `blog.tonsite.com` vers ton instance PikaPods. »

Une fois les 3 pré-requis validés, demande aussi le domaine de l'utilisateur — tu vas t'en servir tout au long de la commande :

> « Quel est ton nom de domaine ? (ex. `monsite.com`) — je vais m'en servir pour personnaliser les instructions. »

Stocke la réponse comme `<DOMAIN>` (ex. `monsite.com`) et utilise-la dans tous les exemples qui suivent (au lieu de `tonsite.com`).

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

### Étape 3 — Sous-domaine `blog.<DOMAIN>`

> « Maintenant on branche ton sous-domaine. Quel sous-domaine veux-tu ? Par défaut je propose `blog.<DOMAIN>` — c'est l'usage standard et c'est le plus court. Tu peux aussi prendre `journal.<DOMAIN>` ou `articles.<DOMAIN>` si tu préfères. »

Attends la réponse, stocke-la comme `<SUBDOMAIN>` (ex. `blog.monsite.com`).

> « Ouvre l'interface de ton registrar et va dans la zone DNS de `<DOMAIN>`. Crée un enregistrement **CNAME** :
>
> - **Nom** : `blog` (ou le préfixe que tu as choisi)
> - **Cible** : `<PIKAPOD_URL>`
> - **TTL** : 3600 (1 h) ou laisse la valeur par défaut
>
> Sauvegarde. La propagation prend 10 min à 1 h.
>
> Pour vérifier la propagation, tu as deux options :
>
> 1. En ligne de commande :
>    ```bash
>    dig +short <SUBDOMAIN>
>    ```
>    Tu dois voir `<PIKAPOD_URL>` apparaître.
>
> 2. Sans terminal : va sur https://dnschecker.org, entre `<SUBDOMAIN>` et choisis « CNAME » dans le menu déroulant. Tu dois voir l'enregistrement résolu sur les serveurs du monde entier.
>
> Confirme-moi quand le CNAME est créé et que `dig` ou dnschecker.org te montre bien la cible. »

Attends la confirmation avant de passer à l'étape 4.

---

### Étape 4 — Custom domain dans PikaPods

> « Maintenant on dit à PikaPods que `<SUBDOMAIN>` est bien le tien :
>
> 1. PikaPods dashboard → ton pod Ghost → onglet **Domains** (ou **Settings** selon la version de l'interface)
> 2. **Add domain** → entre `<SUBDOMAIN>` → valide
> 3. PikaPods provisionne automatiquement un certificat HTTPS via Let's Encrypt (~2 min)
>
> Test : ouvre `https://<SUBDOMAIN>` dans un onglet privé. Tu dois voir l'écran d'accueil Ghost « Welcome » qui te demande de créer le compte admin.
>
> ⚠️ Si tu vois une erreur SSL ou un timeout, attends 5-10 min de plus — Let's Encrypt peut être lent à la première génération. Si ça persiste au-delà de 15 min, vérifie que ton CNAME pointe bien sur `<PIKAPOD_URL>` (pas sur autre chose).
>
> Confirme-moi quand `https://<SUBDOMAIN>` répond avec l'écran Ghost. »

Attends la confirmation.

---

### Étape 5 — Premier login Ghost admin

> « On crée ton compte admin :
>
> 1. URL : `https://<SUBDOMAIN>/ghost`
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
> - **API URL** — c'est `https://<SUBDOMAIN>`
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
> 1. **URL Ghost** : `https://<SUBDOMAIN>`
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

Structure :

```markdown
# Ghost — Configuration

Date setup : <YYYY-MM-DD>

## URL

- **Public** : https://<SUBDOMAIN>
- **Admin** : https://<SUBDOMAIN>/ghost
- **Instance backend (PikaPods)** : <PIKAPOD_URL>

## Mode privé

- Activé : <oui / non>

## API

- **Admin API Key** : *(stockée dans la config MCP, pas ici)*
- **Content API Key** : *(idem)*

## MCP Ghost

- Installé dans Claude Code : oui
- Test ping : OK

## Webhooks (avancé, optionnel)

- Aucun pour l'instant — à configurer au chapitre Production si tu utilises un cache ISR (Next.js, Astro).

---

Setup généré par `/blog:setup-ghost`.
```

Remplace `<YYYY-MM-DD>` par la date du jour, `<SUBDOMAIN>` par le sous-domaine choisi, `<PIKAPOD_URL>` par l'URL technique PikaPods, et `<oui / non>` par la valeur de `<PRIVATE_MODE>`.

## Vérifications finales

Avant d'écrire le fichier `ghost-config.md` et de clôturer, valide la checklist suivante avec l'utilisateur :

- [ ] `https://<SUBDOMAIN>` répond avec l'écran d'accueil Ghost (ou la page publique du blog si le mode privé est désactivé)
- [ ] Compte admin Ghost créé, config initiale faite (title, description, timezone, langue)
- [ ] 2 API keys (Admin + Content) générées via Settings → Integrations → Claude Code
- [ ] Les clés sont stockées **uniquement dans la config MCP** — jamais dans le code, jamais dans git
- [ ] MCP Ghost installé et `mcp ping ghost` répond OK
- [ ] Test fonctionnel passé : Claude Code peut lister les articles Ghost
- [ ] `ghost-config.md` écrit à la racine du projet (sans secrets)

Si un critère n'est pas rempli, **retourne sur l'étape correspondante** avant d'écrire le fichier.

## Clôture

Une fois le fichier écrit :

> « Ghost est en ligne sur `https://<SUBDOMAIN>` et connecté à Claude Code via le MCP. C'est ta plateforme de publication.
>
> Récap de ce qu'on a fait :
> - Instance Ghost lancée sur PikaPods (région Europe, RGPD-compliant)
> - Sous-domaine `<SUBDOMAIN>` branché avec HTTPS auto via Let's Encrypt
> - Compte admin créé, config initiale faite
> - 2 API keys générées et configurées dans le MCP Ghost
> - `ghost-config.md` écrit à la racine du projet pour mémoire
>
> Prochaine étape : `/blog:theme` pour générer un theme Ghost qui matche la charte graphique de ton site existant. »
