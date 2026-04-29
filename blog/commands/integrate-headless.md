---
name: integrate-headless
description: Génère le scaffold de code (lib + routes + sitemap + webhook + .env) pour qu'un site Next.js 16 consomme l'API Ghost et rende le blog directement à <site>/blog/* (scénario C — headless API). Astro et SvelteKit en V1.5.
---

# /blog:integrate-headless — Scaffold du blog headless dans ton site

Tu vas générer dans le projet de l'élève **tout le code nécessaire** pour que son site Next.js 16 (App Router) consomme la Ghost Content API et rende lui-même les pages `/blog` et `/blog/<slug>`. À la fin de cette commande, l'élève a 7 fichiers générés, ses 3 variables d'env documentées dans `.env.example`, et les instructions pour configurer Vercel + le webhook Ghost de revalidation.

Cette commande **n'a de sens qu'en scénario C** (headless API). Si l'élève a choisi le scénario A (URL PikaPods native) ou B (sous-domaine custom) dans `/blog:setup-ghost`, le rendu est servi par Ghost directement et cette commande n'a rien à faire — redirige-le vers `/blog:theme`.

**Temps estimé** : ~10 min (lecture des questions + génération + collage des 3 env vars dans Vercel + config webhook Ghost).

**Coût LLM** : 0 € (pas d'appel modèle, c'est du scaffolding pur).

## Pré-requis

### Pré-requis 1 — `ghost-config.md` existe et scénario = C

Lis `ghost-config.md` à la racine du projet. Vérifie qu'il contient bien `**Type** : headless API rendu par le framework` (ou variante claire indiquant scénario C).

Si absent :

> « Je ne trouve pas `ghost-config.md`. Lance d'abord `/blog:setup-ghost` pour mettre Ghost en place et choisir ton scénario d'hébergement. Reviens ici uniquement si tu as choisi le scénario C (headless API). »

Si le scénario n'est **pas** C :

> « Ton `ghost-config.md` indique que tu es en scénario <A | B>. Cette commande ne sert qu'au scénario C (headless API, où ton framework JS rend le blog à `<site>/blog`). Dans ton cas, Ghost sert directement le rendu — pas besoin de scaffolder du code. Passe à `/blog:theme` pour habiller ton blog Ghost à ta charte. »

### Pré-requis 2 — `<GHOST_BACKEND_URL>` est connue

Récupère depuis `ghost-config.md` la valeur de `Instance backend (PikaPods)` ou `Ghost backend (jamais visible)`. Stocke-la comme `<GHOST_BACKEND_URL>` (ex. `https://wonderful-caribou.pikapod.net`).

Si introuvable, demande-la :

> « Quelle est l'URL technique de ton instance Ghost sur PikaPods ? (ex. `https://wonderful-caribou.pikapod.net`, sans slash final) »

## Préambule

Ouvre la commande par ce message :

> « On va scaffolder le code blog dans ton site Next.js. Plan :
>
> 1. Vérifier ton framework (Next.js 16 supporté en V1, Astro/SvelteKit en V1.5)
> 2. Te demander 2-3 paramètres (path racine, couleur d'accent, police principale)
> 3. Générer 7 fichiers : `lib/ghost.ts`, `app/blog/page.tsx`, `app/blog/[slug]/page.tsx`, `app/blog/styles.css`, `app/blog/list-styles.css`, `app/sitemap.ts` (créé ou patché), `app/api/revalidate-blog/route.ts`
> 4. Mettre à jour `.env.example` avec les 3 variables Ghost
> 5. Te dire comment ajouter ces variables dans Vercel
> 6. Te dire comment configurer le webhook Ghost de revalidation
> 7. Test final : `curl <site>/blog` doit retourner du HTML
>
> À la fin, ton blog est rendu directement à `<ton-site>/blog` par ton framework, avec ISR (1h) et invalidation à la demande via webhook Ghost. Prête ? »

## Protocole

Pose les questions **UNE PAR UNE**. Attends la réponse avant de passer à la suivante.

### Question 1 — Framework

> « Quel framework JS fait tourner ton site sur Vercel ?
>
> 1. **Next.js 16** (App Router) — supporté en V1, génération automatique
> 2. **Astro** — V1.5 ; pour l'instant, lis le code de référence dans le code de démonstration fourni par le formateur (`lib/ghost.ts`, `app/blog/`) et adapte manuellement à `src/pages/blog/`
> 3. **SvelteKit** — V1.5 ; idem, adapte manuellement à `src/routes/blog/`
> 4. **Nuxt / Remix** — non supporté en V1, mais le pattern est identique : un client `lib/ghost.ts` + 2 routes serveur + un sitemap + un webhook
>
> Quel est ton framework ? »

Stocke la réponse comme `<FRAMEWORK>`.

**Si `<FRAMEWORK>` ≠ Next.js** :

> « Très bien. La V1 du plugin scaffolde uniquement Next.js 16. Pour <FRAMEWORK>, voici la marche à suivre manuelle :
>
> 1. Va lire le code de démonstration fourni par le formateur (publié sur GitHub) : c'est l'implémentation Next.js 16 du scénario C qu'on cherche à reproduire.
> 2. Les fichiers à reproduire dans ton projet :
>    - `lib/ghost.ts` (~170 lignes, fetch Content API, fonctions `getAllPosts`, `getPostBySlug`, `getPostSlugs`)
>    - Une page liste des articles (`/blog`)
>    - Une page détail d'article (`/blog/<slug>`) avec rendu du HTML Koenig de Ghost
>    - Un sitemap qui merge les routes statiques + les slugs Ghost
>    - Une route POST `/api/revalidate-blog?token=...` qui invalide le cache à chaque webhook Ghost
>    - 3 env vars : `GHOST_URL`, `GHOST_CONTENT_API_KEY`, `REVALIDATE_SECRET`
> 3. Adapte les conventions à <FRAMEWORK> :
>    - Astro : `src/pages/blog/index.astro` + `src/pages/blog/[slug].astro` + `src/pages/api/revalidate-blog.ts` + `astro.config.mjs` avec `output: 'server'` ou `'hybrid'`
>    - SvelteKit : `src/routes/blog/+page.server.ts` + `src/routes/blog/[slug]/+page.server.ts` + `src/routes/api/revalidate-blog/+server.ts`
>
> Le scaffold automatique <FRAMEWORK> arrive en V1.5. Pour l'instant, je te conseille de copier les fichiers de référence à la main dans ton projet et de demander à Claude de les adapter à la convention de ton framework — il sait faire.
>
> Si tu veux quand même tenter le scaffold automatique en t'imaginant que c'est Next.js, je peux le faire — mais les fichiers générés seront à 80% à réécrire. Tu préfères lire le code de référence ou tenter le scaffold ? »

Si l'élève insiste pour `<FRAMEWORK>` non-Next, **stoppe la commande** proprement et redirige vers le repo de référence. Ne génère rien — ce serait du code mort.

**Si `<FRAMEWORK>` = Next.js**, continue.

### Question 2 — App Router

> « Tu utilises bien l'App Router (dossier `app/` à la racine du projet, pas `pages/`) ? L'App Router est le standard depuis Next.js 13. Le Pages Router n'est pas supporté en V1. »

Si `pages/` :

> « Le Pages Router n'est pas supporté en V1. Tu as deux options :
>
> 1. Migrer vers l'App Router (recommandé, c'est le standard depuis Next.js 13)
> 2. Adapter manuellement le code du code de démonstration fourni par le formateur au Pages Router : `pages/blog/index.tsx` + `pages/blog/[slug].tsx` + `pages/api/revalidate-blog.ts` avec `getStaticProps` + `getStaticPaths` + `revalidate` (ISR classique)
>
> La V1 ne scaffolde que l'App Router. Veux-tu stopper ici et migrer manuellement, ou continuer en sachant que tu devras faire l'adaptation à la main ? »

Stoppe si `pages/`. Continue uniquement si l'App Router est confirmé.

### Question 3 — Path racine du projet

> « Quel est le path racine de ton projet Next.js ? Par défaut `.` (le dossier courant). C'est le dossier qui contient `package.json`, `next.config.js` ou `next.config.ts`, et `app/`. »

Stocke `<PROJECT_ROOT>` (par défaut `.`).

Vérifie rapidement :
- `<PROJECT_ROOT>/package.json` existe et liste `next` dans `dependencies`
- `<PROJECT_ROOT>/app/` existe

Si l'un manque :

> « Je ne trouve pas <fichier manquant> dans <PROJECT_ROOT>. Vérifie le path et redonne-moi-le. »

### Question 4 — Couleur d'accent

> « Quelle couleur d'accent veux-tu pour les liens, tags et CTA du blog ? Donne un hex (ex. `#7C4DFF`) ou laisse vide pour le défaut violet `#7C4DFF` (cohérent avec le code de démonstration fourni par le formateur). »

Stocke `<ACCENT_COLOR>` (par défaut `#7C4DFF`).

### Question 5 — Police principale

> « Quelle famille de police pour le corps du texte ? Donne un nom CSS (ex. `Inter`, `Geist`, `system-ui`) ou laisse vide pour le défaut `system-ui, sans-serif`. »

Stocke `<FONT_FAMILY>` (par défaut `system-ui, sans-serif`).

> Note : tu peux affiner la typographie plus tard en éditant directement `app/blog/styles.css` et `app/blog/list-styles.css`.

## Génération des 7 fichiers

Crée les fichiers ci-dessous dans `<PROJECT_ROOT>`. **Si un fichier existe déjà**, demande à l'élève s'il veut le remplacer ou skipper. Pour `app/sitemap.ts` spécifiquement, regarde la sous-section dédiée plus bas (patch vs création).

Avant chaque écriture, **annonce le fichier**. Après chaque écriture, confirme « ✓ écrit ».

### Fichier 1/7 — `lib/ghost.ts`

```typescript
/**
 * Ghost Content API client (read-only, public key).
 *
 * Docs: https://ghost.org/docs/content-api/
 *
 * All fetches are tagged "blog" for on-demand revalidation via
 * `app/api/revalidate-blog/route.ts` (called by Ghost webhook).
 */

