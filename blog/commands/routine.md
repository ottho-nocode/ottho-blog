---
name: routine
description: Prépare la routine qui rédige un article du cocon chaque semaine, sans toi, et le dépose en brouillon dans Ghost. Génère le prompt autonome et le journal dans .ottho-blog/, le teste dans la session, puis guide la création de la tâche planifiée dans Claude Desktop. Statut Ghost forcé à draft.
---

# /blog:routine, Un article du cocon par semaine, en tâche planifiée

Tu vas installer chez l'étudiant une **routine locale** : une tâche planifiée de Claude Desktop qui, à heure fixe, rédige le prochain article du cocon et le dépose **en brouillon** dans Ghost. L'étudiant relit dans l'admin Ghost et publie à la main.

C'est la variante « sans écran » de `/blog:article`. Même pipeline (brief P1, article P2, sanitizer, link-validation, image fal.ai, draft Ghost, update `cocon.json`), mais **aucune question pendant l'exécution** : à 7 h du matin, personne n'est là pour répondre.

**Charge ces 4 skills** avant tout, identiques à `/blog:article` :

- **`article-quality`** (`skills/article-quality/SKILL.md`), system prompts P1 et P2, schemas JSON, sanitizer cadratin, clamp Ghost.
- **`link-validation`** (`skills/link-validation/SKILL.md`), whitelist des liens internes, fallback vers le pilier-parent.
- **`knowledge-base`** (`skills/knowledge-base/SKILL.md`), KB factuelle injectée dans P2.
- **`cocon-method`** (`skills/cocon-method/SKILL.md`), structure de `cocon.json`.

**Temps estimé** : ~15 min (réglages, test à blanc, création de la tâche dans Desktop, premier Run now).

**Coût** : chaque exécution consomme l'abonnement Claude comme une session normale, plus ~0,02 € d'image fal.ai.

---

## Garde-fous critiques (à ne JAMAIS contourner)

⚠️ **Statut Ghost FORCÉ à `draft`.** La routine crée toujours le post en `status: "draft"`. Elle n'a pas le droit de publier, ni de modifier ou supprimer un post existant.

⚠️ **La relecture humaine se déplace, elle ne disparaît pas.** Dans `/blog:article` et `/blog:batch`, l'étudiant valide le brief puis l'article avant le push Ghost. Ici, il valide **après**, dans l'admin Ghost, avant de cliquer « Publish ». Le prompt de routine accepte donc le brief sans dialogue, mais rien ne part en ligne sans l'étudiant. Dis-le clairement dans le préambule.

⚠️ **Jamais `/blog:article` ni `/blog:batch` dans la routine.** Ces commandes s'arrêtent pour demander une validation : en tâche planifiée, la session resterait bloquée. La routine exécute un prompt dédié, généré par cette commande.

⚠️ **Un article par exécution, deux au plus par jour.** Le prompt de routine s'arrête si le journal compte déjà deux succès datés du jour (protège contre un rattrapage au réveil doublé d'un Run now).

⚠️ **Pas de mode qui saute les autorisations.** La tâche garde le mode d'autorisation par défaut. Les autorisations nécessaires se donnent une fois, au premier Run now, avec « always allow ».

---

## Préambule

Ouvre la commande par ce message :

> « On va mettre ton blog en pilote automatique, avec un garde-fou : chaque semaine, Claude rédige le prochain article de ton cocon et le dépose **en brouillon** dans Ghost. Toi, tu relis et tu publies.
>
> Le déroulé :
>
> 1. Je vérifie que tout ce dont la routine a besoin est en place
> 2. Tu choisis le jour, l'heure et deux réglages
> 3. Je génère le prompt de la routine et son journal dans `.ottho-blog/`
> 4. On le teste ici, une fois, pour de vrai (un article en brouillon dans Ghost)
> 5. Tu crées la tâche planifiée dans Claude Desktop, je te donne chaque valeur à coller
> 6. Premier « Run now » devant l'écran, pour donner les autorisations une fois pour toutes
>
> Compte un quart d'heure. On y va ? »

---

## Pré-requis (à valider AVANT de commencer)

