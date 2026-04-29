---
name: knowledge-base
description: Pattern de knowledge base factuelle pour éviter les hallucinations dans la rédaction d'articles (prix, versions, statistiques, infos entreprise). Structure de la KB. Injection dans le prompt P2. Maintenance (versioning, validated_at, fréquence). Utilisée par /blog:article et /blog:batch.
---

# Skill : knowledge-base

Cette skill définit le **pattern de knowledge base factuelle** : un fichier de vérité injecté dans le prompt de rédaction P2 pour empêcher Claude d'inventer des chiffres, prix, versions ou citations. C'est ton filet anti-hallucination factuelle, utilisé par `/blog:article` et `/blog:batch`.

## 1. Pourquoi une KB factuelle

Les LLM hallucinent **plausiblement** quand ils manquent d'info à jour. Exemple typique : Claude qui te cite « Claude Pro à 18 €/mois » alors que le prix réel est 20 €. Le chiffre semble crédible, l'article semble propre — et tu publies une erreur.

Pour un blog professionnel, une erreur factuelle = perte de crédibilité = mauvais pour la marque. Et tu ne peux pas relire chaque chiffre à chaque article quand tu en publies 30 dans un cocon.

**Solution** : injecter ta **source unique de vérité** dans le contexte du prompt + instruction stricte « tu n'utilises rien d'autre que ces faits ». Le modèle reste créatif sur la forme, mais cadré sur les chiffres.

## 2. Structure d'une KB

Format suggéré : un fichier `knowledge.json` à la racine de ton projet (lisible par un non-dev, versionnable dans git, parsable par le plugin).

Sections recommandées :

- `validated_at` — date ISO de dernière validation manuelle (ex. `"2026-04-29"`)
- `version` — entier incrémenté à chaque mise à jour
- `pricing` — tarifs des outils que tu cites souvent (Claude, ChatGPT, Notion, etc.)
- `models` — versions des modèles IA (Opus 4.7, GPT-5…) avec context window et release date
- `ecosystem` — descriptions courtes des produits/services cités (Claude Code, MCP, Projects)
- `business` — infos sur ton entreprise/offre (formations, prix, certifications, années d'activité, nombres d'apprenants)
- `rules` — règles méta de rédaction (« ne jamais citer un prix sans le qualifier 'à partir de' »)

## 3. Exemple concret

```json
{
  "validated_at": "2026-04-29",
  "version": 1,
  "pricing": {
    "claude": {
      "pro": { "price_eur_per_month": 20, "includes": "Claude.ai illimité, Projects, Artifacts" },
      "max": { "price_eur_per_month": 100 },
      "team": { "price_eur_per_user_per_month": 25, "min_seats": 5 }
    },
    "note": "Les prix évoluent. Préfère 'à partir de' à un montant absolu."
  },
  "models": {
    "claude": {
      "opus_4_7": { "context": "1M tokens", "released": "2026-01", "speciality": "raisonnement long, code agentique" }
    }
  },
  "ecosystem": {
    "claude_code": "Agent CLI officiel Anthropic, lit/écrit/exécute du code en local.",
    "mcp": "Model Context Protocol — standard ouvert créé par Anthropic en 2024 pour connecter des LLM à des outils tiers."
  },
  "business": {
    "founded": 2022,
    "apprenants": "4 000+",
    "qualiopi": true
  },
  "rules": [
    "Ne jamais citer un prix exact sans le qualifier ('à partir de', 'environ').",
    "Ne jamais inventer une statistique. Si elle n'est pas dans la KB : 'la majorité' / 'beaucoup de' / omet.",
    "Ne jamais attribuer une citation à une personne sans source explicite.",
    "Pour les fonctionnalités produit, n'utilise que les descriptions de la section ecosystem."
  ]
}
```

## 4. Injection dans le prompt P2

Le prompt de rédaction P2 reçoit la KB en contexte (sérialisée en JSON dans le user message). L'instruction d'ouverture est stricte :

> **KNOWLEDGE BASE 2026 (validée le {validated_at}) — utilise EXCLUSIVEMENT ces faits pour tout chiffre, prix, statistique, version. Si une info manque : OMETS-LA ou utilise un conditionnel ('environ', 'à partir de', 'selon les données disponibles').**

Le prompt enchaîne ensuite avec le brief de l'article, les instructions de rédaction (ton, format, longueur), puis le `linking_context` (voir skill `cocon-method`).

L'ordre compte : la KB doit arriver **avant** les instructions créatives, pour ancrer le cadre factuel comme contrainte primaire.

## 5. Maintenance — qui, quand, comment

**Qui** : c'est toi (ou ton équipe) qui maintiens ta KB. Le plugin ne la met pas à jour automatiquement — les sources changent trop souvent et de manière imprévisible (annonces Anthropic, refontes tarifaires concurrents, etc.).

**Quand** :
- Trimestriellement minimum
- À chaque grosse annonce (lancement nouveau modèle, refonte tarifaire d'un outil cité, nouveau livrable Ottho)

**Comment** :
1. Édite manuellement `knowledge.json`
2. Bumpe `version` (1 → 2) et `validated_at` (date du jour)
3. Commit dans git avec un message explicite (`chore(kb): bump prix Claude Team de 25→30 €`)

Ce moment de maintenance est aussi l'occasion de relire les articles existants pour vérifier qu'ils sont toujours factuellement à jour. Si un prix a bougé, refresh ces articles avant de publier le suivant.

## 6. Limite à connaître

La KB n'est **pas** une protection à 100 %. Le modèle peut quand même halluciner s'il manque de contexte sur une question annexe (ex. citer une stat marketing du secteur que personne ne lui a fournie).

Couplée avec la **validation humaine en draft Ghost** (statut forcé `draft`, voir skill `article-quality`), c'est suffisant pour 95 % des cas.

Pour les 5 % restants : **relecture humaine attentive** sur les passages comportant chiffres, citations, dates, noms propres. La KB réduit la surface d'erreur, elle ne la supprime pas.

---

## Usage dans les commandes

Cette skill est invoquée par :

- `/blog:article` — injection KB dans le prompt P2 de rédaction
- `/blog:batch` — même injection, répétée pour chaque article du cocon

## Source canonique

Le pattern est implémenté dans le code de démonstration fourni par le formateur (fichier `lib/knowledge.ts`) — version TypeScript de la même structure, utilisée en production.
