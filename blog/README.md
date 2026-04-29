# ottho-blog

> Plugin compagnon du cours [academy.ottho.co/claude-blog](https://academy.ottho.co/claude-blog).
> Donne à ton site déployé sur Vercel un blog Ghost en headless, fidèle à ta charte, structuré en cocon sémantique, et alimenté par Claude, sans jamais taper de clé API.

## À qui ça s'adresse

Ce plugin est le prolongement du cours *Claude + Site web*. Si tu as déjà un site HTML/CSS/JS en ligne et un `brief.md` propre, `ottho-blog` ajoute la couche éditoriale :

- Un Ghost auto-hébergé (PikaPods), configuré en headless et branché sur ton projet
- Un thème Ghost custom qui reprend ta charte (couleurs, typo, navigation), pas le thème Casper par défaut
- Un cocon sémantique généré à partir de ton brief : 1 mère + 3-7 filles + 3-5 petites-filles par fille, avec maillage interne propre
- Une boucle de génération d'articles assistée (un par un avec `/blog:article`) ou en mode batch (`/blog:batch <N>`, max 5)
- Des garde-fous factuels : knowledge base injectée, validation des liens internes, sanitizer (interdit le tiret cadratin, clamp les limites Ghost)
- Un audit SEO continu : 8 critères par article, repérage des opportunités via Search Console
- Un mode debug (`/blog:corrige`) qui diagnostique et propose des fixes communs

**Pré-requis :**
- Avoir suivi (ou lu) le cours *Claude + Site web*
- Avoir un site HTML/CSS/JS déployé sur Vercel
- Avoir un `brief.md` à la racine du projet
- Avoir un `charte.md` (ou pouvoir en créer un)

## Installation

Dans Claude Code :

```
/plugin marketplace add ottho/blog
/plugin install ottho-blog@ottho
```

(Le nom du marketplace pourra changer selon la publication finale, voir le `marketplace.json` de référence.)

Vérification : tape `/` dans Claude Code, tu dois voir les 11 commandes préfixées `/blog:`.

## Pré-requis MCP

Avant utilisation, les MCPs suivants doivent être connectés (procédures couvertes dans le cours) :

| MCP | Rôle | Quand l'installer |
|---|---|---|
| **Ghost MCP** | Création/édition d'articles, tags, themes, images | `/blog:setup-ghost` (chapitre 1) |
| **fal.ai MCP** | Images hero (Nano Banana 2) | Déjà installé au cours précédent |
| **Vercel MCP** | Pas obligatoire pour ce plugin (le blog est sur Ghost, pas Next/Vercel) | Déjà installé au cours précédent |
| **GSC MCP** *(optionnel)* | Mining d'opportunités SEO | Pour `/blog:opportunities` (chapitre 7) |

Tous s'installent par dialogue dans Claude Code (« Installe le MCP [nom] »). Zéro commande bash.

## Les 11 commandes

Toutes les commandes sont namespacées sous `/blog:` pour éviter les conflits avec d'autres plugins.

### `/blog:setup-ghost`

Guide pas-à-pas pour mettre Ghost en place sur PikaPods. Détecte d'abord la techno du site (HTML pur vs framework JS), puis propose les scénarios d'hébergement adaptés :

| Scénario | URL du blog | Techno requise | Notes |
|---|---|---|---|
| **A. PikaPods URL** | `wonderful-caribou.pikapod.net` | toutes | par défaut, marche en 5 min, pas de DNS |
| **B. Sous-domaine custom** | `blog.<domaine>` | toutes | demande un domaine custom + CNAME |
| **C. Headless API** | `<site>/blog` rendu par le framework | Next.js / Astro / SvelteKit / Nuxt / Remix | architecture de référence ; demande d'écrire ~150 lignes de code dans le framework de l'élève (scaffold automatisé via `/blog:integrate-headless`) |

⚠️ **Le subpath `<site>/blog` via rewrite Vercel proxy n'est PAS supporté**, le template Ghost de PikaPods n'expose pas la variable `url` au top-level, ce qui casserait les liens canoniques, le sitemap et les liens internes. Pour avoir une URL `<site>/blog` propre, il faut soit le scénario C (headless API, framework JS requis), soit migrer vers un framework JS d'abord.

Génère `ghost-config.md` à la racine du projet (sans secrets). Compte ~15 min en scénario A, ~25-30 min en scénario B.

### `/blog:theme`

Génère un theme Ghost custom à partir de `charte.md` : fork le theme Source officiel, override CSS variables, adapte `default.hbs` (header/footer) pour matcher la nav du site existant, zip + upload + activate via MCP Ghost. Output : dossier `theme/` + ZIP + theme actif sur le blog. ~20 min.

⚠️ **Inutile en scénario C** (headless API), c'est ton framework qui rend le blog, pas Ghost.

### `/blog:integrate-headless`

**Utile uniquement en scénario C** (headless API). Scaffolde le code dans le projet de l'élève pour que son framework JS consomme la Ghost Content API et rende lui-même `<site>/blog/*`.

Génère 7 fichiers Next.js 16 (App Router) : `lib/ghost.ts`, `app/blog/page.tsx`, `app/blog/[slug]/page.tsx`, 2 stylesheets scopés, `app/sitemap.ts` (créé ou patché), `app/api/revalidate-blog/route.ts`. Met à jour `.env.example` avec les 3 variables Ghost (`GHOST_URL`, `GHOST_CONTENT_API_KEY`, `REVALIDATE_SECRET`) et guide la config Vercel + le webhook Ghost de revalidation.

Support :

| Framework | V1 | Notes |
|---|---|---|
| **Next.js 16** | scaffolding automatique | App Router uniquement (pas de Pages Router) |
| **Astro** | V1.5 | redirige vers le code de démonstration fourni par le formateur à adapter manuellement |
| **SvelteKit** | V1.5 | idem |
| **Nuxt / Remix** | non supporté | pattern identique, à reproduire à la main |

Compte ~10 min (questions + génération + collage des 3 env vars dans Vercel + config webhook).

### `/blog:integrate-admin`

**Utile uniquement en scénario C** (headless API), **complémentaire de `/blog:integrate-headless`**. Scaffolde un back-office UI Next.js dans le projet de l'élève à `<site>/cocon/admin` pour piloter la production d'articles depuis une interface graphique : liste des piliers + petites-filles du cocon, bouton « Générer l'article » qui lance le pipeline P1 brief (Sonnet 4.6) → P2 article (Opus 4.7) → image fal.ai → push Ghost en draft, persistance locale des briefs/articles sous `data/cocon/`, auth simple par cookie + password partagé.

Génère 9 fichiers : `lib/cocon.ts` (types + prompts P1/P2 + KB inline), `lib/cocon-storage.ts` (helpers JSON sur disque), `lib/ghost-admin.ts` (JWT Ghost Admin), `lib/fal.ai.ts` (client fal.ai), `lib/link-validator.ts` (whitelist URLs), 3 routes API (`/api/cocon/{brief,article,publish}`), et 3 fichiers admin (`page.tsx` + `AdminClient.tsx` + `styles.css`). Met à jour `.env.example` avec 4 nouvelles variables (`ANTHROPIC_API_KEY`, `FAL_KEY`, `GHOST_ADMIN_API_KEY`, `COCON_ADMIN_PASSWORD`).

| | `/blog:article` (CLI) | `/blog:integrate-admin` (UI) |
|---|---|---|
| **Où** | Claude Code, terminal | Navigateur, `<site>/cocon/admin` |
| **État** | éphémère par session | persisté côté serveur (`data/cocon/`) |
| **Idéal pour** | rythme régulier solo, rapide | équipe, vue d'ensemble, articles en parallèle |
| **Coût LLM** | ~0,15 € / article | ~0,15 € / article (identique) |

Les deux pipelines coexistent : même `cocon.json`, mêmes prompts, même statut Ghost forcé `draft`. Tu peux mixer librement. Compte ~12 min de scaffolding + collage des 4 env vars dans Vercel.

⚠️ **Sécurité prod** : l'admin expose des routes API qui consomment tes clés Anthropic/fal.ai/Ghost. La commande recommande d'activer la **Vercel Authentication** (Settings → Deployment Protection) ou un middleware maison en plus du password de session. ⚠️ **Filesystem read-only sur Vercel serverless** : le store local fonctionne en `pnpm dev` ou sur un hébergeur avec FS persistant, pour la prod sur Vercel, bascule `lib/cocon-storage.ts` sur Vercel Blob.

### `/blog:cocon`

Lit `brief.md`, propose un cocon sémantique complet (1 mère + 3-7 filles + 3-5 petites-filles par fille). Dialogue de validation : tu accept / modifies / refuses chaque pilier. L'IA propose, tu valides, pas de cartographie manuelle. Output : `cocon.json` finalisé. ~15 min.

### `/blog:article`

Pipeline complet pour 1 article du cocon :
1. Sélectionne une petite-fille planifiée
2. Génère le brief (P1, Sonnet ou Opus)
3. Tu valides le brief
4. Génère l'article HTML + meta SEO (P2, Opus 4.7)
5. Sanitizer (cadratin, limites Ghost)
6. Validation des liens internes (whitelist)
7. Image hero via fal.ai
8. Upload image dans Ghost media library
9. Création post Ghost en **draft** (statut forcé)
10. Update `cocon.json` (planned → draft)

Coût : ~0,15 € par article. Durée : ~60 s. Statut Ghost **toujours draft**, tu relis avant publication.

### `/blog:batch <N>`

Variante batch de `/blog:article` pour générer N articles consécutifs (1 ≤ N ≤ 5). Review human-in-the-loop entre chaque article (brief + article validés avant publication), rate-limit 30 s, abort sur 2 erreurs consécutives, status forcé draft. Coût : ~0,15 € × N. Recommandation : tester `/blog:article` au moins 1× avant le premier batch.

### `/blog:seo-audit`

Audite la qualité SEO d'un article (par slug) ou de tout le cocon. Score sur 8 critères :

1. `meta_title` ≤ 60 chars, keyword bien placé
2. `meta_description` ≤ 155 chars, action + bénéfice
3. `<h1>` unique
4. Structure H2/H3 logique (5-8 H2)
5. 4-6 liens internes (mère, sœurs, CTA)
6. Image hero avec `alt` non vide
7. Au moins 1 bloc `<pre><code>`, `<table>`, ou `<blockquote>`
8. JSON-LD `BlogPosting` + `BreadcrumbList`

Output : `seo-audit-{date}.md` avec suggestions concrètes.

### `/blog:opportunities` *(optionnel, V1.5)*

Fetch Google Search Console (via API ou MCP), filtre les requêtes en position 8-20 avec impressions > 50 sur 28 jours, cross-référence avec `cocon.json`. Pour chaque opportunité : suggestion d'article (refresh d'un existant ou nouvel article dans le pilier le plus proche).

