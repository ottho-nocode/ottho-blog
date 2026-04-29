---
name: batch
description: Génère N articles du cocon en série (max 5 par batch) avec review human-in-the-loop entre chaque, rate-limit 30s, garde-fous renforcés (KB, link-validation, sanitizer). Statut Ghost forcé draft. Variante batch de /blog:article.
---

# /blog:batch — Génération en série de N articles du cocon (auto-pilote avec HITL)

Tu vas accompagner l'étudiant dans la génération en série de **N articles** (max 5) du cocon, avec une **review human-in-the-loop entre chaque article** : à chaque fois, il valide le brief, puis l'article généré, avant publication en draft Ghost. Entre deux articles, **rate-limit obligatoire de 30 s** (anti-spam API + Ghost). Si la pipeline casse 2 fois de suite, le batch s'arrête tout seul.

C'est la variante « auto-pilote encadré » de `/blog:article`. Tu reprends exactement le même pipeline (P1 brief, P2 article, sanitizer, link-validation, image fal.ai, draft Ghost, update cocon.json), mais répété N fois.

**Charge ces 4 skills** avant tout — identiques à `/blog:article` :

- **`article-quality`** (`/Users/thibaultmarty/ottho-blog-plugin/skills/article-quality/SKILL.md`) — system prompts P1 et P2, schemas JSON `BRIEF_SCHEMA` / `ARTICLE_SCHEMA`, sanitizer cadratin, clamp Ghost. ⚠️ **À reproduire intégralement.**
- **`link-validation`** (`/Users/thibaultmarty/ottho-blog-plugin/skills/link-validation/SKILL.md`) — algorithme de validation des `<a href>` internes, fallback vers la pilier-parent.
- **`knowledge-base`** (`/Users/thibaultmarty/ottho-blog-plugin/skills/knowledge-base/SKILL.md`) — pattern de la KB factuelle injectée dans P2.
- **`cocon-method`** (`/Users/thibaultmarty/ottho-blog-plugin/skills/cocon-method/SKILL.md`) — structure de `cocon.json`, règles de maillage, contexte de linking.

**Temps estimé** : ~5 min par article × N (interactif), soit ~15 min pour 3 articles, ~25 min pour 5.

**Coût LLM estimé** : ~0,15 € par article × N. Avec prompt caching activé (system + KB invariants entre articles), descend autour de **~0,05-0,08 € / article au-delà du premier**, soit ~0,40 € pour un batch de 5.

---

## Garde-fous critiques (à ne JAMAIS contourner)

⚠️ **Statut Ghost FORCÉ à `draft`.** Chaque article du batch est créé en `status: "draft"` sans exception. Le plugin n'a **jamais** le droit de publier. C'est l'étudiant qui ouvre Ghost après le batch et clique « Publish » sur chacun, à la main, après relecture.

⚠️ **Review HITL entre chaque article (non-négociable).** Pour chaque article du batch, l'étudiant valide explicitement le brief P1, puis l'article P2, avant push Ghost. Pas de mode « no-prompt » ou « auto-validate ». La review humaine est ce qui rend ce pipeline pédagogiquement défendable — sans elle, on déléguerait à l'IA toute la responsabilité éditoriale, ce qui est interdit par la marque Ottho.

⚠️ **Rate-limit 30 s entre articles (non-négociable).** `sleep 30` entre chaque article du batch (sauf après le dernier). Raison : éviter le rate-limit Anthropic (5xx en chaîne sur les longs runs), éviter les erreurs Ghost sur upload images successifs, et laisser à l'étudiant le temps de respirer entre deux relectures. Pas de fast-mode.

⚠️ **Abort sur 2 erreurs consécutives (non-négociable).** Si la pipeline (P1, P2, sanitizer, link-validation, image, Ghost, update cocon) casse 2 fois de suite sur 2 articles successifs, le batch s'arrête. On affiche un rapport partiel à l'étudiant et on lui demande quoi faire. Raison : éviter de cramer ton budget Anthropic sur une pipeline cassée (ex. KB malformée, slug Ghost en doublon, MCP down). Une erreur isolée ne stoppe pas le batch — deux d'affilée oui.

⚠️ **Max 5 articles par batch (non-négociable).** L'argument N est clampé entre 1 et 5. Au-delà, on refuse. Raison : protéger ton budget Anthropic (5 × 0,15 € = ~0,75 € max par batch), protéger ta capacité de relecture (5 articles = déjà 1h30 de relecture humaine ensuite), éviter de saturer Google avec un dump SEO en bloc (mauvais signal).

---

## Préambule

Ouvre la commande par ce message :

