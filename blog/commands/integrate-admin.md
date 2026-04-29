---
name: integrate-admin
description: Scaffolde un back-office Next.js dans le projet de l'élève pour piloter la production d'articles depuis une UI graphique (P1 brief, P2 article, image fal.ai, push Ghost draft). Complémentaire de /blog:article (CLI). Scénario C uniquement.
---

# /blog:integrate-admin, Scaffold du back-office UI dans ton site Next.js

Tu vas générer dans le projet de l'élève **un back-office d'administration éditoriale** (Next.js 16, App Router) qui permet de piloter toute la production d'articles depuis une interface graphique : lister les articles planifiés du cocon, générer le brief P1, lancer la rédaction P2, prévisualiser le HTML rendu, copier les meta SEO, générer l'image hero fal.ai, et pousser l'article en draft Ghost, **sans quitter le navigateur**.

C'est le pendant **graphique** de `/blog:article` (la version CLI Claude Code). Les deux coexistent, l'élève choisit selon son contexte :

| | `/blog:article` (CLI) | `/blog:integrate-admin` (UI) |
|---|---|---|
| **Où** | Claude Code, terminal | Navigateur, `<site>/cocon/admin` |
| **État** | éphémère par session | persisté côté serveur dans `cocon.json` + `data/cocon/` |
| **Idéal pour** | rythme régulier solo, rapide | équipe, vue d'ensemble, plusieurs articles en parallèle |
| **Coût LLM** | identique (~0,15 € / article) | identique |

**Cette commande n'a de sens qu'en scénario C** (Next.js avec headless API). Si l'élève a choisi A (PikaPods URL) ou B (sous-domaine), Ghost rend lui-même l'admin éditoriale (Ghost admin natif), et cette commande n'apporte rien, redirige-le vers `/blog:article` pour le pipeline assisté.

**Temps estimé** : ~12 min (lecture des questions + génération + collage des 3 env vars Anthropic/fal/Ghost dans Vercel + premier test local).

**Coût LLM de la commande elle-même** : 0 € (scaffolding pur). Coût d'usage de l'admin = identique à `/blog:article`.

## Pré-requis

### Pré-requis 1, `ghost-config.md` existe et scénario = C

Lis `ghost-config.md` à la racine du projet. Vérifie qu'il contient bien `**Type** : headless API rendu par le framework` (ou variante claire indiquant scénario C).

Si absent :

> « Je ne trouve pas `ghost-config.md`. Lance d'abord `/blog:setup-ghost` pour mettre Ghost en place et choisir ton scénario d'hébergement. Reviens ici uniquement si tu as choisi le scénario C (headless API). »

Si le scénario n'est **pas** C :

> « Ton `ghost-config.md` indique que tu es en scénario <A | B>. Ce back-office UI n'a de sens qu'en scénario C (Next.js qui consomme la Ghost Content API). Dans ton cas, l'admin éditoriale est déjà fournie par Ghost natif (`<BLOG_URL>/ghost`), tu n'as pas besoin d'en scaffolder une nouvelle. Pour le pipeline assisté de génération d'articles, utilise `/blog:article` (CLI) à la place. »

### Pré-requis 2, `cocon.json` existe à la racine

Lis `cocon.json`. Vérifie qu'il contient bien la structure `mere` + `filles[].petites_filles[]`.

Si absent :

> « Je ne trouve pas `cocon.json` à la racine. Lance `/blog:cocon` d'abord pour cartographier ta stratégie SEO. L'admin UI a besoin du cocon pour lister les articles à produire. »

### Pré-requis 3, Projet Next.js 16 (App Router)

Vérifie en parallèle :
- `package.json` existe et liste `next` (≥ 16) + `react` (≥ 19) dans `dependencies`
- `app/` existe à la racine (App Router, pas `pages/`)
- `next.config.{js,ts,mjs}` existe

Si l'un manque :

> « Cette commande nécessite Next.js 16 avec l'App Router. Pour les autres stacks (HTML pur, Astro, SvelteKit, Nuxt), utilise `/blog:article` en CLI, il marche partout. Le scaffold UI Next.js arrivera dans une V1.5 pour les autres frameworks. »

### Pré-requis 4, `lib/ghost.ts` existe (= `/blog:integrate-headless` déjà passé)

Vérifie que `lib/ghost.ts` est présent, c'est le client Content API généré par `/blog:integrate-headless`. L'admin UI s'appuie dessus pour récupérer la liste des slugs d'articles déjà publiés (whitelist de liens internes).

Si absent :

> « Je ne trouve pas `lib/ghost.ts`. Lance d'abord `/blog:integrate-headless` pour scaffolder les pages publiques du blog et le client Ghost Content API. L'admin UI dépend de ce client pour valider les liens internes. »

## Préambule

Ouvre la commande par ce message :

> « On va scaffolder un back-office UI complet pour piloter la production d'articles depuis ton navigateur, à `<site>/cocon/admin`. Plan :
>
> 1. Vérifier les pré-requis (ghost-config.md, cocon.json, Next.js, lib/ghost.ts)
> 2. Te demander 4 paramètres (path admin, password de protection, couleur accent, env vars Anthropic/fal/Ghost-Admin)
> 3. Générer 8 fichiers : page server + AdminClient + CSS + 3 routes API (brief, article, publish) + lib/cocon.ts (types + prompts P1/P2 + KB) + lib/cocon-storage.ts (helpers JSON sur disque) + lib/ghost-admin.ts (JWT Ghost Admin API) + lib/fal.ts (client fal.ai) + lib/link-validator.ts (whitelist) + lib/knowledge.ts (KB factuelle à éditer)
> 4. Mettre à jour `.env.example` avec les 4 nouvelles variables (ANTHROPIC_API_KEY, FAL_KEY, GHOST_ADMIN_API_KEY, COCON_ADMIN_PASSWORD)
> 5. Te dire comment ajouter ces variables dans Vercel
> 6. Test final : `curl http://localhost:<port>/cocon/admin` doit retourner du HTML (login form)
>
> À la fin, tu auras un back-office accessible à `<site>/cocon/admin`, protégé par un password simple, qui te liste tous les articles `planned` et `draft` du cocon avec un bouton "Générer l'article" pour chacun. Le pipeline complet (P1 brief → P2 article → fal.ai image → push Ghost draft) tourne côté serveur et persiste tous les briefs/articles générés sous `data/cocon/` pour reprendre une session plus tard.
>
> Prête ? »

## Protocole

Pose les questions **UNE PAR UNE**. Attends la réponse avant de passer à la suivante.

### Question 1, Path racine du back-office