Cette commande nécessite le MCP GSC (à packager, peut nécessiter un service account Google Cloud). À défaut, tu peux faire le mining manuellement dans le dashboard GSC.

### `/blog:status`

État du cocon en 2 secondes : total articles, published, drafts, planned, répartition par pilier, dernière publication, prochain à écrire. Lecture seule, pas de prompt LLM consommé.

### `/blog:corrige`

Diagnostic + fixes communs quand quelque chose plante :

- **Setup Ghost** : DNS, HTTPS, MCP déconnecté, API keys
- **Theme** : ZIP rejeté, theme non activé, rendu cassé
- **Cocon** : `cocon.json` corrompu, brief absent
- **Article** : pipeline qui rate, image fal.ai, erreur 422 Ghost
- **Batch** : interruption en cours, statut bizarre dans le cocon
- **Autre** : génère `bug-{date}.md` à envoyer au support

## Les 7 skills

Les skills sont la matière pédagogique encapsulée. Chaque commande mobilise une ou plusieurs skills.

| Skill | Rôle | Mobilisée par |
|---|---|---|
| `cocon-method` | Méthode Bourrelly + format `cocon.json` + protocole IA→user | `/blog:cocon` |
| `ghost-config` | PikaPods, sous-domaine, custom integration, MCP Ghost | `/blog:setup-ghost` |
| `ghost-theme` | Anatomie theme Ghost, Handlebars 30 min, mapping charte→CSS | `/blog:theme` |
| `article-quality` | System prompts P1+P2 intégraux, schemas JSON, sanitizer | `/blog:article`, `/blog:batch` |
| `link-validation` | Whitelist d'URLs, algorithme regex, fallback pilier-parent | `/blog:article`, `/blog:batch` |
| `seo-blog` | Checklist 8 points + sitemap + GSC + GA4 events | `/blog:seo-audit`, `/blog:opportunities` |
| `knowledge-base` | Pattern KB factuelle pour éviter les hallucinations | `/blog:article`, `/blog:batch` |

