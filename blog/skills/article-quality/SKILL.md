---
name: article-quality
description: System prompts P1 (brief) et P2 (rédaction) pour générer des articles SEO de qualité. Schémas JSON stricts pour les sorties (output_config.format avec json_schema). Sanitizer (cadratin, limites Ghost). Garde-fous factuels (renvoi vers knowledge-base). Status forcé à draft. Utilisée par /blog:article et /blog:batch.
---

# Skill : article-quality

Cette skill packagait les **system prompts P1 et P2** qui pilotent toute la qualité éditoriale du blog Ottho. C'est la skill la plus critique du plugin : c'est elle qui transforme un keyword + contexte cocon en un article SEO publiable. Tout changement ici impacte la totalité du blog.

Les commandes `/blog:article` (un article) et `/blog:batch` (plusieurs en série) chargent cette skill et l'appliquent à chaque génération.

---

## Pipeline éditorial en 2 phases

```
  ┌─────────────────────┐    ┌──────────────┐      ┌──────────────────┐
  │ keyword + cocon ctx │───►│  P1 : BRIEF  │─────►│  P2 : RÉDACTION  │─────► JSON article
  └─────────────────────┘    └──────────────┘      └──────────────────┘            │
                                  Sonnet 4.6           Opus 4.7                    │
                                  ~10 s, ~0,02 €       ~30 s, ~0,13 €              ▼
                                                                          ┌──────────────────┐
                                                                          │  Sanitizer       │
                                                                          │  Link validator  │
                                                                          └──────────────────┘
                                                                                   │
                                                                                   ▼
                                                                          status = draft (Ghost)
```

- **P1 (brief)** : produit un plan d'article structuré (intent, h1, h2, takeaways, ancres) à partir du keyword + linking_context.
- **P2 (rédaction)** : produit l'article HTML complet + tous les méta SEO à partir du brief.
- **Sanitizer + link validator** : post-process local, 0 €.
- **Total** : ~40 secondes, ~0,15 € par article. Sur un cocon de 50 articles : **~7-9 € de coût LLM**.

---

## P1, System prompt (brief)

⚠️ Reproduire **intégralement**. Ce prompt est testé et raffiné, ne pas le réécrire « créativement ».

```
Tu es un stratège SEO francophone expert du cocon sémantique (méthode Laurent Bourrelly). Tu rédiges des briefs d'articles pour le blog d'Ottho, qui forme les entrepreneurs francophones à Claude IA.

Style attendu :
- Direct, sans fioritures, sans "à l'ère du digital".
- Public francophone (FR/CA/BE/CH).
- Pédagogique mais expert. On s'adresse à des praticiens, pas des débutants absolus.
- Les exemples doivent être concrets et exécutables, pas théoriques.

Ponctuation interdite : aucun tiret cadratin (—) dans tout le contenu généré. Remplace-le toujours par une ponctuation adaptée au contexte : deux-points (:) pour une définition ou une explication, virgule pour une incise courte, parenthèses pour une mise à part, point pour scinder en deux phrases. Le tiret demi-cadratin (–) est aussi interdit. Les tirets normaux (-) restent OK pour les mots composés et les listes.

Tu retournes UNIQUEMENT du JSON valide respectant le schéma demandé.
```

### Schéma JSON output P1 (BRIEF_SCHEMA)

À passer dans `output_config.format` de l'appel API Anthropic (format `json_schema`).

```json
{
  "type": "object",
  "properties": {
    "intent_summary": { "type": "string" },
    "promise": { "type": "string" },
    "h1": { "type": "string" },
    "structure": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "h2": { "type": "string" },
          "what": { "type": "string" }
        },
        "required": ["h2", "what"],
        "additionalProperties": false
      }
    },
    "key_takeaways": { "type": "array", "items": { "type": "string" } },
    "internal_links_suggested": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "url": { "type": "string" },
          "anchor_suggestion": { "type": "string" }
        },
        "required": ["url", "anchor_suggestion"],
        "additionalProperties": false
      }
    },
    "word_count_target": { "type": "integer" },
    "tone": { "type": "string" }
  },
  "required": [
    "intent_summary",
    "promise",
    "h1",
    "structure",
    "key_takeaways",
    "internal_links_suggested",
    "word_count_target",
    "tone"
  ],
  "additionalProperties": false
}
```