> « On va générer N articles de ton cocon en série (max 5). Le pipeline pour chaque article :
>
> 1. Sélection auto de la petite-fille suivante (ordre : pilier le plus avancé d'abord)
> 2. P1 brief (~10 s) — **tu valides ou tu ajustes**
> 3. P2 article (~30 s) — **tu valides ou tu ajustes**
> 4. Sanitizer + link-validation + image fal.ai + draft Ghost
> 5. Update `cocon.json`
> 6. Sleep 30 s
> 7. Article suivant
>
> À la fin, tous les articles sont en draft dans Ghost. Tu vas relire et publier chacun à la main, un par un. **Le plugin ne publie jamais directement.**
>
> Si la pipeline casse 2 fois d'affilée, j'arrête le batch et je te dis pourquoi.
>
> Recommandation : avant de lancer un batch, fais au moins **1 fois** `/blog:article` pour t'assurer que tout fonctionne (KB, MCP Ghost, MCP fal.ai, prompts) sur ton setup. Si t'as déjà fait ce test, on y va. »

Si l'étudiant n'a jamais lancé `/blog:article`, **propose-lui de le faire d'abord** :

> « T'as pas encore généré d'article seul. Tu veux d'abord lancer `/blog:article` sur 1 petite-fille pour valider que ton pipeline marche ? Si oui, reviens ici après. Si tu préfères tenter direct le batch, OK, mais on s'arrête au premier vrai blocage. »

---

## Pré-requis (à valider AVANT de commencer)

Vérifie les 4 pré-requis ci-dessous (identiques à `/blog:article`). Si l'un manque, **mets en pause** et redirige.

### Pré-requis 1 — `cocon.json` existe

> « Je vérifie que `cocon.json` existe à la racine du projet. »

Si absent : pause et redirige :

> « Pas de cocon. Lance d'abord `/blog:cocon`. Reviens ici quand c'est fait. »

### Pré-requis 2 — Ghost en place + MCP Ghost dispo

> « Je vérifie que ton instance Ghost et le MCP Ghost sont opérationnels. »

Si absent : pause et redirige :

> « Pas de Ghost. Lance `/blog:setup-ghost`. Reviens ici quand l'admin Ghost s'ouvre dans ton navigateur. »

### Pré-requis 3 — Theme Ghost actif

> « Je vérifie que ton theme Ghost est actif. »

Si absent : pause et redirige :

> « Lance `/blog:theme` avant de lancer un batch (sinon tes drafts seront illisibles dans le rendu Ghost). »

### Pré-requis 4 — MCP fal.ai dispo

> « Je vérifie que le MCP fal.ai est configuré. »

Si absent : warning **mais on continue** :

> « Pas de MCP fal.ai détecté. Le batch va tourner en mode dégradé : tous les articles seront publiés sans `feature_image`. Tu pourras les ajouter à la main dans Ghost après. Continuer ? »

### Pré-requis 5 (spécifique batch) — `knowledge.json` existe

> « Je vérifie que `knowledge.json` existe (KB factuelle obligatoire pour P2). »

Si absent : ⚠️ **stop net**, le batch ne démarre pas :

> « Pas de KB. Sans elle, P2 hallucine prix, versions, stats — la marque Ottho ne tolère pas ça sur 5 articles d'un coup. Lance `/blog:article` une fois (qui te propose de générer la KB), valide-la, et reviens lancer le batch. »

---

## Étape 1 — Capturer l'argument N

L'argument N est passé en argument de la commande (`/blog:batch 3`, `/blog:batch 5`, ou `/blog:batch` tout court).

### 1.1 — Si N est passé en argument

Parse l'argument. Vérifie que c'est un entier entre 1 et 5 inclus.

- Si `N < 1` ou `N > 5` ou `N` n'est pas un entier : refuse.

  > « N invalide. Donne-moi un entier entre 1 et 5. ⚠️ **Max 5 par batch — non-négociable** (protection budget + relecture). Pour plus, lance plusieurs batchs. »

### 1.2 — Si N est absent

Pose la question :

> « Combien d'articles veux-tu générer dans ce batch ? Donne-moi un entier entre 1 et 5 (default : 3). »

Capture la réponse. Si l'étudiant tape `default` ou rien : `N = 3`. Sinon valide comme en 1.1.

Stocke `<N>`.

---

## Étape 2 — Lister et confirmer les N articles à écrire

### 2.1 — Sélection automatique

Lis `cocon.json`. Filtre toutes les petites-filles avec `status: "planned"` (uniquement — pas les `draft`, qui ont déjà été générés).

Ordonne par **pilier le plus avancé d'abord**. Heuristique simple :

- Pour chaque pilier (fille), compte le nombre de petites-filles déjà en `published` ou `draft`.
- Le pilier avec le score le plus élevé passe en premier.
- Égalité : ordre alphabétique du `pilier_slug`.
- À l'intérieur d'un pilier, prends les petites-filles dans l'ordre du fichier `cocon.json`.

Prends les **N premières** de cette liste ordonnée.

⚠️ **Si moins de N petites-filles sont `planned`** : warning et propose à l'étudiant :

> « Il ne reste que {K} petites-filles `planned` dans ton cocon (tu as demandé {N}). Je peux :
>
> - lancer un batch sur les {K} restantes
> - annuler et te suggérer de relancer `/blog:cocon` pour ajouter de nouvelles petites-filles
>
> Que préfères-tu ? »

### 2.2 — Affichage de la liste + confirmation

Affiche la liste compacte des N articles sélectionnés :

```
━━━ Batch de N articles à générer ━━━

[1/N] Pilier : pipeline-blog
       Slug  : cocon-semantique-ia
       Title : Cocon sémantique avec l'IA : la méthode pas-à-pas
       KW    : cocon sémantique ia
       Persona : entrepreneur
       CTA   : /claude-mastery

[2/N] Pilier : pipeline-blog
       Slug  : blog-headless-ghost-nextjs
       Title : Blog headless avec Ghost, Next.js et Claude
       KW    : blog headless ghost
       Persona : entrepreneur
       CTA   : /claude-mastery

[3/N] Pilier : creer-site-web-claude
       ...

━━━ Coût estimé : ~0,{15 × N} € — Temps estimé : ~5 min × N en interactif ━━━
```

Pose la question :

> « C'est cette sélection que tu veux générer ? Tape :
>
> - `confirmer` pour démarrer le batch
> - `modifier` pour échanger un article contre un autre `planned` du cocon
> - `annuler` pour sortir »

Si `modifier` : propose la liste complète des `planned` restants, demande quel article remplacer (`[2/N]` → quel autre slug). Reboucle sur l'affichage.

Si `annuler` : sortie propre.

Si `confirmer` : passe à l'étape 3.

Stocke la liste sélectionnée comme `<BATCH_LIST>` (array de `{ pilier_slug, slug, title, keyword, persona, cta_principal, intent }`).

---

## Étape 3 — Initialiser les compteurs du batch

Avant la boucle, initialise les compteurs qui serviront au rapport final :

```typescript
const batchStats = {
  succeeded: [],          // [{ slug, ghostId, ghostAdminUrl, ghostPublicUrl, linkReport, tokensP1, tokensP2 }]
  skipped: [],            // [{ slug, reason }] — étudiant a tapé "skip" sur brief ou article
  errors: [],             // [{ slug, phase, message }] — erreur pipeline (sanitizer, validator, fal.ai, ghost, ...)
  totalTokens: { p1In: 0, p1Out: 0, p2In: 0, p2Out: 0 },
  totalRewrittenLinks: 0,
  consecutiveErrors: 0,   // ⚠️ compteur d'abort
  abortedAt: null         // index de l'article où on a aborté, null si batch complet
};
```

Stocke `<BATCH_STATS>`. Ce dict est mis à jour à chaque itération.

---

## Étape 4 — Boucle de génération (i = 1 à N)

Pour chaque article `i` de 1 à N, exécute le sous-pipeline complet ci-dessous. Capture le `<CURRENT>` = `BATCH_LIST[i-1]`.

### 4.0 — En-tête article

Affiche un séparateur clair :

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Article {i}/{N} : {CURRENT.title}
  Slug : {CURRENT.slug}  |  Pilier : {CURRENT.pilier_slug}  |  KW : {CURRENT.keyword}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### 4.1 — Construire le `linking_context` (idem `/blog:article` étape 3)

Identifie depuis `cocon.json` :

- **Mère** : `cocon.json` → `mere.slug` → URL `/<mere-slug>`
- **Pilier-parent** : la fille à laquelle appartient `CURRENT`
- **2-3 piliers-sœurs** : autres filles thématiquement proches
- **Outils linkés** : `linked_outils[]` du pilier-parent
- **CTA principal** : `cta_principal` du pilier-parent

Récupère les slugs Ghost déjà publiés via `mcp__ghost__posts_browse` (filter `status:published`, fields `slug`).

Construis la whitelist `<ALLOWED_URLS>` selon la skill `link-validation` (mère, 10 piliers, articles publiés, outils, formations CTA, pages structurelles).

### 4.2 — Phase A : Générer le brief (P1)

#### 4.2.1 — Charger le system prompt P1

Charge **intégralement** le system prompt P1 depuis la skill `article-quality` (section « P1 — System prompt (brief) »). ⚠️ Ne le réécris pas, copie-le tel quel.

#### 4.2.2 — Construire le user message P1

```
KEYWORD CIBLE : <CURRENT.keyword>
TITRE PROPOSÉ : <CURRENT.title>
WORD COUNT TARGET : 1500

CONTEXTE COCON :
- Mère : <MERE_SLUG> (URL : /<MERE_SLUG>) — keyword "<MERE_KEYWORD>"
- Pilier-parent : <PILIER_SLUG> (URL : /blog/pilier/<PILIER_SLUG>) — titre "<PILIER_TITLE>"
- Piliers-sœurs : <SISTERS_LIST_avec_URLs>
- Outils linkés : <LINKED_OUTILS_avec_URLs>
- CTA principal : <CTA_PRINCIPAL>

PERSONA CIBLE : <CURRENT.persona>
INTENT : <CURRENT.intent>
```

#### 4.2.3 — Appel API Anthropic (avec prompt caching pour le batch)

```typescript
try {
  const briefResponse = await anthropic.messages.create({
    model: "claude-sonnet-4-6",     // P1 = Sonnet 4.6, qualité brief suffisante
    max_tokens: 2000,
    system: [
      {
        type: "text",
        text: P1_SYSTEM_PROMPT,
        cache_control: { type: "ephemeral" }   // ⚠️ caching system prompt sur batch
      }
    ],
    messages: [{ role: "user", content: P1_USER_MESSAGE }],
    output_config: {
      format: { type: "json_schema", schema: BRIEF_SCHEMA }
    },
    thinking: { type: "disabled" }
  });

  const brief = JSON.parse(briefResponse.content[0].text);
  batchStats.totalTokens.p1In += briefResponse.usage.input_tokens;
  batchStats.totalTokens.p1Out += briefResponse.usage.output_tokens;
} catch (err) {
  // Voir 4.5 — gestion erreur consécutive
  handlePipelineError(i, "P1_brief", err.message);
  continue;   // skip à l'article suivant si pas d'abort
}
```

**Note prompt caching** : le system prompt P1 (~2K tokens) est invariant entre articles du batch. Avec `cache_control: { type: "ephemeral" }`, économie ~80 % sur les tokens d'entrée à partir du 2e article. Sur un batch de 5, économie estimée ~0,30 € total.

#### 4.2.4 — Affichage du brief + validation HITL

Affiche le brief de manière compacte :

```
┌─ BRIEF P1 ─────────────────────────────────────────────
│ TITRE H1   : <h1 du brief>
│ PROMESSE   : <promise>
│ INTENT     : <intent_summary>
│ TON        : <tone>
│ WORD COUNT : <word_count_target>
│
│ PLAN H2 :
│   1. <h2_1> — <what_1>
│   2. <h2_2> — <what_2>
│   ...
│
│ KEY TAKEAWAYS :
│   • <takeaway_1>
│   • <takeaway_2>
│   ...
│
│ ANCRES SUGGÉRÉES :
│   → <url_1> (anchor: "<anchor_1>")
│   → <url_2> (anchor: "<anchor_2>")
│   ...
└────────────────────────────────────────────────────────
```

Pose la question :

> « Article {i}/{N} — Le brief est OK ? Tape :
>
> - `accepter` pour passer à la rédaction P2
> - `modifier` pour ajuster manuellement (titre, plan, ton)
> - `regenerer` pour relancer P1 avec une instruction supplémentaire (ex. "plus orienté pratique")
> - `skip` pour passer à l'article suivant (cet article reste `planned` dans cocon.json) »

⚠️ **Attendre la réponse de l'étudiant.** Pas de timeout, pas d'auto-validate.

Capture le choix :

- **`accepter`** → continue avec phase B
- **`modifier`** → demande quel champ, applique l'édit manuelle, ré-affiche, redemande
- **`regenerer`** → demande l'instruction additionnelle, ré-appelle P1 avec cette instruction injectée, retour à l'affichage
- **`skip`** → ajoute à `batchStats.skipped` (`{ slug, reason: "skipped_at_brief" }`), incrémente `i`, reset `consecutiveErrors` à 0, **passe au sleep et à l'article suivant** (sans incrémenter le compteur de succès)

Stocke le brief final comme `<BRIEF>`.

### 4.3 — Phase B : Générer l'article (P2)

#### 4.3.1 — Charger le system prompt P2

Charge **intégralement** le system prompt P2 depuis la skill `article-quality` (section « P2 — System prompt (rédaction) »). ⚠️ Reproduis-le tel quel.

#### 4.3.2 — Construire le user message P2

L'ordre des sections compte (KB **avant** brief) :

```
KNOWLEDGE BASE 2026 (validée le <validated_at>) — utilise EXCLUSIVEMENT ces faits pour tout chiffre, prix, statistique, version. Si une info manque : OMETS-LA ou utilise un conditionnel ('environ', 'à partir de', 'selon les données disponibles').

<JSON.stringify(knowledge_base, null, 2)>

---

BRIEF DE L'ARTICLE :
<JSON.stringify(BRIEF, null, 2)>

---

LIENS_AUTORISES (whitelist stricte — ne perds pas de tokens à inventer des URLs, le système les supprime hors whitelist) :
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

#### 4.3.3 — Appel API Anthropic

```typescript
try {
  const articleResponse = await anthropic.messages.create({
    model: "claude-opus-4-7",         // P2 = Opus 4.7 (qualité long-form)
    max_tokens: 8000,
    system: [
      {
        type: "text",
        text: P2_SYSTEM_PROMPT,
        cache_control: { type: "ephemeral" }   // ⚠️ caching system prompt
      }
    ],
    messages: [{
      role: "user",
      content: [
        {
          type: "text",
          text: KB_SECTION,                     // KB sérialisée
          cache_control: { type: "ephemeral" } // ⚠️ caching KB (invariant batch)
        },
        {
          type: "text",
          text: BRIEF_AND_CONSTRAINTS_SECTION   // brief + whitelist + contraintes (variable par article)
        }
      ]
    }],
    output_config: {
      format: { type: "json_schema", schema: ARTICLE_SCHEMA }
    },
    thinking: { type: "disabled" }
  });

  const article = JSON.parse(articleResponse.content[0].text);
  batchStats.totalTokens.p2In += articleResponse.usage.input_tokens;
  batchStats.totalTokens.p2Out += articleResponse.usage.output_tokens;
} catch (err) {
  handlePipelineError(i, "P2_article", err.message);
  continue;
}
```

**Note prompt caching P2** : `system` (~3K tokens) + `KB` (~2K tokens) = ~5K tokens invariants. Cachés sur la durée du batch via `cache_control: { type: "ephemeral" }`. Économie ~80 % sur les tokens d'entrée à partir du 2e article. Sur un batch de 5, économie estimée ~1,50 € sur P2.

Stocke comme `<ARTICLE>` (pré-sanitizer).

#### 4.3.4 — Affichage de l'article + validation HITL

Affiche un résumé compact (pas tout le HTML, pour ne pas saturer la console) :

```
┌─ ARTICLE P2 ───────────────────────────────────────────
│ META TITLE     : <meta_title>     ({len} / 60 chars)
│ META DESC      : <meta_description> ({len} / 155 chars)
│ EXCERPT        : <custom_excerpt>   ({len} / 297 chars)
│ TAGS           : [<tag1>, <tag2>, ...] (primary: <primary_tag>)
│ FEATURE PROMPT : <feature_image_prompt>
│
│ STRUCTURE H2 (extraite du HTML) :
│   1. <h2_extrait_1>
│   2. <h2_extrait_2>
│   ...
│
│ PREMIER PARAGRAPHE :
│   <html_strip → premier <p>, max 300 chars + "...">
│
│ STATS HTML : ~<word_count> mots — <a_count> liens <a> — <table_count> tables — <code_count> blocs <pre>
└────────────────────────────────────────────────────────
```

Pose la question :

> « Article {i}/{N} — Le rendu te convient ? Tape :
>
> - `accepter` pour passer au push Ghost (sanitizer + link-validation + image + draft)
> - `voir` pour afficher le HTML complet (uniquement si tu en as besoin — c'est long)
> - `modifier` pour éditer manuellement un champ (meta_title, meta_description, etc.)
> - `regenerer` pour relancer P2 avec une instruction supplémentaire
> - `skip` pour abandonner cet article (perdu — il restera `planned` dans cocon.json) »

⚠️ **Attendre la réponse.** Pas d'auto-validate.

Capture le choix :

- **`accepter`** → continue avec phase C
- **`voir`** → affiche le HTML complet entre `\`\`\`html ... \`\`\``, redemande
- **`modifier`** → demande quel champ, applique l'édit, ré-affiche le résumé, redemande. Si `html` modifié : ré-applique sanitizer + link-validation après.
- **`regenerer`** → demande l'instruction, ré-appelle P2 avec cette instruction injectée
- **`skip`** → ajoute à `batchStats.skipped` (`{ slug, reason: "skipped_at_article" }`), warning : « article perdu (P2 a coûté ~0,13 €) ». Incrémente `i`, reset `consecutiveErrors`, passe au sleep et à l'article suivant.

Stocke comme `<ARTICLE>` final.

### 4.4 — Phase C : Pipeline post-process et push Ghost (sans HITL)

À partir d'ici, plus de question à l'étudiant — c'est de la mécanique automatique. Si une étape échoue, voir 4.5.

#### 4.4.1 — Sanitizer (idem `/blog:article` étape 7)

Applique `sanitizeArticle(<ARTICLE>)` depuis la skill `article-quality` :

- `stripEmDashes` sur tous les champs string (`html`, `meta_title`, `meta_description`, `custom_excerpt`, `og_title`, `og_description`, `feature_image_prompt`, chaque `tags[]`)
- `clampToWord` pour respecter les limites Ghost : `meta_title ≤ 60`, `meta_description ≤ 155`, `custom_excerpt ≤ 297`, `og_title ≤ 60`, `og_description ≤ 200`

Stocke comme `<ARTICLE_SANITIZED>`.

#### 4.4.2 — Link-validation ⚠️

```typescript
const result = validateLinks(
  ARTICLE_SANITIZED.html,
  ALLOWED_URLS,
  `/blog/pilier/${CURRENT.pilier_slug}`   // fallback = pilier-parent
);

ARTICLE_SANITIZED.html = result.cleaned;
const linkReport = {
  total: result.totalLinks,
  internal: result.internalLinks,
  external: result.externalLinks,
  rewritten: result.rewrittenLinks
};

batchStats.totalRewrittenLinks += linkReport.rewritten.length;
```

#### 4.4.3 — Image hero (fal.ai) — mode dégradé acceptable

```typescript
let ghostImageUrl = null;
if (FAL_MCP_AVAILABLE) {
  try {
    const falResult = await mcp.fal.run({
      endpoint: "fal-ai/nano-banana-2",
      input: {
        prompt: ARTICLE_SANITIZED.feature_image_prompt,
        aspect_ratio: "16:9",
        resolution: "2K",
        output_format: "jpeg",
        num_images: 1
      }
    });
    const imageBlob = await fetch(falResult.images[0].url).then(r => r.blob());
    const ghostImage = await mcp.ghost.images_upload({ file: imageBlob, purpose: "image" });
    ghostImageUrl = ghostImage.url;
  } catch (err) {
    console.warn(`⚠️ Image fal.ai/upload Ghost failed pour ${CURRENT.slug} : ${err.message}`);
    // PAS d'incrément consecutiveErrors — image en échec est dégradation acceptable, pas erreur pipeline
  }
}
```

#### 4.4.4 — Push Ghost en draft ⚠️

```typescript
try {
  const post = await mcp.ghost.posts_add({
    source: "html",
    posts: [{
      title: ARTICLE_SANITIZED.meta_title,
      slug: CURRENT.slug,
      html: ARTICLE_SANITIZED.html,
      status: "draft",                        // ⚠️ FORCÉ — JAMAIS published
      feature_image: ghostImageUrl,           // null si fal.ai a échoué
      tags: [
        { name: ARTICLE_SANITIZED.primary_tag },
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
  const ghostPublicUrl = `https://blog.<DOMAIN>/${CURRENT.slug}`;
} catch (err) {
  handlePipelineError(i, "Ghost_push", err.message);
  continue;
}
```

#### 4.4.5 — Update `cocon.json`

```typescript
const petiteFille = cocon.filles
  .find(f => f.pilier_slug === CURRENT.pilier_slug)
  .petites_filles
  .find(pf => pf.slug === CURRENT.slug);