const GHOST_URL = process.env.GHOST_URL?.replace(/\/$/, "") ?? "";
const GHOST_KEY = process.env.GHOST_CONTENT_API_KEY ?? "";

const DEFAULT_REVALIDATE = 3600; // 1h ISR fallback if webhook misses
const BLOG_TAG = "blog";
const ACCEPT_VERSION = "v5.0";

export const isGhostConfigured = Boolean(GHOST_URL && GHOST_KEY);

export type GhostTag = {
  id: string;
  name: string;
  slug: string;
};

export type GhostAuthor = {
  id: string;
  name: string;
  slug: string;
  profile_image: string | null;
  bio: string | null;
};

export type GhostPost = {
  id: string;
  uuid: string;
  title: string;
  slug: string;
  html: string;
  feature_image: string | null;
  feature_image_alt: string | null;
  feature_image_caption: string | null;
  excerpt: string;
  custom_excerpt: string | null;
  reading_time: number;
  published_at: string;
  updated_at: string;
  tags?: GhostTag[];
  authors?: GhostAuthor[];
  primary_tag?: GhostTag | null;
  primary_author?: GhostAuthor | null;
  meta_title: string | null;
  meta_description: string | null;
  og_image: string | null;
  og_title: string | null;
  og_description: string | null;
  twitter_image: string | null;
  twitter_title: string | null;
  twitter_description: string | null;
  canonical_url: string | null;
};

type Pagination = {
  page: number;
  limit: number;
  pages: number;
  total: number;
  next: number | null;
  prev: number | null;
};

type PostsResponse = { posts: GhostPost[]; meta: { pagination: Pagination } };

function buildUrl(path: string, params: Record<string, string | number> = {}) {
  const url = new URL(`${GHOST_URL}/ghost/api/content${path}`);
  url.searchParams.set("key", GHOST_KEY);
  for (const [k, v] of Object.entries(params)) {
    url.searchParams.set(k, String(v));
  }
  return url.toString();
}

async function ghostFetch<T>(url: string): Promise<T> {
  const res = await fetch(url, {
    headers: { "Accept-Version": ACCEPT_VERSION },
    next: { revalidate: DEFAULT_REVALIDATE, tags: [BLOG_TAG] },
  });
  if (!res.ok) {
    const body = await res.text().catch(() => "");
    throw new Error(
      `Ghost API ${res.status} ${res.statusText} — ${url}\n${body}`,
    );
  }
  return res.json() as Promise<T>;
}

export async function getAllPosts(
  limit: number | "all" = "all",
): Promise<GhostPost[]> {
  if (!isGhostConfigured) return [];
  const url = buildUrl("/posts/", {
    limit,
    include: "tags,authors",
    order: "published_at desc",
  });
  const data = await ghostFetch<PostsResponse>(url);
  return data.posts;
}

export async function getPostBySlug(slug: string): Promise<GhostPost | null> {
  if (!isGhostConfigured) return null;
  const url = buildUrl(`/posts/slug/${encodeURIComponent(slug)}/`, {
    include: "tags,authors",
  });
  try {
    const data = await ghostFetch<PostsResponse>(url);
    return data.posts[0] ?? null;
  } catch (err) {
    const msg = err instanceof Error ? err.message : String(err);
    if (msg.includes("404")) return null;
    throw err;
  }
}

