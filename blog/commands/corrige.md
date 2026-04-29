---
name: corrige
description: Diagnostic et fixes pour les problèmes courants du blog : MCP Ghost déconnecté, theme cassé, image fal.ai non générée, post Ghost en erreur, validation de liens qui rate. Génère bug-{date}.md si rien ne marche.
---

# /blog:corrige — SAV pour ton blog Ghost

Tu vas m'aider à diagnostiquer un problème et à le réparer. Cette commande est faite pour quand quelque chose plante après `/blog:setup-ghost`, `/blog:theme`, `/blog:cocon`, `/blog:article` ou `/blog:batch`.

L'objectif : **identifier la zone du problème, dérouler un diagnostic ciblé, proposer un fix concret**. Si vraiment rien ne marche, on génère un rapport de bug propre à envoyer au support.

## Quand utiliser cette commande

Tu lances `/blog:corrige` quand :

- Tu as essayé `/blog:setup-ghost` et l'instance Ghost ne répond pas, ou le MCP ne ping pas.
- Tu as lancé `/blog:theme` et le ZIP est rejeté par Ghost ou le rendu est cassé.
- `/blog:cocon` plante ou ton `cocon.json` est corrompu.
- `/blog:article` ne va pas au bout : brief vide, image manquante, upload Ghost en erreur 422…
- `/blog:batch` s'est arrêté en plein milieu et tu ne sais pas où en est `cocon.json`.
- N'importe quoi d'autre qui sent le bug.

---

## Étape 1 — Identifier la zone du problème

Première chose à demander à l'utilisateur :

```
Qu'est-ce qui ne marche pas ? Choisis :

  1. Setup Ghost — instance pas accessible, sous-domaine en 404, MCP qui ne ping pas
  2. Theme — rendu cassé, theme non activé, ZIP rejeté par Ghost
  3. Cocon — /blog:cocon plante, cocon.json corrompu
  4. Article — génération qui rate, article qui ne part pas dans Ghost, image manquante
  5. Batch — interrompu en cours, statut bizarre dans cocon.json
  6. Autre (décris-moi)
```

Récupère le numéro et déroule la catégorie correspondante. Ne saute pas d'étape : pose les questions de diagnostic une par une, attends les réponses, et propose un fix à chaque blocage identifié.

---

## Étape 2 — Diagnostic ciblé selon la catégorie

### Catégorie 1 — Setup Ghost

L'instance Ghost ne répond pas, ou le MCP est silencieux. Déroule :

**Questions de diagnostic :**

1. L'URL `xxx.pikapod.net` (ou ton sous-domaine) répond-elle en HTTP ?
   ```bash
   curl -I https://ton-blog.pikapod.net
   ```
   On veut un `HTTP/2 200` ou `301`. Si c'est `connection refused` → l'instance dort ou est down.