### User prompt template P1

Le user prompt fournit `keyword`, `linking_context` (mère, pilier parent, sœurs, CTA, outils techniques) et la `persona` cible. Pas de KB en P1, la KB est injectée seulement en P2 où elle protège les chiffres.

---

## P2, System prompt (rédaction)

⚠️ Reproduire **intégralement**.

```
Tu es rédacteur senior pour le blog Ottho. Tu écris des articles SEO de 1500-2000 mots en français, optimisés pour Claude IA et le cocon sémantique.

Règles non négociables :
1. Tu produis du HTML pur (pas de Markdown). Balises autorisées uniquement : h2, h3, p, ul, ol, li, strong, em, code, pre, a, blockquote, hr, table, thead, tbody, tr, th, td.
2. Pas de h1 (le titre est géré séparément par le système).
3. Tous les liens internes utilisent href avec l'URL relative (ex : href="/blog/serveur-mcp-claude").
4. Tu insères 4 à 6 liens internes obligatoires :
   - 1 lien vers la MÈRE dans les 200 premiers mots
   - 2 ou 3 liens vers des SŒURS dans le corps
   - 1 lien CTA final vers la formation
   - 0 à 2 liens vers les pages outils si le sujet le permet
5. Tu utilises le tutoiement ("tu", pas "vous"), sauf indication contraire dans le tone du brief.
6. Tu cites des chiffres concrets, des exemples exécutables, des outils nommés, des prix datés (2026).
7. Tu inclus 1 bloc <pre><code> de code, prompt Claude, ou commande CLI si c'est pertinent au sujet.
8. Tu termines TOUJOURS par un paragraphe qui contient un <a> contextuel vers le CTA principal de la branche.
9. Pas de "Conclusion" comme dernier H2, termine par un H2 plus actionnable type "Passer à la pratique" ou "Et maintenant".

UTILISE DES TABLES quand c'est pertinent (c'est un format puissant pour le SEO et la lisibilité) :
- Comparaisons (X vs Y, options A/B/C, fonctionnalité par plan tarifaire) → <table> obligatoire
- Décisions (matrice persona → recommandation) → <table>
- Listes longues avec colonnes (outil, cas d'usage, prix, lien) → <table>
- Bénéfices vs limitations → <table> à 2 colonnes

CONTRAINTES FACTUELLES (très strictes, la marque Ottho ne tolère aucune erreur factuelle) :
- Tu reçois plus bas une KNOWLEDGE BASE 2026 vérifiée. Tu n'utilises AUCUN chiffre, prix, statistique, version, date, ou nom propre qui ne soit pas dedans.
- Si une info te manque : OMETS-LA ou utilise un conditionnel ("environ", "à partir de", "selon les données 2026", "la majorité de").
- Pour les prix Claude/ChatGPT/Ottho : reprends les valeurs EXACTES de la KB et préfixe par "à partir de" ou "environ".
- N'invente JAMAIS de citation attribuée à une personne. Pas de "comme le dit X" si X n'est pas dans le contexte.
- N'invente JAMAIS d'URL externe. N'utilise que les URLs internes fournies dans le linking_context.
- N'affirme JAMAIS "Claude est le seul à…" ou "aucun autre outil ne…" sans certitude absolue.
- Pour les fonctionnalités produit (Claude Code, MCP, Projects, Artifacts), reprends les descriptions de la KB section ecosystem.

PONCTUATION INTERDITE : aucun tiret cadratin (—) dans le HTML, ni dans les meta SEO (title, description, excerpt, og_*), ni dans les tags. Remplace-le toujours par la ponctuation adaptée au contexte :
- Deux-points (:) pour une définition, une explication ou un exemple introduit
- Virgule (,) pour une incise courte ou une énumération
- Parenthèses (...) pour une mise à part
- Point (.) pour scinder en deux phrases nettes
Le tiret demi-cadratin (–) est aussi interdit. Les tirets normaux (-) restent OK uniquement pour les mots composés (porte-clé, demi-mesure) et les listes <li>.

Format strict des tables : structure <table><thead><tr><th>...</th></tr></thead><tbody><tr><th>label-de-ligne</th><td>val1</td><td>val2</td></tr>...</tbody></table>. La PREMIÈRE colonne du tbody utilise <th> (label de ligne), les autres <td>. Pas de styling inline, le CSS du blog s'occupe du rendu (eyebrow monospace, dernière colonne mise en avant en orange Claude).

Pour les ancres de lien : varie systématiquement, jamais le keyword cible exact (sur-optimisation = pénalité).

Pour les meta SEO (CONTRAINTES STRICTES, Ghost rejette si dépassé) :
- meta_title : max 60 caractères (SEO Google), contient le keyword une seule fois
- meta_description : max 155 caractères (SEO Google), action + bénéfice + chiffre si possible
- custom_excerpt : max 280 caractères (limite Ghost = 300, laisse 20 char de marge), accroche éditoriale qui donne envie de lire
- og_title : max 60 caractères (LinkedIn / Slack tronquent au-delà)
- og_description : max 200 caractères
- feature_image_prompt : prompt EN ANGLAIS pour fal.ai nano-banana-2, format "editorial magazine photograph, Kodak Portra 400 film grain, [sujet spécifique], muted palette, photorealistic"

ATTENTION : compte les caractères, pas les mots. 80 mots = 350-450 caractères = trop long pour custom_excerpt.

Tu retournes UNIQUEMENT du JSON valide respectant le schéma demandé.
```

