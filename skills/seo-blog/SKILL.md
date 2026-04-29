---
name: seo-blog
description: Checklist SEO spécifique blog (vs landing/site). 8 points par article (meta, structure, ancres, image alt, JSON-LD). Sitemap blog. Soumission Search Console. Mining d'opportunités via GSC API (queries position 8-20). Utilisée par /blog:seo-audit et /blog:opportunities.
---

# Skill : seo-blog

Checklist SEO **spécifique au blog** (Ghost CMS + cocon sémantique). Différente du SEO d'une landing ou d'un site multi-pages, traité dans la skill `seo-checklist` du cours précédent. À utiliser pour `/blog:seo-audit` (audit d'un article ou de tout le cocon) et `/blog:opportunities` (mining GSC pour identifier les requêtes à fort potentiel).

## 1. Différences SEO blog vs landing/site

Un blog n'est **pas** un site classique au sens SEO. Quatre différences fondamentales :

- **Volume** — un blog produit 30, 50, 100+ articles. Le challenge est la **cohérence** entre les articles, pas l'optimisation parfaite d'une page unique.
- **Structure** — chaque article = un **mini-cluster sémantique**. Un seul `<h1>`, des H2 logiques qui font avancer la promesse, pas de juxtaposition de sections décoratives.
- **Fraîcheur** — Google préfère le contenu récent. `published_at` et surtout `updated_at` comptent : un article republié en `updated_at: 2026-04` peut surclasser un article publié en 2024 mais jamais touché.
- **Maillage** — c'est le **cœur du cocon**. Chaque article reçoit des liens d'articles sœurs et de la mère, et envoie des liens vers la mère + sœurs + CTA formation. Voir skills `cocon-method` et `link-validation`.

## 2. Checklist SEO d'un article (8 points)

### Point 1 — `meta_title`

- ≤ **60 caractères** (au-delà, Google tronque)
- Contient le **keyword principal une seule fois** (pas de bourrage)
- Commence par le keyword si possible
- Différent du `<h1>` (le `<h1>` est plus humain, le `meta_title` est calibré pour le SERP)

✅ « Apprendre Claude Code en 5 semaines | Ottho » (47 car.)
❌ « Apprendre Claude Code, formation Claude Code pour devenir builder Claude Code » (bourrage)

### Point 2 — `meta_description`

- ≤ **155 caractères**
- Format : **action + bénéfice + chiffre** quand c'est possible
- Inclut naturellement le keyword principal
- Incite au clic (différenciation vs autres résultats)

✅ « Apprends Claude Code en 5 semaines avec la méthode Ottho. 8 ateliers, 1 projet réel, 1 500 €. Démarrage immédiat. » (138 car.)

### Point 3 — `<h1>` unique

- **Un seul** `<h1>` par page. Ghost l'ajoute automatiquement depuis `title` (ou `meta_title` si défini) — l'éditeur ne doit donc **pas** générer de H1 dans le corps de l'article.
- Si un `<h1>` est trouvé dans le HTML body : c'est une erreur de rédaction → à corriger.

### Point 4 — Structure H2/H3

- **5 à 8 H2** par article (un article < 5 H2 est probablement trop court ou mal structuré)
- Hiérarchie logique : chaque H2 fait avancer la promesse, pas de « Partie 1 / Partie 2 »
- H3 utilisés uniquement pour des sous-points sous un H2 (pas en remplacement de H2)
- Chaque H2 décrit **factuellement** ce que contient la section (Google lit les H2 comme un sommaire)

### Point 5 — 4 à 6 liens internes

Pattern recommandé pour un article fille :
- **1 lien vers la mère** (page pilier `/blog/pilier/<slug>`)
- **2-3 liens vers des sœurs** (autres articles du même pilier ou piliers proches)
- **1 CTA final vers la formation** (Mastery / Builder / Agent selon pertinence)
- **0-2 liens vers les outils** (Claude Code, MCP, etc.) si l'article les évoque

Voir skill `link-validation` pour la validation automatisée.

### Point 6 — Image hero avec `alt` non vide

- Toute image hero doit avoir un `alt` descriptif (8-15 mots)
- ❌ `alt="image"`, `alt="hero"`, `alt=""` (sauf décoration pure)
- ✅ `alt="Capture d'écran d'un terminal montrant Claude Code en train de générer un composant React"`

### Point 7 — Au moins 1 bloc technique

Un signal de profondeur que Google et les lecteurs apprécient :
- Au moins **1 bloc `<pre><code>`** si l'article est technique (commande, config, snippet)
- OU **1 `<table>`** si l'article compare des options
- Sinon : **1 `<blockquote>`** pour citer une source

Un article 100 % paragraphes est plat — il manque un signal de profondeur.

### Point 8 — JSON-LD `BlogPosting` + `BreadcrumbList`

Ghost génère automatiquement le JSON-LD `BlogPosting` si la skill `ghost-theme` est correctement appliquée. Vérifier les champs minimaux :

```json
{
  "@context": "https://schema.org",
  "@type": "BlogPosting",
  "headline": "...",
  "datePublished": "2026-04-15T10:00:00.000Z",
  "dateModified": "2026-04-20T14:00:00.000Z",
  "author": { "@type": "Person", "name": "..." },
  "image": "https://blog.exemple.com/content/images/...",
  "mainEntityOfPage": { "@type": "WebPage", "@id": "https://blog.exemple.com/<slug>/" }
}
```

Inclure également un `BreadcrumbList` :

```
Accueil → Blog → Pilier → Article
```

Si le `primary_tag` correspond à un pilier du cocon, utiliser le slug pilier comme niveau intermédiaire (cf. `app/blog/[slug]/page.tsx` du projet de référence).

## 3. Sitemap blog

Ghost génère **automatiquement** :
- `/sitemap.xml` (index)
- `/sitemap-posts.xml` (articles)
- `/sitemap-pages.xml` (pages)
- `/sitemap-tags.xml` (tags)
- `/sitemap-authors.xml` (auteurs)

L'étudiant **n'a rien à coder**. Vérifications côté audit :
- `https://blog.exemple.com/sitemap.xml` est accessible (HTTP 200)
- Le sitemap liste bien les articles publiés (pas les `status: draft`)
- Les `<lastmod>` correspondent aux `updated_at` réels

À renvoyer dans `/blog:seo-audit` :
> « Va sur `https://blog.exemple.com/sitemap.xml`, vérifie que tu vois bien tes articles publiés et que les dates `<lastmod>` sont récentes. »

## 4. Soumission à Search Console

**Pré-requis** : la propriété `blog.exemple.com` est ajoutée à Google Search Console (cf. bonus du cours — vérification soit via meta tag dans `ghost_head`, soit via fichier HTML uploadé dans `assets/`).

Étapes :
1. Search Console → **Sitemaps** → ajouter `sitemap.xml`
2. Google crawle dans **24-72h**
3. Données utiles disponibles ensuite : queries, position moyenne, clics, impressions, CTR, pages indexées, erreurs

Activer aussi l'**inspection d'URL** : pour un article fraîchement publié, demander une indexation manuelle via « Demander une indexation » accélère le crawl initial.

## 5. Mining d'opportunités SEO via GSC API (`/blog:opportunities`)

Logique du mining :
- **Filtre standard** : queries en **position 8-20** avec **impressions > 50** sur **28 jours**
- **Pourquoi 8-20** : c'est la zone « presque page 1 ». Un effort éditorial (mise à jour, ajout de contenu, internal linking) peut faire passer en position 1-7.
- **Pourquoi > 50 impressions** : seuil de pertinence. En dessous, c'est du bruit (queries trop niches ou ponctuelles).

Exemple de requête GSC API (Search Analytics endpoint) :

```bash
POST https://searchconsole.googleapis.com/webmasters/v3/sites/sc-domain:blog.exemple.com/searchAnalytics/query
Content-Type: application/json
Authorization: Bearer <oauth_token>

{
  "startDate": "2026-04-01",
  "endDate": "2026-04-28",
  "dimensions": ["query", "page"],
  "rowLimit": 1000
}
```

Filtrage côté skill (pseudo-code) :

```
opportunities = rows.filter(r =>
  r.position >= 8 && r.position <= 20 && r.impressions > 50
)
```

Pour chaque query qualifiée :
- **Cross-référencer** avec `cocon.json` : un article du cocon couvre-t-il déjà cette query ?
  - **Si oui** : signal pour mettre à jour l'article (refresh `updated_at`, ajouter 200-400 mots, renforcer le maillage interne sur cette query).
  - **Si non** : suggestion de **nouvel article** dans le pilier le plus proche sémantiquement. Le proposer dans le cocon avec un slug provisoire.

Output type pour `/blog:opportunities` : tableau markdown trié par impressions descendantes, avec colonnes `query | position | impressions | CTR | article existant | action recommandée`.

## 6. Events GA4 utiles pour un blog

Au-delà du SEO pur, instrumenter ces events via le `ghost_head` ou `ghost_foot` du theme custom (skill `ghost-theme`) :

- `page_view` automatique avec **custom dimension `pilier`** (issue du tag Ghost) — voir cours précédent
- `click_cta_blog_to_formation` (custom event, déclenché au clic sur le CTA final de l'article)
- `lecture_article_complete` (scroll 90 %)
- `click_lien_interne_cocon` (clic sur un lien interne — utile pour mesurer la circulation dans le cocon)

Ces events ne remplacent pas Search Console mais permettent de croiser **trafic SEO entrant → conversion vers formation**.

## 7. Audit régulier

- **Cadence recommandée** : lancer `/blog:seo-audit` **1× par mois**, ou après chaque batch de 5 articles publiés.
- **Lancer `/blog:opportunities`** : 1× par mois, idéalement le même jour que l'audit.
- **Output `/blog:seo-audit`** : rapport markdown listant les articles non conformes avec :
  - Numéro du point non respecté (1 à 8)
  - Citation exacte du problème (ex: « `meta_title` fait 73 caractères »)
  - Action corrective concrète (ex: « raccourcir à : "Apprendre Claude Code en 5 semaines | Ottho" »)

Format de rapport type :

```markdown
## /blog:seo-audit — 2026-04-29

**12 articles audités · 8 conformes · 4 à corriger**

### article-1-slug — 2 corrections
- Point 1 (`meta_title` 73 car.) → raccourcir à : "..."
- Point 5 (3 liens internes seulement) → ajouter 1 lien vers /pilier-x

### article-2-slug — 1 correction
- Point 7 (aucun bloc technique) → ajouter snippet ou table comparative
```

## 8. Usage dans les commandes du plugin

Cette skill est invoquée par :
- `/blog:seo-audit` — audit d'un article ou du cocon entier
- `/blog:opportunities` — mining GSC pour identifier les opportunités

Elle complète `seo-checklist` (skill du plugin précédent, focalisée landing/site) sans la dupliquer : `seo-checklist` couvre `<title>`, OG, canonical, robots.txt, etc. — éléments déjà gérés au niveau du theme Ghost (skill `ghost-theme`) et qui ne sont pas spécifiques à un article.
