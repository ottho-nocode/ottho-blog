# ottho-blog

> Marketplace Claude Code pour les plugins du cours [academy.ottho.co/claude-blog](https://academy.ottho.co/claude-blog).

## Plugins disponibles

### `blog`, Compagnon du cours « Claude + Blog »

Met en place un blog Ghost en headless, l'habille à la charte du site existant, structure le contenu en cocon sémantique, génère et publie des articles avec garde-fous factuels :

- **`/blog:start`**, **MASTER** : orchestre les 6 phases du cours de A à Z, avec checkpoint à chaque étape, reprenable
- `/blog:setup-ghost`, Ghost sur PikaPods + détection techno + 3 scénarios d'hébergement (URL PikaPods native / sous-domaine custom / headless API pour Next.js & co) + MCP
- `/blog:theme`, theme Ghost custom à partir de `charte.md` (fork Source + override CSS + Handlebars), *scénarios A et B uniquement*
- `/blog:integrate-headless`, *(scénario C uniquement)* scaffolde le code blog dans ton framework JS (Next.js 16 en V1, Astro/SvelteKit en V1.5) : `lib/ghost.ts` + routes `/blog` + sitemap + webhook revalidation + env vars Vercel
- `/blog:integrate-admin`, *(scénario C uniquement, complémentaire de `integrate-headless`)* scaffolde un back-office UI Next.js à `<site>/cocon/admin` pour piloter la production d'articles depuis le navigateur : pipeline P1 brief (Sonnet 4.6) + P2 article (Opus 4.7) + image fal.ai + push Ghost draft, persistance locale, auth password
- `/blog:cocon`, propose un cocon sémantique depuis `brief.md`, dialogue de validation, écrit `cocon.json`
- `/blog:article`, pipeline 1 article : brief (P1) → article (P2) → image fal.ai → publish draft Ghost
- `/blog:batch <N>`, variante batch (1 ≤ N ≤ 5) avec review human-in-the-loop entre chaque
- `/blog:seo-audit`, audit SEO 8 critères par article ou pour tout le cocon
- `/blog:opportunities`, *(V1.5)* mining des opportunités SEO via Google Search Console
- `/blog:status`, état du cocon (publiés / drafts / planifiés, par pilier)
- `/blog:corrige`, diagnostic + fixes communs (theme cassé, MCP down, article qui rate, etc.)

Détail complet : [blog/README.md](./blog/README.md)

## Installation

Dans Claude Code :

```
/plugin marketplace add ottho-nocode/ottho-blog
/plugin install blog@ottho-blog
```

Pour démarrer le parcours complet en une seule commande :

```
/blog:start
```

Cette commande orchestre les 6 phases du cours (setup Ghost → theme → cocon → premier article → batch → audit SEO) avec un checkpoint à chaque étape. Reprenable via `.ottho-blog/state.json`.

## Pré-requis

- Avoir suivi (ou lu) le cours [Claude + Site web](https://academy.ottho.co/claude-site-web), ce plugin en est le prolongement
- Avoir un site HTML/CSS/JS déployé sur Vercel
- Avoir un `brief.md` et un `charte.md` à la racine du projet

## Pré-requis MCP

Le plugin utilise les MCPs suivants (installation couverte dans le cours) :

- **Ghost MCP** (obligatoire), création/édition d'articles, tags, themes, images
- **fal.ai MCP** (obligatoire), images hero via Nano Banana 2 (déjà installé du cours précédent)
- **Google Search Console MCP** (optionnel), pour `/blog:opportunities` (V1.5)

## Garde-fous non-négociables

Le plugin embarque plusieurs garde-fous pour la production éditoriale :

- **Statut Ghost forcé `draft`**, le plugin ne publie jamais direct. Tu relis dans Ghost admin, tu publies à la main.
- **Knowledge base factuelle**, injectée dans le prompt de rédaction pour éviter les hallucinations (prix, versions, statistiques).
- **Validation des liens internes**, tout lien hors whitelist est réécrit vers le pilier-parent. Aucun 404 ne sort du pipeline.
- **Sanitizer cadratin**, le tiret cadratin (—) est banni, remplacé contextuellement.
- **Rate-limit batch**, 30 secondes entre chaque article pour `/blog:batch`, max 5 articles.

## Coût indicatif

| Action | Coût LLM | Avec image fal.ai |
|---|---|---|
| `/blog:cocon` | ~0,03 € | 0,03 € |
| `/blog:article` | ~0,15 € | ~0,20 € |
| `/blog:batch 5` | ~0,75 € | ~1 € |
| `/blog:seo-audit` | ~0,02 € | 0,02 € |

Cocon complet de 50 articles : **~12,50 €** (vs 200-500 €/article chez un rédacteur SEO pro).

## Cours associé

Le plugin est conçu pour être utilisé en tandem avec le cours « Claude + Blog » sur [academy.ottho.co/claude-blog](https://academy.ottho.co/claude-blog). Chaque chapitre du cours mobilise une commande du plugin.

## Versioning

**v1.2.0**, Ajout de `/blog:integrate-admin` (back-office UI Next.js pour le scénario C). 7 skills, 11 commandes.

**v1.1.0**, Ajout de `/blog:integrate-headless` (scaffold code blog Next.js).

**v1.0.0**, Première version publique. 7 skills, 10 commandes, marketplace prêt à publier.

## Licence

MIT, voir [LICENSE](./LICENSE).

## Liens

- Cours : [academy.ottho.co/claude-blog](https://academy.ottho.co/claude-blog)
- Issues : [github.com/ottho-nocode/ottho-blog/issues](https://github.com/ottho-nocode/ottho-blog/issues)
- Contact : [tools@ottho.co](mailto:tools@ottho.co)
- Ottho : [ottho.co](https://ottho.co)