petiteFille.status = "draft";
petiteFille.ghost_id = ghostPostId;
petiteFille.drafted_at = new Date().toISOString();

fs.writeFileSync("cocon.json", JSON.stringify(cocon, null, 2), "utf-8");
```

#### 4.4.6 — Succès article : update batchStats et reset compteur

```typescript
batchStats.succeeded.push({
  slug: CURRENT.slug,
  ghostId: ghostPostId,
  ghostAdminUrl: ghostAdminEditUrl,
  ghostPublicUrl: ghostPublicUrl,
  linkReport,
  imageOk: ghostImageUrl !== null,
});
batchStats.consecutiveErrors = 0;   // ⚠️ reset compteur d'abort
```

Affiche une confirmation courte :

```
✅ Article {i}/{N} OK — draft Ghost : {ghostAdminEditUrl}
   Liens : {linkReport.total} ({linkReport.internal} internes, {linkReport.rewritten.length} réécrits)
   Image : {imageOk ? "OK" : "KO (sans feature_image)"}
```

### 4.5 — Gestion des erreurs pipeline et abort

Fonction `handlePipelineError(i, phase, message)` appelée à chaque catch :

```typescript
function handlePipelineError(i, phase, message) {
  batchStats.errors.push({
    slug: BATCH_LIST[i-1].slug,
    phase,                                 // "P1_brief" | "P2_article" | "Ghost_push" | ...
    message
  });
  batchStats.consecutiveErrors += 1;

  console.error(`❌ Article ${i}/${N} — Erreur ${phase} : ${message}`);

  if (batchStats.consecutiveErrors >= 2) {
    // ⚠️ ABORT — non-négociable
    batchStats.abortedAt = i;
    console.error(`
⚠️  ABORT du batch après 2 erreurs consécutives.

Dernière erreur     : ${phase} sur ${BATCH_LIST[i-1].slug}
Avant-dernière      : ${batchStats.errors[batchStats.errors.length-2].phase} sur ${batchStats.errors[batchStats.errors.length-2].slug}

Articles déjà OK    : ${batchStats.succeeded.length}/${N}
Articles skippés    : ${batchStats.skipped.length}
Articles à recommencer plus tard : ${N - i}

Causes possibles :
  - ANTHROPIC_API_KEY invalide ou rate-limit
  - MCP Ghost down ou theme cassé
  - KB malformée (cocon.json ou knowledge.json corrompus)
  - Slug Ghost en doublon (déjà publié dans une autre branche)

Va jeter un œil dans Ghost admin pour les drafts déjà créés (${batchStats.succeeded.length}).
Corrige le problème puis relance \`/blog:batch\` sur les ${N - i} restants.
    `);
    // Sortir de la boucle for, passer directement à l'étape 5 (rapport final)
    break;
  }
}
```

### 4.6 — Rate-limit 30 s entre articles ⚠️

À la fin de l'itération `i` (succès, skip, ou erreur isolée), **sauf si c'est le dernier (`i === N`) ou abort** :

```typescript
console.log(`\n⏳ Sleep 30 s avant article suivant... (anti rate-limit + temps de respiration)\n`);
await new Promise(r => setTimeout(r, 30_000));
```

⚠️ **Non-négociable.** Pas de skip-sleep, pas de fast-mode.

---

## Étape 5 — Rapport final du batch

### 5.1 — Affichage console

À la fin de la boucle (succès complet ou abort), affiche un rapport structuré :

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  RAPPORT BATCH — N = <N>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Statut : <BATCH_STATS.abortedAt ? `ABORTED à l'article ${BATCH_STATS.abortedAt}` : `COMPLET`>

