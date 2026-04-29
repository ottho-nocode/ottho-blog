# ottho-blog

> Plugin compagnon du cours [academy.ottho.co/claude-blog](https://academy.ottho.co/claude-blog).
> Donne à ton site déployé sur Vercel un blog Ghost en headless, fidèle à ta charte, structuré en cocon sémantique, et alimenté par Claude — sans jamais taper de clé API.

## À qui ça s'adresse

Ce plugin est le prolongement du cours *Claude + Site web*. Si tu as déjà un site HTML/CSS/JS en ligne et un `brief.md` propre, `ottho-blog` ajoute la couche éditoriale :

- Un Ghost auto-hébergé (PikaPods), configuré en headless et branché sur ton projet
- Un thème Ghost qui hérite de ta charte (couleurs, typo, composants) — pas le thème Casper par défaut
- Un cocon sémantique généré à partir de ton brief : 1 pilier + N enfants + maillage interne propre
- Une boucle de génération d'articles assistée (un par un) ou en auto-pilote (`/blog:batch`)
- Un audit SEO continu : structure, maillage, balises, opportunités de mots-clés
- Une commande de correction chirurgicale qui propage les fixes sur tous les articles concernés

**Pré-requis :** avoir suivi (ou lu) le cours *Claude + Site web*, avoir un site HTML/CSS/JS déployé sur Vercel, et avoir un `brief.md` à la racine du projet.

## Installation

Dans Claude Code :

```
/plugin marketplace add ottho/blog
/plugin install ottho-blog@ottho
```

(Le nom du marketplace pourra changer selon la publication finale — voir le `marketplace.json` de référence.)

## Pré-requis MCP

Avant utilisation, les MCPs suivants doivent être connectés (procédures couvertes dans le cours) :

- **Ghost MCP** (obligatoire) — création/édition d'articles, tags, navigation
- **Vercel MCP** (obligatoire) — env vars, redéploiement après changement de config
- **fal.ai MCP** (optionnel) — illustrations d'articles via Nano Banana

Tous s'installent par dialogue dans Claude Code (« Installe le MCP [nom] »). Zéro commande bash.

## Les 9 commandes (à venir)

Toutes les commandes sont namespacées sous `/blog:` pour éviter les conflits avec d'autres plugins.

### `/blog:setup-ghost` — *à venir*

Provisionne un Ghost auto-hébergé (PikaPods) ou se branche sur une instance existante. Configure le mode headless, ouvre l'API Content + Admin, génère les clés, les pousse dans Vercel via le MCP Vercel. Tu n'écris pas une ligne de config.

### `/blog:theme` — *à venir*

Génère un thème Ghost custom qui reprend ta charte (couleurs, typographies, composants header/footer). Compatible Koenig editor. Build, zip, upload via Ghost MCP.

### `/blog:cocon` — *à venir*

Lit ton `brief.md`, propose une stratégie de cocon sémantique (1 pilier + 6 à 12 enfants), valide le maillage avant de figer. Sortie : `cocon.md` qui pilote toutes les générations suivantes.

### `/blog:article` — *à venir*

Génère un article du cocon, un par un, avec validation à chaque étape (structure, plan, rédaction, illustrations, balises, maillage interne). Publie via Ghost MCP.

### `/blog:batch` — *à venir*

Mode auto-pilote : enchaîne `/blog:article` sur tous les articles non publiés du cocon. Reprenable. Pause à chaque article si tu veux relire.

### `/blog:seo-audit` — *à venir*

Audit complet : balises title/meta, H1/H2/H3, maillage interne, sitemap blog, vitesse, mobile. Liste les fixes, les applique sur accord.

### `/blog:opportunities` — *à venir*

Scan tes articles existants + le cocon prévu, suggère de nouveaux articles à fort potentiel SEO (mots-clés sous-exploités, gaps thématiques).

### `/blog:status` — *à venir*

Tableau de bord du blog : articles publiés, brouillons, articles du cocon non commencés, dernière mise à jour, score SEO moyen.

### `/blog:corrige` — *à venir*

Correction chirurgicale d'un article ou d'un pattern transverse (ex : changer la formulation d'un CTA sur tous les articles enfants d'un pilier). Cascade propre, dry-run par défaut.

## Les 7 skills (à venir)

| Skill | Rôle |
|---|---|
| `cocon-method` | Méthode de cocon sémantique (pilier/enfants/maillage), règles de Laurent Bourrelly adaptées |
| `ghost-config` | Templates de config Ghost headless, env vars, webhooks de revalidation |
| `ghost-theme` | Structure d'un thème Ghost (handlebars), composants Koenig, build/upload |
| `article-quality` | Règles éditoriales : longueur, structure H1-H2-H3, lisibilité, ton de voix |
| `link-validation` | Vérification du maillage interne, détection des liens cassés, suggestions |
| `seo-blog` | Checklist SEO spécifique blog : mots-clés, balises, sitemap, schema Article |
| `knowledge-base` | Lecture du `brief.md` et de `cocon.md` pour alimenter chaque commande |

## Roadmap d'usage

L'ordre suit les chapitres du cours. Tu peux faire chaque étape en dialogue libre, mais les commandes condensent la méthode :

1. `/blog:setup-ghost` — Ghost en place et connecté à ton site
2. `/blog:theme` — Le blog ressemble à ton site, pas à Ghost.org
3. `/blog:cocon` — La stratégie est fixée avant d'écrire une ligne
4. `/blog:article` (×N) ou `/blog:batch` — Les articles arrivent
5. `/blog:seo-audit` — Vérification au fur et à mesure
6. `/blog:opportunities` — Quand le cocon est complet, on étend
7. `/blog:status` — À tout moment, pour voir où on en est
8. `/blog:corrige` — Quand une boulette ou un pattern doit changer

## Philosophie

Ce plugin ne fait rien que Claude Code ne puisse faire en dialogue libre. Chaque commande est un **dialogue structuré** + une **skill associée** qui encapsule la méthode enseignée.

Si une commande échoue ou si tu veux comprendre ce qu'elle fait, tu peux toujours la refaire à la main — c'est même ce qu'enseigne le cours. Le plugin est un raccourci, pas une boîte noire.

## Version

**v1.0.0** — Scaffold initial. Commandes et skills à venir (voir Tasks 9-17 du plan d'implémentation).

## Licence

MIT.

## Liens

- Cours : [academy.ottho.co/claude-blog](https://academy.ottho.co/claude-blog)
- Issues : [github.com/ottho-nocode/ottho-blog-plugin/issues](https://github.com/ottho-nocode/ottho-blog-plugin/issues) *(repo à créer)*
- Contact : tools@ottho.co
- Ottho : [ottho.co](https://ottho.co)