export async function getPostSlugs(): Promise<string[]> {
  if (!isGhostConfigured) return [];
  const url = buildUrl("/posts/", {
    limit: "all",
    fields: "slug",
    order: "published_at desc",
  });
  const data = await ghostFetch<PostsResponse>(url);
  return data.posts.map((p) => p.slug);
}

export function formatPublishedDate(iso: string): string {
  const date = new Date(iso);
  return new Intl.DateTimeFormat("fr-FR", {
    day: "2-digit",
    month: "long",
    year: "numeric",
  }).format(date);
}

export const GHOST_CACHE_TAG = BLOG_TAG;
```

### Fichier 2/7 — `app/blog/page.tsx`

```typescript
import Link from "next/link";
import Image from "next/image";
import {
  getAllPosts,
  isGhostConfigured,
  formatPublishedDate,
  type GhostPost,
} from "@/lib/ghost";
import "./list-styles.css";

const SITE_URL =
  process.env.NEXT_PUBLIC_SITE_URL?.replace(/\/$/, "") ?? "";

export const metadata = {
  title: "Blog — Articles récents",
  description: "Analyses, tutos, retours d'expérience.",
  alternates: { canonical: "/blog" },
};

export default async function BlogPage() {
  const posts = await safeGetPosts();
  const [hero, ...rest] = posts;

  // JSON-LD : ItemList des articles publiés
  const itemListLd = {
    "@context": "https://schema.org",
    "@type": "ItemList",
    name: "Articles du blog",
    itemListElement: posts.map((p, i) => ({
      "@type": "ListItem",
      position: i + 1,
      name: p.title,
      url: `${SITE_URL}/blog/${p.slug}`,
    })),
  };

  return (
    <div className="blog-list-root">
      <script
        type="application/ld+json"
        dangerouslySetInnerHTML={{ __html: JSON.stringify(itemListLd) }}
      />

      <header className="blog-list-header">
        <div className="blog-list-wrap">
          <h1 className="blog-list-title">Blog</h1>
        </div>
      </header>

      {!isGhostConfigured ? (
        <main className="blog-list-section">
          <div className="blog-list-wrap">
            <p className="blog-list-empty">
              Le blog est en cours de configuration. Les articles arrivent
              très bientôt.
            </p>
          </div>
        </main>
      ) : posts.length === 0 ? (
        <main className="blog-list-section">
          <div className="blog-list-wrap">
            <p className="blog-list-empty">
              Aucun article publié pour le moment.
            </p>
          </div>
        </main>
      ) : (
        <>
          {hero && <BlogHeroCard post={hero} />}
          {rest.length > 0 && (
            <main className="blog-list-section">
              <div className="blog-list-wrap">
                <div className="blog-list-grid">
                  {rest.map((post) => (
                    <BlogListCard key={post.id} post={post} />
                  ))}
                </div>
              </div>
            </main>
          )}
        </>
      )}
    </div>
  );
}

function BlogHeroCard({ post }: { post: GhostPost }) {
  const tag = post.primary_tag?.name;
  return (
    <Link href={`/blog/${post.slug}`} className="blog-list-hero">
      {post.feature_image && (
        <div className="blog-list-hero-bg" aria-hidden>
          <Image
            src={post.feature_image}
            alt=""
            fill
            sizes="100vw"
            priority
            unoptimized
            className="blog-list-hero-img"
          />
          <div className="blog-list-hero-overlay" />
        </div>
      )}
      <div className="blog-list-hero-inner">
        <div className="blog-list-hero-eyebrow">À LA UNE</div>
        <div className="blog-list-hero-meta">
          {tag && <span className="blog-list-hero-tag">{tag}</span>}
          <span>{formatPublishedDate(post.published_at)}</span>
          <span>·</span>
          <span>{post.reading_time} min</span>
        </div>
        <h2 className="blog-list-hero-title">{post.title}</h2>
        {post.custom_excerpt && (
          <p className="blog-list-hero-excerpt">{post.custom_excerpt}</p>
        )}
        <span className="blog-list-hero-cta">Lire l'article →</span>
      </div>
    </Link>
  );
}

function BlogListCard({ post }: { post: GhostPost }) {
  const tag = post.primary_tag?.name;
  return (
    <Link href={`/blog/${post.slug}`} className="blog-list-card">
      {post.feature_image && (
        <div className="blog-list-card-image-wrap">
          <Image
            src={post.feature_image}
            alt={post.feature_image_alt ?? post.title}
            width={1200}
            height={800}
            className="blog-list-card-image"
            sizes="(max-width: 720px) 100vw, 33vw"
            unoptimized
          />
          <div className="blog-list-card-image-overlay" />
        </div>
      )}
      <div className="blog-list-card-body">
        <div className="blog-list-card-meta">
          {tag && <span className="blog-list-card-tag">{tag}</span>}
          <span className="blog-list-card-date">
            {formatPublishedDate(post.published_at)}
          </span>
        </div>
        <h2 className="blog-list-card-title">{post.title}</h2>
        <span className="blog-list-card-read">{post.reading_time} min</span>
      </div>
    </Link>
  );
}

async function safeGetPosts(): Promise<GhostPost[]> {
  try {
    return await getAllPosts("all");
  } catch (err) {
    console.error("[blog] Ghost fetch failed:", err);
    return [];
  }
}
```

### Fichier 3/7 — `app/blog/[slug]/page.tsx`

⚠️ **Next.js 16 — `params` est un `Promise<...>` qu'il faut `await`.** C'est le breaking change majeur depuis Next 15. Ne le supprime jamais.

```typescript
import type { Metadata } from "next";
import { notFound } from "next/navigation";
import Link from "next/link";
import Image from "next/image";
import {
  getPostBySlug,
  getPostSlugs,
  formatPublishedDate,
  type GhostPost,
} from "@/lib/ghost";
import "../styles.css";

const SITE_URL =
  process.env.NEXT_PUBLIC_SITE_URL?.replace(/\/$/, "") ?? "";

type Params = { slug: string };

export async function generateStaticParams(): Promise<Params[]> {
  try {
    const slugs = await getPostSlugs();
    return slugs.map((slug) => ({ slug }));
  } catch (err) {
    console.error("[blog] generateStaticParams failed:", err);
    return [];
  }
}

