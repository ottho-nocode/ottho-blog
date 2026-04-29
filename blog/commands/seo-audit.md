---
name: seo-audit
description: Audit SEO d'un article (par slug) ou de tout le cocon. Score sur 8 critères + suggestions concrètes. Output : rapport markdown seo-audit-{date}.md à la racine du projet.
---

# /blog:seo-audit, Audit SEO d'un article ou du cocon entier

Tu vas auditer la qualité SEO d'un article (par slug) ou de tout le cocon, scorer chaque article sur 8 critères, et produire un rapport markdown actionnable. C'est l'outil de contrôle qualité éditorial du chapitre 7.

**Charge la skill `seo-blog`** pour la checklist détaillée (8 points) et les seuils exacts (longueurs, structures, etc.). C'est elle qui définit la grille, tu l'appliques.

**Pré-requis** : Ghost en place (skill `ghost-config`), au moins un article publié ou en draft sur l'instance, et le MCP Ghost installé dans Claude Code.

## Préambule

Ouvre la commande par ce message :

> « On va auditer le SEO de tes articles. Pour chaque article, je vais scorer 8 critères (méta, structure, liens internes, image, bloc technique, JSON-LD) et produire un rapport markdown avec les corrections concrètes à appliquer.
>
> Tu peux auditer un article précis (par son slug) ou tout le cocon. Compte ~30 secondes par article. »

## Protocole

Pose les questions **UNE PAR UNE**, attends la réponse avant de continuer. Ne jamais empiler.

### Étape 1, Choix du scope