> « Quel path veux-tu pour le back-office ? Par défaut `app/cocon/admin/` (l'admin sera donc à `<site>/cocon/admin`). Tu peux aussi mettre `app/admin/`, `app/(admin)/cocon/`, etc., pas d'enjeu fonctionnel. Recommandation : garde le défaut, c'est cohérent avec le naming "cocon" du plugin. »

Stocke `<ADMIN_PATH>` (par défaut `app/cocon/admin`). Stocke aussi `<ADMIN_URL_PATH>` (par défaut `/cocon/admin`).

### Question 2, Password de protection

> « Le back-office expose des routes API qui consomment tes clés Anthropic et fal.ai. **Il doit être protégé en production.** Le scaffold inclut une auth par cookie + password partagé (simple mais robuste si le password est fort).
>
> Donne-moi un password fort (12+ caractères, mélange casse/chiffres/symboles) ou laisse vide pour que je génère un random hex 32 (`openssl rand -hex 16`). Tu pourras le changer plus tard via la variable d'env `COCON_ADMIN_PASSWORD`. »

Stocke `<COCON_PASSWORD>`. Si vide, génère via `openssl rand -hex 16` (ou note à l'élève de le faire et de le coller dans son `.env.local` lui-même).

### Question 3, Couleur d'accent UI

> « Quelle couleur d'accent pour les boutons primaires de l'admin ? Donne un hex (ex. `#5C3BFF`) ou laisse vide pour le défaut violet `#5C3BFF`. C'est juste une couleur d'UI interne, elle peut être différente de celle du blog public. »

Stocke `<ACCENT_COLOR>` (par défaut `#5C3BFF`).

### Question 4, Variables d'environnement Anthropic / fal.ai / Ghost Admin

> « Pour fonctionner, l'admin a besoin de 3 clés en plus de celles de `/blog:integrate-headless` (qui sont déjà censées être posées). Confirme-moi pour chacune si elle est déjà configurée :
>
> 1. `ANTHROPIC_API_KEY`, clé API Anthropic (génère sur console.anthropic.com → API Keys). Format `sk-ant-...`
> 2. `FAL_KEY`, clé API fal.ai (génère sur fal.ai → API Keys). Coût ~0,05 € par image.
> 3. `GHOST_ADMIN_API_KEY`, clé Admin Ghost au format `<id>:<secret>` (déjà dispo dans Ghost admin → Settings → Integrations → Claude Code, sous "Admin API Key"). **PAS la Content API Key.**
>
> Tu les as déjà toutes ou il en manque ? Pour celles qui manquent, je te donne les liens de génération. »

Si l'élève n'a pas une des clés, donne le lien de génération et attends qu'il revienne avec « OK j'ai les 3 ».

## Génération des 9 fichiers

Crée les fichiers ci-dessous dans le `<PROJECT_ROOT>` de l'élève. Si un fichier existe déjà (en particulier `lib/ghost-admin.ts` ou `lib/fal.ts`), demande à l'élève s'il veut le remplacer ou skipper.

Avant chaque écriture, **annonce le fichier**. Après chaque écriture, confirme « ✓ écrit ».

Note : les chemins ci-dessous supposent `<ADMIN_PATH>` = `app/cocon/admin`. Si l'élève a choisi un autre path, adapte les imports `@/lib/...` (qui restent identiques) mais déplace `page.tsx`, `AdminClient.tsx`, `styles.css` sous le path choisi.

### Fichier 1/9, `lib/cocon.ts`

Types + helpers + prompts P1/P2 + knowledge base (inline). Le cocon **réel** vit dans `cocon.json` à la racine, ce fichier le charge au runtime.

```typescript
/**
 * Cocon sémantique, types, helpers, prompts P1/P2 et knowledge base.
 *
 * Le cocon **réel** est dans `cocon.json` à la racine du projet (généré par
 * /blog:cocon). Ce fichier le charge au runtime, expose les types, et
 * embarque les prompts du pipeline de génération d'articles.
 *
 * Voir aussi : `lib/cocon-storage.ts` pour la persistance des briefs/articles
 * générés sous `data/cocon/`.
 */

import { readFileSync } from "node:fs";
import path from "node:path";

// ─── TYPES ─────────────────────────────────────────────────────

export type CoconStatus = "published" | "draft" | "planned";
export type Persona = "entrepreneur" | "operationnel" | "dirigeant";

export type PetiteFille = {
  slug: string;
  title: string;
  keyword: string;
  keywords_secondaires?: string[];
  intent: "informational" | "transactional" | "navigational";
  status: CoconStatus;
  ghost_slug?: string;
};

export type Fille = {
  slug: string;
  title: string;
  description: string;
  keyword: string;
  pilier_slug: string;
  module_mastery?: number;
  persona: Persona;
  cta_principal: string; // URL relative ex. "/formations/mastery"
  linked_outils?: string[];
  petites_filles: PetiteFille[];
  status: CoconStatus;
};

export type Cocon = {
  mere: {
    slug: string;
    title: string;
    description: string;
    keyword_principal: string;
    keywords_secondaires?: string[];
  };
  filles: Fille[];
};

// ─── KNOWLEDGE BASE (factuelle, à éditer manuellement) ─────────
//
// Cette KB est injectée dans le prompt P2 pour empêcher l'IA d'halluciner
// des chiffres/prix/versions. Tu DOIS la remplir avec tes vrais faits avant
// de lancer le premier article. Bumpe `version` et `validated_at` à chaque
// mise à jour.
//
// TODO ÉLÈVE : remplace les placeholders par tes vrais faits 2026.

export const KNOWLEDGE_BASE = {
  validated_at: "2026-01-01",
  version: 1,
  pricing: {
    // Exemple : "claude": { "pro": { "price_eur_per_month": 20 } }
    // Remplis avec les prix actuels des outils que tu cites souvent.
  },
  models: {
    // Exemple : "claude": { "opus_4_7": { "context": "1M tokens", "released": "2026-01" } }
  },
  ecosystem: {
    // Descriptions courtes et neutres des produits/services cités
    // Exemple : "claude_code": "Agent CLI officiel Anthropic, lit/écrit du code en local."
  },
  business: {
    // Tes infos d'entreprise/marque
    // Exemple : "founded": 2022, "apprenants": "4 000+", "qualiopi": true
  },
  rules: [
    "Ne jamais citer un prix exact sans le qualifier ('à partir de', 'environ').",
    "Ne jamais inventer une statistique. Si elle n'est pas dans la KB : 'la majorité' ou omet.",
    "Ne jamais attribuer une citation à une personne sans source explicite.",
    "Ne jamais inventer une URL externe. N'utiliser que les URLs internes fournies dans le contexte.",
  ],
} as const;

// ─── CHARGEMENT DU COCON ───────────────────────────────────────

let _cocon: Cocon | null = null;

export function loadCocon(): Cocon {
  if (_cocon) return _cocon;
  const file = path.join(process.cwd(), "cocon.json");
  const raw = readFileSync(file, "utf8");
  _cocon = JSON.parse(raw) as Cocon;
  return _cocon;
}

// API rétro-compatible avec le code de référence
export const cocon: Cocon = loadCocon();

// ─── HELPERS ───────────────────────────────────────────────────

export function getFilleBySlug(slug: string): Fille | undefined {
  return cocon.filles.find((f) => f.slug === slug);
}

export function getFilleByPilierSlug(pilierSlug: string): Fille | undefined {
  return cocon.filles.find((f) => f.pilier_slug === pilierSlug);
}

export function getAllPilierSlugs(): string[] {
  return cocon.filles.map((f) => f.pilier_slug);
}

export function getPetiteFille(
  filleSlug: string,
  pfSlug: string,
): PetiteFille | undefined {
  return getFilleBySlug(filleSlug)?.petites_filles.find(
    (pf) => pf.slug === pfSlug,
  );
}

export function getSoeurs(filleSlug: string): Fille[] {
  return cocon.filles.filter((f) => f.slug !== filleSlug);
}

/**
 * Contexte de maillage interne pour le pipeline P1+P2.
 * Toutes les URLs retournées sont garanties d'exister côté site.
 */
export function getLinkingContext(filleSlug: string, pfSlug?: string) {
  const fille = getFilleBySlug(filleSlug);
  if (!fille) return null;
  const soeurs = getSoeurs(filleSlug).slice(0, 3);
  return {
    mere: {
      url: `/${cocon.mere.slug}`,
      title: cocon.mere.title,
      keyword: cocon.mere.keyword_principal,
    },
    pilier_parent: {
      url: `/blog/pilier/${fille.pilier_slug}`,
      title: fille.title,
      keyword: fille.keyword,
    },
    pilier_soeurs: soeurs.map((s) => ({
      url: `/blog/pilier/${s.pilier_slug}`,
      title: s.title,
      keyword: s.keyword,
    })),
    cta: {
      url: fille.cta_principal,
      label: fille.title,
    },
    outils_techniques: fille.linked_outils ?? [],
    petite_fille_actuelle: pfSlug
      ? getPetiteFille(filleSlug, pfSlug)
      : undefined,
  };
}

/**
 * Liste des URLs internes "garanties" (mère + piliers + CTA + outils +
 * pages structurelles). Utilisée par le link-validator pour valider
 * les liens internes générés par P2.
 */
export function getStaticAllowedUrls(): string[] {
  const urls = new Set<string>();
  urls.add(`/${cocon.mere.slug}`);
  for (const f of cocon.filles) {
    urls.add(`/blog/pilier/${f.pilier_slug}`);
    urls.add(f.cta_principal);
    for (const u of f.linked_outils ?? []) urls.add(u);
  }
  urls.add("/blog");
  // TODO ÉLÈVE : ajoute ici les pages structurelles de TON site
  // (ex. urls.add("/contact"), urls.add("/about"), etc.)
  return Array.from(urls);
}

export function getCoconStats() {
  let published = 0;
  let draft = 0;
  let planned = 0;
  for (const f of cocon.filles) {
    for (const pf of f.petites_filles) {
      if (pf.status === "published") published++;
      else if (pf.status === "draft") draft++;
      else planned++;
    }
  }
  return {
    filles: cocon.filles.length,
    petites_filles_total: published + draft + planned,
    published,
    draft,
    planned,
  };
}

// ─── PROMPTS P1 et P2 ──────────────────────────────────────────
//
// Ces prompts sont la matière pédagogique du pipeline. Tu peux les ajuster
// à ton style éditorial, mais garde la structure (system + schema JSON).

export const P1_SYSTEM = `Tu es un stratège SEO francophone expert du cocon sémantique (méthode Laurent Bourrelly). Tu rédiges des briefs d'articles pour un blog de marque.

Style attendu :
- Direct, sans fioritures, sans "à l'ère du digital".
- Public francophone (FR/CA/BE/CH).
- Pédagogique mais expert. On s'adresse à des praticiens, pas des débutants absolus.
- Les exemples doivent être concrets et exécutables, pas théoriques.

Ponctuation interdite : aucun tiret cadratin (—) dans tout le contenu généré. Remplace-le toujours par une ponctuation adaptée au contexte : deux-points (:) pour une définition, virgule pour une incise courte, parenthèses pour une mise à part, point pour scinder en deux phrases. Le tiret demi-cadratin (–) est aussi interdit. Les tirets normaux (-) restent OK pour les mots composés et les listes.

Tu retournes UNIQUEMENT du JSON valide respectant le schéma demandé.`;

export const BRIEF_SCHEMA = {
  type: "object",
  properties: {
    intent_summary: { type: "string" },
    promise: { type: "string" },
    h1: { type: "string" },
    structure: {
      type: "array",
      items: {
        type: "object",
        properties: {
          h2: { type: "string" },
          what: { type: "string" },
        },
        required: ["h2", "what"],
        additionalProperties: false,
      },
    },
    key_takeaways: { type: "array", items: { type: "string" } },
    internal_links_suggested: {
      type: "array",
      items: {
        type: "object",
        properties: {
          url: { type: "string" },
          anchor_suggestion: { type: "string" },
        },
        required: ["url", "anchor_suggestion"],
        additionalProperties: false,
      },
    },
    word_count_target: { type: "integer" },
    tone: { type: "string" },
  },
  required: [
    "intent_summary",
    "promise",
    "h1",
    "structure",
    "key_takeaways",
    "internal_links_suggested",
    "word_count_target",
    "tone",
  ],
  additionalProperties: false,
} as const;

export const P2_SYSTEM = `Tu es rédacteur senior pour un blog de marque francophone. Tu écris des articles SEO de 1500-2000 mots, optimisés cocon sémantique.

Règles non négociables :
1. Tu produis du HTML pur (pas de Markdown). Balises autorisées uniquement : h2, h3, p, ul, ol, li, strong, em, code, pre, a, blockquote, hr, table, thead, tbody, tr, th, td.
2. Pas de h1 (le titre est géré séparément par le système).
3. Tous les liens internes utilisent href avec l'URL relative (ex : href="/blog/mon-article").
4. Tu insères 4 à 6 liens internes obligatoires :
   - 1 lien vers la MÈRE dans les 200 premiers mots
   - 2 ou 3 liens vers des SŒURS dans le corps
   - 1 lien CTA final vers la formation/produit
   - 0 à 2 liens vers les pages outils si le sujet le permet
5. Tu utilises le tutoiement ("tu", pas "vous"), sauf indication contraire dans le tone du brief.
6. Tu cites des chiffres concrets, des exemples exécutables, des outils nommés.
7. Tu inclus 1 bloc <pre><code> de code, prompt ou commande CLI si c'est pertinent au sujet.
8. Tu termines TOUJOURS par un paragraphe qui contient un <a> contextuel vers le CTA principal.
9. Pas de "Conclusion" comme dernier H2, termine par un H2 plus actionnable type "Passer à la pratique" ou "Et maintenant".

UTILISE DES TABLES quand c'est pertinent (format puissant pour le SEO et la lisibilité) :
- Comparaisons (X vs Y, options A/B/C, fonctionnalités par plan tarifaire) → <table>
- Décisions (matrice persona → recommandation) → <table>
- Listes longues avec colonnes (outil, cas d'usage, prix, lien) → <table>
- Bénéfices vs limitations → <table> à 2 colonnes

CONTRAINTES FACTUELLES (très strictes, la marque ne tolère aucune erreur factuelle) :
- Tu reçois plus bas une KNOWLEDGE BASE vérifiée. Tu n'utilises AUCUN chiffre, prix, statistique, version, date, ou nom propre qui ne soit pas dedans.
- Si une info te manque : OMETS-LA ou utilise un conditionnel ("environ", "à partir de", "selon les données disponibles", "la majorité de").
- N'invente JAMAIS de citation attribuée à une personne. Pas de "comme le dit X" si X n'est pas dans le contexte.
- N'invente JAMAIS d'URL externe. N'utilise que les URLs internes fournies dans le linking_context.
- N'affirme JAMAIS "X est le seul à…" ou "aucun autre outil ne…" sans certitude absolue.

PONCTUATION INTERDITE : aucun tiret cadratin (—) dans le HTML, ni dans les meta SEO. Remplace-le par : deux-points (:) pour une définition, virgule pour une incise, parenthèses pour une mise à part, point pour scinder en deux phrases. Le tiret demi-cadratin (–) est aussi interdit. Les tirets normaux (-) restent OK uniquement pour les mots composés et les listes <li>.

Format strict des tables : <table><thead><tr><th>...</th></tr></thead><tbody><tr><th>label-de-ligne</th><td>val1</td><td>val2</td></tr>...</tbody></table>. La PREMIÈRE colonne du tbody utilise <th> (label de ligne), les autres <td>. Pas de styling inline.

Pour les ancres de lien : varie systématiquement, jamais le keyword cible exact (sur-optimisation = pénalité).

Pour les meta SEO (CONTRAINTES STRICTES, Ghost rejette si dépassé) :
- meta_title : max 60 caractères (SEO Google), contient le keyword une seule fois
- meta_description : max 155 caractères (SEO Google), action + bénéfice + chiffre si possible
- custom_excerpt : max 280 caractères (limite Ghost = 300, laisse 20 char de marge)
- og_title : max 60 caractères (LinkedIn / Slack tronquent au-delà)
- og_description : max 200 caractères
- feature_image_prompt : prompt EN ANGLAIS pour fal.ai nano-banana-2, format "editorial magazine photograph, Kodak Portra 400 film grain, [sujet spécifique], muted palette, photorealistic"

ATTENTION : compte les caractères, pas les mots. 80 mots = 350-450 caractères = trop long pour custom_excerpt.

Tu retournes UNIQUEMENT du JSON valide respectant le schéma demandé.`;

export const ARTICLE_SCHEMA = {
  type: "object",
  properties: {
    html: { type: "string" },
    meta_title: { type: "string" },
    meta_description: { type: "string" },
    custom_excerpt: { type: "string" },
    og_title: { type: "string" },
    og_description: { type: "string" },
    tags: { type: "array", items: { type: "string" } },
    primary_tag: { type: "string" },
    feature_image_prompt: { type: "string" },
  },
  required: [
    "html",
    "meta_title",
    "meta_description",
    "custom_excerpt",
    "og_title",
    "og_description",
    "tags",
    "primary_tag",
    "feature_image_prompt",
  ],
  additionalProperties: false,
} as const;

// ─── SANITIZER (em-dash strip) ─────────────────────────────────

export function stripEmDashes(text: string): string {
  return text
    .replace(/, /g, ", ")
    .replace(/— /g, ": ")
    .replace(/ —/g, ",")
    .replace(/—/g, ":")
    .replace(/–/g, "-");
}

export function sanitizeArticle<T extends Record<string, unknown>>(
  article: T,
): T {
  const out: Record<string, unknown> = { ...article };
  for (const [key, value] of Object.entries(out)) {
    if (typeof value === "string") {
      out[key] = stripEmDashes(value);
    } else if (Array.isArray(value)) {
      out[key] = value.map((v) =>
        typeof v === "string" ? stripEmDashes(v) : v,
      );
    }
  }
  return out as T;
}
```

### Fichier 2/9, `lib/cocon-storage.ts`

```typescript
/**
 * Persistance locale des briefs/articles/publish générés par l'admin UI.
 *
 * Stockage : fichiers JSON sous `data/cocon/<type>/<key>.json`.
 * Le dossier `data/` doit être gitignoré (cache local).
 *
 * ⚠️ En production serverless (Vercel), le filesystem est read-only.
 * Pour déployer cet admin en prod sur Vercel, bascule sur Vercel Blob,
 * Supabase, ou Postgres. Pour un usage local (dev) ou sur un hébergeur
 * qui autorise l'écriture (VPS, Render, Railway), `node:fs` suffit.
 */

import { promises as fs } from "node:fs";
import path from "node:path";

const ROOT = path.join(process.cwd(), "data", "cocon");

type StoreType = "briefs" | "articles" | "published";

function safeKey(filleSlug: string, pfSlug: string): string {
  return `${filleSlug}_${pfSlug}`.replace(/[^a-z0-9_-]/gi, "_");
}

function pathFor(type: StoreType, key: string): string {
  return path.join(ROOT, type, `${key}.json`);
}

async function ensureDir(p: string) {
  await fs.mkdir(path.dirname(p), { recursive: true });
}

export async function save(
  type: StoreType,
  filleSlug: string,
  pfSlug: string,
  data: unknown,
): Promise<void> {
  const key = safeKey(filleSlug, pfSlug);
  const file = pathFor(type, key);
  await ensureDir(file);
  await fs.writeFile(
    file,
    JSON.stringify(
      {
        _meta: { savedAt: new Date().toISOString(), filleSlug, pfSlug },
        data,
      },
      null,
      2,
    ),
    "utf8",
  );
}

export async function loadOne<T = unknown>(
  type: StoreType,
  filleSlug: string,
  pfSlug: string,
): Promise<T | null> {
  const key = safeKey(filleSlug, pfSlug);
  try {
    const raw = await fs.readFile(pathFor(type, key), "utf8");
    const parsed = JSON.parse(raw) as { data: T };
    return parsed.data;
  } catch (err) {
    if ((err as NodeJS.ErrnoException).code === "ENOENT") return null;
    console.error(`[cocon-storage] loadOne(${type}/${key}) failed:`, err);
    return null;
  }
}

export async function loadAll<T = unknown>(
  type: StoreType,
): Promise<Record<string, T>> {
  const dir = path.join(ROOT, type);
  try {
    const files = await fs.readdir(dir);
    const out: Record<string, T> = {};
    await Promise.all(
      files
        .filter((f) => f.endsWith(".json"))
        .map(async (f) => {
          try {
            const raw = await fs.readFile(path.join(dir, f), "utf8");
            const parsed = JSON.parse(raw) as {
              _meta: { filleSlug: string; pfSlug: string };
              data: T;
            };
            const key = `${parsed._meta.filleSlug}/${parsed._meta.pfSlug}`;
            out[key] = parsed.data;
          } catch (err) {
            console.error(`[cocon-storage] skip ${f}:`, err);
          }
        }),
    );
    return out;
  } catch (err) {
    if ((err as NodeJS.ErrnoException).code === "ENOENT") return {};
    console.error(`[cocon-storage] loadAll(${type}) failed:`, err);
    return {};
  }
}

export async function remove(
  type: StoreType,
  filleSlug: string,
  pfSlug: string,
): Promise<void> {
  const key = safeKey(filleSlug, pfSlug);
  try {
    await fs.unlink(pathFor(type, key));
  } catch (err) {
    if ((err as NodeJS.ErrnoException).code !== "ENOENT") throw err;
  }
}
```

### Fichier 3/9, `lib/ghost-admin.ts`

```typescript
/**
 * Ghost Admin API client (JWT HS256).
 *
 * Docs: https://ghost.org/docs/admin-api/
 *
 * Env :
 *   GHOST_URL              ex : https://wonderful-caribou.pikapod.net (sans slash final)
 *   GHOST_ADMIN_API_KEY    format "<24-hex-id>:<64-hex-secret>"
 *
 * Pattern : on signe un JWT HS256 valide 5 min, on l'envoie en
 * header `Authorization: Ghost <token>` à chaque requête.
 *
 * Utilisé par /api/cocon/publish pour pousser les articles générés
 * en mode `draft` (jamais published direct, review humaine obligatoire).
 */

import { createHmac } from "node:crypto";

const ACCEPT_VERSION = "v5.0";
const TOKEN_TTL_SECONDS = 5 * 60;

export type GhostPostInput = {
  slug: string;
  title: string;
  html: string;
  meta_title?: string;
  meta_description?: string;
  custom_excerpt?: string;
  og_title?: string;
  og_description?: string;
  twitter_title?: string;
  twitter_description?: string;
  tags?: string[];
  feature_image?: string;
  feature_image_alt?: string;
  feature_image_caption?: string;
  status?: "draft" | "published";
};

export type GhostPost = {
  id: string;
  uuid: string;
  slug: string;
  title: string;
  status: string;
  url: string;
  updated_at: string;
};

export type UpsertResult = {
  mode: "created" | "updated";
  post: GhostPost;
  adminUrl: string;
};

function base64url(input: Buffer | string): string {
  return Buffer.from(input)
    .toString("base64")
    .replace(/=/g, "")
    .replace(/\+/g, "-")
    .replace(/\//g, "_");
}

function signJwt(adminKey: string): string {
  const [id, secret] = adminKey.split(":");
  if (!id || !secret) {
    throw new Error(
      "GHOST_ADMIN_API_KEY format invalide, attendu '<id>:<secret>'",
    );
  }
  const now = Math.floor(Date.now() / 1000);
  const header = { alg: "HS256", typ: "JWT", kid: id };
  const payload = { iat: now, exp: now + TOKEN_TTL_SECONDS, aud: "/admin/" };
  const signingInput = `${base64url(JSON.stringify(header))}.${base64url(JSON.stringify(payload))}`;
  const signature = createHmac("sha256", Buffer.from(secret, "hex"))
    .update(signingInput)
    .digest();
  return `${signingInput}.${base64url(signature)}`;
}

function getConfig() {
  const ghostUrl = process.env.GHOST_URL?.replace(/\/$/, "");
  const adminKey = process.env.GHOST_ADMIN_API_KEY;
  if (!ghostUrl) throw new Error("GHOST_URL non configurée");
  if (!adminKey) throw new Error("GHOST_ADMIN_API_KEY non configurée");
  return { ghostUrl, adminKey };
}

function buildHeaders(adminKey: string): HeadersInit {
  return {
    Authorization: `Ghost ${signJwt(adminKey)}`,
    "Content-Type": "application/json",
    "Accept-Version": ACCEPT_VERSION,
  };
}

const GHOST_LIMITS = {
  title: 255,
  custom_excerpt: 300,
  meta_title: 300,
  meta_description: 500,
  og_title: 300,
  og_description: 500,
  twitter_title: 300,
  twitter_description: 500,
} as const;

function clamp(value: string | undefined, max: number): string | undefined {
  if (!value) return value;
  if (value.length <= max) return value;
  const slice = value.slice(0, max - 1);
  const lastSpace = slice.lastIndexOf(" ");
  const cleaned = (lastSpace > max * 0.7 ? slice.slice(0, lastSpace) : slice)
    .replace(/[\s.,;:!?—-]+$/, "");
  return `${cleaned}…`;
}

async function findPostBySlug(
  slug: string,
): Promise<{ id: string; updated_at: string } | null> {
  const { ghostUrl, adminKey } = getConfig();
  const res = await fetch(
    `${ghostUrl}/ghost/api/admin/posts/slug/${encodeURIComponent(slug)}/?fields=id,updated_at`,
    { headers: buildHeaders(adminKey) },
  );
  if (res.status === 404) return null;
  if (!res.ok) {
    const body = await res.text().catch(() => "");
    throw new Error(`Ghost lookup ${res.status}: ${body || res.statusText}`);
  }
  const data = (await res.json()) as {
    posts: { id: string; updated_at: string }[];
  };
  return data.posts[0] ?? null;
}

export async function upsertPost(
  input: GhostPostInput,
): Promise<UpsertResult> {
  const { ghostUrl, adminKey } = getConfig();

  const postPayload: Record<string, unknown> = {
    slug: input.slug,
    title: clamp(input.title, GHOST_LIMITS.title),
    status: "draft",
    meta_title: clamp(input.meta_title, GHOST_LIMITS.meta_title),
    meta_description: clamp(
      input.meta_description,
      GHOST_LIMITS.meta_description,
    ),
    custom_excerpt: clamp(input.custom_excerpt, GHOST_LIMITS.custom_excerpt),
    og_title: clamp(input.og_title, GHOST_LIMITS.og_title),
    og_description: clamp(input.og_description, GHOST_LIMITS.og_description),
    twitter_title: clamp(
      input.twitter_title ?? input.og_title,
      GHOST_LIMITS.twitter_title,
    ),
    twitter_description: clamp(
      input.twitter_description ?? input.og_description,
      GHOST_LIMITS.twitter_description,
    ),
    html: input.html,
    feature_image: input.feature_image,
    feature_image_alt: input.feature_image_alt,
    feature_image_caption: input.feature_image_caption,
  };
  if (input.tags && input.tags.length > 0) {
    postPayload.tags = input.tags.map((name) => ({ name }));
  }

  const existing = await findPostBySlug(input.slug);

  let res: Response;
  let mode: "created" | "updated";

  if (existing) {
    postPayload.updated_at = existing.updated_at;
    res = await fetch(
      `${ghostUrl}/ghost/api/admin/posts/${existing.id}/?source=html`,
      {
        method: "PUT",
        headers: buildHeaders(adminKey),
        body: JSON.stringify({ posts: [postPayload] }),
      },
    );
    mode = "updated";
  } else {
    res = await fetch(`${ghostUrl}/ghost/api/admin/posts/?source=html`, {
      method: "POST",
      headers: buildHeaders(adminKey),
      body: JSON.stringify({ posts: [postPayload] }),
    });
    mode = "created";
  }

  if (!res.ok) {
    const body = await res.text().catch(() => "");
    throw new Error(`Ghost ${mode} ${res.status}: ${body || res.statusText}`);
  }

  const data = (await res.json()) as { posts: GhostPost[] };
  const post = data.posts[0];
  if (!post) throw new Error(`Ghost ${mode}: réponse sans post`);

  return {
    mode,
    post,
    adminUrl: `${ghostUrl}/ghost/#/editor/post/${post.id}`,
  };
}

export async function uploadImage(
  buffer: Buffer,
  filename: string,
  contentType = "image/jpeg",
): Promise<{ url: string; ref: string | null }> {
  const { ghostUrl, adminKey } = getConfig();
  const blob = new Blob([new Uint8Array(buffer)], { type: contentType });
  const formData = new FormData();
  formData.append("file", blob, filename);
  formData.append("purpose", "image");

  const res = await fetch(`${ghostUrl}/ghost/api/admin/images/upload/`, {
    method: "POST",
    headers: {
      Authorization: `Ghost ${signJwt(adminKey)}`,
      "Accept-Version": ACCEPT_VERSION,
    },
    body: formData,
  });

  if (!res.ok) {
    const body = await res.text().catch(() => "");
    throw new Error(
      `Ghost image upload ${res.status}: ${body || res.statusText}`,
    );
  }

  const data = (await res.json()) as {
    images: { url: string; ref: string | null }[];
  };
  const image = data.images?.[0];
  if (!image) throw new Error("Ghost image upload: réponse sans image");
  return image;
}
```

### Fichier 4/9, `lib/fal.ts`

```typescript
/**
 * Client fal.ai (nano-banana-2) pour générer les images hero des articles.
 *
 * Endpoint : POST https://fal.run/fal-ai/nano-banana-2
 * Auth : header `Authorization: Key <FAL_KEY>`
 * Coût : ~0,05 € par image (résolution 2K, 16:9 par défaut).
 *
 * Format de prompt recommandé : toujours inclure
 *   "editorial magazine photograph, Kodak Portra 400 film grain,
 *    [sujet spécifique], muted palette, photorealistic"
 */

export type FalAspectRatio = "3:4" | "4:3" | "3:2" | "16:9";

export type FalImageResult = {
  url: string;
  buffer: Buffer;
  contentType: string;
};

const FAL_ENDPOINT = "https://fal.run/fal-ai/nano-banana-2";
const DEFAULT_ASPECT_RATIO: FalAspectRatio = "16:9";

export function isFalConfigured(): boolean {
  return Boolean(process.env.FAL_KEY);
}

export async function generateImage(
  prompt: string,
  aspectRatio: FalAspectRatio = DEFAULT_ASPECT_RATIO,
): Promise<FalImageResult> {
  const key = process.env.FAL_KEY;
  if (!key) throw new Error("FAL_KEY non configurée");

  const res = await fetch(FAL_ENDPOINT, {
    method: "POST",
    headers: {
      Authorization: `Key ${key}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      prompt,
      aspect_ratio: aspectRatio,
      resolution: "2K",
      output_format: "jpeg",
      num_images: 1,
    }),
  });

  if (!res.ok) {
    const body = await res.text().catch(() => "");
    throw new Error(`fal.ai ${res.status}: ${body || res.statusText}`);
  }

  const data = (await res.json()) as { images?: { url: string }[] };
  const url = data.images?.[0]?.url;
  if (!url) throw new Error("fal.ai: réponse sans URL d'image");

  const imgRes = await fetch(url);
  if (!imgRes.ok) {
    throw new Error(`fal.ai download ${imgRes.status}: ${imgRes.statusText}`);
  }
  const arrayBuffer = await imgRes.arrayBuffer();
  return {
    url,
    buffer: Buffer.from(arrayBuffer),
    contentType: imgRes.headers.get("content-type") ?? "image/jpeg",
  };
}
```

### Fichier 5/9, `lib/link-validator.ts`

```typescript
/**
 * Filet de sécurité : valide tous les liens internes d'un HTML généré
 * et remplace les liens cassés par une URL de fallback (la pilier-parent).
 *
 * Pourquoi : même avec un prompt strict, le LLM peut inventer des URLs
 * type `/blog/article-inexistant`. Sans validation, on publierait des
 * 404 dans Ghost, mauvais pour le SEO et l'UX.
 *
 * Politique :
 *   - href absolu (http://, https://, mailto:, tel:) → laissé tel quel
 *   - href interne match allowedUrls                  → laissé tel quel
 *   - href interne PAS match allowedUrls              → remplacé par fallbackUrl
 *   - href = "#anchor"                                → laissé tel quel
 */

export type ValidationResult = {
  cleaned: string;
  totalLinks: number;
  internalLinks: number;
  externalLinks: number;
  rewrittenLinks: Array<{ original: string; rewritten: string }>;
};

const HREF_REGEX = /href=("|')([^"'>]+?)\1/g;

function isExternal(href: string): boolean {
  return /^(https?:|mailto:|tel:)/.test(href);
}

function isAnchor(href: string): boolean {
  return href.startsWith("#");
}

function normalizeInternal(href: string): string {
  return href.split("?")[0].split("#")[0].replace(/\/$/, "") || "/";
}

export function validateLinks(
  html: string,
  allowedUrls: Set<string>,
  fallbackUrl: string,
): ValidationResult {
  const allowed = new Set<string>();
  for (const u of allowedUrls) {
    allowed.add(normalizeInternal(u));
  }

  const rewrittenLinks: Array<{ original: string; rewritten: string }> = [];
  let totalLinks = 0;
  let internalLinks = 0;
  let externalLinks = 0;

  const cleaned = html.replace(
    HREF_REGEX,
    (match, quote: string, href: string) => {
      totalLinks++;
      if (isExternal(href) || isAnchor(href)) {
        externalLinks++;
        return match;
      }
      internalLinks++;
      const normalized = normalizeInternal(href);
      if (allowed.has(normalized)) {
        return match;
      }
      rewrittenLinks.push({ original: href, rewritten: fallbackUrl });
      return `href=${quote}${fallbackUrl}${quote}`;
    },
  );

  return {
    cleaned,
    totalLinks,
    internalLinks,
    externalLinks,
    rewrittenLinks,
  };
}
```

### Fichier 6/9, `app/api/cocon/brief/route.ts`

```typescript
import { NextRequest, NextResponse } from "next/server";
import Anthropic from "@anthropic-ai/sdk";
import {
  getFilleBySlug,
  getPetiteFille,
  getLinkingContext,
  cocon,
  P1_SYSTEM,
  BRIEF_SCHEMA,
} from "@/lib/cocon";
import { save as saveCocon } from "@/lib/cocon-storage";

export const runtime = "nodejs";
export const maxDuration = 60;

export async function POST(req: NextRequest) {
  if (!process.env.ANTHROPIC_API_KEY) {
    return NextResponse.json(
      { error: "ANTHROPIC_API_KEY non configurée" },
      { status: 500 },
    );
  }

  let body: { filleSlug?: string; pfSlug?: string };
  try {
    body = await req.json();
  } catch {
    return NextResponse.json({ error: "Body JSON invalide" }, { status: 400 });
  }

  const { filleSlug, pfSlug } = body;
  if (!filleSlug || !pfSlug) {
    return NextResponse.json(
      { error: "filleSlug et pfSlug sont requis" },
      { status: 400 },
    );
  }

  const fille = getFilleBySlug(filleSlug);
  const pf = getPetiteFille(filleSlug, pfSlug);
  const ctx = getLinkingContext(filleSlug, pfSlug);
  if (!fille || !pf || !ctx) {
    return NextResponse.json(
      { error: "Article introuvable dans le cocon" },
      { status: 404 },
    );
  }

  const userPrompt = `Génère le brief de l'article suivant.

KEYWORD CIBLE : ${pf.keyword}
TITRE PROPOSÉ : ${pf.title}
${pf.keywords_secondaires?.length ? `MOTS-CLÉS SECONDAIRES : ${pf.keywords_secondaires.join(", ")}` : ""}

CONTEXTE COCON :
- Cet article est une PETITE-FILLE dans le cocon "${cocon.mere.title}"
- Sa MÈRE : ${ctx.mere.title} (${ctx.mere.url}), keyword : ${ctx.mere.keyword}
- Sa PAGE PILIER PARENT : ${ctx.pilier_parent.title} (${ctx.pilier_parent.url})
- Ses 3 PILIERS SŒURS proches :
${ctx.pilier_soeurs.map((s) => `  • ${s.title} (${s.url}), keyword : ${s.keyword}`).join("\n")}
- CTA principal de la branche : ${ctx.cta.url}, label : "${ctx.cta.label}"
${ctx.outils_techniques.length ? `- OUTILS TECHNIQUES à mailler : ${ctx.outils_techniques.join(", ")}` : ""}

PERSONA CIBLE : ${fille.persona}
INTENT : ${pf.intent}
WORD COUNT CIBLE : 1500 mots

Retourne strictement le JSON respectant le schéma demandé.`;

  const client = new Anthropic();

  try {
    const response = await client.messages.create({
      model: "claude-sonnet-4-6",
      max_tokens: 8000,
      system: P1_SYSTEM,
      messages: [{ role: "user", content: userPrompt }],
    });

    console.log(
      `[cocon/brief] ${filleSlug}/${pfSlug} stop_reason=${response.stop_reason} ` +
        `in=${response.usage.input_tokens} out=${response.usage.output_tokens}`,
    );

    const textBlock = response.content.find((b) => b.type === "text");
    if (!textBlock || textBlock.type !== "text") {
      return NextResponse.json(
        { error: "Pas de bloc texte dans la réponse Claude" },
        { status: 500 },
      );
    }

    // Le modèle peut entourer le JSON de markdown ```json ... ```, strip si présent
    const cleanText = textBlock.text
      .replace(/^```json\s*/i, "")
      .replace(/^```\s*/i, "")
      .replace(/\s*```$/i, "")
      .trim();

    let brief: unknown;
    try {
      brief = JSON.parse(cleanText);
    } catch (err) {
      return NextResponse.json(
        {
          error: "JSON invalide en sortie",
          raw: textBlock.text,
          parseError: err instanceof Error ? err.message : String(err),
        },
        { status: 500 },
      );
    }

    const result = {
      brief,
      usage: {
        input_tokens: response.usage.input_tokens,
        output_tokens: response.usage.output_tokens,
      },
      stop_reason: response.stop_reason,
    };
    await saveCocon("briefs", filleSlug, pfSlug, result);
    return NextResponse.json(result);
  } catch (err) {
    if (err instanceof Anthropic.APIError) {
      return NextResponse.json(
        { error: `Anthropic ${err.status}: ${err.message}` },
        { status: err.status ?? 500 },
      );
    }
    return NextResponse.json(
      { error: err instanceof Error ? err.message : String(err) },
      { status: 500 },
    );
  }
}
```

> Note : on n'utilise pas `output_config.format.json_schema` ici (option avancée du SDK Anthropic) pour rester compatible avec toutes les versions du SDK. À la place, le system prompt impose JSON strict et on parse à la main avec strip des fences markdown éventuelles.

### Fichier 7/9, `app/api/cocon/article/route.ts`

```typescript
import { NextRequest, NextResponse } from "next/server";
import Anthropic from "@anthropic-ai/sdk";
import {
  getFilleBySlug,
  getPetiteFille,
  getLinkingContext,
  getStaticAllowedUrls,
  cocon,
  P1_SYSTEM,
  P2_SYSTEM,
  BRIEF_SCHEMA,
  ARTICLE_SCHEMA,
  KNOWLEDGE_BASE,
  sanitizeArticle,
} from "@/lib/cocon";
import { save as saveCocon } from "@/lib/cocon-storage";
import { validateLinks } from "@/lib/link-validator";
import { getPostSlugs } from "@/lib/ghost";

export const runtime = "nodejs";
export const maxDuration = 120;

type Brief = {
  intent_summary: string;
  promise: string;
  h1: string;
  structure: Array<{ h2: string; what: string }>;
  key_takeaways: string[];
  internal_links_suggested: Array<{ url: string; anchor_suggestion: string }>;
  word_count_target: number;
  tone: string;
};

function parseJsonResponse(text: string): unknown {
  const cleaned = text
    .replace(/^```json\s*/i, "")
    .replace(/^```\s*/i, "")
    .replace(/\s*```$/i, "")
    .trim();
  return JSON.parse(cleaned);
}

export async function POST(req: NextRequest) {
  if (!process.env.ANTHROPIC_API_KEY) {
    return NextResponse.json(
      { error: "ANTHROPIC_API_KEY non configurée" },
      { status: 500 },
    );
  }

  let body: { filleSlug?: string; pfSlug?: string };
  try {
    body = await req.json();
  } catch {
    return NextResponse.json({ error: "Body JSON invalide" }, { status: 400 });
  }

  const { filleSlug, pfSlug } = body;
  if (!filleSlug || !pfSlug) {
    return NextResponse.json(
      { error: "filleSlug et pfSlug requis" },
      { status: 400 },
    );
  }

  const fille = getFilleBySlug(filleSlug);
  const pf = getPetiteFille(filleSlug, pfSlug);
  const ctx = getLinkingContext(filleSlug, pfSlug);
  if (!fille || !pf || !ctx) {
    return NextResponse.json(
      { error: "Article introuvable dans le cocon" },
      { status: 404 },
    );
  }

  const client = new Anthropic();

  // ─── PHASE 1 : générer le brief ─────────────────────────────
  const briefUserPrompt = `Génère le brief de l'article suivant.

KEYWORD CIBLE : ${pf.keyword}
TITRE PROPOSÉ : ${pf.title}
${pf.keywords_secondaires?.length ? `MOTS-CLÉS SECONDAIRES : ${pf.keywords_secondaires.join(", ")}` : ""}

CONTEXTE COCON :
- Mère : ${ctx.mere.title} (${ctx.mere.url}), keyword : ${ctx.mere.keyword}
- Pilier parent : ${ctx.pilier_parent.title} (${ctx.pilier_parent.url})
- Piliers sœurs proches :
${ctx.pilier_soeurs.map((s) => `  • ${s.title} (${s.url})`).join("\n")}
- CTA principal : ${ctx.cta.url}, label : "${ctx.cta.label}"
${ctx.outils_techniques.length ? `- Outils à mailler : ${ctx.outils_techniques.join(", ")}` : ""}

PERSONA : ${fille.persona}
INTENT : ${pf.intent}
WORD COUNT CIBLE : 1500 mots`;

  let brief: Brief;
  let briefUsage: { input_tokens: number; output_tokens: number };
  try {
    const briefResponse = await client.messages.create({
      model: "claude-sonnet-4-6",
      max_tokens: 8000,
      system: P1_SYSTEM,
      messages: [{ role: "user", content: briefUserPrompt }],
    });
    const block = briefResponse.content.find((b) => b.type === "text");
    if (!block || block.type !== "text") {
      return NextResponse.json(
        { error: "P1: pas de texte en sortie" },
        { status: 500 },
      );
    }
    brief = parseJsonResponse(block.text) as Brief;
    briefUsage = {
      input_tokens: briefResponse.usage.input_tokens,
      output_tokens: briefResponse.usage.output_tokens,
    };
    console.log(
      `[cocon/article] P1 brief ok in=${briefUsage.input_tokens} out=${briefUsage.output_tokens}`,
    );
  } catch (err) {
    if (err instanceof Anthropic.APIError) {
      return NextResponse.json(
        { error: `P1 Anthropic ${err.status}: ${err.message}` },
        { status: 500 },
      );
    }
    return NextResponse.json(
      {
        error: `P1 failed: ${err instanceof Error ? err.message : String(err)}`,
      },
      { status: 500 },
    );
  }

  // ─── PHASE 2 : générer l'article ────────────────────────────
  let publishedArticleSlugs: string[] = [];
  try {
    publishedArticleSlugs = await getPostSlugs();
  } catch (err) {
    console.warn(`[cocon/article] getPostSlugs failed:`, err);
  }
  const publishedArticleUrls = publishedArticleSlugs.map((s) => `/blog/${s}`);

  const articleUserPrompt = `Voici le brief à exécuter, produis l'article HTML + tous les méta SEO.

KNOWLEDGE BASE (validée le ${KNOWLEDGE_BASE.validated_at}), utilise EXCLUSIVEMENT ces faits pour tout chiffre, prix, statistique, version, ou caractéristique produit :
${JSON.stringify(KNOWLEDGE_BASE, null, 2)}

BRIEF :
${JSON.stringify(brief, null, 2)}

LIENS_AUTORISES (RÈGLE STRICTE : tu ne peux insérer QUE ces URLs internes, aucune autre. Les URLs hors de cette liste sont automatiquement supprimées par le système et remplacées par la page pilier, donc ne perds pas de tokens à en inventer.) :

- MÈRE (obligatoire, dans les 200 premiers mots) : ${ctx.mere.url}
- PAGE PILIER PARENT : ${ctx.pilier_parent.url}
- AUTRES PILIERS du cocon (utilise-en 2 ou 3 dans le corps) :
${ctx.pilier_soeurs.map((s) => `  • ${s.url}, sujet : ${s.keyword}`).join("\n")}
- CTA FINAL OBLIGATOIRE (dans le dernier paragraphe) : ${ctx.cta.url}, label naturel : "${ctx.cta.label}"
${ctx.outils_techniques.length ? `- PAGES OUTILS techniques (utilise si pertinent) :\n${ctx.outils_techniques.map((u) => `  • ${u}`).join("\n")}` : ""}
${
  publishedArticleUrls.length
    ? `- AUTRES ARTICLES BLOG déjà publiés (lie ceux qui sont sémantiquement pertinents) :\n${publishedArticleUrls.map((u) => `  • ${u}`).join("\n")}`
    : ""
}

CONTRAINTES :
- ${brief.word_count_target ?? 1500} mots cible
- Ton : ${brief.tone}
- Structure imposée par les H2 du brief, respecte-les
- HTML pur, pas de markdown
- Pas de h1
- Toute URL hors de LIENS_AUTORISES sera silencieusement remplacée, n'invente AUCUNE URL`;

  try {
    const articleResponse = await client.messages.create({
      model: "claude-opus-4-7",
      max_tokens: 12000,
      system: P2_SYSTEM,
      messages: [{ role: "user", content: articleUserPrompt }],
    });
    const block = articleResponse.content.find((b) => b.type === "text");
    if (!block || block.type !== "text") {
      return NextResponse.json(
        { error: "P2: pas de texte en sortie", brief },
        { status: 500 },
      );
    }
    const articleRaw = parseJsonResponse(block.text) as Record<string, unknown>;
    const article = sanitizeArticle(articleRaw);

    // Filet de sécurité : valide tous les <a href> internes
    const allowedUrls = new Set<string>([
      ...getStaticAllowedUrls(),
      ...publishedArticleUrls,
    ]);
    const validation = validateLinks(
      typeof article.html === "string" ? article.html : "",
      allowedUrls,
      ctx.pilier_parent.url,
    );
    if (validation.rewrittenLinks.length > 0) {
      console.warn(
        `[cocon/article] ${validation.rewrittenLinks.length} lien(s) invalide(s) réécrit(s):`,
        validation.rewrittenLinks,
      );
      article.html = validation.cleaned;
    }

    const articleUsage = {
      input_tokens: articleResponse.usage.input_tokens,
      output_tokens: articleResponse.usage.output_tokens,
    };
    console.log(
      `[cocon/article] P2 article ok in=${articleUsage.input_tokens} out=${articleUsage.output_tokens} ` +
        `links=${validation.totalLinks} (internal=${validation.internalLinks}, rewritten=${validation.rewrittenLinks.length})`,
    );

    const result = {
      brief,
      article,
      usage: {
        brief: briefUsage,
        article: articleUsage,
        total_input: briefUsage.input_tokens + articleUsage.input_tokens,
        total_output: briefUsage.output_tokens + articleUsage.output_tokens,
      },
      stop_reason: articleResponse.stop_reason,
      link_validation: {
        total: validation.totalLinks,
        internal: validation.internalLinks,
        external: validation.externalLinks,
        rewritten: validation.rewrittenLinks,
      },
    };
    await saveCocon("briefs", filleSlug, pfSlug, {
      brief,
      usage: briefUsage,
      stop_reason: "end_turn",
    });
    await saveCocon("articles", filleSlug, pfSlug, result);
    return NextResponse.json(result);
  } catch (err) {
    if (err instanceof Anthropic.APIError) {
      return NextResponse.json(
        { error: `P2 Anthropic ${err.status}: ${err.message}`, brief },
        { status: 500 },
      );
    }
    return NextResponse.json(
      {
        error: `P2 failed: ${err instanceof Error ? err.message : String(err)}`,
        brief,
      },
      { status: 500 },
    );
  }
}
```

### Fichier 8/9, `app/api/cocon/publish/route.ts`

```typescript
import { NextRequest, NextResponse } from "next/server";
import { upsertPost, uploadImage } from "@/lib/ghost-admin";
import { generateImage, isFalConfigured } from "@/lib/fal";
import { getFilleBySlug, getPetiteFille } from "@/lib/cocon";
import { save as saveCocon } from "@/lib/cocon-storage";

export const runtime = "nodejs";
export const maxDuration = 90;

type Article = {
  html: string;
  meta_title: string;
  meta_description: string;
  custom_excerpt: string;
  og_title: string;
  og_description: string;
  tags: string[];
  primary_tag: string;
  feature_image_prompt: string;
};

type Body = {
  filleSlug?: string;
  pfSlug?: string;
  article?: Article;
};

export async function POST(req: NextRequest) {
  let body: Body;
  try {
    body = (await req.json()) as Body;
  } catch {
    return NextResponse.json({ error: "Body JSON invalide" }, { status: 400 });
  }

  const { filleSlug, pfSlug, article } = body;
  if (!filleSlug || !pfSlug || !article) {
    return NextResponse.json(
      { error: "filleSlug, pfSlug et article requis" },
      { status: 400 },
    );
  }

  const fille = getFilleBySlug(filleSlug);
  const pf = getPetiteFille(filleSlug, pfSlug);
  if (!fille || !pf) {
    return NextResponse.json(
      { error: "Article introuvable dans le cocon" },
      { status: 404 },
    );
  }

  // Tags : on FORCE primary_tag = pilier_slug du cocon (pour la page
  // /blog/pilier/<slug>). Les tags secondaires du modèle sont conservés.
  const pilierTag = fille.pilier_slug;
  const secondaryTags = [article.primary_tag, ...article.tags].filter(
    (t) => Boolean(t) && t !== pilierTag,
  );
  const tags = [pilierTag, ...Array.from(new Set(secondaryTags))];

  // ─── Image hero (optionnelle, dépend de FAL_KEY) ────────────
  let featureImageUrl: string | undefined;
  let imageStatus: "skipped" | "generated" | "failed" = "skipped";
  let imageError: string | undefined;

  if (isFalConfigured() && article.feature_image_prompt) {
    try {
      const img = await generateImage(article.feature_image_prompt, "16:9");
      console.log(
        `[cocon/publish] ${pf.slug} fal.ai image générée (${img.buffer.length} bytes)`,
      );
      const uploaded = await uploadImage(
        img.buffer,
        `${pf.slug}.jpg`,
        img.contentType,
      );
      featureImageUrl = uploaded.url;
      imageStatus = "generated";
      console.log(`[cocon/publish] ${pf.slug} image uploadée: ${uploaded.url}`);
    } catch (err) {
      imageStatus = "failed";
      imageError = err instanceof Error ? err.message : String(err);
      console.error(`[cocon/publish] ${pf.slug} image échec: ${imageError}`);
      // On continue sans image, la publication ne doit pas échouer pour ça
    }
  }

  try {
    const result = await upsertPost({
      slug: pf.slug,
      title: pf.title,
      html: article.html,
      meta_title: article.meta_title,
      meta_description: article.meta_description,
      custom_excerpt: article.custom_excerpt,
      og_title: article.og_title,
      og_description: article.og_description,
      tags,
      feature_image: featureImageUrl,
      feature_image_alt: featureImageUrl ? pf.title : undefined,
    });

    console.log(
      `[cocon/publish] ${pf.slug} ${result.mode} en draft (id=${result.post.id})`,
    );

    const response = {
      ok: true,
      mode: result.mode,
      post: {
        id: result.post.id,
        slug: result.post.slug,
        status: result.post.status,
      },
      adminUrl: result.adminUrl,
      featureImage: featureImageUrl
        ? { url: featureImageUrl, status: imageStatus }
        : { url: null, status: imageStatus, error: imageError },
    };
    await saveCocon("published", filleSlug, pfSlug, response);
    return NextResponse.json(response);
  } catch (err) {
    const message = err instanceof Error ? err.message : String(err);
    console.error(`[cocon/publish] ${pf.slug} échec: ${message}`);
    return NextResponse.json({ ok: false, error: message }, { status: 500 });
  }
}
```

### Fichier 9/9, `app/cocon/admin/page.tsx`, `AdminClient.tsx`, `styles.css`

#### `app/cocon/admin/page.tsx`

```typescript
import { cookies } from "next/headers";
import { redirect } from "next/navigation";
import { cocon, getCoconStats } from "@/lib/cocon";
import { loadAll } from "@/lib/cocon-storage";
import AdminClient from "./AdminClient";
import "./styles.css";

const COOKIE_NAME = "cocon-admin";
const COOKIE_TTL_SECONDS = 60 * 60 * 8; // 8 heures

async function loginAction(formData: FormData) {
  "use server";
  const password = formData.get("password");
  const expected = process.env.COCON_ADMIN_PASSWORD;
  if (!expected) {
    throw new Error(
      "COCON_ADMIN_PASSWORD non configuré dans .env.local",
    );
  }
  if (password === expected) {
    const cookieStore = await cookies();
    cookieStore.set(COOKIE_NAME, "1", {
      httpOnly: true,
      sameSite: "lax",
      maxAge: COOKIE_TTL_SECONDS,
      path: "/cocon",
    });
    redirect("/cocon/admin");
  }
  redirect("/cocon/admin?error=wrong");
}

async function logoutAction() {
  "use server";
  const cookieStore = await cookies();
  cookieStore.delete(COOKIE_NAME);
  redirect("/cocon/admin");
}

export const metadata = {
  title: "Cocon Admin",
  robots: { index: false, follow: false },
};

export default async function CoconAdminPage({
  searchParams,
}: {
  searchParams: Promise<{ error?: string }>;
}) {
  const cookieStore = await cookies();
  const isAuthed = cookieStore.get(COOKIE_NAME)?.value === "1";
  const { error } = await searchParams;

  if (!isAuthed) {
    return (
      <div className="cocon-login">
        <form action={loginAction} className="cocon-login-form">
          <h1>Cocon Admin</h1>
          <p className="cocon-login-lede">
            Générateur d'articles, accès protégé.
          </p>
          {error === "wrong" && (
            <div className="cocon-login-error">Mot de passe incorrect.</div>
          )}
          <input
            name="password"
            type="password"
            placeholder="Mot de passe"
            autoFocus
            required
            autoComplete="current-password"
          />
          <button type="submit">Connexion</button>
        </form>
      </div>
    );
  }

  const stats = getCoconStats();
  const [initialBriefs, initialArticles, initialPublished] = await Promise.all([
    loadAll("briefs"),
    loadAll("articles"),
    loadAll("published"),
  ]);
  return (
    <AdminClient
      cocon={cocon}
      stats={stats}
      logoutAction={logoutAction}
      initialBriefs={initialBriefs}
      initialArticles={initialArticles}
      initialPublished={initialPublished}
    />
  );
}
```

#### `app/cocon/admin/AdminClient.tsx`

```typescript
"use client";

import { useState } from "react";
import type { Cocon } from "@/lib/cocon";

type Stats = {
  filles: number;
  petites_filles_total: number;
  published: number;
  draft: number;
  planned: number;
};

type BriefResult = {
  brief?: unknown;
  usage?: { input_tokens: number; output_tokens: number };
  stop_reason?: string;
  error?: string;
  raw?: string;
};

type Article = {
  html: string;
  meta_title: string;
  meta_description: string;
  custom_excerpt: string;
  og_title: string;
  og_description: string;
  tags: string[];
  primary_tag: string;
  feature_image_prompt: string;
};

type ArticleResult = {
  brief?: unknown;
  article?: Article;
  usage?: {
    brief: { input_tokens: number; output_tokens: number };
    article: { input_tokens: number; output_tokens: number };
    total_input: number;
    total_output: number;
  };
  stop_reason?: string;
  error?: string;
};

type View = "brief" | "article" | null;

type PublishResult = {
  ok?: boolean;
  mode?: "created" | "updated";
  post?: { id: string; slug: string; status: string };
  adminUrl?: string;
  featureImage?: {
    url: string | null;
    status: "skipped" | "generated" | "failed";
    error?: string;
  };
  error?: string;
};

export default function AdminClient({
  cocon,
  stats,
  logoutAction,
  initialBriefs = {},
  initialArticles = {},
  initialPublished = {},
}: {
  cocon: Cocon;
  stats: Stats;
  logoutAction: () => Promise<void>;
  initialBriefs?: Record<string, unknown>;
  initialArticles?: Record<string, unknown>;
  initialPublished?: Record<string, unknown>;
}) {
  const [briefs, setBriefs] = useState<Record<string, BriefResult>>(
    initialBriefs as Record<string, BriefResult>,
  );
  const [articles, setArticles] = useState<Record<string, ArticleResult>>(
    initialArticles as Record<string, ArticleResult>,
  );
  const [loadingBrief, setLoadingBrief] = useState<Record<string, boolean>>({});
  const [loadingArticle, setLoadingArticle] = useState<
    Record<string, boolean>
  >({});
  const [openView, setOpenView] = useState<Record<string, View>>({});
  const [copied, setCopied] = useState<string | null>(null);
  const [publishing, setPublishing] = useState<Record<string, boolean>>({});
  const [published, setPublished] = useState<Record<string, PublishResult>>(
    initialPublished as Record<string, PublishResult>,
  );

  async function generateBrief(filleSlug: string, pfSlug: string) {
    const key = `${filleSlug}/${pfSlug}`;
    setLoadingBrief((l) => ({ ...l, [key]: true }));
    setOpenView((v) => ({ ...v, [key]: "brief" }));
    try {
      const res = await fetch("/api/cocon/brief", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ filleSlug, pfSlug }),
      });
      const data: BriefResult = await res.json();
      setBriefs((b) => ({ ...b, [key]: data }));
    } catch (err) {
      setBriefs((b) => ({
        ...b,
        [key]: { error: err instanceof Error ? err.message : String(err) },
      }));
    } finally {
      setLoadingBrief((l) => ({ ...l, [key]: false }));
    }
  }

  async function generateArticle(filleSlug: string, pfSlug: string) {
    const key = `${filleSlug}/${pfSlug}`;
    setLoadingArticle((l) => ({ ...l, [key]: true }));
    setOpenView((v) => ({ ...v, [key]: "article" }));
    try {
      const res = await fetch("/api/cocon/article", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ filleSlug, pfSlug }),
      });
      const data: ArticleResult = await res.json();
      setArticles((a) => ({ ...a, [key]: data }));
    } catch (err) {
      setArticles((a) => ({
        ...a,
        [key]: { error: err instanceof Error ? err.message : String(err) },
      }));
    } finally {
      setLoadingArticle((l) => ({ ...l, [key]: false }));
    }
  }

  async function copy(text: string, label: string) {
    await navigator.clipboard.writeText(text);
    setCopied(label);
    setTimeout(() => setCopied(null), 1500);
  }

  async function publishToGhost(filleSlug: string, pfSlug: string) {
    const key = `${filleSlug}/${pfSlug}`;
    const article = articles[key]?.article;
    if (!article) return;
    setPublishing((p) => ({ ...p, [key]: true }));
    setPublished((p) => {
      const next = { ...p };
      delete next[key];
      return next;
    });
    try {
      const res = await fetch("/api/cocon/publish", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ filleSlug, pfSlug, article }),
      });
      const data: PublishResult = await res.json();
      setPublished((p) => ({ ...p, [key]: data }));
    } catch (err) {
      setPublished((p) => ({
        ...p,
        [key]: { error: err instanceof Error ? err.message : String(err) },
      }));
    } finally {
      setPublishing((p) => ({ ...p, [key]: false }));
    }
  }

  return (
    <div className="cocon-admin">
      <header className="cocon-admin-header">
        <div>
          <h1>Cocon Admin</h1>
          <p className="cocon-admin-lede">
            {stats.filles} piliers · {stats.petites_filles_total} articles
            total · <strong>{stats.published}</strong> publié(s) ·{" "}
            <strong>{stats.draft}</strong> draft(s) · {stats.planned}{" "}
            planifié(s)
          </p>
        </div>
        <form action={logoutAction}>
          <button className="cocon-admin-logout" type="submit">
            Déconnexion
          </button>
        </form>
      </header>

      {cocon.filles.map((fille) => (
        <section key={fille.slug} className="cocon-admin-fille">
          <header className="cocon-admin-fille-head">
            <div>
              <h2>{fille.title}</h2>
              <p className="cocon-admin-fille-meta">
                Mot-clé : <code>{fille.keyword}</code> · Persona :{" "}
                <code>{fille.persona}</code> · CTA :{" "}
                <code>{fille.cta_principal}</code>
              </p>
            </div>
            <span className={`cocon-status cocon-status--${fille.status}`}>
              {fille.status}
            </span>
          </header>

          <ul className="cocon-admin-pf-list">
            {fille.petites_filles.map((pf) => {
              const key = `${fille.slug}/${pf.slug}`;
              const briefResult = briefs[key];
              const articleResult = articles[key];
              const isLoadingBrief = loadingBrief[key];
              const isLoadingArticle = loadingArticle[key];
              const view = openView[key];
              return (
                <li key={pf.slug} className="cocon-admin-pf">
                  <div className="cocon-admin-pf-row">
                    <div className="cocon-admin-pf-info">
                      <span
                        className={`cocon-status cocon-status--${pf.status}`}
                      >
                        {pf.status}
                      </span>
                      <div>
                        <div className="cocon-admin-pf-title">{pf.title}</div>
                        <div className="cocon-admin-pf-keyword">
                          <code>{pf.keyword}</code>
                          {pf.ghost_slug && (
                            <span className="cocon-admin-pf-ghost">
                              · Ghost : <code>{pf.ghost_slug}</code>
                            </span>
                          )}
                        </div>
                      </div>
                    </div>
                    <div className="cocon-admin-pf-actions">
                      {pf.status !== "published" && (
                        <>
                          <button
                            type="button"
                            className="cocon-btn cocon-btn-primary"
                            onClick={() =>
                              generateArticle(fille.slug, pf.slug)
                            }
                            disabled={isLoadingArticle}
                            title="P1 brief + P2 article complet · ~30-60 s · ~0,15 €"
                          >
                            {isLoadingArticle
                              ? "Rédaction..."
                              : articleResult?.article
                                ? "Régénérer l'article"
                                : "Générer l'article"}
                          </button>
                          <button
                            type="button"
                            className="cocon-btn cocon-btn-light"
                            onClick={() => generateBrief(fille.slug, pf.slug)}
                            disabled={isLoadingBrief}
                            title="P1 brief seul · ~10 s · ~0,02 €"
                          >
                            {isLoadingBrief ? "..." : "Brief seul"}
                          </button>
                        </>
                      )}
                      {(briefResult || articleResult) && (
                        <div className="cocon-tabs">
                          {articleResult && (
                            <button
                              type="button"
                              className={`cocon-tab${view === "article" ? " is-active" : ""}`}
                              onClick={() =>
                                setOpenView((v) => ({
                                  ...v,
                                  [key]:
                                    v[key] === "article" ? null : "article",
                                }))
                              }
                            >
                              Article
                            </button>
                          )}
                          {briefResult && (
                            <button
                              type="button"
                              className={`cocon-tab${view === "brief" ? " is-active" : ""}`}
                              onClick={() =>
                                setOpenView((v) => ({
                                  ...v,
                                  [key]: v[key] === "brief" ? null : "brief",
                                }))
                              }
                            >
                              Brief
                            </button>
                          )}
                        </div>
                      )}
                    </div>
                  </div>

                  {view === "brief" && briefResult && (
                    <BriefView result={briefResult} loading={isLoadingBrief} />
                  )}
                  {view === "article" && articleResult && (
                    <ArticleView
                      result={articleResult}
                      loading={isLoadingArticle}
                      onCopy={copy}
                      copiedLabel={copied}
                      uniqueKey={key}
                      onPublish={() => publishToGhost(fille.slug, pf.slug)}
                      isPublishing={publishing[key] ?? false}
                      publishResult={published[key]}
                    />
                  )}
                </li>
              );
            })}
          </ul>
        </section>
      ))}
    </div>
  );
}