export async function generateMetadata({
  params,
}: {
  params: Promise<Params>;
}): Promise<Metadata> {
  const { slug } = await params;
  const post = await safeGetPost(slug);
  if (!post) {
    return { title: "Article introuvable", robots: { index: false } };
  }
  const title = post.meta_title ?? post.title;
  const description =
    post.meta_description ??
    post.custom_excerpt ??
    post.excerpt.slice(0, 160);
  const ogImage = post.og_image ?? post.feature_image ?? undefined;
  return {
    title,
    description,
    alternates: { canonical: post.canonical_url ?? `/blog/${slug}` },
    openGraph: {
      type: "article",
      title: post.og_title ?? title,
      description: post.og_description ?? description,
      url: `${SITE_URL}/blog/${slug}`,
      images: ogImage ? [{ url: ogImage }] : [],
      publishedTime: post.published_at,
      modifiedTime: post.updated_at,
      authors: post.authors?.map((a) => a.name) ?? [],
      tags: post.tags?.map((t) => t.name) ?? [],
    },
    twitter: {
      card: "summary_large_image",
      title: post.twitter_title ?? title,
      description: post.twitter_description ?? description,
      images: post.twitter_image
        ? [post.twitter_image]
        : ogImage
          ? [ogImage]
          : [],
    },
  };
}

async function safeGetPost(slug: string): Promise<GhostPost | null> {
  try {
    return await getPostBySlug(slug);
  } catch (err) {
    console.error(`[blog] getPostBySlug(${slug}) failed:`, err);
    return null;
  }
}

export default async function BlogPostPage({
  params,
}: {
  params: Promise<Params>;
}) {
  const { slug } = await params;
  const post = await safeGetPost(slug);
  if (!post) notFound();

  const tag = post.primary_tag?.name;
  const author = post.primary_author;

  const articleSchema = {
    "@context": "https://schema.org",
    "@type": "BlogPosting",
    headline: post.title,
    description: post.custom_excerpt ?? post.excerpt,
    image: post.feature_image ?? undefined,
    datePublished: post.published_at,
    dateModified: post.updated_at,
    author: author
      ? { "@type": "Person", name: author.name }
      : { "@type": "Organization", name: "Site" },
    mainEntityOfPage: {
      "@type": "WebPage",
      "@id": `${SITE_URL}/blog/${slug}`,
    },
    keywords: post.tags?.map((t) => t.name).join(", "),
  };

  const breadcrumbSchema = {
    "@context": "https://schema.org",
    "@type": "BreadcrumbList",
    itemListElement: [
      {
        "@type": "ListItem",
        position: 1,
        name: "Accueil",
        item: SITE_URL,
      },
      {
        "@type": "ListItem",
        position: 2,
        name: "Blog",
        item: `${SITE_URL}/blog`,
      },
      {
        "@type": "ListItem",
        position: 3,
        name: post.title,
        item: `${SITE_URL}/blog/${slug}`,
      },
    ],
  };

  return (
    <div className="blog-root">
      <script
        type="application/ld+json"
        dangerouslySetInnerHTML={{ __html: JSON.stringify(articleSchema) }}
      />
      <script
        type="application/ld+json"
        dangerouslySetInnerHTML={{
          __html: JSON.stringify(breadcrumbSchema),
        }}
      />

      <article className="blog-post">
        <header
          className={`blog-post-hero${post.feature_image ? " blog-post-hero--with-image" : ""}`}
        >
          {post.feature_image && (
            <div className="blog-post-hero-bg" aria-hidden>
              <Image
                src={post.feature_image}
                alt=""
                fill
                sizes="100vw"
                priority
                unoptimized
                className="blog-post-hero-bg-img"
              />
              <div className="blog-post-hero-bg-overlay" />
            </div>
          )}
          <div className="blog-wrap">
            <nav className="blog-post-breadcrumb" aria-label="Breadcrumb">
              <Link href="/blog">Blog</Link>
              <span className="blog-post-breadcrumb-sep">/</span>
              <span>{post.title}</span>
            </nav>
            <div className="blog-post-meta-top">
              {tag && <span className="blog-card-category">{tag}</span>}
              <span>{formatPublishedDate(post.published_at)}</span>
              <span>·</span>
              <span>{post.reading_time} min de lecture</span>
            </div>
            <h1 className="blog-post-title">{post.title}</h1>
            {post.custom_excerpt && (
              <p className="blog-post-excerpt">{post.custom_excerpt}</p>
            )}
            {author && (
              <div className="blog-post-author">
                {author.profile_image && (
                  <Image
                    src={author.profile_image}
                    alt={author.name}
                    width={48}
                    height={48}
                    className="blog-post-author-avatar"
                    unoptimized
                  />
                )}
                <span>Par {author.name}</span>
              </div>
            )}
          </div>
        </header>

        <section className="blog-post-body-section">
          <div className="blog-wrap">
            <div
              className="blog-prose"
              dangerouslySetInnerHTML={{ __html: post.html }}
            />
          </div>
        </section>
      </article>
    </div>
  );
}
```

### Fichier 4/7 — `app/blog/styles.css`

Adapte la couleur d'accent et la police aux valeurs `<ACCENT_COLOR>` et `<FONT_FAMILY>` répondues plus haut. Le CSS ci-dessous est scopé sous `.blog-root` (page détail).

```css
/* /blog/<slug> — page détail article. Scoped sous .blog-root. */

.blog-root {
  --paper: #FFFFFF;
  --mist: #F1F1F3;
  --ink: #1F1D24;
  --ink-soft: #3a342a;
  --ink-dim: #6B6775;
  --accent: <ACCENT_COLOR>;

  font-family: <FONT_FAMILY>;
  background: var(--paper);
  color: var(--ink);
  line-height: 1.5;
  -webkit-font-smoothing: antialiased;
  min-height: 100vh;
}

.blog-root * { box-sizing: border-box; }

.blog-root .blog-wrap {
  max-width: 1100px;
  margin: 0 auto;
  padding: 0 24px;
}

/* ============ HERO ============ */
.blog-root .blog-post-hero {
  padding: 64px 0 48px;
  background: var(--paper);
  border-bottom: 1px solid var(--mist);
}

.blog-root .blog-post-breadcrumb {
  margin-bottom: 24px;
  font-size: 13px;
  color: var(--ink-dim);
}

.blog-root .blog-post-breadcrumb a {
  color: var(--ink);
  text-decoration: none;
  border-bottom: 1px solid currentColor;
  padding-bottom: 1px;
}

.blog-root .blog-post-breadcrumb a:hover { color: var(--accent); }

.blog-root .blog-post-breadcrumb-sep {
  color: var(--ink-dim);
  margin: 0 8px;
}

.blog-root .blog-post-meta-top {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-bottom: 20px;
  font-size: 12px;
  letter-spacing: 0.08em;
  text-transform: uppercase;
  color: var(--ink-dim);
  flex-wrap: wrap;
}