> « Tu veux auditer **un article spécifique** ou **tout le cocon** ?
>
> - 1 article → tape le slug (ex. `claude-code-installation`)
> - tout le cocon → tape `cocon` (j'auditerai tous les posts `published` + `draft`) »

Selon la réponse :

- **Slug spécifique** → continue avec l'article correspondant. Si le slug n'existe pas dans Ghost (404), redemande.
- **`cocon`** → continue avec tous les posts.

### Étape 2, Récupérer les posts

Via le **MCP Ghost** (skill `ghost-config` couvre l'installation) :

- **1 article** : `posts_read({ slug: '<slug>', include: 'tags,authors' })`
- **Cocon entier** : `posts_browse({ filter: 'status:[published,draft]', limit: 'all', include: 'tags,authors' })`

Pour chaque post, capture :
- `slug`
- `title`, `meta_title`, `meta_description`, `custom_excerpt`
- `og_title`, `og_description`, `og_image`
- `html` (contenu rendu, utile pour l'analyse)
- `feature_image`, `feature_image_alt`
- `primary_tag`, `tags` (pour identifier le pilier)
- `published_at`, `updated_at`

Si l'API retourne une erreur (token invalide, instance offline) : signale-le et redirige vers `/blog:setup-ghost` pour réparer la connexion.

### Étape 3, Auditer chaque article (8 critères)

Pour chaque article, applique la checklist de la skill `seo-blog` (référence-toi au point correspondant pour les seuils détaillés). Score binaire : **0** ou **1** par critère.

#### Critère 1, `meta_title` (≤ 60 caractères)

- Présent et non vide
- ≤ 60 caractères
- Contient le keyword principal une seule fois (pas de bourrage)

Si fail → suggestion : « Raccourcir à `<proposition de 50-58 car. cohérente avec le pilier>` ».

#### Critère 2, `meta_description` (≤ 155 caractères)

- Présente et non vide
- ≤ 155 caractères
- Format action + bénéfice (verbe d'action + ce qu'on apprend / gagne)

Si fail → suggestion : « Réécrire en : `<proposition courte avec verbe + bénéfice + chiffre si possible>` ».

#### Critère 3, `<h1>` unique sur la page

- Compte le nombre de `<h1>` dans le `html` du body
- Doit être **0** dans le body (Ghost ajoute le H1 automatiquement depuis `title`)

Si fail (≥ 1 H1 trouvé dans le body) → suggestion : « Convertir le H1 du corps en H2. Ghost gère le H1 de page automatiquement ».

#### Critère 4, Structure H2/H3 logique

- Compte les `<h2>` du body : doit être entre **5 et 8**
- Vérifie qu'aucun H3 n'apparaît avant un H2 (hiérarchie)
- Si < 5 H2 → article probablement trop court ou mal structuré

Si fail → suggestion : « Ajouter X H2 (recommandé 5-8). Découper la section "<plus longue>" en 2 sous-sections H2 ».

#### Critère 5, 4 à 6 liens internes

- Compte les `<a>` dont le `href` pointe vers le même domaine (ou commence par `/`)
- Cible : entre **4 et 6** liens internes
- Pattern recommandé (cf. skill) : 1 mère + 2-3 sœurs + 1 CTA formation + 0-2 outils

Si fail → suggestion concrète : « Tu as N liens internes. Recommandé : 4-6. Suggestion : ajouter 1 lien vers le pilier mère `/blog/pilier/<slug-pilier>` et 1-2 liens vers des articles sœurs du même pilier ».

> Si la skill `link-validation` est disponible, tu peux croiser avec `cocon.json` pour suggérer des slugs sœurs précis.

#### Critère 6, Image hero avec `alt` non vide

- `feature_image` présente
- `feature_image_alt` non vide, ≥ 8 mots, descriptif (pas `image`, `hero`, `cover`)

Si fail → suggestion : « Ajouter un `alt` descriptif (8-15 mots) qui décrit factuellement le contenu de l'image, pas son rôle ».

#### Critère 7, Au moins 1 bloc `<pre><code>` ou `<table>`

- Détecte au moins **1** des éléments suivants dans le `html` du body :
  - `<pre><code>` (snippet, commande, config)
  - `<table>` (comparaison)
  - `<blockquote>` (citation source), fallback acceptable

Si aucun → suggestion : « Article 100 % paragraphes, manque un signal de profondeur. Ajouter au moins 1 bloc technique (snippet, table comparative, ou citation source) ».

#### Critère 8, JSON-LD `BlogPosting` + `BreadcrumbList`

- Vérifie que le **theme custom** (skill `ghost-theme`) injecte bien :
  - `BlogPosting` (avec `headline`, `datePublished`, `dateModified`, `author`, `image`, `mainEntityOfPage`)
  - `BreadcrumbList` (Accueil → Blog → Pilier → Article)

Comme ces balises sont injectées via `post.hbs`, valide en récupérant le HTML rendu de l'URL publique de l'article :

```bash
curl -s https://blog.<domaine>/<slug>/ | grep -i 'application/ld+json'
```

Si pas de `BlogPosting` → suggestion : « `BlogPosting` absent du rendu. Vérifier que `theme/post.hbs` contient le bloc `<script type="application/ld+json">` (skill `ghost-theme`) et redéployer le theme ».

Si pas de `BreadcrumbList` → suggestion : « `BreadcrumbList` absent. Ajouter le bloc breadcrumb dans `theme/post.hbs` avec les 4 niveaux Accueil → Blog → Pilier → Article ».

#### Score article

Total : **N / 8**, où N = somme des critères passés.

### Étape 4, Score global du cocon (si scope = `cocon`)

À la fin de l'audit de tous les articles, calcule :

- **Score moyen** : moyenne arithmétique des scores / 8
- **Articles 8/8** : liste des slugs parfaits
- **Top 3 à corriger** : les 3 articles avec le score le plus bas (en cas d'égalité, départage par nombre d'impressions GSC si disponible, sinon par `published_at` le plus ancien)

### Étape 5, Générer le rapport

Écris le fichier `seo-audit-{YYYY-MM-DD}.md` à la racine du projet (date du jour, format ISO).

Structure :

```markdown
# SEO Audit, {YYYY-MM-DD}

Scope : {1 article : <slug> | cocon entier}

## Résumé

- Articles audités : X
- Score moyen : Y / 8
- Articles 8/8 : Z (liste : slug1, slug2, …)
- Top 3 à corriger : slug-bas-1 (3/8), slug-bas-2 (4/8), slug-bas-3 (4/8)

---

## Article : <slug-1>

**URL** : https://blog.<domaine>/<slug-1>/
**Pilier** : <primary_tag>
**Score** : 6 / 8

### Points forts (✅)

- ✅ **meta_title**, 52 car., keyword bien placé en début
- ✅ **meta_description**, 140 car., action claire (« Apprends … en 5 semaines »)
- ✅ **H1 unique**, 0 H1 dans le body, Ghost gère
- ✅ **Structure H2**, 6 H2 logiques
- ✅ **Image hero alt**, 12 mots descriptifs
- ✅ **Bloc technique**, 2 `<pre><code>` détectés

### Points à corriger (⚠️)

- ⚠️ **Critère 5, Liens internes** : 2 trouvés, recommandé 4-6.
  → **Suggestion** : ajouter 2 liens vers les piliers sœurs `/blog/pilier/claude-code` et `/blog/pilier/mcp`, et 1 CTA final vers `/claude-builders`.

- ⚠️ **Critère 8, JSON-LD BreadcrumbList** : absent du rendu.
  → **Suggestion** : vérifier que `theme/post.hbs` inclut le bloc `BreadcrumbList` (skill `ghost-theme`). Redéployer le theme après correction.

---

## Article : <slug-2>

…

---

Audit généré le {YYYY-MM-DD} par le plugin ottho-blog (`/blog:seo-audit`).
```

Pour un audit **mono-article**, le rapport ne contient qu'une seule section `## Article` et pas de section `## Résumé` (à remplacer par la ligne `**Score** : N / 8`).

## Vérifications finales

Avant de clôturer :

- [ ] Le rapport `seo-audit-{date}.md` est écrit à la racine.
- [ ] Chaque article audité a un score affiché sur 8.
- [ ] Chaque critère failed a une **suggestion concrète et actionnable** (pas « améliorer le SEO »).
- [ ] Les piliers et URLs cités dans les suggestions existent (croiser avec `cocon.json` si présent).
- [ ] Si scope = cocon : le top 3 à corriger est listé dans le résumé.

Si un de ces points n'est pas validé, complète avant d'écrire le fichier.

## Clôture

Une fois le fichier écrit :

> « Audit terminé. Rapport : `./seo-audit-{date}.md`.
>
> Prochaine étape : ouvre les top 3 articles à score bas dans Ghost admin et applique les suggestions une par une. Pour les corrections de structure (liens internes, blocs techniques), édite le draft. Pour les corrections de meta (title, description, alt), édite directement les champs du panneau « SEO » de Ghost.
>
> Quand tu as fini, relance `/blog:seo-audit` pour valider que les scores sont remontés. Cadence recommandée : audit complet 1× par mois, ou après chaque batch de 5 articles publiés. »

## Cas particuliers

- **Article en draft** : audite normalement. Précise dans le rapport « (draft) » à côté du slug. La validation JSON-LD via `curl` n'est pas possible (pas d'URL publique), marque le critère 8 comme `n/a, draft non publié`.
- **Theme Ghost par défaut (Casper) au lieu du theme custom** : critère 8 sera systématiquement fail. Signale-le en tête du rapport et redirige vers la skill `ghost-theme` pour installer le theme custom avant le prochain audit.
- **Aucun article publié ou en draft** : interromps la commande avec :
  > « Aucun article trouvé sur ton instance Ghost. Lance d'abord `/blog:create-article` (ou rédige un draft dans Ghost admin), puis reviens. »
- **Erreur MCP Ghost** : redirige vers `/blog:setup-ghost` pour vérifier les API keys et la connectivité.
