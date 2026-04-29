---
name: status
description: Affiche l'état du cocon : total articles, published, drafts, planned, répartition par pilier, dernière publication, prochaine suggestion. Lecture seule, pas d'API LLM consommée.
---

# /blog:status, État du cocon

Tu vas afficher à l'étudiant un instantané de son cocon : combien d'articles total, combien publiés, combien en draft Ghost, combien encore à écrire, et un découpage par pilier. **Lecture seule**, pas de prompt LLM, pas d'image, pas d'écriture sur disque, pas d'appel à une skill. Juste de la donnée croisée entre `cocon.json` et Ghost.

L'objectif : une vue d'ensemble en 2 secondes, comme un `git status` mais pour le cocon éditorial.

## Pré-requis

Avant de démarrer, vérifie qu'un fichier `cocon.json` existe à la racine du projet. Sinon, dis-lui :

> « Je ne trouve pas de `cocon.json` à la racine du projet. Cette commande lit l'état du cocon, il faut donc qu'il existe d'abord. Lance `/blog:cocon` pour le cartographier, puis reviens sur `/blog:status`. »

Et arrête-toi là.

Pas besoin de `ghost-config.md` à proprement parler, si le MCP Ghost n'est pas branché, tu te contentes des status présents dans `cocon.json` et tu signales l'absence de cross-référence.

## Protocole

Tu vas dérouler **4 étapes** silencieusement (pas de dialogue), puis afficher le résultat.

### Étape 1, Lire `cocon.json`

1. Ouvre `cocon.json` à la racine du projet.
2. Parse le JSON.
3. Pour chaque fille (pilier), récupère son `pilier_slug`, son `title`, et la liste des `petites_filles` (slug + status).
4. Pour chaque petite-fille, note son `slug` et son `status` actuel (`planned`, `draft` ou `published`).

Si le JSON est invalide ou si la structure attendue manque (`mere`, `filles`, `petites_filles`), signale-le à l'étudiant et arrête-toi :

> « Ton `cocon.json` est cassé ou incomplet (clé manquante : `<clé>`). Relance `/blog:cocon` pour le régénérer proprement. »

### Étape 2, Lire l'état Ghost (via MCP)

Si le MCP Ghost est disponible :

