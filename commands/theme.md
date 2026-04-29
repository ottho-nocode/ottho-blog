---
name: theme
description: Génère un theme Ghost custom à partir de charte.md (fork Source + override CSS + adaptation default.hbs/post.hbs). Uploade et active dans Ghost via MCP. Output : dossier theme/ + ZIP + theme actif sur le blog.
---

# /blog:theme — Theme Ghost custom à partir de la charte

Tu vas accompagner l'étudiant dans la génération d'un theme Ghost sur mesure qui fait ressembler `blog.exemple.com` à `exemple.com` : même typographie, même palette, même nav, même ton.

La stratégie est claire et **non négociable** : on **fork le theme officiel Source** (TryGhost, MIT), on remplace les CSS variables par celles de la charte, on ajuste header + footer + post layout, on zippe, on uploade via MCP Ghost. Pas de réécriture from scratch.

> **Skill à charger** : avant toute action, lis et applique la skill `ghost-theme` (`skills/ghost-theme/SKILL.md` du plugin). Elle contient l'anatomie complète d'un theme Ghost, les 5 helpers Handlebars indispensables, le mapping `charte.md` → CSS variables, la procédure exacte de fork de Source, et les pièges à éviter (`{{ghost_head}}`, `{{ghost_foot}}`, `{{body_class}}`, ZIP racine).

## Pré-requis

Avant de démarrer, vérifie **dans cet ordre** :

1. **`/blog:setup-ghost` a été exécuté** — il existe un fichier `ghost-config.md` à la racine du projet courant.
   - Si non : stoppe et demande à l'étudiant de lancer `/blog:setup-ghost` d'abord. Cette commande dépend de l'instance Ghost configurée.
2. **MCP Ghost répond** — fais un appel léger (ex. `mcp__ghost__tags_browse` avec `{ "limit": 1 }`) pour vérifier que le serveur MCP Ghost est bien connecté et authentifié.
   - Si erreur : indique à l'étudiant de relancer Claude Code après avoir vérifié `~/.mcp.json` et la variable d'environnement `GHOST_ADMIN_API_KEY`.
3. **Tu te trouves bien à la racine du projet** — `pwd` doit pointer sur le dossier de l'étudiant (où vivent `brief.md`, `charte.md`, `ghost-config.md`).

Si une vérif échoue, n'avance pas. Corrige avant.

## Étape 0 — Vérification de `charte.md`

Pose **une seule question** à l'étudiant :

> « As-tu déjà un fichier `charte.md` dans ton projet ? Il doit contenir tes couleurs (palette principale + texte + fond), tes typographies (display + body, idéalement Google Fonts) et tes espacements de base. C'est la sortie du chapitre Design du cours précédent. »

Selon la réponse :

- **Oui** : lis `charte.md` à la racine. Si le fichier existe mais qu'il manque la palette **OU** les fonts, demande à l'étudiant de compléter avant de continuer (impossible de générer un theme cohérent sans ces deux blocs).
- **Non** : guide-le brièvement (3-4 lignes max — pas d'atelier complet, c'est le rôle du chapitre Design). Donne-lui ce squelette à remplir, puis attends qu'il revienne :

  ```markdown
  # Charte graphique

  ## Couleurs
  - Primary : #5C3BFF
  - Texte : #1F1D24
  - Texte soft : #6B6775
  - Fond : #FFFFFF
  - Fond soft : #F1F1F3
  - Accent : #D97757

  ## Typographies
  - Display (titres) : "Instrument Serif", Georgia, serif
  - Body (texte) : "Inter", system-ui, sans-serif
  - Mono (code) : "JetBrains Mono", ui-monospace, monospace

  ## Espacements
  - Section : 80px
  - Bloc : 32px
  - Radius cartes : 8px
  - Max-width prose : 720px
  ```

  Précise que les valeurs doivent **matcher exactement** celles du site existant (`exemple.com`) — c'est tout l'enjeu du theme.

Une fois `charte.md` validé, extrais-en mentalement les valeurs : tu vas les coller dans le bloc `:root` du CSS à l'étape 3.

## Étape 1 — Récupérer le theme Source

Source est le theme par défaut de Ghost, MIT, maintenu par TryGhost. On part d'un build propre depuis les releases (pas un clone du repo, qui contiendrait des fichiers de dev inutiles).

Annonce ce que tu fais, puis exécute :

