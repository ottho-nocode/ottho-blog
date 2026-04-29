---
name: ghost-config
description: Procédure complète de mise en place de Ghost en headless sur PikaPods. Création compte, lancement instance, branchement sous-domaine, configuration custom integration (API keys), installation MCP Ghost dans Claude Code, mode privé, webhooks. Utilisée par /blog:setup-ghost.
---

# Skill : ghost-config

Cette skill contient la procédure complète pour mettre en place une instance Ghost en mode headless, hébergée sur PikaPods, et exposée à Claude Code via le MCP Ghost. C'est le socle technique du chapitre « Installation du blog » du cours academy.ottho.co/claude-blog.

## 3 scénarios d'hébergement supportés

Le choix dépend de la **techno du site existant** ET de la **disponibilité d'un nom de domaine custom** :

| Scénario | URL publique | Techno requise | Quand l'utiliser |
|---|---|---|---|
| **A — PikaPods URL** | `wonderful-caribou.pikapod.net` | toute (HTML pur OK) | par défaut, pas de DNS, pas de domaine custom |
| **B — Sous-domaine custom** | `blog.<domaine>` | toute (HTML pur OK) | l'élève a un domaine custom, veut un blog branding clean |
| **C — Headless API** | `<site>/blog` (rendu par le framework de l'élève) | Next.js / Astro / SvelteKit / Nuxt / Remix uniquement | architecture du repo de référence `ottho-reforged` ; demande d'écrire ~150 lignes côté framework |

⚠️ **Anti-pattern à NE PAS proposer** : un rewrite Vercel proxy `/blog/* → ghost.pikapod.net/blog/*`. Ghost servirait directement les pages mais croirait toujours vivre à son URL native (le template Ghost de PikaPods n'expose pas la variable `url` au top-level), ce qui casserait les `<link rel="canonical">`, le sitemap et les liens internes du theme. Pour avoir l'URL `<site>/blog` proprement, c'est scénario C uniquement (= rendu par framework JS, pas par Ghost).

Les 9 étapes ci-dessous décrivent la procédure pour le scénario B (sous-domaine custom, le plus complet). Pour le scénario A, sauter l'étape 3 (DNS + Custom Domain) — l'URL Ghost est l'URL PikaPods native. Pour le scénario C, sauter aussi l'étape 3 et utiliser l'URL PikaPods comme `GHOST_BACKEND_URL` (le visiteur ne la verra jamais ; c'est le framework qui rend `<site>/blog`).

## Pourquoi Ghost (et pas WordPress, Notion, Substack)

Ghost est un CMS pensé **pour les rédacteurs**, open-source, écrit en Node.js. Il a quatre forces décisives pour notre usage :

- **Headless natif.** Ghost expose deux APIs REST (Content API en lecture publique, Admin API en lecture/écriture authentifiée). Tu peux donc rendre tes articles côté Claude Code (HTML/CSS/JS, Next, Astro, peu importe) et n'utiliser Ghost que comme back-office d'écriture. WordPress demande un plugin REST + plein de configuration. Notion n'a pas d'API publique de lecture stable. Substack est fermé.
- **Open-source et portable.** Tu possèdes ta base, tu peux migrer vers un autre hébergeur en 10 minutes. Substack ne te laisse pas exporter le HTML propre. Notion non plus.
- **SEO solide par défaut.** Sitemap auto-généré, balises canonical correctes, OG/Twitter cards configurables par article, redirections 301 paramétrables. Tu n'as rien à coder pour avoir un blog SEO-ready.
- **Editor Koenig minimaliste.** Le seul éditeur WYSIWYG qui produit du HTML propre (pas de spans imbriqués façon Word, pas de divs vides). Les classes Koenig (`.kg-callout-card`, `.kg-bookmark-card`) sont stables et stylables.

## Pourquoi PikaPods (et pas Ghost-Pro, ni VPS self-hosted)

- **Hébergeur européen, RGPD-compliant.** Datacenters en Allemagne et aux Pays-Bas. Aucun transfert de données vers les US. Critique si tu collectes des emails newsletter.
- **Une instance Ghost lancée en 2 minutes.** Catalogue de pods pré-configurés. Tu cliques, tu choisis une taille, c'est en ligne.
- **~5 €/mois pour démarrer** (à vérifier sur la page tarifs PikaPods, les prix évoluent). Ghost-Pro démarre à 9 $/mois mais limite les intégrations sur le plan de base. Un VPS self-hosted (Hetzner, OVH) coûte moins cher mais demande de gérer les mises à jour, les backups, le HTTPS, le firewall — pas le scope du cours.
- **Pas de gestion serveur.** Mises à jour Ghost gérées en un clic, backups quotidiens automatiques, HTTPS Let's Encrypt automatique.

## Étape 1 — Créer un compte PikaPods

1. Aller sur https://www.pikapods.com → bouton **Sign up**
2. Email + mot de passe fort (utilise un gestionnaire de mots de passe : 1Password, Bitwarden, le trousseau Apple)
3. PikaPods offre des crédits gratuits à l'inscription pour tester (montant à vérifier sur leur page d'accueil, ça évolue). C'est suffisant pour valider l'installation avant de passer à un paiement mensuel.

