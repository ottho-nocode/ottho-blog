---
name: ghost-theme
description: Anatomie d'un theme Ghost (default.hbs, post.hbs, package.json, routes.yaml). Handlebars en 30 minutes. Procédure pour générer un theme depuis charte.md (mapping charte → CSS variables). Upload + activation du theme dans Ghost. Utilisée par /blog:theme.
---

# Skill : ghost-theme

Cette skill packe tout ce qu'il faut savoir pour générer un theme Ghost sur mesure à partir de la charte graphique de l'étudiant. Le theme produit doit faire ressembler `blog.exemple.com` à `exemple.com` — typo, palette, espacements, ton.

L'entrée principale est un fichier **`charte.md`** déjà produit lors du cours précédent (couleurs, typographies, layout). La sortie est un dossier `theme/` zippable, prêt à être uploadé dans Ghost.

## 1. Anatomie d'un theme Ghost

Un theme Ghost est un dossier qui contient des templates Handlebars (`.hbs`), un manifeste, des assets, et optionnellement un fichier de routing.

```
mon-theme/
├── package.json              ← manifeste obligatoire
├── routes.yaml               ← routing custom (optionnel)
├── default.hbs               ← layout principal (header + footer)
├── index.hbs                 ← liste des articles (page d'accueil du blog)
├── post.hbs                  ← page d'un article
├── tag.hbs                   ← page d'un tag = équivalent du pilier dans le cocon
├── page.hbs                  ← page statique (à propos, contact)
├── author.hbs                ← page d'un auteur
├── partials/
│   ├── header.hbs            ← réutilisable
│   ├── footer.hbs
│   └── card-article.hbs
└── assets/
    ├── css/
    │   └── screen.css        ← styles principaux
    └── js/
        └── main.js           ← optionnel
```

### `package.json` — le manifeste

```json
{
  "name": "mon-theme",
  "description": "Theme custom pour blog.exemple.com",
  "version": "1.0.0",
  "engines": {
    "ghost": ">=5.0.0"
  },
  "author": { "email": "tools@exemple.com" },
  "config": {
    "posts_per_page": 12,
    "image_sizes": {
      "s": { "width": 480 },
      "m": { "width": 960 },
      "l": { "width": 1600 }
    }
  }
}
```

`engines.ghost` est lu par Ghost pour vérifier la compatibilité — toujours `>=5.0.0` aujourd'hui. `posts_per_page` contrôle la pagination.

### `routes.yaml` — facultatif

Le routing par défaut de Ghost couvre 95 % des cas (`/`, `/blog/`, `/tag/<slug>/`, `/<slug>/`). On ne crée un `routes.yaml` que pour des collections custom ou pour faire vivre un site marketing depuis Ghost. **Pour un blog standard, ne pas en créer** — Ghost utilisera ses routes par défaut.

## 2. Handlebars en 30 minutes

Ghost utilise une variante de Handlebars étendue avec des helpers personnalisés. Les 5 à connaître :

### `{{content}}` — insère le HTML d'un post ou d'une page

```handlebars
<article class="post">
  <h1>{{title}}</h1>
  {{content}}
</article>
```

### `{{#foreach posts}} ... {{/foreach}}` — itère sur une collection

```handlebars
{{#foreach posts}}
  <a href="{{url}}" class="card-article">
    <h2>{{title}}</h2>
    <p>{{excerpt}}</p>
    <time>{{date published_at format="LL"}}</time>
  </a>
{{/foreach}}
```

À l'intérieur du `{{#foreach}}`, le contexte change : `{{title}}` réfère au post courant. `{{@index}}` donne l'index, `{{@first}}` et `{{@last}}` sont des booléens utiles.

### `{{date ... format="LL"}}` — formatage de date

```handlebars
{{date published_at format="DD MMMM YYYY"}}
{{!-- Affiche : 29 avril 2026 --}}
```

Format basé sur Moment.js. `LL` = format long localisé.

### `{{img_url ... size="m"}}` — URL d'image avec resize

```handlebars
<img src="{{img_url feature_image size="m"}}" alt="{{title}}">
```

Génère une URL vers une version resize de l'image (en s'appuyant sur `image_sizes` du `package.json`). Économise la bande passante sur les listings.

### `{{> partial-name}}` — inclut un partial

```handlebars
{{!-- default.hbs --}}
<body>
  {{> header}}
  {{{body}}}
  {{> footer}}
</body>
```