```bash
# Depuis la racine du projet
mkdir -p theme
curl -L -o /tmp/source-theme.zip https://github.com/TryGhost/Source/releases/latest/download/source.zip
unzip -q /tmp/source-theme.zip -d theme/
rm /tmp/source-theme.zip
```

Si l'URL `latest/download/source.zip` ne répond pas (le nom de l'asset peut changer entre releases), fallback :

```bash
# Liste les assets de la dernière release et identifie le ZIP
curl -s https://api.github.com/repos/TryGhost/Source/releases/latest \
  | grep -E '"browser_download_url".*\.zip"' \
  | head -1
```

Récupère l'URL exacte du ZIP, puis re-télécharge avec `curl -L -o`.

Une fois `theme/` peuplé, vérifie la structure attendue :

```bash
ls theme/
# Tu dois voir au minimum :
# package.json
# default.hbs
# index.hbs
# post.hbs
# tag.hbs
# page.hbs
# author.hbs
# assets/
# partials/
```

Si la structure est anormale (par exemple un sous-dossier `Source-x.y.z/` qui contient les fichiers), aplatie :

```bash
# Si présent — adapte le nom du sous-dossier
mv theme/Source-*/* theme/
rmdir theme/Source-*
```

> **Marqueur : action hors Claude Code** — fallback manuel si le réseau bloque GitHub : demande à l'étudiant de télécharger manuellement le ZIP depuis https://github.com/TryGhost/Source/releases (latest) et de l'extraire dans `theme/` à la racine du projet, puis annonce « ok » pour continuer.

## Étape 2 — Adapter `theme/package.json`

Lis `theme/package.json`. Tu vas modifier 3 champs et garder le reste tel quel.

1. **`name`** : remplace par un slug du projet, ex. `ottho-blog-theme-<slug>` (en kebab-case, sans espace ni caractère spécial). Demande le slug à l'étudiant en une question :

   > « Quel slug court pour ton theme ? (kebab-case, ex. `monsite-blog`) — sera le `name` du theme dans Ghost et le nom du dossier visible dans l'admin. »

2. **`description`** : adapte selon le projet, ex. `"Theme custom pour blog.<domaine>, dérivé de Source"`.

3. **`config.posts_per_page`** : default à 5. Demande optionnellement :

   > « Combien d'articles par page sur la liste du blog ? (5 par défaut, dis "ok" pour garder, ou donne un autre chiffre comme 8 ou 12) »

**Garde tel quel** :
- `engines.ghost` (typiquement `>=5.0.0`) — c'est lu par Ghost pour la compat.
- `config.image_sizes` — utilisé par `{{img_url ... size="m"}}`.
- Toute autre clé que Source ajoute (ex. `config.custom`).

Édite le fichier avec l'outil `Edit` en remplaçant uniquement les lignes ciblées.

## Étape 3 — Adapter `theme/assets/css/screen.css`

C'est l'étape qui fait 80 % du visuel. **Ne réécris pas le CSS** — tu fais un override minimal au top.

Lis `theme/assets/css/screen.css` (juste le début, 100 premières lignes suffisent) pour identifier où Source définit ses CSS variables (cherche un bloc `:root { ... }`).

**Stratégie** : garde le bloc `:root` existant de Source intact, et **ajoute juste après** un second bloc `:root` qui override avec les valeurs de la charte. Le second gagne par cascade.

Insère en haut du fichier (après les éventuels `@charset` / `@font-face` mais avant tout sélecteur normal) :

```css
/* === Override Charte (généré par /blog:theme) === */
:root {
  /* Palette — extraite de charte.md */
  --color-primary: <valeur charte>;
  --color-text: <valeur charte>;
  --color-text-soft: <valeur charte>;
  --color-bg: <valeur charte>;
  --color-bg-soft: <valeur charte>;
  --color-accent: <valeur charte>;

  /* Typographies — extraites de charte.md */
  --font-display: <valeur charte>;
  --font-body: <valeur charte>;
  --font-mono: <valeur charte>;

  /* Espacements */
  --space-section: <valeur charte>;
  --space-block: <valeur charte>;
  --radius-card: <valeur charte>;
  --max-width-prose: <valeur charte>;
}

body {
  font-family: var(--font-body);
  color: var(--color-text);
  background: var(--color-bg);
}

h1, h2, h3, h4, h5, h6 {
  font-family: var(--font-display);
}

a {
  color: var(--color-primary);
}
/* === Fin override Charte === */
```