## Étape 2 — Lancer une instance Ghost

1. Une fois connecté → **Catalog** → tape « Ghost » dans la recherche
2. Clique sur la fiche Ghost → bouton **Run pod**
3. **Taille** : choisis **S (256 MB RAM)** — largement suffisant pour un blog qui démarre. Tu pourras upgrader si tu publies des centaines d'articles avec beaucoup d'images.
4. **Région** : choisis **Europe** (Allemagne ou Pays-Bas) — meilleure latence pour tes lecteurs européens et RGPD direct
5. Clique **Deploy**. L'instance démarre en 2 à 3 minutes. Tu obtiens une URL technique du type `https://wonderful-caribou.pikapod.net`

À ce stade ton Ghost tourne, mais tu vas le brancher sur ton vrai sous-domaine avant la première connexion admin.

## Étape 3 — Sous-domaine `blog.tonsite.com`

Chez ton registrar de domaine (OVH, Gandi, Namecheap, Cloudflare DNS, Infomaniak) :

1. Va dans la zone DNS du domaine principal
2. Crée un enregistrement **CNAME** :
   - **Nom** : `blog`
   - **Cible** : l'URL technique fournie par PikaPods (ex : `wonderful-caribou.pikapod.net`)
   - **TTL** : 3600 (1 h) ou laisse la valeur par défaut
3. Sauvegarde

La propagation DNS prend de **10 minutes à 1 heure**. Vérifie avec :

```bash
dig blog.tonsite.com CNAME
# ou
dig +short blog.tonsite.com
```

Tu peux aussi utiliser https://dnschecker.org en y entrant `blog.tonsite.com` — tu dois voir l'enregistrement CNAME pointer vers l'URL PikaPods sur les serveurs du monde entier.

## Étape 4 — Configurer le custom domain dans Ghost (côté PikaPods)