Le partial est chargé depuis `partials/header.hbs`. Triple accolades `{{{body}}}` = pas d'échappement HTML (nécessaire pour le contenu rendu).

### Bonus : `{{#if}}` / `{{else}}` / `{{#unless}}`

```handlebars
{{#if feature_image}}
  <img src="{{img_url feature_image size="l"}}" alt="">
{{else}}
  <div class="post-hero-fallback"></div>
{{/if}}

{{#unless @first}}
  <hr>
{{/unless}}
```

## 3. Variables Ghost disponibles dans les templates

### `{{@site}}` — méta-données globales

```handlebars
<title>{{@site.title}}</title>
<meta name="description" content="{{@site.description}}">
<a href="{{@site.url}}">{{@site.title}}</a>
{{!-- Logo --}}
{{#if @site.logo}}<img src="{{@site.logo}}" alt="">{{/if}}
{{!-- Navigation principale --}}
{{#foreach @site.navigation}}
  <a href="{{url}}">{{label}}</a>
{{/foreach}}
```

### `{{post}}` — disponible dans `post.hbs`

Champs : `title`, `excerpt`, `content`, `feature_image`, `feature_image_caption`, `primary_tag` (objet avec `name`, `slug`, `url`), `tags` (array), `authors` (array), `published_at`, `reading_time`, `url`.

### `{{posts}}` — disponible dans `index.hbs` et `tag.hbs`

Collection à itérer avec `{{#foreach posts}}`. Filtrée et paginée automatiquement par Ghost selon le contexte.

### `{{pagination}}` — liens next/prev

```handlebars
{{#if pagination.pages}}
  <nav class="pagination">
    {{#if pagination.prev}}<a href="{{page_url pagination.prev}}">Précédent</a>{{/if}}
    <span>Page {{pagination.page}} / {{pagination.pages}}</span>
    {{#if pagination.next}}<a href="{{page_url pagination.next}}">Suivant</a>{{/if}}
  </nav>
{{/if}}
```

## 4. Mapping `charte.md` → CSS variables

L'étudiant a une `charte.md` qui définit ses couleurs, fonts, espacements. **Pattern recommandé** : extraire ces valeurs en CSS custom properties au top de `screen.css`. Le reste du CSS référence ces variables → cohérence visuelle assurée + facile à itérer.

```css
/* assets/css/screen.css */

:root {
  /* Palette — extraite de charte.md */
  --color-primary: #5C3BFF;
  --color-text: #1F1D24;
  --color-text-soft: #6B6775;
  --color-bg: #FFFFFF;
  --color-bg-soft: #F1F1F3;
  --color-accent: #D97757;

  /* Typographies — extraites de charte.md */
  --font-display: "Instrument Serif", Georgia, serif;
  --font-body: "Inter", system-ui, sans-serif;
  --font-mono: "JetBrains Mono", ui-monospace, monospace;

  /* Espacements */
  --space-section: 80px;
  --space-block: 32px;
  --radius-card: 8px;
  --max-width-prose: 720px;
}

body {
  font-family: var(--font-body);
  color: var(--color-text);
  background: var(--color-bg);
}

h1, h2, h3 { font-family: var(--font-display); }

a { color: var(--color-primary); }
```

L'agent qui exécute `/blog:theme` doit **lire `charte.md`**, **extraire** les valeurs, et **les coller** dans le bloc `:root`. Le reste du CSS hérite — pas besoin de tout réécrire à chaque itération.

## 5. Stratégie de génération du theme custom

Trois options possibles :

- **Option A — from scratch** : écrire tous les `.hbs` à la main. Beaucoup de boulot, gros risque d'oublier des helpers Ghost (`{{ghost_head}}`, `{{ghost_foot}}`, navigation membres). **Déconseillé.**
- **Option B — fork du theme officiel Source** : Source est le theme par défaut de Ghost, MIT, maintenu par TryGhost. On clone, on remplace le CSS, on ajuste le markup. **C'est ce que `/blog:theme` doit faire.**
- **Option C — adopter un theme tiers gratuit** : Ghost Marketplace propose des themes MIT. Utile pour des cas spécifiques mais on perd le contrôle.

### Procédure Option B (recommandée)