Vérifie les 5 pré-requis ci-dessous. Si l'un des 4 premiers manque, **mets en pause** et redirige.

### Pré-requis 1, `cocon.json` avec au moins un article à écrire

Lis `cocon.json` à la racine du projet. Compte les petites-filles au statut `planned`.

- Fichier absent ou invalide : « Lance d'abord `/blog:cocon`. La routine pioche ses articles dans le cocon. »
- Aucune `planned` : « Ton cocon est entièrement écrit. Ajoute des petites-filles avec `/blog:cocon` avant de planifier quoi que ce soit. »

### Pré-requis 2, `knowledge.json` existe

Obligatoire pour P2, comme dans `/blog:article`. Si absent : « Sans knowledge base, la routine inventerait des chiffres sans personne pour les relire avant Ghost. Crée-la d'abord : lance `/blog:article`, il te guide. »

### Pré-requis 3, MCP Ghost opérationnel

Appelle le MCP Ghost (`posts_browse`, `limit: 1`). S'il ne répond pas : « Lance `/blog:setup-ghost` ou vérifie le MCP avec `/mcp`, puis reviens. »

### Pré-requis 4, au moins un article déjà écrit à la main

Cherche dans `cocon.json` une petite-fille en `draft` ou `published`. Si aucune :

> « Tu n'as encore jamais lancé `/blog:article`. Fais-le une fois avant d'automatiser : c'est là que tu vérifies que ta KB, tes liens et ton image sortent bien, avec la main sur chaque étape. Une routine qui part sur un pipeline jamais testé, c'est trois brouillons ratés avant que tu t'en aperçoives. »

Mets en pause. Continue seulement si l'étudiant insiste explicitement.

### Pré-requis 5, MCP fal.ai (optionnel)

Si absent : warning, on continue. La routine tournera sans image hero (réglage « image » forcé à non).

### Cas particulier, routine déjà installée

Si `.ottho-blog/routine/prompt.md` existe déjà, demande :

> « Une routine existe déjà dans ce projet. Tu veux **régénérer** son prompt (les réglages actuels sont remplacés, le journal est conservé) ou **annuler** ? »

---

## Étape 1, Réglages

Pose les questions en un seul message :

> « Trois réglages, réponds en une ligne (ou tape `défaut`) :
>
> 1. **Quand ?** Jour et heure. Défaut : lundi 7:00. Une fois par semaine suffit : un blog qui publie à rythme régulier vaut mieux qu'un blog qui déverse dix articles d'un coup, et chaque exécution consomme ton abonnement.
> 2. **Image hero ?** oui / non. Défaut : oui (fal.ai).
> 3. **Quel article en premier ?**
>    - `avance` (défaut) : la prochaine petite-fille du pilier le plus avancé, pour finir un pilier avant d'en ouvrir un autre
>    - `ordre` : les petites-filles dans l'ordre du `cocon.json`
>    - `alterne` : un pilier différent à chaque exécution »

Capture `<JOUR>`, `<HEURE>`, `<IMAGE>` (oui/non), `<REGLE_CHOIX>`. Si l'étudiant veut une fréquence quotidienne, accepte mais rappelle le quota et le risque SEO.

Détermine aussi :

- `<PROJECT_PATH>` : le chemin absolu du dossier courant (`pwd`)
- `<TASK_NAME>` : `article-blog-hebdo` (ou `article-blog-quotidien` si quotidien)
- `<DATE>` : la date du jour, ISO

---

## Étape 2, Générer les fichiers de la routine

### 2.1, Le prompt : `.ottho-blog/routine/prompt.md`

Écris ce fichier en remplaçant les variables. Le bloc `<REGLE_CHOIX_TEXTE>` dépend du réglage 3 :

| Réglage | Texte à insérer |
|---|---|
| `avance` | Prends la première petite-fille au statut "planned" du pilier le plus avancé (celui qui compte le plus de petites-filles en "draft" ou "published"). |
| `ordre` | Prends la première petite-fille au statut "planned" en lisant cocon.json dans l'ordre (filles puis petites-filles). |
| `alterne` | Lis le dernier succès du journal pour connaître le pilier utilisé. Prends la première petite-fille "planned" d'un autre pilier ; s'il n'y en a pas, reste sur le même. |