1. Liste tous les posts via `mcp__ghost__posts_browse` avec un filtre `filter: "status:[published,draft]"` et un `limit: all` (ou paginé jusqu'à épuisement).
2. Pour chaque post Ghost, récupère son `slug` et son `status` (`published` ou `draft`) et la `published_at` si disponible.
3. **Cross-référence** avec les petites-filles du cocon (matching par `slug`) :
   - Si une petite-fille du cocon est trouvée en `published` sur Ghost → status effectif = `published`
   - Si elle est trouvée en `draft` sur Ghost → status effectif = `draft`
   - Si elle n'est pas trouvée du tout sur Ghost → status effectif = `planned`
4. **Articles hors cocon** : repère les posts Ghost dont le `slug` ne correspond à **aucune** petite-fille du cocon. Liste-les à part.

Si le MCP Ghost n'est **pas** disponible (erreur de connexion ou MCP non installé) :

- Utilise les `status` tels quels dans `cocon.json` (sans cross-référence).
- Note dans l'output final que la cross-référence Ghost n'a pas pu être faite.

### Étape 3, Calculs

À partir des status effectifs, calcule :

- **Total articles dans le cocon** = nombre total de petites-filles
- **Total `published`** = petites-filles dont le status effectif est `published`
- **Total `draft`** = petites-filles dont le status effectif est `draft`
- **Total `planned`** = petites-filles dont le status effectif est `planned`
- **Par pilier** : pour chaque fille, ratio `<published>/<total>` et la liste des slugs avec leur status effectif (groupés par status : `published` puis `draft` puis `planned`).
- **Dernière publication** : parmi les posts Ghost `published`, prends celui avec la `published_at` la plus récente. Note la date (ISO court `YYYY-MM-DD`) et le `slug`. Si aucune publication, affiche `aucune pour l'instant`.
- **Prochain à écrire** : parmi les petites-filles `planned`, prends celle du pilier le plus avancé (= pilier avec le plus haut ratio `published/total`, hors piliers déjà 100 % terminés). En cas d'égalité, prends celle qui apparaît en premier dans le JSON. Affiche son `title` et son `pilier_slug`.
- **Articles hors cocon** : la liste collectée à l'étape 2 (slugs Ghost non rattachés).

### Étape 4, Output

Affiche le résultat en un seul bloc, exactement dans le format ci-dessous (français, sobre, type CLI, ASCII pour les arborescences). Remplace `<...>` par les valeurs calculées. Aligne les colonnes proprement.

```
Cocon : <titre de la mère, depuis cocon.json>
────────────────────────
<TOTAL> articles total
  ✓ <N_PUBLISHED> publiés
  ◐ <N_DRAFT> en draft Ghost (à relire)
  ○ <N_PLANNED> planifiés (à écrire)

Par pilier :
  <pilier_slug 1>           : <X>/<Y>   ✓ <slug published 1>, <slug published 2>
                                         ◐ <slug draft 1>
                                         ○ <slug planned 1>, <slug planned 2>
  <pilier_slug 2>           : <X>/<Y>   ✓ <...>
                                         ◐ <...>
                                         ○ <...>
  ...

Dernière publication : <YYYY-MM-DD> (<slug>)
Prochain à écrire    : "<title>" (pilier <pilier_slug>)
```

**Règles d'affichage** :

- Les `pilier_slug` sont alignés sur la même colonne (padding à droite pour atteindre la largeur du plus long pilier_slug).
- Pour chaque pilier, la première ligne contient le ratio `<X>/<Y>` (publiés / total) suivi des slugs `✓`. Les lignes suivantes (`◐` puis `○`) sont indentées au même niveau que la première occurrence de `✓`, sans répéter le pilier_slug.
- **N'affiche pas une ligne de status si elle est vide.** Si un pilier n'a aucun draft, ne mets pas de ligne `◐`. Idem pour `✓` et `○`.
- Si **toutes** les petites-filles d'un pilier sont planifiées (0 publié, 0 draft), affiche-les toutes sur la ligne `○` à côté du ratio `0/Y`.
- Sépare les slugs sur une même ligne par `, ` (virgule + espace). Si la liste est très longue (>5 slugs), passe à la ligne en gardant l'indentation, en regroupant par paquets de 3 par ligne.

**Légende** (à afficher sous l'output, sur une ligne de séparation) :

```
────────────────────────
✓ publié sur le blog public  ◐ draft sur Ghost (en attente de relecture)  ○ planifié dans le cocon, à écrire
```

**Section optionnelle, Articles hors cocon**

Si la cross-référence Ghost a renvoyé des posts non rattachés au cocon, ajoute après la légende :

```

⚠️  <N> articles hors cocon dans Ghost :
  - <slug 1>
  - <slug 2>
  - <slug 3>
```

Sinon, omets entièrement cette section.

**Section conditionnelle, MCP Ghost indisponible**

Si l'étape 2 n'a pas pu se faire (MCP Ghost absent ou en erreur), ajoute après la légende :

```

ℹ️  Cross-référence Ghost indisponible, affichage basé sur les status de cocon.json uniquement.
   Branche le MCP Ghost (`/blog:setup-ghost`) pour un état temps réel.
```

## Vérifications finales

Avant de clôturer, vérifie que :

- [ ] L'affichage est clair, lisible, aligné en colonnes.
- [ ] Tous les piliers du cocon sont listés (même ceux à 0/Y).
- [ ] Le total `published + draft + planned` est égal au total d'articles annoncé.
- [ ] Les articles hors cocon sont signalés si présents.
- [ ] Aucun secret n'est affiché (slugs publics uniquement, jamais l'API key).

## Clôture

Une fois l'output affiché, ajoute en pied de réponse :

> « Pour écrire le prochain : `/blog:article` (par défaut, prend le premier `planned`).
> Pour batcher 5 articles d'un coup : `/blog:batch 5`. »

## Règles de comportement

- **Lecture seule, toujours.** Cette commande ne modifie jamais `cocon.json`, ne publie rien sur Ghost, ne déclenche aucun prompt LLM coûteux. Si l'étudiant te demande pendant l'exécution « peux-tu corriger un slug ? », arrête-toi et redirige : « Pour modifier le cocon, lance `/blog:cocon` (mode édition à l'étape concernée). Ici, je ne fais que lire. »
- **Pas de dialogue interactif.** Tu déroules les 4 étapes en silence et tu affiches le résultat. Si quelque chose bloque (cocon.json absent ou cassé), tu signales et tu t'arrêtes, pas de protocole de réparation ici.
- **Tutoiement direct.**
- **ASCII propre.** Pas d'emoji superflu en dehors des 3 marqueurs de status (`✓`, `◐`, `○`) et des 2 marqueurs de section (`⚠️`, `ℹ️`). Pas de couleurs ANSI (l'output doit rester lisible dans n'importe quel terminal et copiable en markdown).
- **Si le cocon est vide** (mère sans filles, ou filles sans petites-filles), affiche le message minimal :
  > « Ton cocon est cartographié mais aucune petite-fille n'est définie pour l'instant. Lance `/blog:cocon` pour les ajouter, puis reviens. »
  Et arrête-toi.