✅ Articles publiés en draft Ghost : <BATCH_STATS.succeeded.length>
   <pour chaque succeeded :>
   - <slug> → <ghostAdminUrl>
     Liens : <linkReport.total> total, <linkReport.rewritten.length> réécrits
     Image hero : <imageOk ? "OK" : "❌ KO (à ajouter à la main)">

⏭️  Articles skippés par toi : <BATCH_STATS.skipped.length>
   <pour chaque skipped :>
   - <slug> (reason: <reason>)

❌ Articles en erreur pipeline : <BATCH_STATS.errors.length>
   <pour chaque error :>
   - <slug> — phase <phase> — <message>

━━━ Tokens consommés ━━━

P1 (Sonnet 4.6) : <p1In> in / <p1Out> out
P2 (Opus 4.7)   : <p2In> in / <p2Out> out

━━━ Coût estimé ━━━

LLM           : ~<0,15 × succeeded.length> €
fal.ai images : ~<0,02 × succeeded.length> € (mode normal) ou 0 € (mode dégradé)
TOTAL         : ~<somme> €

━━━ Liens internes ━━━

Total réécrits par link-validator : <BATCH_STATS.totalRewrittenLinks>
   (idéalement 0 — si > 5 sur le batch, ajuste la whitelist du prompt P2)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### 5.2 — Écriture du fichier de rapport