Si `<IMAGE>` = non, remplace l'étape 8 du prompt par : « 8. Pas d'image hero : crée le post sans feature_image. »

```markdown
# Routine <TASK_NAME>

Générée par /blog:routine le <DATE>. Ce fichier est lu à chaque exécution :
le modifier change la routine, sans toucher à Claude Desktop.

Tu tournes en tâche planifiée, sans personne pour te répondre.
Ne pose aucune question. Si une information manque, si une autorisation
t'est refusée ou si un outil ne répond pas, arrête-toi et explique
pourquoi dans le journal.

## Objectif

Rédiger UN article du cocon et le déposer en brouillon dans Ghost. Un seul.

## Avant de commencer

1. Lis .ottho-blog/journal-auto.md. S'il contient déjà deux lignes OK
   datées d'aujourd'hui, ajoute la ligne « LIMITE | limite du jour
   atteinte » et arrête-toi.
2. Charge les skills du plugin blog : article-quality, knowledge-base,
   link-validation, cocon-method. Si l'une d'elles est introuvable,
   ajoute la ligne « ÉCHEC | skills du plugin blog indisponibles » et
   arrête-toi : ne rédige jamais sans la méthode.
3. Lis brief.md, cocon.json, knowledge.json et ghost-config.md à la
   racine du dossier.

## Choix de l'article

4. <REGLE_CHOIX_TEXTE>
   Ne touche jamais une petite-fille en "draft" ou "published".
   S'il ne reste aucune "planned", ajoute la ligne « FIN | cocon épuisé »
   au journal et arrête-toi.

## Rédaction (méthode de /blog:article, sans les dialogues)

5. Liens autorisés : construis la whitelist de la skill link-validation
   (mère, piliers, articles déjà publiés dans Ghost via le MCP Ghost,
   pages structurelles).
6. Brief : applique le system prompt P1 de la skill article-quality.
   Pas de validation humaine : le brief est retenu tel quel s'il respecte
   BRIEF_SCHEMA. Sinon, régénère-le une fois.
7. Article : applique le system prompt P2, avec knowledge.json injectée
   en premier. Tout chiffre, prix ou version vient de la KB ; sinon,
   omets-le. Puis applique le sanitizer (aucun tiret cadratin ni
   demi-cadratin, limites Ghost de la skill) et la validation des liens
   (lien hors whitelist = réécrit vers le pilier-parent).

## Image et Ghost

8. Image : génère l'image hero avec fal.ai (16:9) à partir du
   feature_image_prompt, puis envoie-la dans Ghost (images_upload).
   Si fal.ai ou l'upload échoue, continue sans image et note-le.
9. Crée le post avec le MCP Ghost : slug = slug de la petite-fille,
   status "draft". Ne publie jamais. Ne modifie et ne supprime aucun
   post existant. Si un post existe déjà avec ce slug, ne crée rien,
   note-le et arrête-toi.
10. Dans cocon.json, passe la petite-fille en "draft" et ajoute
    ghost_id et drafted_at. Ne change rien d'autre dans le fichier.

## Trace

11. Ajoute une ligne au tableau de .ottho-blog/journal-auto.md :
    - succès : | <date et heure> | OK | <titre> | <lien admin Ghost du brouillon> | <remarques : sans image, liens réécrits> |
    - échec : | <date et heure> | ÉCHEC | <étape> | | <message d'erreur> |

## Garde-fous

- Un seul article par exécution.
- Si une étape échoue deux fois de suite, arrête-toi et note l'erreur.
- Jamais de publication, de suppression, ni de modification d'un post
  Ghost existant.
- Sur le disque, tu n'écris que dans cocon.json et
  .ottho-blog/journal-auto.md (plus le fichier image temporaire).
```

### 2.2, Le journal : `.ottho-blog/journal-auto.md`

S'il n'existe pas, crée-le :

```markdown
# Journal de la routine <TASK_NAME>

Une ligne par exécution, ajoutée par la routine. La plus récente en bas.

| Date | Statut | Article | Brouillon Ghost | Remarques |
|---|---|---|---|---|
```

S'il existe déjà, ne le touche pas.