**Important** :
- Si Source utilise des noms de variables différents (ex. `--ghost-accent-color` au lieu de `--color-primary`), ajoute aussi des alias :
  ```css
  --ghost-accent-color: var(--color-primary);
  ```
  Repère les noms exacts en lisant le `:root` de Source.
- Si la charte impose une font Google non chargée par Source, note-le pour l'étape 4 (tu ajouteras un `<link>` dans `default.hbs`).
- **Ne supprime aucune règle existante de Source.** Tu surcharges, tu ne remplaces pas.

## Étape 4 — Adapter `theme/default.hbs` et les partials

`default.hbs` est le layout principal. Lis-le entièrement avant d'éditer.

**Vérifications obligatoires** (si une est absente, ajoute-la — c'est non négociable) :

- `<head>` contient `{{ghost_head}}` — généralement juste avant `</head>`. Ce helper injecte les balises meta, OG, JSON-LD, code injection, analytics.
- `<body>` a `class="{{body_class}}"` — Ghost s'en sert pour appliquer des classes contextuelles (`post-template`, `tag-template`, `home-template`).
- `</body>` est précédé de `{{ghost_foot}}` — injecte le code injection footer + barre membres.

**Si une font Google custom est requise par la charte** (étape 3), ajoute le `<link>` **avant** `{{ghost_head}}` dans `<head>` :

```handlebars
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Instrument+Serif&family=Inter:wght@400;500;700&display=swap" rel="stylesheet">
{{ghost_head}}
```

Adapte la query string `family=...` à la font de la charte.

**Header** : repère le markup du header dans `default.hbs` ou dans `partials/site-header.hbs` (Source décompose souvent en partials). Adapte pour matcher la nav du site existant — mêmes liens, même branding.

Pose la question :

> « Donne-moi la liste des liens de nav du site existant (label + URL), au format `Label → /url` ligne par ligne. Tu peux aussi me dire "garde la nav par défaut de Ghost" et on utilise `{{#foreach @site.navigation}}` (la nav configurée dans Ghost admin → Settings → Navigation). »

- Si l'étudiant fournit des liens : remplace le markup nav par une liste `<nav>` statique avec ces liens, en gardant le branding `{{@site.title}}` ou `{{@site.logo}}`.
- Si l'étudiant veut la nav Ghost : laisse le `{{#foreach @site.navigation}}` en place et précise qu'il devra configurer la nav dans Ghost admin.

**Footer** : même logique. Demande :

> « Footer : minimaliste (juste `© <année> <site> · Propulsé par Ghost`) ou clone du footer du site existant ? Si clone, donne-moi les liens. »

Adapte `partials/site-footer.hbs` (ou inline dans `default.hbs` selon la structure de Source).

**Préserve absolument** :
- `{{{body}}}` (triple accolades) où le contenu de chaque page s'injecte.
- Tout `{{> partial-name}}` que tu ne touches pas explicitement.

## Étape 5 — Adapter `theme/post.hbs` et `theme/index.hbs`

`post.hbs` rend une page d'article. `index.hbs` rend la liste (page d'accueil du blog).

**Pour `post.hbs`** :
- Vérifie que la prose est bien englobée dans un container avec `max-width: var(--max-width-prose)` — si Source utilise une autre variable, ajoute la règle dans `screen.css` :
  ```css
  .post-content { max-width: var(--max-width-prose); }
  ```
