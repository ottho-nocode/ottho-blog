---
name: link-validation
description: Validation des liens internes dans le HTML d'un article généré. Whitelist d'URLs (mère, piliers, articles publiés, outils, formations). Algorithme regex. Politique externe / ancres / interne. Fallback vers pilier-parent. Utilisée par /blog:article et /blog:batch après le sanitizer.
---

# Skill : link-validation

Cette skill définit le protocole de validation des liens internes après génération d'un article par P2 (le prompt de rédaction). Elle est appliquée juste après le sanitizer HTML, avant la publication dans Ghost.

## Pourquoi valider les liens internes

Même avec un prompt strict qui liste les URLs autorisées, le LLM peut inventer des liens, un slug qui sonne juste mais n'existe pas (`/blog/le-bon-prompt`), un pilier mal orthographié (`/blog/pilier/setups`), une formation qui n'existe pas (`/claude-pro`).

Sans filet de validation, ces liens passent dans Ghost et l'article publié contient des **404** :

- **SEO détérioré** : Google détecte les 404 internes et baisse la note de qualité du site.
- **UX cassée** : le lecteur clique, atterrit sur une page d'erreur, perd confiance, quitte.
- **Maillage interne perdu** : tout le bénéfice du cocon (transmettre le PageRank) disparaît si les liens cassent.

La validation est un **filet de sécurité automatique** : 0 lien cassé qui sort du pipeline, garanti.

---

## Politique de validation (3 catégories)

Pour chaque attribut `href` rencontré dans le HTML, le validator classe le lien dans une de ces 3 catégories :