.blog-root .blog-card-category {
  color: var(--accent);
  border: 1px solid var(--accent);
  padding: 4px 8px;
}

.blog-root .blog-post-title {
  font-weight: 700;
  font-size: clamp(32px, 5vw, 56px);
  line-height: 1.05;
  letter-spacing: -0.02em;
  margin: 0 0 20px;
  max-width: 900px;
}

.blog-root .blog-post-excerpt {
  font-size: clamp(17px, 2vw, 21px);
  line-height: 1.45;
  color: var(--ink-soft);
  max-width: 760px;
  margin: 0 0 24px;
}

.blog-root .blog-post-author {
  display: flex;
  align-items: center;
  gap: 12px;
  font-size: 14px;
  color: var(--ink-soft);
}

.blog-root .blog-post-author-avatar {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  object-fit: cover;
}

/* Hero with cover image */
.blog-root .blog-post-hero--with-image {
  position: relative;
  isolation: isolate;
  overflow: hidden;
  padding: 96px 0 56px;
  min-height: 420px;
  display: flex;
  align-items: center;
  border-bottom: none;
}

.blog-root .blog-post-hero-bg { position: absolute; inset: 0; z-index: -1; }
.blog-root .blog-post-hero-bg-img { object-fit: cover; }

.blog-root .blog-post-hero-bg-overlay {
  position: absolute;
  inset: 0;
  background: linear-gradient(180deg, rgba(0, 0, 0, 0.35) 0%, rgba(0, 0, 0, 0.75) 100%);
}

.blog-root .blog-post-hero--with-image .blog-post-title,
.blog-root .blog-post-hero--with-image .blog-post-excerpt,
.blog-root .blog-post-hero--with-image .blog-post-author,
.blog-root .blog-post-hero--with-image .blog-post-meta-top {
  color: #FFFFFF;
}

.blog-root .blog-post-hero--with-image .blog-post-breadcrumb a {
  color: rgba(255, 255, 255, 0.85);
}

/* ============ BODY / PROSE ============ */
.blog-root .blog-post-body-section {
  padding: 48px 0 96px;
}

.blog-root .blog-prose {
  max-width: 720px;
  margin: 0 auto;
  font-size: 18px;
  line-height: 1.75;
  color: var(--ink);
}

.blog-root .blog-prose > * + * { margin-top: 1.2em; }

.blog-root .blog-prose h2 {
  font-weight: 700;
  font-size: clamp(24px, 3vw, 32px);
  line-height: 1.2;
  margin-top: 1.8em;
  margin-bottom: 0.5em;
}

.blog-root .blog-prose h3 {
  font-weight: 600;
  font-size: clamp(20px, 2.4vw, 24px);
  line-height: 1.3;
  margin-top: 1.6em;
  margin-bottom: 0.4em;
}

.blog-root .blog-prose h4 {
  font-weight: 600;
  font-size: 18px;
  margin-top: 1.4em;
  margin-bottom: 0.4em;
}

.blog-root .blog-prose p { margin: 0; }

.blog-root .blog-prose a {
  color: var(--accent);
  text-decoration: none;
  border-bottom: 1px solid var(--accent);
}

.blog-root .blog-prose a:hover { opacity: 0.7; }

.blog-root .blog-prose strong { font-weight: 600; }
.blog-root .blog-prose em { font-style: italic; }

.blog-root .blog-prose blockquote {
  margin: 1.6em 0;
  padding: 0 0 0 20px;
  border-left: 3px solid var(--accent);
  font-style: italic;
  font-size: clamp(18px, 2vw, 22px);
  line-height: 1.5;
  color: var(--ink-soft);
}

.blog-root .blog-prose ul,
.blog-root .blog-prose ol {
  padding-left: 1.4em;
  margin: 1em 0;
}

.blog-root .blog-prose ul { list-style: disc; }
.blog-root .blog-prose ol { list-style: decimal; }
.blog-root .blog-prose li + li { margin-top: 0.4em; }
.blog-root .blog-prose li::marker { color: var(--accent); }

.blog-root .blog-prose code {
  font-family: ui-monospace, monospace;
  font-size: 0.92em;
  background: var(--mist);
  padding: 2px 6px;
  border-radius: 3px;
}

.blog-root .blog-prose pre {
  background: var(--ink);
  color: #FFFFFF;
  padding: 18px 22px;
  overflow-x: auto;
  border-radius: 8px;
  margin: 1.6em 0;
  font-family: ui-monospace, monospace;
  font-size: 14px;
  line-height: 1.5;
}

.blog-root .blog-prose pre code {
  background: transparent;
  padding: 0;
  color: inherit;
}

.blog-root .blog-prose img {
  width: 100%;
  height: auto;
  display: block;
  margin: 1.6em 0;
  border-radius: 6px;
}

.blog-root .blog-prose figure { margin: 1.6em 0; }
.blog-root .blog-prose figcaption {
  font-size: 13px;
  color: var(--ink-dim);
  margin-top: 8px;
  text-align: center;
}

.blog-root .blog-prose hr {
  border: none;
  border-top: 1px solid var(--mist);
  margin: 2.4em 0;
}

.blog-root .blog-prose table {
  width: 100%;
  border-collapse: collapse;
  margin: 1.6em 0;
  font-size: 15px;
}

.blog-root .blog-prose th,
.blog-root .blog-prose td {
  border: 1px solid var(--mist);
  padding: 12px 16px;
  text-align: left;
  vertical-align: top;
}

.blog-root .blog-prose thead th {
  background: var(--mist);
  font-weight: 600;
}

/* ============ Ghost Koenig embeds ============ */
.blog-root .blog-prose .kg-callout-card,
.blog-root .blog-prose .kg-toggle-card,
.blog-root .blog-prose .kg-bookmark-card {
  border: 1px solid var(--mist);
  border-radius: 8px;
  padding: 18px 22px;
  margin: 1.6em 0;
  background: var(--mist);
}

.blog-root .blog-prose .kg-image-card { margin: 1.6em 0; }
.blog-root .blog-prose .kg-image-card img { margin: 0; }

.blog-root .blog-prose .kg-bookmark-card {
  display: block;
  text-decoration: none;
  color: var(--ink);
  background: var(--paper);
}

.blog-root .blog-prose .kg-bookmark-title { font-weight: 600; margin-bottom: 4px; }
.blog-root .blog-prose .kg-bookmark-description {
  font-size: 14px;
  color: var(--ink-dim);
}

@media (max-width: 640px) {
  .blog-root .blog-post-hero { padding: 40px 0 32px; }
  .blog-root .blog-post-body-section { padding: 32px 0 64px; }
  .blog-root .blog-prose { font-size: 16px; line-height: 1.7; }
}
```

### Fichier 5/7 — `app/blog/list-styles.css`

```css
/* /blog (listing) — Scoped sous .blog-list-root. */