Chaque skill est un fichier `skills/<name>/SKILL.md` autonome. Tu peux les lire pour comprendre la méthode même sans utiliser les commandes.

## Roadmap d'usage

L'ordre suit les chapitres du cours :

1. **Chapitre 1**, `/blog:setup-ghost` → Ghost en ligne, connecté à Claude Code
2. **Chapitre 2**, `/blog:theme` *(scénarios A/B)* OU `/blog:integrate-headless` *(scénario C)* → blog ressemble à ton site
3. **Chapitre 3**, *(théorie cocon, pas de commande)*
4. **Chapitre 4**, `/blog:cocon` → stratégie SEO cartographiée
5. **Chapitre 5**, `/blog:article` ×3 → premiers articles écrits manuellement
6. **Chapitre 6**, `/blog:batch 3` → mode auto-pilote
7. **Chapitre 7**, `/blog:seo-audit` + `/blog:opportunities` → optimisation continue
8. **Transverse**, `/blog:status` à tout moment, `/blog:corrige` quand ça coince

## Philosophie

Ce plugin ne fait rien que Claude Code ne pourrait faire en dialogue libre. Chaque commande est un **dialogue structuré** + une **skill associée** qui encapsule la méthode enseignée.

Si une commande échoue ou si tu veux comprendre ce qu'elle fait, tu peux toujours la refaire à la main, c'est même ce qu'enseigne le cours. Le plugin est un raccourci, pas une boîte noire.