- Garde `{{content}}` intact — c'est lui qui injecte le HTML rendu par Ghost.
- Garde les helpers de méta : `{{date published_at format="DD MMMM YYYY"}}`, `{{reading_time}}`, `{{primary_tag}}`, `{{authors}}`.
- Si la charte impose un layout très différent (ex. titre énorme bordé d'une image), ajuste le markup mais **conserve les helpers**.

**Pour `index.hbs`** :
- Le `{{#foreach posts}} ... {{/foreach}}` est le cœur. Garde-le.
- À l'intérieur, garde les helpers : `{{title}}`, `{{excerpt}}`, `{{url}}`, `{{date published_at format="..."}}`, `{{img_url feature_image size="m"}}`.
- Si tu changes le markup d'une carte article, fais-le dans `partials/article-card.hbs` ou équivalent — Source utilise généralement un partial.
- Garde la pagination en bas : `{{pagination}}` ou le bloc `{{#if pagination.pages}}...{{/if}}`.

**Tag.hbs et page.hbs** : ne les touche pas en première itération. Ils héritent du CSS via le bloc `:root` et fonctionnent out of the box. L'étudiant pourra les ajuster plus tard.

## Étape 6 — Construire le ZIP

Le ZIP doit avoir `package.json` à la **racine** — pas dans un sous-dossier. C'est une contrainte stricte de Ghost.

```bash
# Depuis la racine du projet
cd theme && zip -r ../theme.zip . -x "*.DS_Store" "node_modules/*" "*.git*"
cd ..
ls -la theme.zip
```

Vérifie le contenu du ZIP :

```bash
unzip -l theme.zip | head -20
# La première entrée doit être package.json (pas mon-theme/package.json)
```

Si tu vois un wrapper de dossier dans le listing, recommence en zippant **depuis l'intérieur** de `theme/` (le `cd theme &&` est essentiel).

## Étape 7 — Upload du theme via MCP Ghost

Le MCP Ghost ne semble pas exposer d'outil dédié à l'upload de theme dans la liste standard (`mcp__ghost__themes_*` n'existe pas dans la version courante). Deux pistes selon ce que ton instance MCP propose :

1. **Si un outil `mcp__ghost__themes_upload` (ou équivalent) est disponible** : appelle-le avec le chemin absolu vers `theme.zip`. Lis sa signature avec `ToolSearch query="select:mcp__ghost__themes_upload" max_results=1` au besoin.

2. **Sinon, fallback Admin API direct** — utilise `curl` avec un JWT généré depuis `GHOST_ADMIN_API_KEY` :

   ```bash
   # Lis l'URL et la clé depuis ghost-config.md, exporte-les
   GHOST_URL="<url depuis ghost-config.md, sans slash final>"
   GHOST_ADMIN_API_KEY="<clé depuis .env ou variables d'env>"

   # Génère un JWT (5min de validité, signé HMAC-SHA256 avec la moitié secrète de la clé)
   # Le format de la clé est <id>:<secret>
   KEY_ID="${GHOST_ADMIN_API_KEY%%:*}"
   KEY_SECRET="${GHOST_ADMIN_API_KEY##*:}"
   IAT=$(date +%s)
   EXP=$((IAT + 300))
   HEADER=$(echo -n "{\"alg\":\"HS256\",\"typ\":\"JWT\",\"kid\":\"$KEY_ID\"}" | base64 | tr -d '=' | tr '/+' '_-')
   PAYLOAD=$(echo -n "{\"iat\":$IAT,\"exp\":$EXP,\"aud\":\"/admin/\"}" | base64 | tr -d '=' | tr '/+' '_-')
   SIG=$(echo -n "$HEADER.$PAYLOAD" | openssl dgst -sha256 -mac HMAC -macopt hexkey:$KEY_SECRET -binary | base64 | tr -d '=' | tr '/+' '_-')
   JWT="$HEADER.$PAYLOAD.$SIG"

   # Upload
   curl -X POST "$GHOST_URL/ghost/api/admin/themes/upload/" \
     -H "Authorization: Ghost $JWT" \
     -F "file=@theme.zip"
   ```

   Vérifie la réponse : un objet `{ "themes": [{ "name": "...", "active": false, ... }] }` signifie succès. Une erreur `400` mentionnant `package.json` signifie que le ZIP a un wrapper de dossier — refais l'étape 6.

> **Marqueur : action hors Claude Code** — si ni le MCP ni le curl ne marchent (clé API mal configurée, instance qui rejette JWT) : demande à l'étudiant d'aller manuellement dans Ghost admin → **Settings** → **Design & branding** → **Change theme** → **Upload theme** → sélectionner `theme.zip` → activer. Annonce-lui le chemin absolu du ZIP pour qu'il sache quoi uploader.

## Étape 8 — Activer le theme

Après upload, le theme existe dans l'admin mais n'est pas activé.

1. **Si un outil MCP `mcp__ghost__themes_activate` existe** : appelle-le avec `{ "name": "<slug du theme défini en étape 2>" }`.

2. **Sinon, via Admin API** :

   ```bash
   curl -X PUT "$GHOST_URL/ghost/api/admin/themes/<slug-du-theme>/activate/" \
     -H "Authorization: Ghost $JWT"
   ```

   Réponse attendue : `{ "themes": [{ "name": "<slug>", "active": true, ... }] }`.

> **Marqueur : action hors Claude Code** — fallback : Ghost admin → Settings → Design & branding → liste des themes → bouton **Activate** sur le theme uploadé.

**Test live** : ouvre `<URL du blog>` (l'URL est dans `ghost-config.md`) dans le navigateur. Tu dois voir le rendu custom — typo charte, couleurs charte, nav adaptée. Compare côte à côte avec le site principal.

## Étape 9 — Génération du récap (`theme/README.md`)

Crée `theme/README.md` à la racine du dossier theme avec ce contenu (adapté aux valeurs réelles) :

```markdown
# Theme — <nom du projet>

Theme Ghost custom dérivé de [Source](https://github.com/TryGhost/Source) (MIT).

## Source

- Theme de base : Ghost Source v<version récupérée depuis package.json original>
- Date de fork : <date du jour, format YYYY-MM-DD>
- Generator : `/blog:theme` (plugin ottho-blog)

## Adaptations

- **Palette** : extraite de `../charte.md` (primary <hex>, accent <hex>, etc.)
- **Typographies** : <font-display> (titres), <font-body> (texte)
- **Navigation** : <description courte — clone du site / nav Ghost dynamique>
- **Footer** : <description courte>

## Structure

- `package.json` — manifeste (name: <slug>, posts_per_page: <n>)
- `default.hbs` — layout (header/footer adaptés)
- `assets/css/screen.css` — bloc `:root` override Charte au top
- `post.hbs` / `index.hbs` — typo + spacing alignés charte
- `partials/` — partials de Source, conservés

## Itérer

Pour modifier le theme :

1. Édite les fichiers dans `theme/`.
2. Re-zippe : `cd theme && zip -r ../theme.zip . -x "*.DS_Store" && cd ..`.
3. Re-uploade et active :
   - via MCP Ghost (relance `/blog:theme` → étapes 7-8)
   - ou Ghost admin → Settings → Design & branding → Upload theme.

Ghost recharge le theme à la volée — pas de restart de pod. L'étudiant voit ses changements en moins d'une minute.
```

## Vérifications finales

Avant de clôturer, coche **chaque case** :

- [ ] `theme/` existe à la racine du projet et contient `package.json` + `default.hbs` + `index.hbs` + `post.hbs` + `assets/css/screen.css`.
- [ ] `theme/package.json` a un `name` adapté (slug projet), une `description` adaptée, `engines.ghost` intact.
- [ ] `theme/assets/css/screen.css` a un bloc `:root` override en tête avec les valeurs de `charte.md`.
- [ ] `theme/default.hbs` contient `{{ghost_head}}` dans `<head>`, `{{body_class}}` sur `<body>`, `{{ghost_foot}}` avant `</body>`.
- [ ] `theme/default.hbs` a un `<link>` Google Fonts si la charte impose une font custom.
- [ ] `theme.zip` créé à la racine du projet, `package.json` est à la racine du ZIP (pas dans un sous-dossier).
- [ ] Theme uploadé dans Ghost (réponse API 200 ou confirmation manuelle).
- [ ] Theme activé dans Ghost (réponse API 200 ou confirmation manuelle).
- [ ] `<URL du blog>` rend le theme custom (test visuel par l'étudiant).
- [ ] `theme/README.md` généré.

Si une case ne passe pas, corrige avant de clôturer.

## Clôture

Une fois toutes les vérifications validées :

> « Ton theme Ghost custom est en ligne. Le blog respecte ta charte — typo, palette, nav, spacing. Pour itérer : édite les fichiers dans `theme/`, re-zippe, ré-uploade. Ghost recharge à la volée, pas de redémarrage.
>
> Pièges classiques pour les prochaines itérations :
> - Toujours zipper **depuis l'intérieur** de `theme/` (sinon Ghost rejette : `package.json` doit être à la racine du ZIP).
> - Ne jamais supprimer `{{ghost_head}}`, `{{ghost_foot}}`, `{{body_class}}` — ça casse tracking, membres, code injection.
> - Si tu changes la palette, modifie uniquement le bloc `:root` au top de `screen.css` — le reste hérite.
>
> Prochaine étape : `/blog:cocon` pour cartographier la stratégie SEO de ton blog (piliers thématiques + maillage interne). »