### Schéma JSON output P2 (ARTICLE_SCHEMA)

À passer dans `output_config.format` de l'appel API Anthropic.

```json
{
  "type": "object",
  "properties": {
    "html": {
      "type": "string",
      "description": "Article complet en HTML (1500-2000 mots), sans balise <h1>, avec 4-6 liens internes obligatoires."
    },
    "meta_title": { "type": "string" },
    "meta_description": { "type": "string" },
    "custom_excerpt": { "type": "string" },
    "og_title": { "type": "string" },
    "og_description": { "type": "string" },
    "tags": { "type": "array", "items": { "type": "string" } },
    "primary_tag": { "type": "string" },
    "feature_image_prompt": { "type": "string" }
  },
  "required": [
    "html",
    "meta_title",
    "meta_description",
    "custom_excerpt",
    "og_title",
    "og_description",
    "tags",
    "primary_tag",
    "feature_image_prompt"
  ],
  "additionalProperties": false
}
```

### User prompt template P2

Le user prompt P2 contient :
1. **KNOWLEDGE BASE 2026** sérialisée, voir skill `knowledge-base`. C'est la garantie factuelle.
2. **BRIEF** sérialisé, sortie de P1.
3. **LIENS_AUTORISES**, whitelist stricte. Toute URL hors whitelist sera silencieusement réécrite par le link validator post-process. Annonce-le dans le prompt : « ne perds pas de tokens à inventer des URLs ».
4. **CONTRAINTES**, word_count_target, tone (du brief), rappel HTML-pas-markdown, rappel pas-de-h1.

---

## Sanitizer côté plugin (post-process)

⚠️ **Filet de sécurité obligatoire**. Même avec la consigne explicite dans P1 et P2, le modèle peut occasionnellement glisser un cadratin. Strip-le systématiquement après parsing du JSON, avant de l'envoyer à Ghost.

### Fonction `stripEmDashes(text)`

Heuristiques contextuelles, à appliquer **dans cet ordre** (ordre = priorité) :

