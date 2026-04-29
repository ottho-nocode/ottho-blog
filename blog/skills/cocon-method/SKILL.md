---
name: cocon-method
description: Méthode du cocon sémantique (Bourrelly). Cartographie un cocon à partir d'un brief : 1 mère + 3-7 filles + 3-5 petites-filles. Règles de maillage interne. Format JSON cocon.json. Protocole d'interaction IA → utilisateur (proposer / valider / ajuster). Utilisée par /blog:cocon.
---

# Skill : cocon-method

Cette skill packe la méthode du cocon sémantique de Laurent Bourrelly pour qu'un agent (typiquement `/blog:cocon`) puisse proposer un cocon à un étudiant et dialoguer avec lui jusqu'à un `cocon.json` validé.

## La méthode Bourrelly en 30 secondes

Un cocon sémantique = un **arbre thématique de pages** que tu fais converger vers une page racine, avec un **maillage interne strict** entre les pages. Au lieu d'écrire 50 articles isolés en espérant que Google les classe, tu écris 50 articles **organisés**, où chaque article renforce ses voisins par des liens internes.

Le bénéfice : Google ne te classe plus juste sur une page (« mon article sur X »), il te reconnaît comme **expert d'un sujet entier** (« ce site sait tout sur X »). On parle d'**autorité thématique** ou *topic authority*.

Une seule règle vraiment importante : **le cocon est un silo étanche**. Pas de raccourcis, pas de liens internes anarchiques, pas de pages orphelines.

## Anatomie d'un cocon : mère, filles, petites-filles

Un cocon a **trois étages** :

### La mère (1 seule)
La page racine du cocon. Elle vise la **requête commerciale la plus large** sur laquelle tu veux te positionner. C'est la page hub vers laquelle tout converge.

Exemple Ottho : `/apprendre-claude` cible le keyword `apprendre claude`.

### Les filles (3 à 7)
Les **piliers**. Chacune couvre un **sous-thème orthogonal** de la mère. C'est aussi une page (souvent un long article ou une page-pilier de blog), pas juste une catégorie.

Exemple Ottho, 3 filles parmi les 10 :
- `Construire un pipeline de blog SEO avec Claude` → keyword `blog claude seo`
- `Créer un site web avec Claude` → keyword `créer site web claude`
- `Automatiser sa stack avec Claude et MCP` → keyword `claude automatisation`

### Les petites-filles (3 à 5 par fille)
Les **articles spécifiques longue traîne**. Chaque petite-fille répond à une **question précise** d'un internaute, avec une intention claire.

Exemple sous la fille « pipeline blog » :
- `Cocon sémantique avec l'IA : la méthode pas-à-pas` → keyword `cocon sémantique ia`
- `Blog headless avec Ghost, Next.js et Claude` → keyword `blog headless ghost`
- `Rédiger un article SEO avec Claude` → keyword `rédiger article seo claude`

## Les 3 règles de maillage (à tatouer)

Le maillage est ce qui transforme une simple liste de pages en cocon. **Toujours dans le contenu, pas seulement dans le menu.**

1. **Une fille remonte vers sa mère.** Au moins un lien dans le corps de l'article (pas dans le footer ou le menu seulement) qui pointe vers la mère.
2. **Les filles pointent vers leurs sœurs (latéral).** Chaque fille référence 2-3 sœurs dans son contenu, pour tisser la nappe.
3. **Une petite-fille remonte vers sa mère ET sa fille parente.** Double remontée systématique.

**Règle d'étanchéité :** pas de lien direct d'une petite-fille vers une autre branche du cocon (par exemple, une petite-fille de la branche « blog » ne pointe pas directement vers une petite-fille de la branche « ecommerce »). Si tu veux bouger latéralement, tu remontes vers la fille parente, qui pointe vers sa sœur. C'est ça, le silo étanche.

## Trouver la mère depuis le brief

Tu pars du `brief.md` produit par la skill `brief-method` (objectif business + audience + ICP + zone de chalandise). Tu en extrais la **requête commerciale max** : la phrase la plus large que ton ICP pourrait taper sur Google quand il a le problème que tu résous.

Critères de choix entre 3 candidates :
- **Volume de recherche** suffisant (sinon personne ne te trouvera)
- **Intention commerciale** (pas juste informative, ton ICP doit être prêt à acheter à terme)
- **Concurrence accessible** (si la SERP est verrouillée par Le Monde et Wikipedia, choisis un cran en dessous)

L'IA ne tranche pas seule : elle **propose 3 candidates** au minimum (avec une justification courte sur volume/intention/concurrence pour chacune) et l'utilisateur valide celle qu'il veut viser. Si aucune ne convient, l'IA en propose 3 nouvelles.

