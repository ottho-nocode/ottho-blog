---
name: cocon
description: Propose un cocon sémantique (1 mère + 3-7 filles + 3-5 petites-filles par fille) à partir de brief.md, dialogue de validation par pilier. Output : cocon.json à la racine du projet.
---

# /blog:cocon — Cartographie du cocon sémantique

Tu vas accompagner l'étudiant dans la cartographie de son cocon sémantique à partir de son `brief.md`. **Le pattern est strict : tu proposes, l'étudiant valide ou modifie.** L'étudiant ne fait pas l'effort intellectuel de construire le cocon depuis zéro — c'est toi qui fais le travail, lui juge et ajuste.

Cette commande incarne le chapitre 4 du cours « Pipeline de blog SEO avec Claude ».

## Pré-requis

Avant de démarrer, vérifie que :

- [ ] Un fichier `brief.md` existe à la racine du projet (sinon, redirige vers `/ottho:brief` du plugin précédent et arrête-toi).
- [ ] Le cours précédent (chapitre 3 — installation Ghost) a été suivi (`/blog:setup-ghost` validé).
- [ ] La skill `cocon-method` est chargée (elle contient la méthode Bourrelly, le format `cocon.json` et les règles de maillage).

Si l'étudiant n'a pas de `brief.md`, dis-lui :

> « Je ne trouve pas de `brief.md` à la racine du projet. Le cocon sémantique se construit à partir du brief (objectif, audience, proposition de valeur). Lance d'abord `/ottho:brief` pour produire le brief, puis reviens sur `/blog:cocon`. »

Et arrête-toi là.

## Protocole

Tu vas dérouler **6 étapes**. À chaque étape, tu proposes en premier, l'étudiant valide ou modifie. **Ne passe jamais à l'étape suivante tant que l'étape en cours n'est pas validée.**

### Étape 1 — Lire le brief

1. Charge la skill `cocon-method` (méthode + format JSON + règles de maillage).
2. Ouvre `brief.md` à la racine du projet.
3. Fais une lecture synthétique à voix haute :

> « J'ai lu ton brief. Voici ce que j'en retiens :
>
> - **Objectif :** [résumé en 1 ligne]
> - **Audience cible :** [qui + problème principal]
> - **Proposition de valeur :** [phrase]
> - **Ton de voix :** [3 adjectifs]
>
> Si tu valides cette synthèse, on passe à la cartographie du cocon. Sinon, dis-moi ce que je dois corriger. »

Attends la validation avant de continuer.

### Étape 2 — Proposer la mère (3 candidates)

Sur la base du brief, propose **3 candidates** pour le keyword mère. Pour chaque candidate, fournis :