.blog-list-root {
  --paper: #FFFFFF;
  --mist: #F1F1F3;
  --ink: #1a1510;
  --ink-soft: #3a342a;
  --ink-dim: #6a6358;
  --accent: <ACCENT_COLOR>;

  font-family: <FONT_FAMILY>;
  background: var(--paper);
  color: var(--ink);
  -webkit-font-smoothing: antialiased;
  min-height: 100vh;
}

.blog-list-root * { box-sizing: border-box; }

.blog-list-root .blog-list-wrap {
  max-width: 1320px;
  margin: 0 auto;
  padding: 0 32px;
}

/* ============ HEADER ============ */
.blog-list-root .blog-list-header {
  padding: 64px 0 32px;
  background: var(--mist);
  border-bottom: 1px solid var(--ink);
}

.blog-list-root .blog-list-title {
  font-weight: 700;
  font-size: clamp(36px, 5vw, 64px);
  line-height: 1;
  letter-spacing: -0.02em;
  margin: 0;
  color: var(--ink);
}

/* ============ HERO FULL-BLEED ============ */
.blog-list-root .blog-list-hero {
  position: relative;
  display: flex;
  align-items: flex-end;
  min-height: 480px;
  padding: 100px 32px 48px;
  overflow: hidden;
  isolation: isolate;
  text-decoration: none;
  color: #FFFFFF;
  background: var(--ink);
}

.blog-list-root .blog-list-hero-bg { position: absolute; inset: 0; z-index: -1; }
.blog-list-root .blog-list-hero-img { object-fit: cover; transition: transform 0.6s ease; }
.blog-list-root .blog-list-hero:hover .blog-list-hero-img { transform: scale(1.03); }

.blog-list-root .blog-list-hero-overlay {
  position: absolute;
  inset: 0;
  background: linear-gradient(180deg, rgba(0, 0, 0, 0.3) 0%, rgba(0, 0, 0, 0.85) 100%);
}

.blog-list-root .blog-list-hero-inner {
  max-width: 1100px;
  margin: 0 auto;
  width: 100%;
  position: relative;
  z-index: 2;
  display: flex;
  flex-direction: column;
  gap: 14px;
}

.blog-list-root .blog-list-hero-eyebrow {
  font-size: 11px;
  font-weight: 700;
  letter-spacing: 0.25em;
  color: var(--accent);
  text-transform: uppercase;
}

.blog-list-root .blog-list-hero-meta {
  display: flex;
  align-items: center;
  gap: 10px;
  flex-wrap: wrap;
  font-size: 12px;
  letter-spacing: 0.08em;
  text-transform: uppercase;
  color: rgba(255, 255, 255, 0.78);
}

.blog-list-root .blog-list-hero-tag {
  font-size: 10px;
  font-weight: 700;
  letter-spacing: 0.16em;
  text-transform: uppercase;
  background: var(--accent);
  color: #FFFFFF;
  padding: 4px 10px;
}

.blog-list-root .blog-list-hero-title {
  font-weight: 600;
  font-size: clamp(26px, 3.4vw, 44px);
  line-height: 1.1;
  margin: 0;
  text-wrap: balance;
}

.blog-list-root .blog-list-hero-excerpt {
  font-size: clamp(16px, 1.8vw, 20px);
  line-height: 1.45;
  margin: 0;
  max-width: 720px;
  color: rgba(255, 255, 255, 0.92);
  text-wrap: balance;
}

.blog-list-root .blog-list-hero-cta {
  font-size: 12px;
  font-weight: 700;
  letter-spacing: 0.18em;
  text-transform: uppercase;
  color: #FFFFFF;
  margin-top: 12px;
  padding-bottom: 4px;
  border-bottom: 1px solid #FFFFFF;
  align-self: flex-start;
}

/* ============ GRID ============ */
.blog-list-root .blog-list-section {
  padding: 56px 0 96px;
  background: var(--paper);
}

.blog-list-root .blog-list-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(320px, 1fr));
  gap: 20px;
}

.blog-list-root .blog-list-card {
  position: relative;
  display: flex;
  flex-direction: column;
  text-decoration: none;
  color: var(--ink);
  background: var(--mist);
  overflow: hidden;
  border-radius: 8px;
  transition: transform 0.25s, box-shadow 0.25s;
  min-height: 320px;
}

.blog-list-root .blog-list-card:hover {
  transform: translateY(-3px);
  box-shadow: 0 14px 30px rgba(0, 0, 0, 0.12);
}

.blog-list-root .blog-list-card-image-wrap {
  position: absolute;
  inset: 0;
  overflow: hidden;
}

.blog-list-root .blog-list-card-image {
  width: 100%;
  height: 100%;
  object-fit: cover;
  transition: transform 0.5s ease;
}

.blog-list-root .blog-list-card:hover .blog-list-card-image {
  transform: scale(1.04);
}

.blog-list-root .blog-list-card-image-overlay {
  position: absolute;
  inset: 0;
  background: linear-gradient(to bottom, rgba(0, 0, 0, 0.2) 0%, rgba(0, 0, 0, 0.55) 50%, rgba(0, 0, 0, 0.92) 100%);
  pointer-events: none;
}

.blog-list-root .blog-list-card-body {
  position: relative;
  z-index: 2;
  padding: 22px;
  margin-top: auto;
  display: flex;
  flex-direction: column;
  color: #FFFFFF;
  gap: 10px;
}

.blog-list-root .blog-list-card:not(:has(.blog-list-card-image-wrap)) .blog-list-card-body {
  color: var(--ink);
  margin-top: 0;
  height: 100%;
  justify-content: space-between;
  padding: 24px;
}

.blog-list-root .blog-list-card-meta {
  display: flex;
  align-items: center;
  gap: 10px;
  flex-wrap: wrap;
}

.blog-list-root .blog-list-card-tag {
  font-size: 10px;
  letter-spacing: 0.18em;
  text-transform: uppercase;
  background: var(--accent);
  color: #FFFFFF;
  padding: 4px 10px;
  font-weight: 600;
}

.blog-list-root .blog-list-card-date {
  font-size: 10px;
  letter-spacing: 0.12em;
  text-transform: uppercase;
  opacity: 0.85;
}

.blog-list-root .blog-list-card-title {
  font-weight: 600;
  font-size: 20px;
  line-height: 1.2;
  margin: 0;
  text-wrap: balance;
}

.blog-list-root .blog-list-card-read {
  font-size: 10px;
  letter-spacing: 0.18em;
  text-transform: uppercase;
  opacity: 0.78;
}