| Pattern entrée | Sortie | Cas typique |
|---|---|---|
| `", "` (espace-cadratin-espace) | `", "` | Incise courte (le plus fréquent) |
| `"— "` (cadratin-espace, début) | `": "` | Introduction de définition |
| `" —"` (espace-cadratin, fin) | `","` | Fin d'incise |
| `"—"` standalone | `":"` | Cas par défaut |
| `"–"` (en-dash) | `"-"` | Tiret demi-cadratin → tiret normal |

Implémentation TypeScript de référence (depuis `app/api/cocon/article/route.ts`) :

```ts
function stripEmDashes(text: string): string {
  return text
    .replace(/, /g, ", ")
    .replace(/— /g, ": ")
    .replace(/ —/g, ",")
    .replace(/—/g, ":")
    .replace(/–/g, "-");
}
```

### Fonction `sanitizeArticle(article)`

Applique `stripEmDashes` sur **tous les champs string** du JSON parsé (html, meta_*, og_*, custom_excerpt, feature_image_prompt) ET sur chaque string des arrays (`tags`).

```ts
function sanitizeArticle<T extends Record<string, unknown>>(article: T): T {
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

### Clamp Ghost (limites caractères)

Ghost rejette les payloads avec `custom_excerpt > 300 caractères`. Le prompt P2 demande `≤ 280` (marge de sécurité), mais on ajoute un filet :

- Si `custom_excerpt.length > 297`, **trim au mot entier le plus proche** sous 297 caractères et **append `…`** (ellipsis simple, pas trois points).
- Idem pour `meta_title > 60`, `meta_description > 155`, `og_title > 60`, `og_description > 200`. Trim en dernier recours.

Pseudo-code :

```ts
function clampToWord(s: string, max: number): string {
  if (s.length <= max) return s;
  const truncated = s.slice(0, max - 1);
  const lastSpace = truncated.lastIndexOf(" ");
  const cut = lastSpace > max * 0.7 ? truncated.slice(0, lastSpace) : truncated;
  return cut + "…";
}
```

---

## Garde-fous factuels (KB obligatoire)

⚠️ **Le prompt P2 doit toujours recevoir une knowledge base** dans son user message. Sans KB, le modèle hallucine des prix, des versions, des stats. Voir skill `knowledge-base` pour le format et le contenu de la KB Ottho 2026.

L'instruction critique du system prompt :

> Tu reçois plus bas une KNOWLEDGE BASE 2026 vérifiée. Tu n'utilises AUCUN chiffre, prix, statistique, version, date, ou nom propre qui ne soit pas dedans.

Si une info manque dans la KB, le modèle a deux issues acceptables :
- **Omettre** le chiffre/fait
- **Conditionner** : « environ X », « à partir de Y », « selon les données 2026 »

Ce qui est **interdit** :
- Inventer un prix précis
- Citer une version produit non listée dans la KB
- Attribuer une citation à une personne réelle
- Inventer une URL externe (les URLs internes sont gérées séparément par la whitelist)

---

## Validation des liens (post-process)

Voir skill `link-validation` pour le détail. En résumé :

- Toute balise `<a href="/...">` interne du HTML généré est extraite et confrontée à une whitelist (`getStaticAllowedUrls()` du cocon + slugs Ghost déjà publiés).
- Toute URL **non-whitelistée** est **réécrite vers la pilier-parent** (l'URL la plus contextuelle pour ne jamais 404).
- Les URLs externes (`http://`, `https://`) ne sont pas réécrites, elles sont seulement comptées et loggées.
- Le HTML nettoyé remplace `article.html` avant push Ghost.

C'est ce qui permet d'écrire dans le prompt P2 : « ne perds pas de tokens à inventer des URLs, le système les supprime ». Le modèle se concentre sur la prose.

---

## ⚠️ Statut Ghost forcé à `draft`

**Non-négociable.** Le plugin n'a **JAMAIS** le droit de publier un article directement. Tous les articles générés sont créés dans Ghost avec `status: "draft"`.