Écris un fichier markdown à la racine du projet utilisateur :

**Path** : `batch-{ISO_DATE}-report.md` (ex. `batch-2026-04-29T14-32-15-report.md`)

**Contenu** :

```markdown
# Rapport batch /blog:batch — {ISO_DATETIME}

**Statut** : {COMPLET | ABORTED}
**N demandé** : {N}
**N traité** : {succeeded.length + skipped.length + errors.length}

## Articles publiés en draft Ghost

{tableau markdown : slug | ghostAdminUrl | ghostPublicUrl | imageOk | linkReport}

## Articles skippés

{liste : slug — reason}

## Erreurs pipeline

{liste : slug — phase — message}

## Stats tokens

| Phase | In | Out |
|---|---|---|
| P1 (Sonnet 4.6) | {p1In} | {p1Out} |
| P2 (Opus 4.7) | {p2In} | {p2Out} |

## Coût estimé

- LLM : ~{0,15 × succeeded.length} €
- fal.ai images : ~{0,02 × succeeded.length} €
- **Total** : ~{somme} €

## Liens internes

Total réécrits par link-validator : {totalRewrittenLinks}

## Prochaines actions

- Aller dans Ghost admin pour relire les {succeeded.length} drafts
- Publier 1 par 1 après relecture (~15-20 min par article)
- {if errors > 0 : "Corriger les erreurs pipeline puis relancer /blog:batch sur les articles en erreur"}
```

