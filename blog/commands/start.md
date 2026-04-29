---
name: start
description: Commande MASTER — orchestre les 6 phases du cours « Claude + Blog » de A à Z (setup Ghost → theme → cocon → premier article → batch auto → audit SEO) avec un checkpoint à chaque étape. Reprenable via .ottho-blog/state.json.
---

# /start — Orchestrateur complet du cours « Claude + Blog »

Tu vas orchestrer la création **complète** d'un blog headless avec cocon sémantique, de A à Z, en enchaînant les phases dans l'ordre, avec un **checkpoint** à chaque étape. L'utilisateur peut dire STOP à tout moment et reprendre plus tard.

## Étape 0 — Gestion de l'état

### Si `.ottho-blog/state.json` existe

Lis le fichier. Il contient la progression (phase courante, choix faits, fichiers générés). Propose à l'utilisateur :

```
Je vois que tu as déjà avancé sur ton blog.
Progression actuelle : phase [X] sur 6.
Dernière étape complétée : [nom].

Tu veux :
1. Reprendre là où tu en étais (phase [X+1])
2. Recommencer depuis le début
3. Sauter à une phase spécifique (précise laquelle)
```

### Si le fichier n'existe pas

Crée `.ottho-blog/state.json` :

```json
{
  "project_name": "[nom du projet, demandé à l'utilisateur]",
  "created_at": "[date ISO]",
  "current_phase": 0,
  "completed_phases": [],
  "ghost_url": null,
  "ghost_subdomain": null,
  "cocon_path": null,
  "articles_published_drafts": 0,
  "choices": {}
}
```

## Étape 1 — Annonce du parcours

Affiche :

```
🚀 /blog:start — Création de blog Ghost, de A à Z

Tu vas enchaîner 6 phases, avec une validation à chaque étape.
Tu peux dire STOP à tout moment, je garde ta progression.

Phase 1 — Setup Ghost (20-30 min)     [PikaPods + branchement subpath /blog OU sous-domaine + MCP Ghost]
Phase 2 — Theme à ta charte (20 min)  [fork Source + override CSS + upload]
Phase 3 — Cocon sémantique (15 min)   [propose + valide → cocon.json]
Phase 4 — Premier article (5 min)     [pipeline complet : brief → article → image → draft Ghost]
Phase 5 — Batch 3 articles (15 min)   [variante batch avec review HITL]
Phase 6 — Audit SEO (5 min)           [score 8 critères + suggestions]

Temps total estimé : ~1h30 (si tu as déjà ton site avec brief.md et charte.md prêts).

Coût LLM total estimé : ~0,75 € (4 articles générés × ~0,15 €) + ~0,20 € d'images fal.ai.

Pré-requis vérifiés ?
- [ ] Site HTML/CSS/JS déployé sur Vercel (URL Vercel par défaut `<projet>.vercel.app` OK, custom domain optionnel)
- [ ] brief.md à la racine
- [ ] charte.md à la racine
- [ ] Compte PikaPods (gratuit ou ~5 €/mois)
- [ ] Compte fal.ai (déjà dans Claude Code, du cours précédent)

⚠️ **Pas besoin d'avoir un nom de domaine custom.** Le scénario par défaut (`/blog:setup-ghost` te demandera de choisir) marche avec n'importe quelle URL Vercel — y compris la `vercel.app` par défaut. Le blog vivra à `<ton-site>/blog` via un rewrite Vercel transparent.

On y va ? (oui / non / précise un blocage)
```

Si l'utilisateur n'a pas un des pré-requis, redirige :
- Pas de site déployé → cours « Claude + Site web »
- Pas de `brief.md` → `/ottho:brief` du plugin précédent
- Pas de `charte.md` → cours « Claude + Site web » chapitre Design
- Pas de compte PikaPods → continue, on en crée un en Phase 1