.blog-list-root .blog-list-empty {
  font-style: italic;
  font-size: 18px;
  color: var(--ink-soft);
  text-align: center;
  padding: 64px 0;
}

@media (max-width: 720px) {
  .blog-list-root .blog-list-wrap { padding: 0 20px; }
  .blog-list-root .blog-list-hero { min-height: 360px; padding: 72px 24px 36px; }
  .blog-list-root .blog-list-grid { grid-template-columns: 1fr; gap: 16px; }
  .blog-list-root .blog-list-card { min-height: 260px; }
}
```

### Fichier 6/7 — `app/sitemap.ts` (création OU patch)

#### Cas A — `app/sitemap.ts` n'existe pas encore

Crée le fichier minimal :

```typescript
import type { MetadataRoute } from "next";
import { getAllPosts } from "@/lib/ghost";

const SITE_URL =
  process.env.NEXT_PUBLIC_SITE_URL?.replace(/\/$/, "") ?? "";

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const now = new Date();

  // Routes statiques : adapte cette liste à ton site.
  const staticRoutes = [
    { path: "/", priority: 1.0, changeFrequency: "weekly" as const },
    { path: "/blog", priority: 0.7, changeFrequency: "weekly" as const },
  ];

  const staticEntries: MetadataRoute.Sitemap = staticRoutes.map((r) => ({
    url: `${SITE_URL}${r.path}`,
    lastModified: now,
    changeFrequency: r.changeFrequency,
    priority: r.priority,
  }));

  let blogEntries: MetadataRoute.Sitemap = [];
  try {
    const posts = await getAllPosts("all");
    blogEntries = posts.map((p) => ({
      url: `${SITE_URL}/blog/${p.slug}`,
      lastModified: new Date(p.updated_at),
      changeFrequency: "monthly" as const,
      priority: 0.6,
    }));
  } catch (err) {
    console.error(
      "[sitemap] Ghost fetch failed, skipping blog entries:",
      err,
    );
  }

  return [...staticEntries, ...blogEntries];
}
```

#### Cas B — `app/sitemap.ts` existe déjà

⚠️ **Ne remplace pas le fichier.** Affiche son contenu actuel à l'élève et propose un patch précis :

> « Ton `app/sitemap.ts` existe déjà. Pour intégrer les articles Ghost, voici le patch à appliquer :
>
> 1. Vérifie que la fonction par défaut est `async` (sinon transforme-la en `async`)
> 2. Importe `getAllPosts` : `import { getAllPosts } from "@/lib/ghost";`
> 3. Avant le `return`, ajoute :
>
> ```typescript
> let blogEntries: MetadataRoute.Sitemap = [];
> try {
>   const posts = await getAllPosts("all");
>   blogEntries = posts.map((p) => ({
>     url: `${SITE_URL}/blog/${p.slug}`,
>     lastModified: new Date(p.updated_at),
>     changeFrequency: "monthly" as const,
>     priority: 0.6,
>   }));
> } catch (err) {
>   console.error("[sitemap] Ghost fetch failed:", err);
> }
> ```
>
> 4. Inclue `...blogEntries` dans le tableau retourné.
>
> Veux-tu que j'applique ce patch automatiquement ? Si oui, je le fais ; si non, tu peux le copier-coller à la main. »

Si l'élève accepte, applique le patch via Edit. Sinon, écris le snippet et passe à la suite.

### Fichier 7/7 — `app/api/revalidate-blog/route.ts`

```typescript
import { NextRequest, NextResponse } from "next/server";
import { revalidateTag, revalidatePath } from "next/cache";
import { GHOST_CACHE_TAG } from "@/lib/ghost";

/**
 * Ghost webhook → revalidate blog cache.
 *
 * Configure on Ghost (Settings → Integrations → Claude Code → Add webhook):
 *   Events: Post published, Post updated, Post unpublished, Post deleted
 *   URL:    https://<site>/api/revalidate-blog?token=<REVALIDATE_SECRET>
 *
 * Env var: REVALIDATE_SECRET — shared secret matching the ?token query param.
 */
export async function POST(req: NextRequest) {
  const token = req.nextUrl.searchParams.get("token");
  const secret = process.env.REVALIDATE_SECRET;

  if (!secret) {
    console.error("[revalidate-blog] REVALIDATE_SECRET not configured");
    return NextResponse.json(
      { ok: false, error: "REVALIDATE_SECRET not configured" },
      { status: 500 },
    );
  }

  if (token !== secret) {
    console.warn("[revalidate-blog] unauthorized — token mismatch");
    return NextResponse.json(
      { ok: false, error: "invalid token" },
      { status: 401 },
    );
  }

  let slug: string | undefined;
  try {
    const body = await req.json();
    slug = body?.post?.current?.slug ?? body?.post?.previous?.slug;
  } catch (err) {
    console.warn("[revalidate-blog] could not parse webhook body:", err);
  }

  try {
    // Next.js 16: revalidateTag à 1 argument est déprécié.
    // Utiliser le 2e argument 'max' (stale-while-revalidate).
    revalidateTag(GHOST_CACHE_TAG, "max");
    revalidatePath("/blog");
    if (slug) revalidatePath(`/blog/${slug}`);
    console.log(
      `[revalidate-blog] ok — tag=${GHOST_CACHE_TAG} slug=${slug ?? "(none)"}`,
    );
  } catch (err) {
    console.error("[revalidate-blog] revalidation failed:", err);
    return NextResponse.json(
      { ok: false, error: "revalidation failed" },
      { status: 500 },
    );
  }

  return NextResponse.json({
    ok: true,
    revalidated: {
      tag: GHOST_CACHE_TAG,
      paths: ["/blog", slug && `/blog/${slug}`].filter(Boolean),
    },
  });
}
```

## Mise à jour de `.env.example`

Ajoute (ou crée) `<PROJECT_ROOT>/.env.example` avec ces 3 entrées en plus de l'existant :

```dotenv
# === Ghost Content API (blog headless) ===

# URL de ton instance Ghost sur PikaPods (sans slash final)
GHOST_URL=https://<ton-instance>.pikapod.net

# Clé Content API (read-only) — Ghost admin → Settings → Integrations → Claude Code
# Format : 24 caractères hexadécimaux
GHOST_CONTENT_API_KEY=

# Secret partagé pour le webhook de revalidation
# Génère-le avec: openssl rand -hex 32
REVALIDATE_SECRET=