### 2.3, L'état : `.ottho-blog/state.json`

Si le fichier existe (projet lancé avec `/blog:start`), ajoute ou remplace la clé `routine` :

```json
"routine": {
  "name": "<TASK_NAME>",
  "schedule": "<JOUR> <HEURE>",
  "image": true,
  "choice_rule": "<REGLE_CHOIX>",
  "prompt_file": ".ottho-blog/routine/prompt.md",
  "created_at": "<DATE>"
}
```

S'il n'existe pas, ne le crée pas.

---

## Étape 3, Test à blanc, dans cette session

Annonce le test :

> « Le prompt est prêt. Avant de le confier à Desktop, on le lance ici, une fois. C'est un vrai test : il va créer un vrai brouillon dans Ghost et passer une petite-fille en `draft`. Rien n'est publié. On y va ? »

Si l'étudiant accepte, **lis `.ottho-blog/routine/prompt.md` et exécute-le à la lettre, sans poser aucune question**, comme le ferait la tâche planifiée. Les demandes d'autorisation de Claude Code peuvent apparaître : c'est normal en session interactive.

Puis contrôle avec l'étudiant :

- le brouillon est dans l'admin Ghost (donne le lien) ;
- `cocon.json` a changé (la petite-fille est en `draft`, avec `ghost_id`) ;
- le journal a sa ligne `OK`.

Si le test s'est arrêté sur une question ou une information manquante, **corrige le prompt** (pas l'exécution) et relance le test. Le prompt est prêt quand il va au bout sans rien demander.

---

## Étape 4, Créer la tâche planifiée dans Claude Desktop

La tâche se crée dans l'app **Claude Desktop** (version 1.1.5368 ou plus récente). Depuis le terminal, Claude Code ne sait pas créer de tâche locale durable : ses tâches planifiées s'arrêtent avec la session.

Affiche à l'étudiant le pas-à-pas et les valeurs exactes :

> « Dans Claude Desktop :
>
> 1. Onglet **Code**, clique **Scheduled** dans la barre latérale (ou dans le menu **More**), puis **New task**. Si un menu apparaît, choisis **Set up manually**.
> 2. **Name** : `<TASK_NAME>`
> 3. **Instructions** : colle exactement ce texte
>
>    ```
>    Tu tournes en tâche planifiée. Lis le fichier .ottho-blog/routine/prompt.md à la racine de ce dossier et exécute-le à la lettre, sans poser de question. Si le fichier est introuvable, arrête-toi.
>    ```
>
>    Dans la barre sous le champ : modèle **Sonnet** (suffisant pour rédiger), mode d'autorisation **par défaut**, celui qui demande avant d'agir.
> 4. **Advanced settings**, **General** : active **Only on this computer**.
> 5. **Local folder** : `<PROJECT_PATH>`
> 6. **Trigger** : **Weekly**, `<JOUR>`, `<HEURE>` (ou **Daily**, `<HEURE>`, si tu as choisi une routine quotidienne).
> 7. **Description** : « Un article du cocon en brouillon Ghost ».
> 8. Enregistre. Si Desktop n'a jamais ouvert ce dossier, il te demande de lui faire confiance : accepte. »

Propose aussi la voie en langage naturel, à coller dans n'importe quelle session de Claude Desktop :

> « Crée une tâche planifiée locale nommée <TASK_NAME>, chaque <JOUR> à <HEURE>, dans le dossier <PROJECT_PATH>, uniquement sur cet ordinateur, avec ces instructions : "Tu tournes en tâche planifiée. Lis le fichier .ottho-blog/routine/prompt.md à la racine de ce dossier et exécute-le à la lettre, sans poser de question. Si le fichier est introuvable, arrête-toi." »

Explique pourquoi l'instruction pointe vers un fichier : pour changer la routine (une règle, le ton, l'image), on modifie `.ottho-blog/routine/prompt.md` dans le projet. La tâche Desktop, elle, ne bouge pas.

---

## Étape 5, Premier Run now, devant l'écran