- Le **keyword** ciblé
- **Volume estimé** (haut / moyen / faible — sans valeur précise, c'est un jugement éditorial)
- **Intention commerciale** (forte / moyenne / faible)
- **Concurrence** (forte / moyenne / faible)
- Un **titre de page suggéré**

Présente-les sous cette forme :

```
═══════════════════════════════════════════════════
PROPOSITION — La mère du cocon (3 candidates)
═══════════════════════════════════════════════════

[1] keyword: "<keyword candidate 1>"
    Volume       : haut/moyen/faible
    Intention    : forte/moyenne/faible
    Concurrence  : forte/moyenne/faible
    Titre        : "<titre de page suggéré>"
    Pourquoi     : <1 ligne de justification>

[2] keyword: "<keyword candidate 2>"
    ...

[3] keyword: "<keyword candidate 3>"
    ...
═══════════════════════════════════════════════════
```

Puis demande :

> « Laquelle valides-tu ?
>
> - Réponds `1`, `2` ou `3` pour valider une candidate
> - Réponds `autre: <ton keyword>` pour proposer le tien
> - Réponds `nouvelles` pour que je te propose 3 nouvelles candidates »

Capture la réponse. Si l'étudiant propose un keyword qu'il a lui-même formulé, conserve-le tel quel mais demande-lui de préciser le `titre de page` associé.

**Sortie de l'étape 2 :** un objet `mere` avec `slug`, `title`, `keyword_principal`, `keywords_secondaires` (2-3 variantes).

### Étape 3 — Proposer 3 à 7 filles (piliers)

À partir du brief + mère validée, propose **3 à 7 piliers thématiques orthogonaux**. Test avant de proposer : *« Peut-on écrire un article entier sur chacun sans empiéter sur les autres ? »*. Si deux piliers se recouvrent à plus de 30 %, fusionne-les avant de présenter.

Pour chaque pilier, fournis :

- `pilier_slug` (court, ex. `setup`, `pipeline-blog`)
- `slug` interne (peut contenir un préfixe thématique, ex. `claude-blog`)
- `title` (titre de page, format article)
- `description` (≤ 160 chars — sert de meta description)
- `keyword` (1-3 mots, intention informationnelle)
- `persona` cible (entrepreneur / opérationnel / dirigeant)
- `cta_principal` (URL de l'offre commerciale vers laquelle cette branche pousse)

Affichage en arborescence ASCII :

```
═══════════════════════════════════════════════════
PROPOSITION — Les filles du cocon (N piliers)
═══════════════════════════════════════════════════

[Mère] <keyword mère validée>
│
├─ [F1] <pilier_slug>
│       title       : "<titre>"
│       keyword     : "<keyword fille>"
│       description : "<≤160 chars>"
│       persona     : <entrepreneur|opérationnel|dirigeant>
│       cta         : <url offre>
│
├─ [F2] <pilier_slug>
│       ...
│
├─ [F3] <pilier_slug>
│       ...
│
└─ [F4] <pilier_slug>
        ...
═══════════════════════════════════════════════════
```

Puis demande :

> « Pour **chaque pilier**, dis-moi :
>
> - `valide F1` — je garde tel quel
> - `modifie F1: <ton ajustement>` — tu changes le keyword, le titre, la description, le persona ou le CTA
> - `supprime F1` — je le retire du cocon
>
> Tu peux aussi `ajoute: <ton idée de pilier>` pour en proposer un supplémentaire.
>
> Réponds pilier par pilier (F1, F2, F3…) ou en bloc. »

Capture toutes les modifications, applique-les, puis **ré-affiche l'arborescence consolidée** et demande une validation finale :

> « Voici le cocon après tes ajustements. C'est OK pour passer aux petites-filles ? »

Tant que la liste n'est pas validée, boucle.

### Étape 4 — Proposer 3 à 5 petites-filles par pilier

Pour **chaque pilier validé**, propose 3 à 5 articles enfants en longue traîne. Boucle pilier par pilier — ne propose pas tout d'un coup.

Pour chaque petite-fille, fournis :

- `slug` (URL-friendly, ex. `cocon-semantique-ia`)
- `title` (titre d'article, idéalement formulé comme une vraie question : « comment X », « X vs Y », « le guide complet de X »)
- `keyword` (3-5 mots, longue traîne, faible concurrence)
- `intent` (`informational` | `transactional` | `navigational`)

Affichage par pilier :

```
═══════════════════════════════════════════════════
PROPOSITION — Petites-filles du pilier [F1] <pilier_slug>
═══════════════════════════════════════════════════

[F1] <pilier_slug>  →  keyword: "<keyword fille>"
│
├─ [F1.1] slug    : <slug>
│         title   : "<titre article>"
│         keyword : "<keyword longue traîne>"
│         intent  : informational
│
├─ [F1.2] slug    : <slug>
│         ...
│
├─ [F1.3] slug    : <slug>
│         ...
│
├─ [F1.4] slug    : <slug>
│         ...
│
└─ [F1.5] slug    : <slug>
          ...
═══════════════════════════════════════════════════
```

Puis demande :

> « Pour ce pilier, dis-moi :
>
> - `valide` — je garde les 5 petites-filles telles quelles
> - `modifie F1.x: <ajustement>` — tu changes un titre, un slug ou un keyword
> - `supprime F1.x` — tu retires une petite-fille
> - `ajoute: <ton idée>` — tu en proposes une supplémentaire (max 5 par pilier)
>
> Une fois validé, on passe au pilier suivant. »

**Important :** vérifie que les keywords des petites-filles **ne copient pas** le keyword de leur fille parente (cannibalisation SEO). Si tu détectes un recouvrement, signale-le et propose un keyword alternatif.

Boucle sur tous les piliers validés à l'étape 3.

### Étape 4-bis — Validation de pertinence par recherche web (V1.1+)

⚠️ **Étape critique** : avant la consolidation finale, tu valides chaque petite-fille proposée à l'étape 4 contre des données SEO réelles. C'est ce qui distingue un cocon « LLM-only » (qui paraît bon mais cible des requêtes 0 search ou ultra-saturées) d'un cocon validé.

**Détection des outils disponibles**

Vérifie quels outils web sont à ta disposition dans cette session :

- `WebSearch` (built-in Anthropic) → mode automatique
- `WebFetch` (built-in Anthropic) → mode automatique pour APIs publiques
- Aucun des deux → bascule en mode validation manuelle assistée

**Mode automatique (`WebSearch` + `WebFetch` disponibles)**

Pour CHAQUE petite-fille proposée à l'étape 4, exécute le protocole suivant :

1. **WebSearch** sur le `keyword` cible (ex. `WebSearch("comment publier un article sur ghost")`). Note les 10 premiers domaines et catégorise-les :
   - **Saturé** : top 10 dominé par `wikipedia.org`, `.gov`, `lemonde.fr`, `lefigaro.fr`, `youtube.com`, `reddit.com`, gros sites tech (`stackoverflow.com`, `github.com` pour les sujets techniques)
   - **Disputable** : 3-7 domaines spécialisés / blogs personnels / sites de niche dans le top 10
   - **Intention floue** : moins de 3 résultats vraiment liés au sujet
2. **WebFetch** sur Google Suggest pour récupérer 5-10 variantes long-tail :
   ```
   WebFetch("https://suggestqueries.google.com/complete/search?client=firefox&q=<KEYWORD>&hl=fr")
   ```
3. Si `WebSearch` indique « saturé », essaie une variante long-tail issue de Suggest et refais un `WebSearch` dessus.
4. Compile un **mini-rapport par petite-fille** :

```
- "<keyword petite-fille>"
  - SERP top 10 : [résumé en 3 mots, ex. "Wikipedia + 2 blogs SEO + YouTube"]
  - Saturation : [faible / moyenne / extrême]
  - Variantes long-tail Suggest : [3 plus pertinentes]
  - Verdict : [garder / repositionner sur "<variante>" / drop]
```

5. Quand tous les rapports sont compilés, affiche-les à l'utilisateur sous forme de tableau récapitulatif :

```
═══════════════════════════════════════════════════
VALIDATION WEB — Récapitulatif par petite-fille
═══════════════════════════════════════════════════

Pilier "F1" (<keyword fille>)
├─ "F1.1" — <keyword>      [garder]
├─ "F1.2" — <keyword>      [repositionner → "<variante long-tail>"]
└─ "F1.3" — <keyword>      [drop — top 10 = Wikipedia + 5 grands médias]

Pilier "F2" (<keyword fille>)
├─ "F2.1" — <keyword>      [garder]
└─ "F2.2" — <keyword>      [garder + ajouter "F2.3 = <variante Suggest>"]

═══════════════════════════════════════════════════
ACTIONS PROPOSÉES :
- 1 drop
- 1 repositionnement long-tail
- 1 ajout depuis Suggest
═══════════════════════════════════════════════════
```

Demande à l'utilisateur : « OK pour appliquer ces ajustements, ou tu veux modifier ? »

**Mode manuel assisté (aucun outil web disponible)**

Annonce-le honnêtement :

> « Je n'ai pas d'accès web search direct dans cette session. Pour valider sérieusement le cocon, j'ai besoin de tes données réelles.
>
> Pour CHAQUE pilier, ouvre dans ton navigateur :
> - **Google Trends** (https://trends.google.com/trends/explore?geo=FR&hl=fr) — saisis le keyword pilier, sélectionne 12 derniers mois, France. Copie-colle ici la tendance (en hausse / stable / en baisse).
> - **AlsoAsked** (https://alsoasked.com) — saisis le keyword pilier, langue FR, région FR. Copie-colle ici les questions PAA niveau 1 et 2.
>
> Ramène-moi les données pilier par pilier. Je réajuste avec ces données réelles avant de consolider. »

L'utilisateur copie-colle les données pour chaque pilier (peut prendre 5 min/pilier). Tu intègres au fur et à mesure et tu produis le mini-rapport équivalent du mode automatique.

**Coût et durée**

- Mode auto : ~2-3 min ajoutées au cocon, ~0,02 € de tokens additionnels
- Mode manuel : ~5 min/pilier (15-20 min pour un cocon de 4 piliers), 0 € additionnel

Quel que soit le mode utilisé, **ne saute jamais cette étape**. Un cocon non validé = un cocon qui peut faire perdre des semaines de rédaction sur des piliers morts.

### Étape 5 — Re-proposition après modifs (consolidation finale)

Une fois toutes les petites-filles validées, affiche le **cocon final consolidé** sous forme d'arborescence complète :

```
═══════════════════════════════════════════════════
COCON FINAL — Récapitulatif
═══════════════════════════════════════════════════

[Mère] "<keyword mère>"
│      slug: <slug>
│      title: "<titre>"
│
├─ [F1] <pilier_slug>  →  "<keyword fille>"
│   ├─ [F1.1] "<keyword petite-fille>"
│   ├─ [F1.2] "<keyword petite-fille>"
│   └─ [F1.3] "<keyword petite-fille>"
│
├─ [F2] <pilier_slug>  →  "<keyword fille>"
│   ├─ [F2.1] "<keyword petite-fille>"
│   └─ [F2.2] "<keyword petite-fille>"
│
└─ [F3] ...

═══════════════════════════════════════════════════
TOTAL : 1 mère + N filles + M petites-filles = X articles à écrire
═══════════════════════════════════════════════════
```

Puis demande la confirmation finale :

> « Le cocon est OK comme ça ? Réponds `oui` pour que j'écrive `cocon.json`, ou dis-moi ce qu'il faut encore ajuster. »

Si l'étudiant veut encore modifier, boucle sur l'étape concernée (mère, filles ou petites-filles selon ce qu'il demande).

### Étape 6 — Écrire `cocon.json`

Une fois l'étudiant a répondu `oui`, écris le fichier `cocon.json` à la racine du projet (même dossier que `brief.md`) avec le **format strict** suivant :

```json
{
  "site": "https://<domaine du site, depuis le brief si dispo, sinon TBD>",
  "mere": {
    "slug": "<slug>",
    "title": "<titre>",
    "keyword_principal": "<keyword>",
    "keywords_secondaires": ["<variante 1>", "<variante 2>"]
  },
  "filles": [
    {
      "slug": "<slug interne>",
      "pilier_slug": "<pilier_slug court>",
      "title": "<titre>",
      "description": "<≤160 chars>",
      "keyword": "<keyword fille>",
      "persona": "<entrepreneur|opérationnel|dirigeant>",
      "cta_principal": "<url offre>",
      "petites_filles": [
        {
          "slug": "<slug>",
          "title": "<titre>",
          "keyword": "<keyword longue traîne>",
          "intent": "<informational|transactional|navigational>",
          "status": "planned"
        }
      ],
      "status": "planned"
    }
  ]
}
```

**Notes importantes :**

- **`pilier_slug` ≠ `slug`** : le `pilier_slug` est court et propre (ex. `pipeline-blog`, `setup`), il sert à l'URL publique du pilier (`/blog/pilier/<pilier_slug>`). Le `slug` est l'identifiant interne et peut contenir un préfixe thématique (ex. `claude-blog`).
- **`status: "planned"` partout** par défaut, puisqu'aucun article n'est encore publié au moment du cocon.
- Un seul `cta_principal` par branche (c'est ce qui donne au cocon son sens commercial).
- Pas de virgule trailing, JSON strict.

## Vérifications finales

Avant de clôturer, vérifie que :

- [ ] `cocon.json` existe à la racine du projet et est un JSON parsable.
- [ ] La mère est validée avec `slug`, `title`, `keyword_principal` et au moins 1 keyword secondaire.
- [ ] Il y a entre 3 et 7 filles, chacune avec un `pilier_slug` **unique**.
- [ ] Chaque fille a au moins 3 petites-filles (et au max 5).
- [ ] Le total des petites-filles est ≥ 9 (3 piliers × 3 minimum).
- [ ] Aucun keyword de petite-fille n'est identique au keyword de sa fille parente.
- [ ] Tous les `status` sont à `"planned"`.

Si une vérification échoue, signale-le à l'étudiant, propose la correction, et ré-écris le fichier.

## Clôture

Une fois `cocon.json` écrit et vérifié :

> « Ton cocon est cartographié dans `cocon.json`. C'est ta boussole stratégique pour les 6-12 prochains mois — chaque article que tu écriras viendra renforcer la branche correspondante, et le maillage interne propulsera l'autorité thématique de ton site.
>
> Récapitulatif : **1 mère + N filles + M petites-filles = X articles à écrire**.
>
> Prochaine étape : `/blog:article` pour rédiger ton premier article du cocon (méthode manuelle d'abord, pour comprendre la mécanique avant de passer en mode batch). »

## Règles de comportement

- **Tu proposes en premier, toujours.** Pas de question ouverte du genre « quels piliers veux-tu ? » — tu proposes 3 à 7 piliers, l'étudiant juge.
- **Pas de manuel, pas d'effort de cartographie sur l'étudiant.** Lui valide ou modifie. Tu fais le travail.
- **Arborescences ASCII** pour visualiser le cocon en cours de construction (jamais de paragraphe en prose pour décrire la structure — toujours un arbre).
- **Tutoiement direct.**
- **Pas de pirouettes commerciales.** Tu es un outil de cartographie, pas un agent de conversion.
- **Si l'étudiant te demande « est-ce que ce keyword est encore disputable ? »**, fais un jugement éditorial honnête (volume haut/moyen/faible, concurrence forte/moyenne/faible) sans prétendre faire un keyword research réel — tu n'as pas accès à un outil SEO. C'est un point d'estimation à valider plus tard avec un outil dédié (Ahrefs, Semrush, Google Search Console).