function BriefView({
  result,
  loading,
}: {
  result: BriefResult;
  loading: boolean;
}) {
  return (
    <div className="cocon-admin-output">
      {loading && (
        <div className="cocon-admin-loading">
          Génération du brief en cours (~10 s)...
        </div>
      )}
      {result.error && (
        <div className="cocon-admin-error">
          <strong>Erreur :</strong> {result.error}
          {result.raw && <pre className="cocon-admin-raw">{result.raw}</pre>}
        </div>
      )}
      {result.brief != null && (
        <>
          <div className="cocon-admin-usage">
            Tokens, entrée : {result.usage?.input_tokens ?? "?"} · sortie :{" "}
            {result.usage?.output_tokens ?? "?"} · stop:{" "}
            <strong
              style={{
                color:
                  result.stop_reason === "end_turn" ? "#065f46" : "#b00",
              }}
            >
              {result.stop_reason ?? "?"}
            </strong>
          </div>
          <pre className="cocon-admin-json">
            {JSON.stringify(result.brief, null, 2)}
          </pre>
        </>
      )}
    </div>
  );
}

function ArticleView({
  result,
  loading,
  onCopy,
  copiedLabel,
  uniqueKey,
  onPublish,
  isPublishing,
  publishResult,
}: {
  result: ArticleResult;
  loading: boolean;
  onCopy: (text: string, label: string) => void;
  copiedLabel: string | null;
  uniqueKey: string;
  onPublish: () => void;
  isPublishing: boolean;
  publishResult?: PublishResult;
}) {
  return (
    <div className="cocon-admin-output">
      {loading && (
        <div className="cocon-admin-loading">
          Pipeline en cours : P1 brief → P2 rédaction (~30-60 s)...
        </div>
      )}
      {result.error && (
        <div className="cocon-admin-error">
          <strong>Erreur :</strong> {result.error}
        </div>
      )}
      {result.article && (
        <>
          <div className="cocon-admin-usage">
            Tokens, total entrée :{" "}
            <strong>{result.usage?.total_input ?? "?"}</strong> · total sortie :{" "}
            <strong>{result.usage?.total_output ?? "?"}</strong> · stop:{" "}
            <strong
              style={{
                color:
                  result.stop_reason === "end_turn" ? "#065f46" : "#b00",
              }}
            >
              {result.stop_reason ?? "?"}
            </strong>
            {result.usage && (
              <span className="cocon-admin-usage-detail">
                {" "}
                (P1 : {result.usage.brief.input_tokens}→
                {result.usage.brief.output_tokens} · P2 :{" "}
                {result.usage.article.input_tokens}→
                {result.usage.article.output_tokens})
              </span>
            )}
          </div>

          <div className="cocon-meta-grid">
            <MetaField
              label="Meta title"
              value={result.article.meta_title}
              max={60}
              copyKey={`${uniqueKey}-meta-title`}
              onCopy={onCopy}
              copiedLabel={copiedLabel}
            />
            <MetaField
              label="Meta description"
              value={result.article.meta_description}
              max={155}
              copyKey={`${uniqueKey}-meta-desc`}
              onCopy={onCopy}
              copiedLabel={copiedLabel}
            />
            <MetaField
              label="Custom excerpt (Ghost)"
              value={result.article.custom_excerpt}
              copyKey={`${uniqueKey}-excerpt`}
              onCopy={onCopy}
              copiedLabel={copiedLabel}
            />
            <MetaField
              label="OG title"
              value={result.article.og_title}
              copyKey={`${uniqueKey}-og-title`}
              onCopy={onCopy}
              copiedLabel={copiedLabel}
            />
            <MetaField
              label="OG description"
              value={result.article.og_description}
              copyKey={`${uniqueKey}-og-desc`}
              onCopy={onCopy}
              copiedLabel={copiedLabel}
            />
            <MetaField
              label="Tags Ghost"
              value={result.article.tags.join(", ")}
              copyKey={`${uniqueKey}-tags`}
              onCopy={onCopy}
              copiedLabel={copiedLabel}
            />
            <MetaField
              label="Primary tag"
              value={result.article.primary_tag}
              copyKey={`${uniqueKey}-ptag`}
              onCopy={onCopy}
              copiedLabel={copiedLabel}
            />
            <MetaField
              label="Prompt image hero (fal.ai)"
              value={result.article.feature_image_prompt}
              copyKey={`${uniqueKey}-img`}
              onCopy={onCopy}
              copiedLabel={copiedLabel}
            />
          </div>

          <div className="cocon-article-actions">
            <button
              type="button"
              className="cocon-btn cocon-btn-publish"
              onClick={onPublish}
              disabled={isPublishing}
              title="Génère l'image hero (fal.ai) puis crée/met à jour le draft Ghost"
            >
              {isPublishing
                ? "Image + Ghost (~30-60 s)..."
                : publishResult?.ok
                  ? `✓ ${publishResult.mode === "created" ? "Créé" : "Mis à jour"} dans Ghost, republier`
                  : "Publier en draft sur Ghost"}
            </button>
            <button
              type="button"
              className="cocon-btn cocon-btn-primary"
              onClick={() =>
                onCopy(result.article!.html, `${uniqueKey}-html`)
              }
            >
              {copiedLabel === `${uniqueKey}-html`
                ? "✓ HTML copié"
                : "Copier le HTML"}
            </button>
            <button
              type="button"
              className="cocon-btn cocon-btn-light"
              onClick={() =>
                onCopy(
                  JSON.stringify(result.article, null, 2),
                  `${uniqueKey}-json`,
                )
              }
            >
              {copiedLabel === `${uniqueKey}-json`
                ? "✓ JSON copié"
                : "Copier le JSON"}
            </button>
          </div>

          {publishResult?.ok && publishResult.adminUrl && (
            <div className="cocon-publish-success">
              <strong>
                ✓{" "}
                {publishResult.mode === "created"
                  ? "Article créé"
                  : "Article mis à jour"}{" "}
                dans Ghost (statut : draft).
              </strong>
              <br />
              <a
                href={publishResult.adminUrl}
                target="_blank"
                rel="noopener noreferrer"
              >
                → Ouvrir dans Ghost admin pour relire et publier
              </a>
              {publishResult.featureImage?.status === "generated" &&
                publishResult.featureImage.url && (
                  <div className="cocon-publish-image">
                    <span className="cocon-publish-image-label">
                      Image hero générée et attachée :
                    </span>
                    {/* eslint-disable-next-line @next/next/no-img-element */}
                    <img
                      src={publishResult.featureImage.url}
                      alt="Feature image"
                      className="cocon-publish-image-preview"
                    />
                  </div>
                )}
              {publishResult.featureImage?.status === "failed" && (
                <div className="cocon-publish-image-warn">
                  Image non générée :{" "}
                  {publishResult.featureImage.error ?? "fal.ai a échoué"}
                  <span className="cocon-publish-image-hint">
                    {" "}
                   , l'article est publié sans image, tu peux en ajouter une à la main dans Ghost.
                  </span>
                </div>
              )}
              {publishResult.featureImage?.status === "skipped" && (
                <div className="cocon-publish-image-warn">
                  Image non générée (FAL_KEY non configurée). Ajoute{" "}
                  <code>FAL_KEY=...</code> dans <code>.env.local</code> pour
                  activer la génération automatique.
                </div>
              )}
            </div>
          )}
          {publishResult?.error && (
            <div className="cocon-publish-error">
              <strong>Échec publication Ghost :</strong> {publishResult.error}
            </div>
          )}

          <div className="cocon-article-preview-label">
            Aperçu (rendu HTML)
          </div>
          <div
            className="cocon-article-preview"
            dangerouslySetInnerHTML={{ __html: result.article.html }}
          />
        </>
      )}
    </div>
  );
}