# URL publique de ton site (utilisée par le sitemap, OG, JSON-LD)
NEXT_PUBLIC_SITE_URL=https://<ton-site>.vercel.app
```

⚠️ **Ne mets jamais les vraies valeurs dans `.env.example`** — c'est un fichier commité. Les vraies valeurs vont dans `.env.local` (jamais commité, déjà gitignoré par défaut dans Next.js) ou directement dans le dashboard Vercel.

## Configuration sur Vercel

> « Maintenant, ajoute les 3 variables dans Vercel pour que ton site déployé puisse parler à Ghost :
>
> 1. Va sur **vercel.com** → ton projet → **Settings → Environment Variables**
> 2. Ajoute ces 3 (pour les 3 environnements : Production + Preview + Development) :
>    - `GHOST_URL` = `<GHOST_BACKEND_URL>` *(récupéré de ghost-config.md)*
>    - `GHOST_CONTENT_API_KEY` = ta clé Content API (24 hex)
>    - `REVALIDATE_SECRET` = un secret généré avec `openssl rand -hex 32` (note-le, tu vas le réutiliser dans Ghost)
> 3. Si pas déjà défini : `NEXT_PUBLIC_SITE_URL` = ton URL publique (ex. `https://mon-site.vercel.app`)
> 4. Redéploie : **Deployments → ⋯ → Redeploy** sur le dernier déploiement (sans cache)
>
> Confirme-moi quand les 3 variables sont ajoutées et que le redeploy est lancé. »

Attends la confirmation avant de passer à l'étape suivante.

## Configuration du webhook Ghost

> « Dernière étape côté Ghost : on lui dit d'appeler ton site à chaque publication/édition d'article, pour que le cache se rafraîchisse automatiquement.
>
> 1. Ghost admin → **Settings → Integrations**
> 2. Section **Custom integrations** → ouvre l'intégration `Claude Code` (créée dans `/blog:setup-ghost`)
> 3. En bas, section **Webhooks** → bouton **Add webhook**
> 4. Pour chaque événement ci-dessous, ajoute un webhook séparé :
>    - **Name** : `Revalidate <event>`
>    - **Event** : choisis l'événement
>    - **Target URL** : `<SITE_URL>/api/revalidate-blog?token=<REVALIDATE_SECRET>`
>
> Les 4 événements à brancher :
>
> - `Post published`
> - `Post updated` (libellé Ghost : « Post published, edited »)
> - `Post unpublished`
> - `Post deleted`
>
> Sauvegarde. Test : édite un article (juste corriger une virgule), sauve. Tu dois voir dans les logs Vercel (`Functions → revalidate-blog`) un appel `200 OK` avec ton slug.
>
> Confirme-moi quand les 4 webhooks sont créés et que tu vois bien l'appel `200 OK` dans les logs Vercel. »

Attends la confirmation.

## Test final

> « On vérifie que tout marche.
>
> 1. Ouvre `<SITE_URL>/blog` dans ton navigateur. Tu dois voir la liste des articles Ghost (ou « Aucun article publié pour le moment » si Ghost est encore vide — c'est OK).
> 2. Ouvre `<SITE_URL>/blog/<un-slug>` (clique sur un article). Tu dois voir le rendu détail avec ton CSS.
> 3. Ouvre `<SITE_URL>/sitemap.xml`. Tu dois y voir tes URLs `/blog/<slug>`.
>
> Test depuis le terminal :
>
> ```bash
> curl -sI <SITE_URL>/blog | head -5
> # Doit retourner : HTTP/2 200
>
> curl -s <SITE_URL>/sitemap.xml | grep -c "<loc>"
> # Doit retourner un nombre > 0 (au moins tes routes statiques)
> ```
>
> Si une étape échoue, on debug. Sinon, on clôture. »

## Vérifications

Avant clôture, valide :

- [ ] Les 7 fichiers sont créés (ou patché pour `sitemap.ts`)
- [ ] `<PROJECT_ROOT>/.env.example` contient les 3 variables Ghost
- [ ] Les 3 env vars sont ajoutées dans Vercel (Production + Preview + Development)
- [ ] Le redeploy Vercel a réussi (build vert)
- [ ] Les 4 webhooks Ghost sont créés et pointent sur `<SITE_URL>/api/revalidate-blog?token=<REVALIDATE_SECRET>`
- [ ] `<SITE_URL>/blog` répond `200` avec du HTML
- [ ] Test webhook : un édit d'article déclenche bien un `200 OK` dans les logs Vercel

Si un point ne passe pas, **retourne sur l'étape correspondante** avant de clôturer.

## Clôture

> « Ton blog headless est en ligne. Récap :
>
> - 7 fichiers générés dans ton projet (`lib/ghost.ts`, 2 routes blog, 2 CSS, sitemap, route revalidate)
> - 3 env vars ajoutées dans Vercel
> - 4 webhooks Ghost branchés pour invalidation à la demande
> - Cache Next.js : 1h ISR fallback + invalidation immédiate via webhook (`revalidateTag('blog', 'max')`)
> - SEO : metadata par article + JSON-LD `BlogPosting` + `BreadcrumbList` + sitemap mergé
>
> Ton blog vit maintenant à `<SITE_URL>/blog`. Ghost reste sur PikaPods et fait office de CMS — il n'est jamais visible des visiteurs.
>
> Prochaine étape : si tu n'as pas encore de cocon, lance `/blog:cocon`. Sinon `/blog:article` pour publier ton premier article. »

## Notes de design

- **Pas de theme Ghost en scénario C** : tu rends avec le HTML/CSS de ton site. Les fichiers `app/blog/styles.css` et `app/blog/list-styles.css` générés ici sont à customiser pour matcher 100% ta charte. Ce sont des bases volontairement neutres.
- **`<Image unoptimized>`** : on évite de configurer `remotePatterns` dans `next.config.ts` (Ghost-hosted images). Si tu publies beaucoup d'images, bascule en `remotePatterns` explicite et enlève `unoptimized` pour bénéficier de l'optimisation Vercel.
- **JSON-LD inline** : pas de wrapper `<JsonLd />` factorisé pour rester autonome. Si tu veux mutualiser, crée `components/JsonLd.tsx` qui prend `{ data }: { data: object | object[] }` et émet `<script type="application/ld+json">{JSON.stringify(data)}</script>`.
- **Pas de TOC sticky / progress bar** : le code de démonstration fourni par le formateur en a une (`BlogReadingChrome.tsx`) mais elle dépend de `transformBlogHtml` + `extractHeadings` (`lib/blog-html.ts`). Volontairement omis pour rester en scaffolding minimal. Tu peux l'ajouter ensuite en lisant le repo de référence.
- **Pas de pages pilier (`/blog/pilier/<slug>`)** : également présentes dans le code de démonstration fourni par le formateur, dépendent de `lib/cocon.ts`. À ajouter manuellement si tu veux des pages pilier — la donnée vient de ton `cocon.json` généré par `/blog:cocon`.