Workflow pédagogique :
1. Le plugin génère l'article (P1 + P2 + sanitizer + link validator).
2. Push Ghost via `mcp__ghost__posts_add` avec `status: "draft"`.
3. **L'étudiant ouvre Ghost**, relit, corrige les détails, ajuste l'image hero.
4. **L'étudiant publie à la main** (bouton "Publish" dans Ghost).

Cette règle est aussi rappelée dans `/blog:article` et `/blog:batch`, mais elle figure ici car c'est une **garantie pédagogique** : on apprend l'étudiant à reviewer son contenu avant publication, jamais à laisser une IA pousser sur le blog public.

Le draft mode protège aussi contre :
- Les hallucinations factuelles qui auraient échappé à la KB
- Les liens internes étranges
- Les meta SEO mal calibrés
- Les images hero hors-marque

---

## Coût estimé par article (prix API 2026)

| Étape | Modèle | Tokens in | Tokens out | Coût |
|---|---|---|---|---|
| P1 brief | Sonnet 4.6 | ~1 500 | ~800 | ~0,01-0,02 € |
| P2 rédaction | Opus 4.7 | ~3 000-5 000 | ~4 000-6 000 | ~0,10-0,15 € |
| Sanitizer | local | 0 | 0 | 0 € |
| Link validator | local | 0 | 0 | 0 € |
| **Total / article** | | | | **~0,13-0,17 €** |

Pour un cocon complet de **50 articles** : **~7-9 € de coût LLM total**.

Si on active le **prompt caching Anthropic** (`cache_control: "ephemeral"` sur le system message + KB qui sont invariants entre articles), l'économie est de l'ordre de **80 %** sur les tokens d'entrée. Sur un batch de 50 articles, ça descend à **~2-3 €**.

---

## Notes d'implémentation

### Modèle réellement utilisé

Dans `app/api/cocon/article/route.ts` du repo de référence, **les deux phases utilisent Opus 4.7** (qualité maximale). Pour le plugin déployé chez les étudiants, on peut :
- **Recommandé** : Sonnet 4.6 pour P1 (économie significative, qualité brief suffisante) + Opus 4.7 pour P2.
- **Budget** : Sonnet 4.6 pour les deux phases (qualité légèrement inférieure mais ~0,03-0,05 € / article).

### Structured output

Anthropic SDK supporte le format `output_config.format = { type: "json_schema", schema: ... }` qui force le modèle à produire un JSON conforme au schéma. Plus fiable qu'un simple « retourne du JSON » dans le prompt. Toujours utiliser cette feature.

### Thinking désactivé

`thinking: { type: "disabled" }` dans l'appel API. Pour la rédaction d'article, le thinking étendu n'apporte pas de gain mesurable et coûte 2-3× plus cher en tokens.

### Caching ANTHROPIC_API_KEY

Le plugin doit lire `ANTHROPIC_API_KEY` depuis l'env (jamais en dur). Si manquante, retourne une erreur claire avant d'appeler quoi que ce soit. Voir skill `knowledge-base` pour le pattern d'env vars.

---

## Lien avec les autres skills du plugin

- **`cocon-method`** : fournit `getLinkingContext(filleSlug, pfSlug)`, la mère, les sœurs, le CTA, les outils techniques. Sans le contexte cocon, P1 ne sait pas où placer l'article.
- **`knowledge-base`** : fournit `KNOWLEDGE_BASE`, la source factuelle de P2. Sans la KB, P2 hallucine.
- **`link-validation`** : post-process le `<a href>` interne. Sans le validator, le modèle peut inventer des URLs 404.
- **`ghost-config`** : push le résultat de P2 dans Ghost en `status: "draft"`.
- **`/blog:article`** (commande) : orchestrateur d'un seul article.
- **`/blog:batch`** (commande) : orchestrateur en série, avec rate-limit (max 5 articles/jour pour ne pas spammer Google).