function MetaField({
  label,
  value,
  max,
  copyKey,
  onCopy,
  copiedLabel,
}: {
  label: string;
  value: string;
  max?: number;
  copyKey: string;
  onCopy: (text: string, label: string) => void;
  copiedLabel: string | null;
}) {
  const overLimit = max != null && value.length > max;
  return (
    <div className="cocon-meta-field">
      <div className="cocon-meta-label">
        <span>{label}</span>
        {max != null && (
          <span className={`cocon-meta-count${overLimit ? " is-over" : ""}`}>
            {value.length}/{max}
          </span>
        )}
        <button
          type="button"
          className="cocon-meta-copy"
          onClick={() => onCopy(value, copyKey)}
          aria-label={`Copier ${label}`}
        >
          {copiedLabel === copyKey ? "✓" : "Copier"}
        </button>
      </div>
      <div className="cocon-meta-value">{value}</div>
    </div>
  );
}
```

#### `app/cocon/admin/styles.css`

Adapte la couleur d'accent à `<ACCENT_COLOR>` (par défaut `#5C3BFF`).

```css
/* Cocon admin, UI interne minimale, non destinée au public. */

.cocon-login,
.cocon-admin {
  font-family:
    ui-sans-serif, system-ui, -apple-system, "Segoe UI", Roboto, sans-serif;
  color: #111;
  background: #f7f7f8;
  min-height: 100vh;
  padding: 24px;
}

/* ============ Login ============ */

.cocon-login {
  display: flex;
  align-items: center;
  justify-content: center;
}

.cocon-login-form {
  background: #fff;
  padding: 40px;
  border-radius: 6px;
  border: 1px solid #e5e5e8;
  box-shadow: 0 4px 24px rgba(0, 0, 0, 0.04);
  width: 100%;
  max-width: 380px;
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.cocon-login-form h1 { margin: 0; font-size: 24px; font-weight: 600; }
.cocon-login-lede { margin: 0; font-size: 14px; color: #6b6b75; }

.cocon-login-error {
  background: #fee;
  color: #c00;
  padding: 10px 12px;
  border-radius: 4px;
  font-size: 13px;
}

.cocon-login-form input {
  padding: 10px 12px;
  border: 1px solid #d4d4d8;
  border-radius: 4px;
  font-size: 14px;
}

.cocon-login-form button {
  background: #1a1a1f;
  color: #fff;
  border: none;
  padding: 10px 16px;
  border-radius: 4px;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
}

.cocon-login-form button:hover { background: #000; }

/* ============ Admin layout ============ */

.cocon-admin { max-width: 1100px; margin: 0 auto; }

.cocon-admin-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  gap: 24px;
  margin-bottom: 32px;
  padding-bottom: 16px;
  border-bottom: 1px solid #e5e5e8;
}

.cocon-admin-header h1 { margin: 0; font-size: 22px; font-weight: 600; }
.cocon-admin-lede { margin: 4px 0 0; font-size: 13px; color: #6b6b75; }

.cocon-admin-logout {
  background: transparent;
  border: 1px solid #d4d4d8;
  color: #6b6b75;
  padding: 6px 12px;
  border-radius: 4px;
  font-size: 12px;
  cursor: pointer;
}

.cocon-admin-logout:hover { border-color: #1a1a1f; color: #1a1a1f; }

/* ============ Fille ============ */

.cocon-admin-fille {
  background: #fff;
  border: 1px solid #e5e5e8;
  border-radius: 6px;
  margin-bottom: 16px;
  overflow: hidden;
}

.cocon-admin-fille-head {
  padding: 16px 20px;
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  gap: 16px;
  border-bottom: 1px solid #ececef;
  background: #fafafb;
}

.cocon-admin-fille-head h2 {
  margin: 0;
  font-size: 17px;
  font-weight: 600;
  line-height: 1.3;
}

.cocon-admin-fille-meta { margin: 4px 0 0; font-size: 12px; color: #6b6b75; }
.cocon-admin-fille-meta code {
  background: #f0f0f3;
  padding: 1px 6px;
  border-radius: 3px;
  font-size: 11px;
  color: #1a1a1f;
}

/* ============ Status badges ============ */

.cocon-status {
  display: inline-block;
  padding: 3px 8px;
  border-radius: 12px;
  font-size: 10px;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  font-weight: 600;
  flex-shrink: 0;
}
.cocon-status--published { background: #d1fae5; color: #065f46; }
.cocon-status--draft { background: #fef3c7; color: #78350f; }
.cocon-status--planned { background: #e5e5e8; color: #4b4b53; }

/* ============ Petites-filles ============ */

.cocon-admin-pf-list { list-style: none; margin: 0; padding: 0; }
.cocon-admin-pf { border-bottom: 1px solid #f0f0f3; }
.cocon-admin-pf:last-child { border-bottom: none; }

.cocon-admin-pf-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 16px;
  padding: 14px 20px;
}

.cocon-admin-pf-info {
  display: flex;
  align-items: center;
  gap: 12px;
  flex: 1;
  min-width: 0;
}

.cocon-admin-pf-title { font-size: 14px; font-weight: 500; line-height: 1.3; }
.cocon-admin-pf-keyword { margin-top: 2px; font-size: 11px; color: #6b6b75; }
.cocon-admin-pf-keyword code {
  background: #f0f0f3;
  padding: 1px 6px;
  border-radius: 3px;
  font-size: 11px;
  color: #1a1a1f;
}
.cocon-admin-pf-ghost { margin-left: 6px; }

.cocon-admin-pf-actions { display: flex; gap: 8px; flex-shrink: 0; }

/* ============ Buttons ============ */

.cocon-btn {
  background: <ACCENT_COLOR>;
  color: #fff;
  border: none;
  padding: 6px 12px;
  border-radius: 4px;
  font-size: 12px;
  font-weight: 500;
  cursor: pointer;
}

.cocon-btn:hover:not(:disabled) { filter: brightness(0.85); }
.cocon-btn:disabled { opacity: 0.6; cursor: not-allowed; }

.cocon-btn-primary { background: <ACCENT_COLOR>; color: #fff; }
.cocon-btn-light {
  background: #fff;
  color: #1a1a1f;
  border: 1px solid #d4d4d8;
}
.cocon-btn-light:hover:not(:disabled) {
  border-color: <ACCENT_COLOR>;
  color: <ACCENT_COLOR>;
}

/* ============ Tabs ============ */

.cocon-tabs {
  display: flex;
  gap: 4px;
  border: 1px solid #d4d4d8;
  border-radius: 4px;
  padding: 2px;
  background: #fff;
}

.cocon-tab {
  background: transparent;
  border: none;
  padding: 4px 10px;
  font-size: 11px;
  font-weight: 500;
  color: #6b6b75;
  border-radius: 3px;
  cursor: pointer;
}

.cocon-tab.is-active { background: #1a1a1f; color: #fff; }
.cocon-tab:hover:not(.is-active) { background: #f0f0f3; color: #1a1a1f; }

/* ============ Output ============ */

.cocon-admin-output { padding: 0 20px 16px; background: #fafafb; }

.cocon-admin-loading {
  padding: 12px;
  font-size: 13px;
  color: #6b6b75;
  font-style: italic;
}

.cocon-admin-error {
  padding: 12px;
  background: #fee;
  color: #b00;
  font-size: 13px;
  border-radius: 4px;
}

.cocon-admin-raw {
  margin: 8px 0 0;
  padding: 8px;
  background: #fff;
  border: 1px solid #f0d0d0;
  font-size: 11px;
  overflow-x: auto;
  white-space: pre-wrap;
}

.cocon-admin-usage {
  font-size: 11px;
  color: #6b6b75;
  margin-bottom: 8px;
  font-family: ui-monospace, "SF Mono", monospace;
}

.cocon-admin-usage-detail {
  font-size: 10px;
  color: #9b9ba5;
  display: block;
  margin-top: 2px;
}

.cocon-admin-json {
  background: #1a1a1f;
  color: #e5e5e8;
  padding: 16px;
  border-radius: 4px;
  font-size: 12px;
  font-family: ui-monospace, "SF Mono", monospace;
  line-height: 1.5;
  overflow-x: auto;
  white-space: pre-wrap;
  word-break: break-word;
}

/* ============ Meta SEO grid ============ */

.cocon-meta-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 8px;
  margin: 16px 0;
}

@media (max-width: 720px) {
  .cocon-meta-grid { grid-template-columns: 1fr; }
}

.cocon-meta-field {
  background: #fff;
  border: 1px solid #e5e5e8;
  border-radius: 4px;
  padding: 10px 12px;
}

.cocon-meta-label {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 10px;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  color: #6b6b75;
  font-weight: 600;
  margin-bottom: 4px;
}

.cocon-meta-label > span:first-child { flex: 1; }

.cocon-meta-count {
  background: #f0f0f3;
  color: #6b6b75;
  padding: 1px 6px;
  border-radius: 8px;
  font-size: 9px;
  font-weight: 600;
  font-family: ui-monospace, "SF Mono", monospace;
}

.cocon-meta-count.is-over { background: #fee; color: #c00; }

.cocon-meta-copy {
  background: transparent;
  border: 1px solid #e5e5e8;
  color: #6b6b75;
  padding: 1px 6px;
  border-radius: 3px;
  font-size: 10px;
  cursor: pointer;
  font-weight: 500;
}

.cocon-meta-copy:hover { border-color: #1a1a1f; color: #1a1a1f; }

.cocon-meta-value {
  font-size: 13px;
  line-height: 1.4;
  color: #1a1a1f;
  word-break: break-word;
}

/* ============ Article preview ============ */

.cocon-article-actions { display: flex; gap: 8px; margin: 16px 0 12px; }

.cocon-article-preview-label {
  font-size: 10px;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  color: #6b6b75;
  font-weight: 600;
  margin: 16px 0 8px;
}

.cocon-article-preview {
  background: #fff;
  border: 1px solid #e5e5e8;
  border-radius: 6px;
  padding: 32px 36px;
  max-width: 720px;
  margin: 0 auto;
  font-family: Georgia, "Times New Roman", serif;
  font-size: 17px;
  line-height: 1.65;
  color: #1a1a1f;
}

.cocon-article-preview h2 {
  font-family:
    ui-sans-serif, system-ui, -apple-system, sans-serif;
  font-size: 24px;
  font-weight: 600;
  line-height: 1.25;
  margin: 32px 0 12px;
}

.cocon-article-preview h2:first-child { margin-top: 0; }

.cocon-article-preview h3 {
  font-family:
    ui-sans-serif, system-ui, -apple-system, sans-serif;
  font-size: 18px;
  font-weight: 600;
  line-height: 1.3;
  margin: 24px 0 8px;
}

.cocon-article-preview p { margin: 0 0 16px; }
.cocon-article-preview ul,
.cocon-article-preview ol { margin: 0 0 16px; padding-left: 24px; }
.cocon-article-preview li { margin-bottom: 6px; }

.cocon-article-preview a {
  color: <ACCENT_COLOR>;
  text-decoration: underline;
  text-underline-offset: 2px;
}

.cocon-article-preview strong { font-weight: 700; }
.cocon-article-preview em { font-style: italic; }

.cocon-article-preview code {
  background: #f0f0f3;
  padding: 1px 6px;
  border-radius: 3px;
  font-family: ui-monospace, "SF Mono", monospace;
  font-size: 14px;
}

.cocon-article-preview pre {
  background: #1a1a1f;
  color: #e5e5e8;
  padding: 16px;
  border-radius: 6px;
  overflow-x: auto;
  margin: 0 0 16px;
  font-size: 13px;
  line-height: 1.55;
}

.cocon-article-preview pre code {
  background: transparent;
  padding: 0;
  color: inherit;
}

.cocon-article-preview blockquote {
  border-left: 3px solid <ACCENT_COLOR>;
  padding-left: 16px;
  margin: 16px 0;
  font-style: italic;
  color: #4b4b53;
}

.cocon-article-preview hr {
  border: none;
  border-top: 1px solid #e5e5e8;
  margin: 24px 0;
}

.cocon-article-preview table {
  width: 100%;
  border-collapse: collapse;
  border: 1px solid #1a1a1f;
  margin: 24px 0;
  font-size: 14px;
  line-height: 1.5;
  background: #fff;
}

.cocon-article-preview thead th {
  background: #f5f3f7;
  padding: 14px 16px;
  text-align: left;
  font-weight: 700;
  font-size: 11px;
  letter-spacing: 0.14em;
  text-transform: uppercase;
}

.cocon-article-preview tbody tr { border-top: 1px solid #e5e5e8; }
.cocon-article-preview tbody th,
.cocon-article-preview tbody td { padding: 14px 16px; vertical-align: top; }
.cocon-article-preview tbody th { font-weight: 600; background: #fafafb; }

/* ============ Publish ============ */

.cocon-btn-publish { background: #15171a; color: #fff; font-weight: 600; }
.cocon-btn-publish:hover:not(:disabled) { background: #000; }

.cocon-publish-success {
  margin: 12px 0 0;
  padding: 12px 16px;
  background: #d1fae5;
  color: #065f46;
  border-radius: 4px;
  font-size: 13px;
  line-height: 1.5;
}

.cocon-publish-success a {
  color: #047857;
  text-decoration: underline;
  font-weight: 500;
}

.cocon-publish-error {
  margin: 12px 0 0;
  padding: 12px 16px;
  background: #fee;
  color: #b00;
  border-radius: 4px;
  font-size: 13px;
  line-height: 1.5;
}

.cocon-publish-image { margin-top: 12px; }
.cocon-publish-image-label {
  display: block;
  font-size: 11px;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  color: #047857;
  font-weight: 600;
  margin-bottom: 6px;
}

.cocon-publish-image-preview {
  max-width: 100%;
  max-height: 280px;
  border-radius: 4px;
  border: 1px solid #047857;
  display: block;
}

.cocon-publish-image-warn {
  margin-top: 10px;
  padding: 8px 12px;
  background: #fef3c7;
  color: #78350f;
  border-radius: 4px;
  font-size: 12px;
  line-height: 1.45;
}

.cocon-publish-image-warn code {
  background: #fff;
  padding: 1px 6px;
  border-radius: 3px;
}

.cocon-publish-image-hint { color: #92400e; opacity: 0.85; }
```