| Catégorie | Pattern | Action |
|---|---|---|
| **Externe** | `http://`, `https://`, `mailto:`, `tel:` | Laissé tel quel (pas notre rôle de valider l'extérieur) |
| **Ancre** | `#section` (commence par `#`) | Laissé tel quel (lien interne au document) |
| **Interne** | Commence par `/` | Vérifié contre la whitelist → laissé OU réécrit vers le fallback |

---

## Construction de la whitelist d'URLs autorisées

La whitelist est l'ensemble des URLs que le LLM est autorisé à citer. Elle est construite au moment de la validation, à partir de plusieurs sources :

| Source | URLs ajoutées | Origine |
|---|---|---|
| **Mère du cocon** | `/<mere-slug>` (ex. `/apprendre-claude`) | `cocon.json` → `mere.slug` |
| **10 piliers du cocon** | `/blog/pilier/<pilier-slug>` (ex. `/blog/pilier/setup`) | `cocon.json` → `piliers[].slug` |
| **Articles déjà publiés** | `/blog/<slug>` | Fetch Ghost Content API au moment de la validation |
| **Outils du site** | `/outils/<slug>` (ex. `/outils/claude-code`) | `cocon.json` → `linked_outils[]` ou liste statique |
| **Formations CTA** | `/claude-mastery`, `/claude-builders`, `/claude-agent` | Liste statique (pages CTA des piliers) |
| **Pages structurelles** | `/`, `/blog`, `/contact`, `/methode`, `/formations`, `/atelier`, `/avis` | Liste statique |

La whitelist est passée au validator sous forme de `Set<string>` pour une vérification en O(1).

---

## Algorithme regex (pseudocode)

L'implémentation parcourt le HTML une seule fois avec un regex global sur les attributs `href`, et substitue les liens cassés à la volée.

```typescript
// Regex qui capture tous les attributs href avec leurs guillemets
const HREF_REGEX = /href=("|')([^"'>]+?)\1/g;

// Résultat retourné au caller
type ValidationResult = {
  cleaned: string;                                          // HTML corrigé
  totalLinks: number;
  internalLinks: number;
  externalLinks: number;
  rewrittenLinks: Array<{ original: string; rewritten: string }>;
};

function validateLinks(
  html: string,
  allowedUrls: Set<string>,
  fallbackUrl: string,                                      // ex. URL du pilier parent
): ValidationResult {
  // Normaliser la whitelist (strip slash final, query, hash)
  const allowed = new Set<string>();
  for (const u of allowedUrls) allowed.add(normalizeUrl(u));

  const rewrittenLinks: Array<{ original: string; rewritten: string }> = [];
  let total = 0, internal = 0, external = 0;

  const cleaned = html.replace(HREF_REGEX, (match, quote, href) => {
    total++;
    if (isExternal(href) || isAnchor(href)) {
      external++;
      return match;                                         // laissé tel quel
    }
    internal++;
    if (allowed.has(normalizeUrl(href))) return match;      // OK, lien valide
    rewrittenLinks.push({ original: href, rewritten: fallbackUrl });
    return `href=${quote}${fallbackUrl}${quote}`;           // réécrit
  });

  return { cleaned, totalLinks: total, internalLinks: internal,
           externalLinks: external, rewrittenLinks };
}

function normalizeUrl(href: string): string {
  // Strip query string + hash + slash final, pour comparer ce qui compte
  return href.split("?")[0].split("#")[0].replace(/\/$/, "") || "/";
}

function isExternal(href: string): boolean {
  return /^(https?:|mailto:|tel:)/.test(href);
}

function isAnchor(href: string): boolean {
  return href.startsWith("#");
}
```

---

## Fallback URL, choix du remplacement

**Par défaut : la page pilier parent** de l'article validé.

- L'article appartient à un pilier (ex. pilier `setup`). Le fallback est `/blog/pilier/setup`.
- **Pourquoi ce choix** : la pilier-parent est sémantiquement la plus proche de l'article. Si le LLM voulait pointer vers un article cousin du même pilier mais a inventé son URL, rediriger vers le pilier permet au lecteur d'arriver au bon endroit thématique.
- **Perte minimale d'intention** : on ne casse pas le maillage, on le rabat vers le hub thématique le plus pertinent.

**Variantes possibles (déconseillées pour MVP)** :
- Fallback vers la **mère** (`/<mere-slug>`) → trop large, perd la spécificité du pilier.
- Fallback vers une **formation CTA** (`/claude-mastery` etc.) → biais commercial fort, à ne pas mettre par défaut.

---

## Cas particuliers à connaître

| Cas | Exemple | Comportement |
|---|---|---|
| Query string seule | `<a href="?param=x">` | Considéré comme ancre (même page), laissé |
| Protocol-relative | `<a href="//exemple.com">` | Traité comme externe, laissé |
| Href vide | `<a href="">` | Laissé (pas de remplacement, signaler en warning) |
| `javascript:` | `<a href="javascript:alert(1)">` | Blacklisté (XSS), supprimer le href via le sanitizer en amont |
| Slash final | `/blog/pilier/setup/` vs `/blog/pilier/setup` | Normalisés identiques (strip du `/` final) |

Note : le filtrage `javascript:` doit idéalement être fait par le **sanitizer HTML** (étape précédente), pas par le link-validator. Le validator part du principe que le HTML est déjà nettoyé des protocoles dangereux.

---

## Reporting

Le validator retourne un rapport structuré, exploitable par la commande appelante :

```
{
  total: 14,           // nombre total de liens dans le HTML
  internal: 8,         // dont liens internes
  external: 6,         // dont liens externes / ancres
  rewritten: [
    { original: "/blog/article-bidon", rewritten: "/blog/pilier/setup" },
    { original: "/claude-pro",         rewritten: "/blog/pilier/setup" }
  ]
}
```

**Affichage à l'étudiant** à la fin de la génération :

```
Liens vérifiés : 14 (8 internes, 6 externes)
2 liens internes cassés réécrits vers le pilier parent :
  /blog/article-bidon → /blog/pilier/setup
  /claude-pro         → /blog/pilier/setup
```

**Signal d'amélioration** : si `rewritten.length > 0` régulièrement sur les générations, c'est un signal pour ajuster le prompt P2, soit la whitelist n'a pas été bien fournie en entrée, soit le LLM est sous-instruit sur la liste des URLs autorisées.

---

## Source canonique

Une implémentation TypeScript de référence (fichier `lib/link-validator.ts`) est fournie dans le code de démonstration du formateur. Cette skill décrit le contrat ; le code est le contrat.

## Usage dans les commandes du plugin

Cette skill est invoquée par :

- `/blog:article`, après le sanitizer HTML, avant publication Ghost
- `/blog:batch`, appliquée individuellement à chaque article généré dans le lot