### Garde-fous non-négociables

- **Statut Ghost forcé à `draft`**, le plugin ne publie JAMAIS direct. Tu relis dans Ghost admin, tu publies à la main.
- **Knowledge base obligatoire** dans le prompt P2, pour éviter les hallucinations factuelles (prix, versions, statistiques).
- **Validation des liens internes**, tout `<a href="/...">` interne hors whitelist est réécrit vers la pilier-parent. Aucun 404 ne sort du pipeline.
- **Sanitizer cadratin**, le tiret cadratin (—) est banni dans tous les outputs. Remplacé contextuellement par `:`, `,`, `(`, ou `.`.
- **Rate limit 30 s** entre articles dans `/blog:batch`, pour éviter les spam d'API.
- **Max 5 articles par batch**, pour éviter de cramer le budget Anthropic en une fois.

## Coût estimé

| Action | Coût LLM | Coût total (avec image) |
|---|---|---|
| `/blog:cocon` | ~0,03 € (Sonnet) | 0,03 € |
| `/blog:article` | ~0,15 € (Opus 4.7) | ~0,20 € avec fal.ai |
| `/blog:batch 5` | ~0,75 € (Opus 4.7 × 5) | ~1 € avec images |
| `/blog:seo-audit` (cocon entier) | ~0,02 € | 0,02 € |

Un cocon complet de 50 articles : **~10 € de LLM** + **~2,50 € d'images fal.ai**. Soit **~12,50 €** pour démarrer un blog SEO complet.

À mettre en regard du coût d'un rédacteur (200-500 € par article SEO professionnel).

## Version

**v1.0.0**, Plugin complet (7 skills + 10 commandes). Production-ready après le test E2E manuel.

## État de livraison

- [x] Plugin scaffolded (`.claude-plugin/plugin.json`, structure)
- [x] 7 skills rédigées
- [x] 10/11 commandes (toutes sauf `/blog:opportunities` qui est V1.5)
- [ ] Test end-to-end sur un projet pilote (à faire avec un compte Ghost réel)
- [ ] Publication marketplace

## Licence

MIT.

## Liens

- Cours : [academy.ottho.co/claude-blog](https://academy.ottho.co/claude-blog)
- Issues : [github.com/ottho-nocode/ottho-blog-plugin/issues](https://github.com/ottho-nocode/ottho-blog-plugin/issues) *(repo à créer)*
- Contact : tools@ottho.co
- Ottho : [ottho.co](https://ottho.co)