## Mise à jour de `.env.example`

Ajoute (ou crée) `<PROJECT_ROOT>/.env.example` avec ces 4 entrées en plus de l'existant (les 3 variables Ghost de `/blog:integrate-headless` doivent déjà être là) :

```dotenv
# === Cocon Admin (génération d'articles assistée) ===

# Anthropic, clé API pour P1 (Sonnet 4.6) et P2 (Opus 4.7)
# Génère sur https://console.anthropic.com → API Keys
ANTHROPIC_API_KEY=sk-ant-...

# fal.ai, clé API pour la génération d'image hero (nano-banana-2)
# Génère sur https://fal.ai → API Keys
FAL_KEY=

# Ghost Admin API, pour pousser les articles en draft
# Format : "<24-hex-id>:<64-hex-secret>" (Ghost admin → Settings → Integrations → Claude Code)
GHOST_ADMIN_API_KEY=

# Mot de passe d'accès au back-office /cocon/admin
# Génère un random hex 32 avec : openssl rand -hex 16
COCON_ADMIN_PASSWORD=
```

⚠️ **Ne mets jamais les vraies valeurs dans `.env.example`**, c'est un fichier commité. Les vraies valeurs vont dans `.env.local` (jamais commité, déjà gitignoré par défaut dans Next.js) ou directement dans le dashboard Vercel.