1. PikaPods dashboard → ton pod Ghost → onglet **Domains** (ou **Settings** selon la version de l'interface)
2. **Add domain** → entre `blog.tonsite.com`
3. PikaPods détecte le CNAME et provisionne automatiquement un certificat **HTTPS via Let's Encrypt**
4. Test : ouvre `https://blog.tonsite.com` dans un onglet privé. Tu dois voir l'écran d'accueil Ghost « Welcome » qui te demande de créer le compte admin.

Si tu vois une erreur SSL ou un timeout, attends 5-10 minutes de plus (le certificat peut prendre du temps à se générer la première fois).

## Étape 5 — Premier login Ghost admin

1. URL : `https://blog.tonsite.com/ghost`
2. Crée le compte admin (owner) :
   - **Email** : ton email pro
   - **Nom** : ton prénom + nom
   - **Mot de passe** : fort (12+ caractères, mélange casse + chiffres + symboles)
3. Configuration de base, à faire tout de suite :
   - **Settings → General → Site title** : nom du blog (ex : « Mon Blog »)
   - **Settings → General → Site description** : 1 phrase, 150 caractères max — utilisée comme meta description par défaut
   - **Settings → General → Timezone** : `Europe/Paris`
   - **Settings → General → Publication language** : `fr`
   - **Settings → General → Navigation** : 3-4 entrées max pour démarrer (Accueil, À propos, Contact)

## Étape 6 — Créer une Custom Integration (Admin + Content API keys)

C'est l'étape qui débloque l'usage de Ghost depuis Claude Code.

1. Ghost admin → **Settings → Integrations** (icône de prise dans la barre latérale)
2. Section **Custom integrations** → bouton **Add custom integration**
3. **Nom** : `Claude Code` (ou le nom de ton projet)
4. Une fois créée, tu vois 3 valeurs critiques :

   - **Content API Key** — 24 caractères hexadécimaux (ex : `2b1c4d5e6f7a8b9c0d1e2f3a`). Clé **publique en lecture seule**. C'est elle qu'on utilise pour afficher les articles côté front (lib/ghost.ts dans le code de référence).
   - **Admin API Key** — format `id:secret` où `id` fait 24 hex et `secret` fait 64 hex (ex : `5a1b...:6c2d...`). Clé **privée en lecture/écriture**. C'est elle qui permet à Claude Code de **créer et publier des articles** via l'API Admin.
   - **API URL** — l'URL de ton instance Ghost, donc `https://blog.tonsite.com`

5. **Stockage de ces valeurs** :
   - Pour un projet HTML/CSS/JS pur sans build : copie-les dans un fichier local non commité (ex : `secrets.md` dans `.gitignore`) et passe-les à la config MCP (étape 7).
   - Pour un projet Next/Astro avec déploiement Vercel : ajoute-les en variables d'env (`GHOST_URL`, `GHOST_CONTENT_API_KEY`, `GHOST_ADMIN_API_KEY`).

**Ne jamais committer ces clés dans git.** L'Admin API Key permet de tout faire sur ton blog, y compris supprimer des articles.

## Étape 7 — Installer le MCP Ghost dans Claude Code

Le MCP Ghost donne à Claude Code l'accès direct à ton instance — il pourra lister, lire, créer, éditer des articles via les outils `mcp__ghost__posts_*`.

1. Dans Claude Code, lance la commande :

   ```
   /mcp install ghost
   ```

   (ou utilise la commande équivalente proposée par le repo MCP Ghost officiel ou un fork compatible Admin API + Content API)

2. Le serveur MCP te demandera 3 valeurs :
   - **URL Ghost** : `https://blog.tonsite.com`
   - **Admin API Key** : la clé `id:secret` de l'étape 6
   - **Content API Key** : la clé hex de l'étape 6

3. Vérifie l'installation :

   ```
   /mcp ping ghost
   ```

   Tu dois recevoir une réponse `OK` ou équivalent. Si tu obtiens une erreur 401, tes clés ne sont pas correctes — repasse par Settings → Integrations dans Ghost admin pour les régénérer.

4. Test fonctionnel : demande à Claude Code « liste mes 5 derniers articles Ghost ». Il doit utiliser `mcp__ghost__posts_browse` et te répondre. Sur une instance fraîche, tu auras les 3 articles de démo Ghost (« Coming soon », « Start here », etc.).

## Étape 8 — Mode privé (optionnel mais recommandé pendant l'installation)

Tant que ton blog est vide ou en chantier, tu ne veux pas que Google indexe les articles de démo.

1. Ghost admin → **Settings → General → Make this site private**
2. Active le toggle **Private**
3. Ghost te génère un mot de passe partagé (à donner aux personnes qui veulent voir le site en preview)

Quand le désactiver : **dès que tu publies ton premier vrai article**. Avant, tu indexes du contenu vide ou de mauvaise qualité, ce qui dégrade ton référencement futur.

## Étape 9 — Webhooks (avancé, pour plus tard)

Les webhooks Ghost servent à **notifier ton site qu'un article a été publié/modifié**, pour qu'il rafraîchisse son cache.

- **Si tu utilises Next.js / Astro avec un cache ISR** : tu auras besoin des webhooks pour la revalidation à la publication. Voir `Settings → Integrations → ton intégration → Add webhook`.
- **Si ton site est en HTML/CSS/JS pur sans cache serveur** : tu n'en as pas besoin. Les articles sont fetchés à chaque visite.

À voir au chapitre « Production » du cours, après que ton premier article est publié.

## Output attendu de `/blog:setup-ghost`

À la fin de l'exécution de la commande, l'agent doit avoir créé un fichier `ghost-config.md` à la racine du projet de l'étudiant, **sans aucun secret**, contenant :

```markdown
# Ghost — Configuration de mon blog

- **URL admin** : https://blog.tonsite.com/ghost
- **URL publique** : https://blog.tonsite.com
- **Hébergeur** : PikaPods (région Europe)
- **Date de création de l'instance** : YYYY-MM-DD
- **Mode privé activé** : oui / non
- **MCP Ghost installé dans Claude Code** : oui / non
- **Webhooks configurés** : non (à faire plus tard)

## Notes

- Les API Keys sont stockées dans [emplacement choisi par l'étudiant, sans valeur].
- Ne jamais committer les clés dans git.
```

Ce fichier sert de mémoire au projet — l'étudiant saura toujours d'où vient son blog et ce qui reste à faire.