> « Sur la page de la tâche, clique **Run now** et reste là. À chaque demande d'autorisation (écrire dans `cocon.json` et le journal, créer le brouillon et envoyer l'image dans Ghost, appeler fal.ai), choisis **always allow**. Les exécutions suivantes réutiliseront ces autorisations sans te déranger. Tu les retrouves, et tu peux les retirer, dans le panneau **Always allowed** de la tâche.
>
> Ce Run now écrit un deuxième article : c'est voulu, c'est le test en conditions réelles. »

Si un outil redemande l'autorisation à chaque appel sans proposer « always allow », la routine bloquera à chaque exécution sur cet outil : signale-le et cherche une alternative (autre outil, ou étape retirée du prompt).

Rappelle enfin les deux conditions de fonctionnement :

> « La tâche ne tourne que si l'app Desktop est ouverte et l'ordinateur réveillé. Active **Keep computer awake** (**Settings**, **This computer**, **System**). Un portable écran fermé se met quand même en veille. Si l'ordinateur dormait à l'heure prévue, Desktop lance une seule exécution de rattrapage au réveil. »

---

## Vérifications finales (avant clôture)

Confirme avec l'étudiant que chaque ligne est cochée :

- [ ] `.ottho-blog/routine/prompt.md` généré avec les réglages choisis
- [ ] `.ottho-blog/journal-auto.md` existe, avec au moins une ligne `OK`
- [ ] Test à blanc allé au bout sans aucune question
- [ ] Brouillon visible dans l'admin Ghost, **status draft** (rien de publié)
- [ ] Tâche créée dans Desktop : **Only on this computer**, bon dossier, bonne fréquence, statut **Active**
- [ ] Run now terminé, autorisations données en « always allow »
- [ ] Keep computer awake activé

---

## Clôture

Termine par ce message :

> « Ta routine est en place : chaque <JOUR> à <HEURE>, un article du cocon t'attend en brouillon dans Ghost.
>
> Ton rituel, une vingtaine de minutes :
>
> 1. Ouvre `.ottho-blog/journal-auto.md` : la dernière ligne te donne le titre et le lien du brouillon, ou l'erreur.
> 2. Relis le brouillon dans l'admin Ghost (faits, ton, liens, image), corrige, puis clique **Publish**.
> 3. Lance `/blog:status` pour voir l'état du cocon.
>
> Si rien n'apparaît : la session de la tâche est listée sous **Scheduled** dans Desktop, avec l'historique des exécutions (y compris celles sautées, et pourquoi). Pour une pause, passe la tâche en **Paused** depuis sa page. Pour changer ce que fait la routine, modifie `.ottho-blog/routine/prompt.md`. »

---

## Notes d'implémentation

- **Pourquoi un prompt dédié.** `/blog:article` et `/blog:batch` ont une review HITL non négociable avant le push Ghost. La routine la déplace après le push, en gardant le statut `draft` forcé : aucun texte n'est publié sans relecture humaine.
- **Rédaction.** Dans la routine, P1 et P2 sont produits par le modèle de la session, avec les system prompts et schémas de la skill `article-quality`. Aucune clé `ANTHROPIC_API_KEY` à faire passer à Desktop.
- **Pourquoi pas une tâche créée depuis le terminal.** Les outils de planification du CLI (`/loop`, CronCreate) sont liés à la session : ils s'arrêtent avec elle et expirent au bout de 7 jours. La tâche durable sur l'ordinateur se crée dans Desktop.
- **Routine cloud.** Pour un ordinateur souvent éteint, une tâche cloud (`/schedule` dans le CLI, ou claude.ai/code/routines) tourne sur les serveurs d'Anthropic. Elle demande le projet dans un dépôt GitHub, le MCP Ghost en connecteur ou dans un `.mcp.json` du dépôt, l'adresse Ghost autorisée dans l'accès réseau de l'environnement, et elle pousse sur des branches `claude/` : sans fusion, la mise à jour de `cocon.json` est perdue et la routine suivante reprend le même article. Hors périmètre de cette commande.
- **Idempotence.** Relancer `/blog:routine` régénère le prompt et conserve le journal. La tâche Desktop n'a pas besoin d'être recréée, sauf pour changer le jour ou l'heure (bouton **Edit** sur sa page).
- **Doc de référence** : https://code.claude.com/docs/en/desktop-scheduled-tasks