Vérifie aussi que `data/` est dans `.gitignore` (le store local des briefs/articles ne doit pas être commité) :

```gitignore
# Cocon admin, cache local des briefs/articles générés
data/
```

## Configuration sur Vercel

> « Maintenant, ajoute les 4 variables dans Vercel pour que ton admin déployé puisse fonctionner :
>
> 1. Va sur **vercel.com** → ton projet → **Settings → Environment Variables**
> 2. Ajoute ces 4 (pour les 3 environnements : Production + Preview + Development) :
>    - `ANTHROPIC_API_KEY` = ta clé `sk-ant-...`
>    - `FAL_KEY` = ta clé fal.ai
>    - `GHOST_ADMIN_API_KEY` = la clé `id:secret` de Ghost
>    - `COCON_ADMIN_PASSWORD` = ton password fort
> 3. Redéploie : **Deployments → ⋯ → Redeploy** sur le dernier déploiement
>
> Confirme-moi quand les 4 variables sont ajoutées et que le redeploy est lancé. »

Attends la confirmation.

## Test final

> « On vérifie que tout marche.
>
> **Test local** (avec `pnpm dev` ou `npm run dev`) :
>
> ```bash
> # Vérifie d'abord que les 4 env vars sont dans .env.local
> grep -c '^ANTHROPIC_API_KEY=' .env.local
> grep -c '^FAL_KEY=' .env.local
> grep -c '^GHOST_ADMIN_API_KEY=' .env.local
> grep -c '^COCON_ADMIN_PASSWORD=' .env.local
> # Tu dois avoir "1" pour chaque
>
> # Puis test HTTP
> curl -sI http://localhost:3000/cocon/admin | head -5
> # Doit retourner : HTTP/1.1 200 (page de login)
> ```
>
> *(Adapte le port 3000 au port que ton `pnpm dev` utilise, c'est probablement 3000 par défaut, ou autre si tu l'as personnalisé.)*
>
> **Test fonctionnel** :
>
> 1. Ouvre `http://localhost:3000/cocon/admin` dans ton navigateur
> 2. Tu vois un formulaire de login. Saisis ton `COCON_ADMIN_PASSWORD`.
> 3. Tu arrives sur le dashboard : tous tes piliers du cocon listés, avec chaque petite-fille `planned` ou `draft` et un bouton « Générer l'article ».
> 4. Clique « Brief seul » sur un article test pour valider la chaîne Anthropic. Tu dois voir un JSON brief s'afficher en ~10 s.
> 5. Si OK : clique « Générer l'article » sur le même (ou un autre), tu vois P1+P2 tourner en ~30-60 s, puis le HTML rendu en preview.
> 6. Clique « Publier en draft sur Ghost », image fal.ai générée + post créé/mis à jour en draft Ghost. Tu reçois un lien direct vers l'éditeur Ghost.
>
> Si une étape échoue, on debug. Sinon, on clôture. »