1. **Télécharger Source** depuis `https://github.com/TryGhost/Source/releases` — prendre le ZIP de la dernière release (pas un clone du repo, pour avoir un build propre).
2. **Décompresser** dans un dossier de travail `theme/`.
3. **Renommer** dans `package.json` : `"name": "mon-theme"` (slug du blog).
4. **Adapter `assets/css/screen.css`** :
   - Remplacer le bloc `:root` au top par les variables extraites de `charte.md`.
   - Si la charte impose une typo précise non chargée par défaut, ajouter un `<link>` vers Google Fonts dans `default.hbs` (juste avant `{{ghost_head}}`).
   - Garder le reste du CSS de Source — il référence déjà des CSS variables et s'adapte automatiquement.
5. **Adapter `default.hbs`** : remplacer le markup du header pour matcher la nav du site existant (logo + liens identiques). Idem pour le footer.
6. **Adapter `post.hbs`** : ajuster typo et spacing pour matcher la charte (taille des H2, line-height, max-width prose).
7. **Garder `index.hbs` et `tag.hbs` quasi tels quels** — ils utilisent les helpers standard et fonctionnent en s'appuyant sur les CSS variables.

## 6. Upload du theme dans Ghost

### Méthode A — Ghost admin UI

1. Compresser le dossier `theme/` en `theme.zip` (depuis le dossier — le ZIP doit avoir `package.json` à sa racine, pas dans un sous-dossier).
2. Ghost admin → **Settings** → **Design & branding** → **Change theme** → **Upload theme**.
3. Activer le theme uploadé.

### Méthode B (préférée par le plugin) — via Admin API + MCP

`/blog:theme` utilise le serveur MCP Ghost pour uploader sans passer par l'UI :

- `POST /themes/upload` (multipart/form-data) — le MCP gère l'authentification JWT et le multipart.
- `POST /themes/<name>/activate` — active le theme en une requête.

Cycle d'itération : 2 minutes par changement (modifier `screen.css`, re-zipper, ré-uploader). Ghost recharge le theme à la volée — pas besoin de redémarrer le pod.

## 7. Test + itération

Après activation :

1. Visiter `blog.exemple.com` → vérifier rendu page d'accueil.
2. Visiter un post → vérifier typo, spacing, couleur des liens.
3. Visiter une page tag → vérifier que la liste s'affiche et que la nav matche.
4. Comparer côte à côte avec `exemple.com` → ajuster les variables de `screen.css` si écart.

**Pièges courants :**

- ⚠️ **Oublier `{{ghost_head}}` dans `<head>` ou `{{ghost_foot}}` avant `</body>` de `default.hbs`** : casse le tracking, les analytics, le code injection, la barre membres. Ces deux helpers sont **obligatoires**, jamais à supprimer.
- ⚠️ **`{{body_class}}` manquant sur `<body>`** : empêche Ghost d'appliquer les classes de contexte (`post-template`, `tag-template`, etc.) et casse le CSS conditionnel.
- ⚠️ **ZIP avec un wrapper de dossier** : si le ZIP contient `mon-theme/package.json` au lieu de `package.json` à la racine, Ghost rejette l'upload. Compresser depuis l'intérieur du dossier.
- ⚠️ **Polices Google non chargées** : si la charte utilise une font custom, vérifier que le `<link>` Google Fonts est bien dans `default.hbs` au-dessus de `{{ghost_head}}`.
- ⚠️ **Image `feature_image` énorme servie en pleine taille** : toujours utiliser `{{img_url feature_image size="m"}}` ou `size="l"` dans les listings — pas `{{feature_image}}` brut.

## 8. Variations supportées (mode sombre, accents, fonts)

Le theme est **vivant** — l'étudiant peut le faire évoluer sans rebuild :

- **Mode sombre** : ajouter un bloc `@media (prefers-color-scheme: dark)` dans `screen.css` qui override les CSS variables. Re-upload, c'est tout.
- **Changer la couleur primaire** : modifier `--color-primary` dans le bloc `:root`, ré-uploader. Toutes les références héritent.
- **Changer la font de titre** : remplacer `--font-display` + ajuster le `<link>` Google Fonts dans `default.hbs`.
- **Ajouter une page custom** (ex. `/manifesto`) : créer un `page-manifesto.hbs` (Ghost le détecte automatiquement par la convention de nommage `page-<slug>.hbs`).

Aucun rebuild de site nécessaire. Ghost recharge le theme à la volée à chaque upload — l'étudiant voit ses changements en moins d'une minute.

---

## Usage dans les commandes du plugin

Cette skill est invoquée par :

- `/blog:theme` — usage principal (lit `charte.md`, fork Source, génère le theme, upload via MCP Ghost).
- `/blog:setup` — peut référencer cette skill pour expliquer le rôle du theme dans le pipeline d'installation.