Stocke le path et affiche-le à l'étudiant :

```
📄 Rapport écrit : ./batch-{ISO_DATE}-report.md
```

---

## Vérifications finales

Confirme à l'étudiant que chaque ligne de la checklist est cochée :

- [ ] N articles traités (ou batch aborté avec compteur explicite)
- [ ] Chaque succès a `status: "draft"` dans Ghost (jamais `published`)
- [ ] `cocon.json` mis à jour (`status: "draft"`, `ghost_id`, `drafted_at`) pour chaque succès
- [ ] Sanitizer cadratin appliqué sur chaque article publié
- [ ] Link-validation appliquée sur chaque article publié
- [ ] Rapport batch écrit (`batch-{date}-report.md`)
- [ ] Tous les drafts visibles dans Ghost admin (l'étudiant peut vérifier via le lien admin)
- [ ] Si abort : raison expliquée à l'étudiant et reste à reprocesser identifié

---

## Clôture

Termine par ce message :

> « Batch terminé. **{X} articles en draft Ghost** à relire.
>
> Va dans Ghost admin pour finaliser et publier (1 par 1, après relecture) :
>
> {liste des admin URLs des succès}
>
> ⚠️ **Recommandation forte** : 15-20 min de relecture par article. Le batch te fait gagner les ~30 min de génération par article, pas la relecture. Si tu publies sans relire, tu prends le risque d'une erreur factuelle ou d'un lien étrange en ligne.
>
> Checklist relecture par article :
> 1. Lis l'intro — la promesse est claire ?
> 2. Vérifie chaque chiffre/prix/version (la KB peut avoir des trous)
> 3. Clique chaque lien interne — pas de 404 (le link-validator a fait le gros, mais re-vérifier)
> 4. Vérifie l'image hero — elle colle au sujet ?
> 5. Lis le H2 final — il pousse bien vers le CTA ?
> 6. Click "Publish" dans Ghost une fois que tu es OK.
>
> Prochain step :
>
> - `/blog:status` pour voir l'état du cocon (combien `planned`, `draft`, `published`)
> - `/blog:seo-audit` pour vérifier la qualité SEO des derniers publiés
> - `/blog:batch <N>` pour le prochain lot quand t'as fini de relire
> - Si erreur récurrente sur la pipeline : `/blog:article` (mode 1-par-1) pour debug avant de relancer un batch »

---

## Notes d'implémentation

### Prompt caching — pourquoi c'est crucial pour le batch

Sur `/blog:article` seul, le caching est marginal. Sur `/blog:batch` avec N=5, c'est **massif** :

| Élément caché | Tokens | Présent dans tous les articles ? |
|---|---|---|
| System prompt P1 | ~1 500 | Oui |
| System prompt P2 | ~3 000 | Oui |
| KB sérialisée (P2 user msg) | ~2 000 | Oui |
| Linking context (P1 + P2) | ~500 | Variable (whitelist + mère + sœurs partagées) |

Total ~6,5K tokens cachés × 4 articles cache-hits (à partir du 2e) × prix d'entrée Opus 4.7 (~3 €/M tokens) × 0,9 (économie 90 % du tarif input sur cache hit) = **~0,07 € par article économisé**, soit **~0,30 € sur un batch de 5**.

⚠️ **Important** : `cache_control` doit être placé sur les blocs **invariants** uniquement. Les sections variables par article (`BRIEF`, `CONTRAINTES` spécifiques, `PERSONA`) sont en dernière position, sans `cache_control`. C'est ce qui permet au cache de fonctionner.

### Persistance entre sleeps

Le `await sleep(30000)` ne doit **pas** réinitialiser l'état du batch. Si la commande tourne dans une session unique (cas standard pour un slash command Claude Code), c'est automatique. Si pour une raison ou une autre la session redémarre pendant le sleep : le batch reprend depuis le début (pas de checkpointing implémenté en MVP — accepter cette limite).

### Idempotence — relancer un batch après abort

Si l'étudiant relance `/blog:batch <N>` après un abort, l'étape 2.1 (sélection auto) filtre uniquement les `status: "planned"`. Les articles qui ont réussi pendant l'abort sont déjà passés à `draft` dans `cocon.json`, donc ils sont automatiquement exclus de la sélection. Pas de doublon.

### Comportement si l'étudiant interrompt (Ctrl+C / quit)

Si l'étudiant interrompt manuellement la commande pendant la boucle :

- Les articles déjà passés à `draft` sont conservés dans Ghost et `cocon.json` est à jour pour eux
- L'article en cours de traitement (P1 ou P2) est perdu (rollback automatique : pas d'écriture si pas de succès final)
- Pas de rapport écrit (la commande s'arrête avant)

Recommandation : laisser tourner les batchs jusqu'au bout (max 25 min interactif pour N=5). Ne pas interrompre.

### `ANTHROPIC_API_KEY` et `FAL_KEY`

Lus depuis l'env. Si `ANTHROPIC_API_KEY` manquante : **stop net** avant la boucle, erreur claire. Si `FAL_KEY` manquante : warning mode dégradé, on continue sans images (idem `/blog:article`).

### Modèles utilisés

- **P1** : Sonnet 4.6 (économie, qualité brief suffisante)
- **P2** : Opus 4.7 (qualité long-form indispensable sur du SEO 1500-2000 mots)

L'étudiant peut basculer P2 sur Sonnet 4.6 pour économiser ~0,10 € par article, mais qualité éditoriale dégradée — déconseillé sur le batch (l'effet se cumule).

### Pourquoi pas plus de 5 par batch ?

- **Budget** : 5 × 0,15 € = 0,75 € — perte acceptable si pipeline cassée. Au-delà (10+ articles), le coût d'un abort devient lourd.
- **Relecture humaine** : 5 × 20 min = 1h40 de relecture avant publication. Au-delà, l'étudiant publie en mode automatique = perte de qualité.
- **SEO** : Google Search Console détecte les dumps de contenu (10+ articles publiés en 1h). Mauvais signal de fraîcheur. Mieux vaut espacer.
- **Charge cognitive** : 5 dialogues HITL d'affilée = limite haute avant fatigue de validation. Au-delà, l'étudiant valide en mode robot.

Si l'étudiant insiste pour 10+ : refuse poliment, propose 2 batchs de 5 espacés d'au moins 24h.