## Vérifications

Avant clôture, valide :

- [ ] Les 9 fichiers sont écrits (5 lib + 3 routes API + 3 fichiers admin)
- [ ] `<PROJECT_ROOT>/.env.example` contient les 4 nouvelles variables
- [ ] `<PROJECT_ROOT>/.env.local` (local) contient les vraies valeurs
- [ ] `data/` est dans `.gitignore`
- [ ] Les 4 env vars sont ajoutées dans Vercel (Production + Preview + Development)
- [ ] Le redeploy Vercel a réussi (build vert)
- [ ] `<localhost>/cocon/admin` répond `200` avec la page de login
- [ ] Login + génération brief test = OK
- [ ] Génération article + publish Ghost draft = OK

Si un point ne passe pas, **retourne sur l'étape correspondante** avant de clôturer.

## Clôture

> « Ton back-office UI est prêt. Récap :
>
> - 9 fichiers générés dans ton projet (5 helpers `lib/`, 3 routes API, 3 fichiers admin)
> - 4 env vars ajoutées dans Vercel (Anthropic, fal.ai, Ghost Admin, password admin)
> - Pipeline P1 brief → P2 article → fal.ai image → push Ghost draft, piloté depuis ton navigateur
> - État persisté dans `data/cocon/` (briefs/articles/published), tu retrouves ton travail entre sessions
> - Auth simple par cookie + password (8h de session)
>
> URL admin :
> - Local : `http://localhost:<port>/cocon/admin`
> - Production : `<SITE_URL>/cocon/admin`
>
> ⚠️ **Sécurité production** : l'admin expose des routes API qui consomment tes clés Anthropic, fal.ai et Ghost Admin. Le simple password protège l'accès à l'UI mais **ne protège pas les routes API en cas de fuite du chemin `/api/cocon/...`**. Pour durcir :
>
> 1. **Mode privé Vercel** (le plus simple) : Settings → Deployment Protection → activer **Vercel Authentication** sur Production. Tu te logues une fois avec ton compte Vercel pour accéder au site.
> 2. **Middleware Auth maison** (intermédiaire) : crée `middleware.ts` à la racine qui rejette les requêtes vers `/cocon/admin` ET `/api/cocon/*` sans cookie valide. Garde une exception pour la page de login.
> 3. **Basic Auth Cloudflare/nginx** (avancé) : si tu as un reverse proxy devant Vercel.
>
> ⚠️ **Filesystem read-only sur Vercel** : `lib/cocon-storage.ts` écrit dans `data/cocon/`. Vercel serverless est read-only, l'admin **fonctionnera à la lecture initiale** (briefs/articles déjà commités) mais **pas à l'écriture**. Trois options pour la prod :
>
> 1. **Usage local uniquement** (recommandé) : tu lances l'admin en `pnpm dev` sur ta machine, tu génères les articles, tu pushes les drafts dans Ghost. Le déploiement Vercel ne sert qu'à l'affichage public du blog (`/blog`), pas à la génération.
> 2. **Vercel Blob** : remplace `lib/cocon-storage.ts` par un client `@vercel/blob` qui stocke les JSON dans Vercel Blob. ~5 lignes à changer.
> 3. **Hébergeur avec FS persistant** : Render, Railway, VPS, `node:fs` marche tel quel.
>
> Coexistence avec `/blog:article` (CLI) : les deux pipelines partagent le même `cocon.json` et les mêmes prompts P1/P2. Tu peux utiliser l'admin UI pour la prod (équipe, vue d'ensemble) et `/blog:article` pour les itérations rapides en CLI. Aucun conflit.
>
> Prochaine étape : ouvre l'admin, génère ton premier article, et publie-le dans Ghost. »