2. Le DNS du sous-domaine est-il propagé ?
   ```bash
   dig blog.exemple.com +short
   ```
   On veut une IP en sortie. Si c'est vide → la config DNS n'est pas remontée (peut prendre jusqu'à 24h, en général 5-10 min).

3. Le HTTPS est-il valide ?
   ```bash
   curl -I https://blog.exemple.com
   ```
   Si `SSL certificate problem` → Let's Encrypt n'a pas encore généré le cert. Attends 1 à 3 minutes après la config du domaine et retente.

4. Le MCP Ghost est-il listé ?
   ```bash
   claude mcp list
   ```
   Tu dois voir une ligne `ghost`. Si pas → il n'est pas installé, ou Claude Code a besoin d'un redémarrage pour le charger.

5. Les API keys sont-elles bien copiées (sans espace, format `id:secret` pour l'Admin API, format hex 26 chars pour la Content API) ?

**Fix communs :**

- **Redémarrer Claude Code** : 8 fois sur 10, le MCP qui ne ping pas se règle en relançant la session.
- **Re-tester le ping après 2 min** : Let's Encrypt et la propagation DNS prennent un peu de temps.
- **Recopier les API keys** : ouvre Ghost admin → Settings → Integrations → ton intégration custom → recopie sans espace, sans saut de ligne.
- **Vérifier la variable d'env du MCP** : ouvre ton fichier de config MCP (souvent `~/.mcp.json` ou la config plugin) et confirme que `GHOST_API_URL` et `GHOST_ADMIN_API_KEY` sont remplis.

---

### Catégorie 2 — Theme

Le theme ne s'applique pas, ou Ghost rejette le ZIP. Déroule :

**Questions de diagnostic :**

1. Le ZIP a-t-il bien été généré ?
   ```bash
   ls -lh theme.zip
   ```
   On veut voir un fichier d'au moins 5-10 KB. Si absent → la commande `/blog:theme` a planté avant la fin.

2. L'upload a-t-il renvoyé un message d'erreur dans Ghost ? Ghost rejette les themes invalides avec un message du type :
   - `"Theme is missing default.hbs or index.hbs"` → le squelette de base manque
   - `"package.json is invalid"` → JSON cassé
   - `"Theme uses unsupported features"` → version de Ghost trop ancienne pour les helpers utilisés

3. Le `package.json` du theme est-il un JSON valide ?
   ```bash
   cat theme/package.json | jq
   ```
   Si `jq` retourne `parse error` → JSON cassé, regarde la ligne pointée par l'erreur.

4. Le theme est-il activé dans Ghost admin → Settings → Design → Change theme ? L'upload ne suffit pas, il faut cliquer "Activate".

5. Le rendu est-il cassé seulement sur certaines pages (post seul, page liste, page tag) ? Ça oriente vers le `.hbs` correspondant (`post.hbs`, `index.hbs`, `tag.hbs`).

**Fix communs :**

- **Re-générer le ZIP** : `/blog:theme` régénère proprement le ZIP, écrase l'ancien.
- **Valider la structure du theme avec gscan** : si tu as Node sous la main, `npx gscan theme.zip` te donne un rapport détaillé des erreurs (Ghost utilise gscan en interne).
- **Vider le cache navigateur** : Ghost peut continuer à servir l'ancien CSS pendant quelques minutes, fais un Cmd+Shift+R.
- **Réactiver Casper temporairement** : en cas de doute total, repasse Ghost sur le theme par défaut Casper, vérifie que le contenu s'affiche, puis ré-active ton theme custom.

---

### Catégorie 3 — Cocon

`/blog:cocon` plante ou le `cocon.json` est cassé. Déroule :

**Questions de diagnostic :**

1. Le `brief.md` existe-t-il bien à la racine du projet ?
   ```bash
   ls -lh brief.md
   ```
   Si absent → `/blog:cocon` a besoin de ce fichier en entrée. Crée-le ou relance `/blog:setup` si la commande existe dans ton workflow.

2. `cocon.json` est-il un JSON valide ?
   ```bash
   cat cocon.json | jq
   ```
   Si `parse error` → le fichier est corrompu (souvent un caractère orphelin, une virgule en trop, ou un caractère non échappé).

3. Quelle est la dernière version saine ? Regarde l'historique git :
   ```bash
   git log --oneline cocon.json
   ```

**Fix communs :**

- **`cocon.json` corrompu** : restaure une version précédente depuis git.
  ```bash
  git checkout HEAD~1 -- cocon.json
  ```
  Puis vérifie avec `cat cocon.json | jq` que c'est valide.
- **Pas de version git saine** : relance `/blog:cocon`. Elle écrase le fichier — tu perds l'avancement éventuel mais tu repars d'un fichier propre.
- **`brief.md` malformé** : ouvre-le, vérifie qu'il contient bien les sections attendues (le format dépend de ce que `/blog:cocon` lit). Si vide ou trop court, étoffe-le.

---

### Catégorie 4 — Article

`/blog:article` ne va pas au bout. Le pipeline Article a 4 étapes (brief P1 → article P2 → image fal.ai → upload Ghost) et chacune peut planter. Déroule :

**Questions de diagnostic :**

1. L'API Anthropic répond-elle ? Vérifie que `ANTHROPIC_API_KEY` est bien set dans ton environnement :
   ```bash
   echo $ANTHROPIC_API_KEY | head -c 12
   ```
   On veut voir `sk-ant-api...`. Si vide → la variable n'est pas exportée dans ce shell.

2. Le brief P1 a-t-il été généré ? Regarde la sortie de la commande, ou ton `cocon.json` (champ `brief` sur l'article concerné).

3. L'article P2 a-t-il été généré ? Le `stop_reason` de la réponse Claude est-il `end_turn` (OK) ou `max_tokens` (article tronqué) ? Si tronqué → augmenter `max_tokens` dans le pipeline ou demander à Claude un article plus court.

4. L'image fal.ai a-t-elle été générée ?
   - Vérifie `FAL_KEY` :
     ```bash
     echo $FAL_KEY | head -c 12
     ```
   - Si la clé est OK mais l'image ne sort pas → fal.ai peut être en surcharge. L'article doit publier **sans image** (mode dégradé), pas planter le pipeline complet.

5. L'upload Ghost a-t-il renvoyé une erreur 422 (validation) ? Causes courantes :
   - `custom_excerpt` > 300 caractères → tronque le résumé.
   - `slug` déjà existant → ajoute un suffixe ou supprime l'ancien post.
   - `tags` invalides → vérifie le format des tags Ghost (string ou objet `{name}`).
   - Image en `feature_image` qui pointe vers une URL inaccessible → utilise l'upload natif Ghost ou un URL publique.

**Fix communs :**

- **Relancer `/blog:article`** : le pipeline est idempotent. Le `cocon.json` marquera le re-essai comme `draft` au lieu de `published` pour que tu vérifies en admin Ghost avant de publier.
- **Mode dégradé image** : si fal.ai plante, l'article doit partir sans `feature_image`. Tu pourras en ajouter une plus tard manuellement dans Ghost admin.
- **Couper l'article si max_tokens** : reformule le P2 avec une cible plus courte, ou augmente `max_tokens` côté pipeline.
- **Erreur 422 Ghost** : copie le payload JSON envoyé et le message d'erreur, ça pointe direct vers le champ fautif.

---

### Catégorie 5 — Batch

`/blog:batch` s'est arrêté en cours. Déroule :

**Questions de diagnostic :**

1. Combien d'articles ont déjà été publiés ? Regarde `cocon.json` :
   ```bash
   cat cocon.json | jq '.articles | map(select(.status == "published")) | length'
   ```

2. Combien restent en `draft` ou `pending` ?
   ```bash
   cat cocon.json | jq '.articles | map(select(.status != "published")) | length'
   ```

3. Les articles déjà publiés sont-ils OK dans Ghost admin ? Connecte-toi à `https://ton-blog.pikapod.net/ghost/` et vérifie visuellement les 2-3 derniers.

4. Quel article exactement a planté ? Le dernier non-published dans `cocon.json` te donne le coupable.

**Fix communs :**

- **NE PAS re-lancer le même `/blog:batch`** : tu vas dupliquer les articles déjà publiés. Le batch n'est pas idempotent côté Ghost (chaque appel `posts_add` crée un nouveau post).
- **Lancer `/blog:status`** d'abord pour voir précisément où on en est et quels articles manquent.
- **Reprendre article par article** : lance `/blog:article` sur chaque article manquant, un par un. C'est plus lent mais tu gardes le contrôle.
- **Si le `cocon.json` est désynchronisé** (un article publié mais marqué `draft` dans le json, ou l'inverse) : édite le `cocon.json` à la main pour refléter la réalité Ghost, puis reprends.

---

### Catégorie 6 — Autre

L'utilisateur décrit le problème en texte libre. Pose 2-3 questions de diagnostic pour comprendre, propose un fix au cas par cas. Si tu n'arrives pas à diagnostiquer, passe direct à l'Étape 3.

---

## Étape 3 — Si aucune des solutions ne marche

Si après diagnostic et tentatives de fix, le problème persiste, génère un fichier `bug-{YYYY-MM-DD-HHmm}.md` à la racine du projet avec ce contenu :

```markdown
# Rapport de bug — Blog Ghost — {date ISO}

## Description du problème

[Ce que l'utilisateur a essayé de faire, et ce qui s'est passé à la place.]

## Catégorie identifiée

[1. Setup Ghost / 2. Theme / 3. Cocon / 4. Article / 5. Batch / 6. Autre]

## Étapes tentées

1. [étape 1]
2. [étape 2]
3. [...]

## Output des commandes de diagnostic

```
[colle ici la sortie brute de chaque commande lancée pendant le diagnostic]
```

## Versions

- **Plugin ottho-blog** : [lire `.claude-plugin/plugin.json` → champ `version`]
- **Ghost** : [admin Ghost → bas de page → "About Ghost"]
- **Claude Code** : [output de `claude --version`]
- **Node** : [output de `node --version`]
- **OS** : [output de `uname -a` ou équivalent]

## Contexte

[Tout autre détail pertinent : commande exacte lancée, fichiers en jeu, captures d'erreur…]
```

Récupère ces infos via les outils dispo (lecture de fichiers, exécution de commandes shell). Une fois le fichier écrit, dis à l'utilisateur :

> « J'ai généré `bug-{date}.md` à la racine du projet. Colle son contenu sur le support Ottho (lien à venir / Slack / Notion — selon ce qui est en place). Reviens-moi avec le ticket si tu as besoin de creuser plus. »

---

## Vérifications finales

Avant de clôturer, valide :

- [ ] Catégorie identifiée (1-6).
- [ ] Au moins une suggestion de fix proposée à l'utilisateur.
- [ ] Si bloqué après plusieurs tentatives : fichier `bug-{date}.md` généré.

---

## Clôture

Si le fix a fonctionné, dis simplement :

> « Si le fix a marché, super. Sinon, le rapport `bug-{date}.md` est prêt à être envoyé au support Ottho. Reviens-moi avec le ticket si besoin. »

Pas de jargon obscur, pas de blabla. SAV qui en a vu d'autres.
