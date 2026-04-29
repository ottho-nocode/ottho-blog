---
name: article
description: Pipeline complet de génération d'un article du cocon. Sélectionne une petite-fille planifiée, génère brief (P1) puis article (P2), image hero (fal.ai), publie en draft sur Ghost. Met à jour cocon.json. Statut Ghost forcé à draft.
---

# /blog:article, Génération d'un article du cocon (P1 + P2 + image + Ghost draft)

Tu vas accompagner l'étudiant dans la génération d'un article complet : sélection d'une petite-fille du cocon, brief stratégique, rédaction long-form, image hero, publication en **draft** sur Ghost. À la fin, l'étudiant a un article relisable dans l'admin Ghost, qu'il valide et publie à la main.

**Charge ces 4 skills** avant tout, elles contiennent les system prompts, schemas, sanitizers, validateurs et garde-fous factuels :

- **`article-quality`** (`/Users/thibaultmarty/ottho-blog-plugin/skills/article-quality/SKILL.md`), system prompts P1 et P2, schemas JSON `BRIEF_SCHEMA` / `ARTICLE_SCHEMA`, sanitizer cadratin, clamp Ghost. ⚠️ **Cette skill contient les prompts à reproduire intégralement.**
- **`link-validation`** (`/Users/thibaultmarty/ottho-blog-plugin/skills/link-validation/SKILL.md`), algorithme de validation des `<a href>` internes contre une whitelist, fallback vers la pilier-parent.
- **`knowledge-base`** (`/Users/thibaultmarty/ottho-blog-plugin/skills/knowledge-base/SKILL.md`), pattern de la KB factuelle (`knowledge.json`) injectée dans P2 pour empêcher les hallucinations chiffrées.
- **`cocon-method`** (`/Users/thibaultmarty/ottho-blog-plugin/skills/cocon-method/SKILL.md`), structure de `cocon.json` (mère, filles, petites-filles), règles de maillage, contexte de linking.