## Notes de design

- **Modèles Anthropic** : P1 utilise Sonnet 4.6 (économie, ~0,02 €/brief), P2 utilise Opus 4.7 (qualité long-form non négociable, ~0,13 €/article). Tu peux passer P1 sur Opus 4.7 si tu veux maximum qualité, modifie `model: "claude-sonnet-4-6"` en `"claude-opus-4-7"` dans `app/api/cocon/brief/route.ts` et `app/api/cocon/article/route.ts` (phase P1).
- **Parsing JSON robuste** : le scaffold strip les fences markdown éventuelles (```` ```json ... ``` ````) avant `JSON.parse`. C'est un filet de sécurité, l'instruction "JSON pur" du system prompt suffit dans 99% des cas, mais pas 100%.
- **Knowledge base inline** : la KB factuelle vit dans `lib/cocon.ts` (variable `KNOWLEDGE_BASE`). Tu peux la déporter dans `knowledge.json` à la racine et la lire au runtime, bouge alors le bloc dans son propre fichier et exporte-la. Choix ici : inline pour simplifier le scaffold, à toi de la versionner via git.
- **Pas de `output_config.format.json_schema`** : option Anthropic SDK avancée non utilisée pour rester compatible avec toutes les versions du SDK. Le system prompt impose JSON strict et on parse à la main. Si tu veux la garantie schéma stricte côté Anthropic, ajoute l'option dans les 2 routes API.
- **Stockage local par défaut** : `data/cocon/` permet de retrouver ses briefs/articles entre sessions. Pas de DB à gérer. À l'échelle (équipe, prod), bascule sur Vercel Blob ou Postgres.
- **Statut Ghost forcé draft** : `lib/ghost-admin.ts` force `status: "draft"` dans `upsertPost()`. Non négociable, relecture humaine obligatoire avant publication. Aligné avec `/blog:article` CLI.
- **Pas de TOC, pas de progress bar** : l'admin est une UI interne minimale. Pas d'enjeu de polish UX, fonctionnel d'abord.
- **Coexistence avec `/blog:article` CLI** : les deux pipelines lisent le même `cocon.json` et utilisent les mêmes prompts. Tu peux mixer librement, par exemple : générer en CLI sur ta machine de dev, garder l'admin UI ouverte pour suivre l'état d'avancement à plusieurs.