**Ne pose JAMAIS de question sur un nom de domaine custom à ce stade.** Le scénario d'hébergement (subpath `/blog` vs sous-domaine) est choisi à l'intérieur de `/blog:setup-ghost` (Phase 1). Ta seule pré-requis est qu'un site soit déployé sur Vercel — peu importe son URL.

Sinon, attends `oui` et passe à la Phase 1.

## Phase 1 — Setup Ghost

**Si phase non complétée :**

1. Annonce : « Phase 1/6 — Setup Ghost. ~20-30 min. On va créer ton instance Ghost sur PikaPods, brancher le blog à ton site (subpath `/blog` par défaut, ou sous-domaine custom si tu en as un), et installer le MCP Ghost dans Claude Code. »
2. Lance la logique de **`/blog:setup-ghost`** (charge la skill `ghost-config`). C'est cette commande qui demandera à l'utilisateur quel scénario d'hébergement choisir (A = subpath / B = sous-domaine). Tu n'as PAS à pré-juger ici.
3. À la fin de cette phase, vérifie que `ghost-config.md` existe à la racine et que le MCP Ghost répond.
4. **Checkpoint :** « Ghost est en ligne sur `<BLOG_URL>` (l'URL exacte dépend du scénario choisi dans setup-ghost). Le MCP répond bien. On passe au theme ? (oui / stop) »
5. Marque Phase 1 complétée dans le state, note `ghost_url` (l'URL publique) et `ghost_scenario` (`A` ou `B`).

## Phase 2 — Theme à ta charte

**Si phase non complétée :**

1. Annonce : « Phase 2/6 — Theme. ~20 min. On va générer un theme Ghost custom à partir de ta `charte.md`, pour que ton blog ressemble à ton site existant. »
2. Vérifie que `charte.md` est lisible. Si absent, propose-en la création rapide en lisant les couleurs/fonts depuis le site existant.
3. Lance la logique de **`/blog:theme`** (charge la skill `ghost-theme`).
4. À la fin, le theme est uploadé et activé dans Ghost. L'utilisateur visite l'URL publique du blog (`<BLOG_URL>` notée en Phase 1) pour valider visuellement.
5. **Checkpoint :** « Le rendu visuel est OK ? On passe au cocon ? (oui / itère sur le theme / stop) »
6. Si l'utilisateur veut itérer : reste en Phase 2, applique les modifs CSS demandées, ré-uploade.
7. Marque Phase 2 complétée dans le state.

## Phase 3 — Cocon sémantique

**Si phase non complétée :**

1. Annonce : « Phase 3/6 — Cocon sémantique. ~15 min. Je vais lire ton `brief.md`, te proposer une structure de cocon (1 mère + 3-7 piliers + 3-5 articles par pilier), tu valides chaque pilier. »
2. Lance la logique de **`/blog:cocon`** (charge la skill `cocon-method`).
3. À la fin, `cocon.json` est écrit à la racine.
4. **Checkpoint :** « Le cocon est cartographié dans `cocon.json`. On écrit ton premier article ? (oui / stop) »
5. Marque Phase 3 complétée, note `cocon_path: ./cocon.json`.

## Phase 4 — Premier article (méthode manuelle assistée)

**Si phase non complétée :**

1. Annonce : « Phase 4/6 — Ton premier article. ~5 min. On va générer un article du cocon de A à Z : brief, rédaction, image hero, publication en draft sur Ghost. Tu valides à chaque étape pour comprendre la mécanique. »
2. Lance la logique de **`/blog:article`** (charge `article-quality`, `link-validation`, `knowledge-base`).
3. **Important** : ce premier article doit être fait **manuellement assisté** (pas en batch) pour que l'utilisateur comprenne le pipeline.
4. À la fin, l'article est en draft sur Ghost. L'utilisateur va dans Ghost admin pour relire et publier.
5. **Checkpoint :** « Ton premier article est en draft sur Ghost. Tu l'as relu et publié ? On passe au mode auto-pilote ? (oui / stop / refais un article manuellement avant) »
6. Marque Phase 4 complétée, incrémente `articles_published_drafts`.

## Phase 5 — Batch 3 articles (mode auto-pilote)

**Si phase non complétée :**

1. Annonce : « Phase 5/6 — Mode auto-pilote. ~15 min. On va lancer un batch de 3 articles d'un coup, avec une review humaine entre chaque. Tu apprends à gérer un batch propre. »
2. Recommandation : « Si tu n'es pas à l'aise avec le résultat de l'article unique, refais 1-2 articles manuellement avant ce batch. »
3. Lance la logique de **`/blog:batch 3`** (charge les mêmes skills que Phase 4 + rate-limit + HITL renforcé).
4. À la fin, 3 articles supplémentaires sont en draft Ghost. L'utilisateur les relit et publie depuis Ghost admin.
5. **Checkpoint :** « 3 articles sont en draft. Relis-les et publie ceux qui te plaisent (15 min par article environ). On passe à l'audit SEO ? (oui / stop) »
6. Marque Phase 5 complétée, incrémente `articles_published_drafts` de 3.

## Phase 6 — Audit SEO

**Si phase non complétée :**

1. Annonce : « Phase 6/6 — Audit SEO. ~5 min. On va checker la qualité SEO des articles déjà publiés (8 critères), pour identifier ce qui mérite des corrections. »
2. Lance la logique de **`/blog:seo-audit`** (charge la skill `seo-blog`).
3. À la fin, un rapport `seo-audit-{date}.md` est écrit à la racine.
4. **Checkpoint :** « L'audit est dans `seo-audit-{date}.md`. Tu peux maintenant corriger les top 3 articles à score bas dans Ghost admin. »
5. Marque Phase 6 complétée.

## Étape finale — Clôture

Affiche :

```
🎉 /blog:start — Parcours terminé.

Bilan :
- Ghost en ligne sur {ghost_url}
- Theme custom à ta charte
- Cocon sémantique cartographié ({nombre} piliers, {nombre} articles planifiés)
- {articles_published_drafts} articles publiés/drafts
- Audit SEO disponible : ./seo-audit-{date}.md

Prochaine étape : itère.
- Tu peux relancer `/blog:article` une fois par semaine pour publier régulièrement.
- Ou `/blog:batch 5` une fois par mois pour un gros sprint.
- Surveille `/blog:status` pour voir l'avancement.
- Lance `/blog:opportunities` (V1.5, dès que dispo) pour identifier les requêtes SEO à creuser via Search Console.

Le cocon va prendre du momentum SEO en 2-3 mois. Patience et régularité.

Bravo. 🚀
```

Marque le state comme complet : `current_phase: 7` (= terminé).

## Règles de comportement

- **Une phase à la fois.** Pas d'enchaînement automatique sans checkpoint validé.
- **Reprenable.** Si l'utilisateur dit STOP, sauvegarde le state et arrête proprement.
- **Pas de surprise de coût.** Chaque phase avec coût LLM significatif (Phase 4 = ~0,20 €, Phase 5 = ~0,75 €) est annoncée AVANT lancement.
- **Le statut Ghost reste TOUJOURS draft.** L'utilisateur publie à la main depuis Ghost admin. Non négociable (cf. skill `article-quality`).
- **Si une phase échoue, propose `/blog:corrige`** pour diagnostiquer avant d'avancer.

## Quand utiliser `/blog:start` vs les commandes individuelles

- **Première fois** sur le projet → `/blog:start` (orchestré, pédagogique, avec checkpoints)
- **Reprise après un break** → `/blog:start` aussi (lit le state, propose de reprendre)
- **Itération sur un projet établi** → commandes individuelles (`/blog:article`, `/blog:batch 5`, `/blog:seo-audit`)
- **Debug** → `/blog:corrige`
- **Vue d'ensemble** → `/blog:status`