**Temps estimé** : ~3 min en interactif (40 s d'API + dialogues de validation + 1 image fal.ai).

**Coût LLM estimé** : ~0,15 € par article (P1 Sonnet 4.6 + P2 Opus 4.7 + 1 image fal.ai ~0,02 €).

---

## Garde-fous critiques (à ne JAMAIS contourner)

⚠️ **Statut Ghost FORCÉ à `draft`.** L'article généré est **toujours** créé en `status: "draft"`. Le plugin n'a pas le droit de publier directement. C'est l'étudiant qui relit dans l'admin Ghost et clique « Publish » à la main.

⚠️ **KB obligatoire en P2.** Sans `knowledge.json` injecté dans le user message du prompt P2, le modèle hallucine prix, versions, statistiques. Si la KB est absente, **arrête le pipeline** et propose à l'étudiant de la créer (voir Étape 1 ci-dessous).

⚠️ **Link-validation obligatoire avant push Ghost.** Sans le validateur, le HTML peut contenir des `<a href="/...">` inventés qui finissent en 404 sur le blog publié. La validation est non-négociable.

---

## Préambule

Ouvre la commande par ce message :

> « On va générer un article complet de ton cocon. Le pipeline :
>
> 1. Charger ton `cocon.json` et ta `knowledge.json`
> 2. Sélectionner la petite-fille à écrire
> 3. Générer le brief stratégique (P1, ~10 s, ~0,02 €)
> 4. Te montrer le brief, tu valides ou tu ajustes
> 5. Générer l'article long-form (P2, ~30 s, ~0,13 €)
> 6. Sanitizer le cadratin + valider tous les liens internes
> 7. Générer l'image hero via fal.ai (nano-banana-2)
> 8. Pousser dans Ghost en `status: draft`
> 9. Mettre à jour `cocon.json`
>
> À la fin, l'article est dans ton admin Ghost. Tu relis, tu corriges, tu publies à la main. **Le plugin ne publie jamais directement.**
>
> Prête ? On y va. »

---

## Pré-requis (à valider AVANT de commencer)

Avant toute exécution, vérifie les 4 pré-requis ci-dessous. Si l'un manque, **mets en pause** et redirige.

### Pré-requis 1, `cocon.json` existe

> « Je vérifie que `cocon.json` existe à la racine du projet. »

Si absent : pause et redirige :

> « Tu n'as pas encore de cocon. Lance d'abord `/blog:cocon` pour cartographier ta mère + 3-7 filles + 3-5 petites-filles par fille. Reviens ici quand le fichier `cocon.json` est généré et validé. »

### Pré-requis 2, Ghost en place + MCP Ghost dispo

> « Je vérifie que ton instance Ghost et le MCP Ghost sont opérationnels. »

Si absent : pause et redirige :

> « Tu n'as pas encore Ghost. Lance `/blog:setup-ghost` pour créer ton instance PikaPods, brancher le sous-domaine et installer le MCP Ghost. Reviens ici quand l'admin Ghost s'ouvre dans ton navigateur. »

### Pré-requis 3, Theme Ghost actif

> « Je vérifie que ton theme Ghost est actif. »

Si absent : pause et redirige :

> « Lance `/blog:theme` pour activer le theme Ottho (ou ton theme custom) avant de générer le premier article. »

### Pré-requis 4, MCP fal.ai dispo (vu au cours précédent)

> « Je vérifie que le MCP fal.ai est configuré (vu dans le chapitre précédent du cours). »

Si absent : warning **mais on continue** :

> « Pas de MCP fal.ai détecté. On peut continuer sans image hero (mode dégradé). L'article sera publié sans `feature_image`, tu pourras en ajouter une à la main dans Ghost. Tu veux continuer sans image, ou tu préfères revenir installer le MCP fal.ai d'abord ? »

---

## Étape 1, Charger le contexte

### 1.1, Lire `cocon.json`

Lis `cocon.json` à la racine du projet. Parse-le. Vérifie qu'il a bien la structure attendue (`mere`, `filles[]`, `filles[].petites_filles[]`). Si parsing échoue ou structure invalide, propose à l'étudiant de relancer `/blog:cocon`.

### 1.2, Lire `knowledge.json` (KB)

Lis `knowledge.json` à la racine du projet.

⚠️ **Si `knowledge.json` n'existe pas** : c'est obligatoire pour P2. Propose à l'étudiant de la créer maintenant :

> « Tu n'as pas encore de `knowledge.json`. Sans elle, P2 hallucine des chiffres et des versions, ta marque ne tolère pas ça.
>
> Je peux te générer un squelette à partir du pattern de la skill `knowledge-base`. Tu auras juste à remplir les sections `pricing`, `business`, `ecosystem` avec tes propres données. Ça prend 5 minutes.
>
> On y va, ou tu préfères créer le fichier toi-même avant de relancer ? »

Si l'étudiant accepte : génère `knowledge.json` avec la structure-type de la skill `knowledge-base` (sections `validated_at`, `version`, `pricing`, `models`, `ecosystem`, `business`, `rules`), pré-remplie avec des placeholders explicites (`"REMPLIR : prix mensuel de ton offre principale en €"`). Pause le pipeline tant que la KB n'est pas remplie.

### 1.3, Charger les 4 skills

Charge en mémoire les contenus des 4 skills mentionnées en haut. Tu vas en avoir besoin :

- `article-quality` → system prompts P1 et P2 + schemas JSON + sanitizer
- `link-validation` → algorithme de whitelist + fallback
- `knowledge-base` → format d'injection KB
- `cocon-method` → structure de `cocon.json` pour identifier mère/sœurs/CTA

---

## Étape 2, Sélectionner la petite-fille à écrire

Liste toutes les petites-filles avec `status: "planned"` ou `status: "draft"`, **groupées par pilier**. Affiche en table compacte :

```
Pilier : pipeline-blog (3 articles à écrire)
  [1] cocon-semantique-ia         , keyword "cocon sémantique ia"     [planned]
  [2] blog-headless-ghost-nextjs  , keyword "blog headless ghost"     [planned]
  [3] rediger-article-seo-claude  , keyword "rédiger article seo claude" [planned]

Pilier : creer-site-web-claude (4 articles à écrire)
  [4] ...
```

Pose la question :

> « Laquelle écrire ? Donne-moi le numéro, ou tape `default` pour la première planifiée du pilier le plus avancé (= le pilier qui a déjà le plus d'articles publiés/draft). »

Capture le choix. Stocke `<FILLE_SLUG>`, `<PETITE_FILLE_SLUG>`, `<KEYWORD>`, `<PILIER_SLUG>`, `<PERSONA>`, `<CTA_PRINCIPAL>`.

---

## Étape 3, Construire le `linking_context`

Identifie depuis `cocon.json` :

- **Mère** : `cocon.json` → `mere.slug` → URL `/<mere-slug>` (ex. `/apprendre-claude`)
- **Pilier-parent** (la fille à laquelle appartient la petite-fille sélectionnée) : `pilier_slug`, `title`, `cta_principal`
- **2-3 piliers-sœurs** : prends 2 ou 3 autres filles du `cocon.json`, les plus thématiquement proches du pilier-parent (heuristique simple : descriptions partageant des keywords)
- **Outils linkés** : `cocon.json` → `linked_outils[]` du pilier-parent, ou liste statique si vide
- **CTA principal** : `cta_principal` du pilier-parent (ex. `/claude-mastery`)

### 3.1, Récupérer les articles déjà publiés dans Ghost

Appelle le MCP Ghost (Content API) pour lister les slugs de tous les posts déjà publiés. Pseudo-code :

```typescript
const publishedSlugs = await mcp.ghost.posts_browse({
  filter: "status:published",
  fields: "slug",
  limit: "all"
});
// → ["serveur-mcp-claude", "premier-prompt-claude", ...]
```

### 3.2, Construire la whitelist d'URLs

Selon le protocole de la skill `link-validation`, agrège dans un `Set<string>` toutes les URLs autorisées :

| Source | Format URL |
|---|---|
| Mère | `/<mere-slug>` |
| 10 piliers du cocon | `/blog/pilier/<pilier-slug>` (×10) |
| Articles déjà publiés (Ghost) | `/blog/<slug>` (récupérés à l'étape 3.1) |
| Outils du site | `/outils/<slug>` |
| Formations CTA | `/claude-mastery`, `/claude-builders`, `/claude-agent` |
| Pages structurelles | `/`, `/blog`, `/contact`, `/methode`, `/formations`, `/atelier`, `/avis` |

Stocke comme `<ALLOWED_URLS>` pour l'étape 8 (link-validation).

---

## Étape 4, Générer le brief (P1)

### 4.1, Charger le system prompt P1

Charge **intégralement** le system prompt P1 depuis la skill `article-quality` (section « P1, System prompt (brief) »). ⚠️ **Ne le réécris pas, copie-le tel quel.**

### 4.2, Construire le user message P1

```
KEYWORD CIBLE : <KEYWORD>
TITRE PROPOSÉ : <PETITE_FILLE_TITLE>
WORD COUNT TARGET : 1500

CONTEXTE COCON :
- Mère : <MERE_SLUG> (URL : /<MERE_SLUG>), keyword "<MERE_KEYWORD>"
- Pilier-parent : <PILIER_SLUG> (URL : /blog/pilier/<PILIER_SLUG>), titre "<PILIER_TITLE>"
- Piliers-sœurs : <SISTERS_LIST_avec_URLs>
- Outils linkés : <LINKED_OUTILS_avec_URLs>
- CTA principal : <CTA_PRINCIPAL>

PERSONA CIBLE : <PERSONA>
INTENT : <intent_de_la_petite_fille>
```

### 4.3, Appeler l'API Anthropic

```typescript
const briefResponse = await anthropic.messages.create({
  model: "claude-sonnet-4-6",     // ~10 s, ~0,02 €, qualité brief suffisante
  max_tokens: 2000,
  system: P1_SYSTEM_PROMPT,
  messages: [{ role: "user", content: P1_USER_MESSAGE }],
  output_config: {
    format: {
      type: "json_schema",
      schema: BRIEF_SCHEMA           // depuis skill article-quality
    }
  },
  thinking: { type: "disabled" }
});

const brief = JSON.parse(briefResponse.content[0].text);
```

> **Note** : si l'étudiant veut maximum qualité, propose Opus 4.7 à la place (~0,06 € au lieu de 0,02 €). Par défaut Sonnet 4.6.

Parse le JSON. Vérifie qu'il respecte `BRIEF_SCHEMA` (champs `intent_summary`, `promise`, `h1`, `structure`, `key_takeaways`, `internal_links_suggested`, `word_count_target`, `tone`).

---

## Étape 5, Validation user du brief

Affiche le brief de manière lisible et compacte :

```
TITRE H1 : <h1 du brief>
PROMESSE : <promise>
INTENT : <intent_summary>
TON : <tone>
WORD COUNT TARGET : <word_count_target>

PLAN H2 :
  1. <h2_1>, <what_1>
  2. <h2_2>, <what_2>
  3. <h2_3>, <what_3>
  ...

KEY TAKEAWAYS :
  • <takeaway_1>
  • <takeaway_2>
  ...

ANCRES SUGGÉRÉES (liens internes) :
  → <url_1> (anchor: "<anchor_1>")
  → <url_2> (anchor: "<anchor_2>")
  ...
```

Pose la question :

> « Le brief est OK ? Tape :
>
> - `accepter` pour passer à la rédaction
> - `modifier` pour ajuster manuellement (titre, plan, ton)
> - `regenerer` pour relancer P1 avec une instruction supplémentaire (ex. "plus orienté pratique", "ton plus expert") »

Capture le choix :

- **`accepter`** → passe à l'étape 6
- **`modifier`** → demande quel champ modifier, applique l'édition manuelle
- **`regenerer`** → demande l'instruction additionnelle, ré-appelle P1 avec cette instruction injectée dans le user message

Stocke le brief final comme `<BRIEF>`.

---

## Étape 6, Générer l'article (P2)

### 6.1, Charger le system prompt P2

Charge **intégralement** le system prompt P2 depuis la skill `article-quality` (section « P2, System prompt (rédaction) »). ⚠️ **Reproduis-le tel quel.**

### 6.2, Construire le user message P2

L'ordre des sections compte (KB **avant** les instructions créatives, voir skill `knowledge-base`) :

```
KNOWLEDGE BASE 2026 (validée le <validated_at>), utilise EXCLUSIVEMENT ces faits pour tout chiffre, prix, statistique, version. Si une info manque : OMETS-LA ou utilise un conditionnel ('environ', 'à partir de', 'selon les données disponibles').

<JSON.stringify(knowledge_base, null, 2)>

---

BRIEF DE L'ARTICLE :
<JSON.stringify(BRIEF, null, 2)>

---

LIENS_AUTORISES (whitelist stricte, ne perds pas de tokens à inventer des URLs, le système les supprime hors whitelist) :
<liste des URLs depuis ALLOWED_URLS, une par ligne>

---

CONTRAINTES :
- Word count target : <BRIEF.word_count_target> mots (~1500-2000)
- Tone : <BRIEF.tone>
- Format : HTML pur (pas de Markdown)
- Pas de balise <h1> (titre géré séparément)
- 4 à 6 liens internes obligatoires (1 vers mère + 2-3 sœurs + 1 CTA final + 0-2 outils)
- Tutoiement
- Aucun tiret cadratin (—) ni demi-cadratin (–)
- Inclure 1 bloc <pre><code> si pertinent
- Inclure 1 <table> si comparaison/décision/listes longues pertinentes
- Terminer par un H2 actionnable type "Passer à la pratique" ou "Et maintenant"
```

### 6.3, Appeler l'API Anthropic

```typescript
const articleResponse = await anthropic.messages.create({
  model: "claude-opus-4-7",         // ~30 s, ~0,13 €, qualité long-form
  max_tokens: 8000,
  system: P2_SYSTEM_PROMPT,
  messages: [{ role: "user", content: P2_USER_MESSAGE }],
  output_config: {
    format: {
      type: "json_schema",
      schema: ARTICLE_SCHEMA         // depuis skill article-quality
    }
  },
  thinking: { type: "disabled" },    // structure stricte, pas de raisonnement profond
  // Effort : pour les modèles supportant `effort`, mettre "high".
  // Optionnel : prompt caching (cache_control: "ephemeral") sur system + KB
  // pour un batch ; pour un article seul, l'économie est marginale.
});

const article = JSON.parse(articleResponse.content[0].text);
```

Parse le JSON. Vérifie qu'il respecte `ARTICLE_SCHEMA` (champs `html`, `meta_title`, `meta_description`, `custom_excerpt`, `og_title`, `og_description`, `tags`, `primary_tag`, `feature_image_prompt`).

Stocke comme `<ARTICLE>` (pré-sanitizer).

---

## Étape 7, Sanitizer

⚠️ **Filet de sécurité non-négociable**, même si le prompt P2 a déjà l'instruction « pas de cadratin ». Le modèle peut occasionnellement glisser un `—`, et Ghost rejette les payloads avec `custom_excerpt > 300 caractères`.

### 7.1, Strip cadratin sur tous les champs string

Applique `sanitizeArticle(<ARTICLE>)` (depuis skill `article-quality`) qui passe `stripEmDashes` sur :

- `html` (toute la prose)
- `meta_title`, `meta_description`
- `custom_excerpt`
- `og_title`, `og_description`
- `feature_image_prompt`
- Chaque string de `tags[]`

Heuristiques (ordre = priorité, voir skill) :

| Pattern | Sortie |
|---|---|
| `", "` | `", "` |
| `"— "` | `": "` |
| `" —"` | `","` |
| `"—"` | `":"` |
| `"–"` | `"-"` |

### 7.2, Clamp limites Ghost (caractères, pas mots)

Pour chaque champ avec une limite, applique `clampToWord(s, max)` (trim au mot entier le plus proche + ellipsis `…`) :

| Champ | Max chars |
|---|---|
| `meta_title` | 60 |
| `meta_description` | 155 |
| `custom_excerpt` | 297 (Ghost = 300, marge de 3) |
| `og_title` | 60 |
| `og_description` | 200 |

Stocke comme `<ARTICLE_SANITIZED>`.

---

## Étape 8, Validation des liens internes ⚠️

⚠️ **Non-négociable.** Sans validation, les `<a href="/...">` inventés par le modèle deviennent des 404 dans Ghost.

### 8.1, Charger le validator

Charge l'algorithme depuis la skill `link-validation`. Le contrat :

```typescript
type ValidationResult = {
  cleaned: string;          // HTML corrigé
  totalLinks: number;
  internalLinks: number;
  externalLinks: number;
  rewrittenLinks: Array<{ original: string; rewritten: string }>;
};

function validateLinks(
  html: string,
  allowedUrls: Set<string>,    // <ALLOWED_URLS> de l'étape 3.2
  fallbackUrl: string,         // = /blog/pilier/<PILIER_SLUG>
): ValidationResult;
```

### 8.2, Exécuter la validation

```typescript
const result = validateLinks(
  ARTICLE_SANITIZED.html,
  ALLOWED_URLS,
  `/blog/pilier/${PILIER_SLUG}`   // fallback = pilier-parent
);

ARTICLE_SANITIZED.html = result.cleaned;
const linkReport = {
  total: result.totalLinks,
  internal: result.internalLinks,
  external: result.externalLinks,
  rewritten: result.rewrittenLinks
};
```

Stocke `<LINK_REPORT>` pour le rapport final.

---

## Étape 9, Image hero (fal.ai)

### 9.1, Récupérer le prompt image depuis P2

Le `feature_image_prompt` du JSON P2 est en anglais, format « editorial magazine photograph, Kodak Portra 400 film grain, [sujet spécifique], muted palette, photorealistic ».

### 9.2, Appeler le MCP fal.ai

```typescript
try {
  const falResult = await mcp.fal.run({
    endpoint: "fal-ai/nano-banana-2",
    input: {
      prompt: ARTICLE_SANITIZED.feature_image_prompt,
      aspect_ratio: "16:9",          // format hero classique
      resolution: "2K",
      output_format: "jpeg",
      num_images: 1
    }
  });

  const falUrl = falResult.images[0].url;
  // Télécharger l'image localement (binaire, pas base64) avant upload Ghost
  const imageBlob = await fetch(falUrl).then(r => r.blob());
  // Stocker temporairement
} catch (err) {
  // Mode dégradé acceptable
  console.warn("⚠️ Génération image fal.ai échouée :", err.message);
  console.warn("L'article sera publié sans feature_image. Tu pourras en ajouter une à la main dans Ghost.");
  imageBlob = null;
}
```

⚠️ **En cas d'échec fal.ai** : continue le pipeline **sans bloquer**. L'article sera publié sans `feature_image`. Affiche un warning clair à l'étudiant à la fin.

---

## Étape 10, Upload image dans Ghost

Si `imageBlob` n'est pas null :

```typescript
const ghostImage = await mcp.ghost.images_upload({
  file: imageBlob,
  purpose: "image"
});

const ghostImageUrl = ghostImage.url;
// → https://blog.tonsite.com/content/images/2026/04/<filename>.jpeg
// L'image est désormais hébergée par Ghost (plus de dépendance fal.ai).
```

Stocke `<GHOST_IMAGE_URL>`. Si pas d'image : `GHOST_IMAGE_URL = null`.

---

## Étape 11, Création post draft dans Ghost ⚠️

⚠️ **Statut FORCÉ à `draft`.** Quoi qu'il arrive, on ne publie pas directement.

```typescript
const post = await mcp.ghost.posts_add({
  source: "html",                                // input HTML pur
  posts: [{
    title: ARTICLE_SANITIZED.meta_title,         // titre éditorial
    slug: PETITE_FILLE_SLUG,                     // slug de la petite-fille du cocon
    html: ARTICLE_SANITIZED.html,                // HTML sanitizé + links validés
    status: "draft",                             // ⚠️ FORCÉ, JAMAIS published
    feature_image: GHOST_IMAGE_URL,              // null si fal.ai a échoué
    tags: [
      { name: ARTICLE_SANITIZED.primary_tag },   // = pilier_slug
      ...ARTICLE_SANITIZED.tags.map(t => ({ name: t }))
    ],
    meta_title: ARTICLE_SANITIZED.meta_title,
    meta_description: ARTICLE_SANITIZED.meta_description,
    custom_excerpt: ARTICLE_SANITIZED.custom_excerpt,
    og_title: ARTICLE_SANITIZED.og_title,
    og_description: ARTICLE_SANITIZED.og_description
  }]
});

const ghostPostId = post.posts[0].id;
const ghostAdminEditUrl = `https://blog.<DOMAIN>/ghost/#/editor/post/${ghostPostId}`;
const ghostPublicUrl = `https://blog.<DOMAIN>/${PETITE_FILLE_SLUG}`;
```

Stocke `<GHOST_POST_ID>`, `<GHOST_ADMIN_EDIT_URL>`, `<GHOST_PUBLIC_URL>`.

---

## Étape 12, Update `cocon.json`

Met à jour la petite-fille concernée dans `cocon.json` :

```typescript
// Trouver la petite-fille dans cocon.filles[].petites_filles[]
const petiteFille = cocon.filles
  .find(f => f.pilier_slug === PILIER_SLUG)
  .petites_filles
  .find(pf => pf.slug === PETITE_FILLE_SLUG);

petiteFille.status = "draft";                    // planned → draft
petiteFille.ghost_id = GHOST_POST_ID;            // ID Ghost pour traçabilité
petiteFille.drafted_at = new Date().toISOString(); // timestamp ISO

// Réécrire cocon.json (formaté)
fs.writeFileSync(
  "cocon.json",
  JSON.stringify(cocon, null, 2),
  "utf-8"
);
```

---

## Étape 13, Rapport final

Affiche un rapport structuré et lisible :

```
✅ Article généré et poussé en DRAFT sur Ghost.

TITRE       : <ARTICLE_SANITIZED.meta_title>
SLUG        : <PETITE_FILLE_SLUG>
PILIER      : <PILIER_SLUG>
STATUS      : draft (à relire et publier à la main)

ADMIN GHOST (relecture) :
  <GHOST_ADMIN_EDIT_URL>

URL PUBLIQUE (quand publié) :
  <GHOST_PUBLIC_URL>

LIENS INTERNES :
  Total       : <LINK_REPORT.total>
  Internes    : <LINK_REPORT.internal>
  Externes    : <LINK_REPORT.external>
  Réécrits    : <LINK_REPORT.rewritten.length> (idéalement 0)
  <Si rewritten > 0, lister chaque pair "original → rewritten">

IMAGE HERO :
  <Si OK : "✅ Générée par fal.ai + uploadée Ghost (URL : <GHOST_IMAGE_URL>)">
  <Si KO : "⚠️ Échec fal.ai, article sans feature_image. Ajoute-en une à la main dans Ghost.">

TOKENS UTILISÉS :
  P1 brief    : in <X> / out <Y> (Sonnet 4.6)
  P2 article  : in <X> / out <Y> (Opus 4.7)

COÛT ESTIMÉ : ~0,15 €
```

---

## Vérifications finales (avant clôture)

Confirme à l'étudiant que chaque ligne de la checklist est cochée :

- [ ] Brief P1 généré et validé par l'utilisateur
- [ ] Article HTML P2 généré
- [ ] Sanitizer cadratin appliqué (tous champs string)
- [ ] Clamp limites Ghost appliqué (custom_excerpt ≤ 297, meta_title ≤ 60, etc.)
- [ ] Link-validation appliquée (whitelist + fallback pilier-parent)
- [ ] Image hero attachée (ou warning explicite si échec fal.ai)
- [ ] Post Ghost créé avec **`status: "draft"`** (jamais `published`)
- [ ] `cocon.json` mis à jour (`status: "draft"`, `ghost_id`, `drafted_at`)
- [ ] Rapport final affiché

Si l'une des cases ne peut pas être cochée : signale clairement à l'étudiant ce qui n'a pas marché et propose une remédiation (relancer P2, regen image, retry Ghost upload).

---

## Clôture

Termine par ce message :

> « Ton article est en draft sur Ghost : <GHOST_ADMIN_EDIT_URL>.
>
> Va le relire dans l'admin Ghost, ajuste si besoin (image hero, ponctuation, paragraphe à reformuler), puis **publie depuis l'admin Ghost en cliquant sur "Publish"**. Le plugin ne publie jamais à ta place, c'est ton garde-fou éditorial.
>
> Prochaine étape :
>
> - `/blog:status` pour voir l'état du cocon (combien d'articles planned / draft / published)
> - `/blog:article` à nouveau pour le suivant
> - Pour faire 3-5 articles d'un coup : `/blog:batch 5` (rate-limit auto à 5/jour) »

---

## Notes d'implémentation

- **`ANTHROPIC_API_KEY`** doit être lue depuis l'env. Si manquante, retourne une erreur claire **avant** d'appeler quoi que ce soit (pas d'appel inutile).
- **Modèles utilisés** : Sonnet 4.6 pour P1 (économie, ~0,02 €), Opus 4.7 pour P2 (qualité long-form, ~0,13 €). Si l'étudiant veut tout en Opus : confirme l'écart de coût (~0,06 € au lieu de 0,02 € pour P1).
- **Structured output** : utilise `output_config.format = { type: "json_schema", schema: ... }` (Anthropic SDK). Plus fiable qu'un simple « retourne du JSON » dans le prompt.
- **Thinking** : `thinking: { type: "disabled" }` pour les deux phases. Structure stricte JSON, pas de gain mesurable et 2-3× plus de tokens.
- **Prompt caching** : pour `/blog:article` seul, l'économie est marginale (~0,03 €). Ça vaut surtout pour `/blog:batch` (système + KB invariants entre articles, économie ~80 % sur le batch).
- **Idempotence** : si l'étudiant relance `/blog:article` sur la même petite-fille déjà en `status: "draft"`, demande s'il veut **régénérer** (écrase l'ancien post Ghost via `posts_edit`) ou **annuler**. Ne crée pas de doublon.