## Trouver les filles

3 à 7 sous-thèmes **orthogonaux** de la mère. Test simple : *« Peut-on écrire un article entier sur chacun sans empiéter sur les autres ? »* Si deux filles se recouvrent à plus de 30 %, fusionne-les.

Pour chaque fille, tu fixes :
- un `slug` (URL : `/blog/pilier/<pilier_slug>`)
- un `keyword` principal (1-3 mots, intention informationnelle)
- une `description` (≤ 160 caractères, sert aussi à la meta description)
- un `persona` cible (entrepreneur, opérationnel, dirigeant, détermine le ton)
- un `cta_principal` (URL de l'offre commerciale vers laquelle cette branche pousse)

## Trouver les petites-filles

Pour chaque fille, 3 à 5 niches en **longue traîne** (3-5 mots, faible concurrence, intention claire). Une petite-fille qui marche a souvent la forme d'une **vraie question** (« comment faire X », « X vs Y », « le guide complet de X »).

Critères :
- mot-clé spécifique, peu disputé
- intention identifiable : `informational`, `transactional`, `navigational`
- statut : `published` (déjà en ligne), `draft` (rédigé non publié), `planned` (à écrire)

## Format `cocon.json`

Voici le schéma que la skill produit à la fin du dialogue. C'est ce fichier que les autres skills (`brief-method`, `article-method`) consomment pour générer chaque article.

```json
{
  "site": "https://exemple.com",
  "mere": {
    "slug": "apprendre-claude",
    "title": "Apprendre Claude, Le guide complet pour francophones",
    "keyword_principal": "apprendre claude",
    "keywords_secondaires": ["formation claude", "claude ia français"]
  },
  "filles": [
    {
      "slug": "claude-blog",
      "pilier_slug": "pipeline-blog",
      "title": "Construire un pipeline de blog SEO avec Claude",
      "description": "...",
      "keyword": "blog claude seo",
      "persona": "entrepreneur",
      "cta_principal": "/claude-mastery",
      "petites_filles": [
        {
          "slug": "cocon-semantique-ia",
          "title": "Cocon sémantique avec l'IA : la méthode pas-à-pas",
          "keyword": "cocon sémantique ia",
          "intent": "informational",
          "status": "planned"
        }
      ],
      "status": "planned"
    }
  ]
}
```

Notes :
- `slug` (interne) ≠ `pilier_slug` (URL publique du pilier). Le `slug` peut contenir le préfixe thématique (`claude-blog`), le `pilier_slug` est la version courte et propre (`pipeline-blog`).
- `status` se propage : une fille `planned` n'a que des petites-filles `planned` ou `draft` ; une fille passe `published` quand au moins 1 petite-fille est publiée.
- Un seul `cta_principal` par branche, c'est ce qui donne au cocon son sens commercial.

## Protocole d'interaction IA → utilisateur

C'est ce que `/blog:cocon` doit exécuter, dans cet ordre :

1. **Lecture du brief.** L'IA ouvre `brief.md` à la racine du projet utilisateur. Si le fichier n'existe pas, elle redirige vers la skill `brief-method` et s'arrête.

2. **Proposition de la mère.** L'IA propose **3 candidates** pour la requête mère, avec pour chacune : keyword cible, justification courte (volume/intention/concurrence accessible), titre de page suggéré. L'utilisateur choisit une candidate, en demande de nouvelles, ou propose la sienne. Le dialogue continue jusqu'à validation.

3. **Proposition des filles.** L'IA propose **3 à 7 filles**, présentées sous forme de liste avec `pilier_slug`, `keyword`, `description`, `persona`, `cta_principal`. Pour chaque fille, l'utilisateur peut : `valider`, `modifier` (changer le keyword ou la description), `supprimer`, ou demander à l'IA d'en `ajouter` une nouvelle. Tant que toutes les filles ne sont pas validées (ou explicitement rejetées), on ne passe pas à l'étape suivante.

4. **Proposition des petites-filles, fille par fille.** Pour chaque fille validée, l'IA propose **3 à 5 petites-filles** : `slug`, `title`, `keyword`, `intent`. Même boucle valider/modifier/supprimer/ajouter.

5. **Validation de pertinence par recherche web** *(nouvelle étape, V1.1+)*. Avant la consolidation finale, l'IA passe en revue chaque petite-fille proposée et appelle des outils de recherche web pour vérifier la pertinence SEO de chaque keyword. Voir la section « Validation de pertinence, protocole web search » plus bas.

6. **Consolidation finale.** L'IA reproposa l'arbre complet en intégrant les ajustements issus de la validation web (drops, repositionnements long-tail, ajouts depuis les suggestions). L'utilisateur valide le tout en bloc.

7. **Écriture du fichier.** Une fois tout validé, l'IA écrit `cocon.json` à la racine du projet utilisateur (même dossier que `brief.md`), avec `status: "planned"` partout par défaut. L'IA confirme la création en récapitulant : `1 mère + N filles + M petites-filles = X articles à écrire`.

8. **Sortie.** L'IA suggère l'étape suivante : passer à `/blog:article` pour générer chaque petite-fille, ou éditer manuellement `cocon.json` si l'utilisateur veut ajuster plus tard.

## Validation de pertinence, protocole web search

Avant de finaliser le cocon, l'IA doit valider chaque petite-fille proposée par recherche web. Cette étape évite le piège du « cocon LLM-only » : un arbre qui paraît bon mais qui cible des requêtes 0 search ou ultra-saturées (Wikipedia + .gov + grands médias dans le top 10).

### Outils mobilisés (par ordre de préférence)

1. **`WebSearch` (built-in Anthropic, dispo dans Claude Code)**, pour chaque petite-fille, fais une recherche Google sur le `keyword` cible. Note les domaines du top 10 et évalue :
   - **Saturation** : top 10 dominé par `wikipedia.org`, `.gov`, `lemonde.fr`, `lefigaro.fr`, `youtube.com`, `reddit.com` = saturation extrême → drop ou repositionne sur une variante long-tail
   - **Pertinence** : moins de 3 résultats vraiment liés au sujet = niche faible ou intention floue → drop ou requalifie
   - **Format dominant** : présence d'un featured snippet (paragraphe court / liste / table) ou de PAA = opportunité de cibler ce format dans l'article
2. **`WebFetch` sur Google Suggest** : `https://suggestqueries.google.com/complete/search?client=firefox&q=<keyword>&hl=fr`
   - Retourne 10 suggestions en JSON public (autocomplete Google)
   - Permet d'enrichir les petites-filles avec des variantes long-tail réelles, ou de remplacer une petite-fille saturée par sa variante long-tail dérivée
3. **`WebFetch` sur Wikipedia FR** : `https://fr.wikipedia.org/w/api.php?action=opensearch&search=<keyword>&limit=5&namespace=0&format=json`
   - Confirme l'existence d'une entité encyclopédique du sujet
   - Si Wikipedia n'a aucun article proche, c'est probablement un keyword inventé ou trop niche pour porter un pilier

### Bascule manuelle (si WebSearch et WebFetch indisponibles)

Si l'agent n'a accès à aucun outil web, **bascule en mode validation assistée par l'utilisateur** :

> « Je n'ai pas d'accès web search direct. Pour valider la pertinence de chaque pilier, ouvre **Google Trends** (trends.google.com) + **AlsoAsked** (alsoasked.com) avec ton mot-clé pilier, copie-colle ici les graphes d'intérêt 12 mois et la liste des PAA. Je réajuste avec ces données réelles, pilier par pilier. »

C'est l'option de repli, moins fluide mais aussi sérieuse, car les données sont les vraies.

### Output attendu (format minirapport)

Pour chaque petite-fille de l'étape 4, l'IA produit avant la consolidation :

```
- "<keyword petite-fille>"
  - SERP top 10 : [3 mots résumant qui rank]
  - Saturation : [faible / moyenne / extrême]
  - Variantes long-tail récupérées : [liste si Suggest API a marché]
  - Verdict : [garder / repositionner sur "<variante>" / drop]
```

### Coût additionnel

5 à 10 web searches au total pour un cocon de 4 piliers × 4 petites-filles. ~2-3 min ajoutées au temps total. Tokens additionnels : ~0,02 € par cocon. Ratio valeur/coût excellent : la validation évite des heures de rédaction sur des piliers morts.

## Pièges à éviter

- **Filles qui se recouvrent.** Si deux filles partagent plus de 30 % de leur champ sémantique, le cocon perd sa structure et Google se mélange. Fusionne ou redécoupe.
- **Trop de filles d'un coup (> 7).** Mieux vaut 4 piliers solides que 10 piliers vides. On peut toujours ajouter une fille plus tard.
- **Petites-filles qui copient le keyword de leur fille.** La fille = sujet large, les petites-filles = niches. Si la petite-fille reprend le keyword de la fille, il y a cannibalisation SEO.
- **Maillage uniquement dans le menu.** Le cocon vit dans le **contenu** des articles, pas dans la navigation. Un menu n'est pas un maillage.
- **Pas de `cta_principal`.** Sans CTA commercial par branche, le cocon devient un blog sans monétisation. Toujours rattacher chaque branche à une offre.
