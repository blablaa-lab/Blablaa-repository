---
name: we-finalise
description: Procédure Werocket de finalisation d'un site WordPress + Breakdance avant mise en ligne, exécutée via WP-CLI en SSH et Playwright. Couvre cookies (werocket tools), liens réseaux sociaux, alt/légendes/descriptions de la médiathèque, cohérence des alt côté Breakdance, responsive, meta title/description Rank Math, SEO local, GEO, modules Rank Math (Local SEO, Schema/données structurées, robots.txt, llms.txt), données structurées JSON-LD adaptées aux CPT et au secteur du site, favicon, Independent Analytics, et produit un rapport de finalisation. Utilise cette skill dès que l'utilisateur parle de finaliser, livrer, mettre en ligne, préparer la mise en prod, faire la checklist de fin de projet, "passer we-finalise", ou demande une seule de ces étapes (alt des images, meta Rank Math, données structurées / schema / rich results, favicon, responsive check, SEO local) sur un site Werocket — même s'il ne prononce pas le mot "finalisation".
---

# we-finalise — finalisation d'un site Werocket

Tu exécutes la checklist de fin de projet Werocket sur un site WordPress construit avec Breakdance. L'objectif : un site propre, techniquement prêt et correctement optimisé pour le SEO local, sans casser un contenu que le client a validé.

Accès au site : **WP-CLI via SSH uniquement** (alias `wp @<alias>`) pour tout ce qui est données et réglages, et **Playwright en local** pour tout ce qui doit être vu côté front (responsive, alt rendus, liens sociaux, favicon). Pas d'admin WP en navigateur.

## Deux règles qui priment sur tout le reste

1. **Le contenu visible n'est pas ta propriété.** Les métadonnées invisibles (alt, légendes, descriptions médias, meta title/description, réglages plugins, favicon) s'appliquent directement. Les textes visibles s'appliquent directement **sauf** sur les pages clés listées dans la config (`key_pages`) et jamais sur les pages légales : là, tu proposes dans le rapport, tu ne modifies pas. Pourquoi : un client qui découvre ses textes réécrits au lancement, c'est un ticket, pas de la valeur.
2. **Un site Breakdance ne stocke pas ses textes dans `post_content`.** Les pages Breakdance ont leur contenu dans la postmeta `breakdance_data` (JSON). Toute modification de texte passe par `scripts/bd-text-replace.sh` (remplacement exact dans l'arbre JSON), jamais par `wp post update --post_content` et jamais par `wp search-replace` (l'échappement JSON ferait rater les occurrences). Tu ne changes que du texte, jamais la structure des éléments.

## Étape 0 — Intake et sauvegarde

Cherche `we-finalise.json` à la racine du projet. S'il n'existe pas, pose les questions manquantes en une seule fois, puis crée-le :

```json
{
  "wp_alias": "@staging",
  "site_url": "https://staging.client.fr",
  "client": {
    "name": "Nom commercial",
    "legal_name": "Raison sociale",
    "sector": "ostéopathe",
    "business_type": "Physician",
    "city": "Lyon",
    "zone": ["Lyon", "Villeurbanne", "Caluire"],
    "address": { "street": "", "postal_code": "", "region": "", "country": "FR" },
    "phone": "",
    "email": "",
    "opening_hours": [],
    "keywords": ["ostéopathe lyon", "ostéopathe villeurbanne"],
    "social": { "facebook": "", "instagram": "", "linkedin": "" }
  },
  "key_pages": ["accueil"],
  "legal_pages": ["mentions-legales", "politique-de-confidentialite"],
  "post_types": ["page", "post"],
  "logo_source": ""
}
```

`wp_alias` doit exister dans le `wp-cli.yml` du projet :

```yaml
@staging:
  ssh: user@host/chemin/vers/wordpress
```

Vérifie la connexion : `wp @staging option get blogname`. `business_type` est un type Schema.org accepté par Rank Math (voir `references/rankmath.md`). Prérequis locaux : `wp`, `jq`, `node` ≥ 18, ImageMagick (favicon), Playwright (étape 4).

Puis **sauvegarde la base avant toute écriture** : `wp @staging db export we-finalise-backup-$(date +%Y%m%d).sql` (le fichier reste sur le serveur, note son chemin dans le rapport). Sans ce backup, n'écris rien.

Crée un dossier de travail local `.we-finalise/` (à ajouter au `.gitignore`) pour les JSON intermédiaires et les captures.

## Étape 1 — Inventaire

```bash
scripts/list-urls.sh @staging page post <cpts>   # → .we-finalise/urls.json
wp @staging plugin list --format=json            # → état des plugins
scripts/media-audit.sh @staging                  # → .we-finalise/media.json
scripts/schema-audit.sh @staging                 # → .we-finalise/schema.json
```

`urls.json` liste chaque contenu publié (ID, type, titre, slug, URL, `breakdance` oui/non) plus les templates Breakdance (header, footer, templates, popups — `template: true`, sans URL) et une entrée `_meta` avec la clé postmeta Breakdance détectée, `blog_public` et `site_icon`. `media.json` liste chaque image de la médiathèque avec alt, légende, description, URL, et les posts qui l'utilisent (détection dans `post_content` et dans `breakdance_data`). C'est la base des étapes 3 et 5.

`schema.json` liste les post types publics réellement présents (slug, label, nombre de contenus publiés, `has_archive`, taxonomies, schema Rank Math déjà configuré) plus l'état du module Schema. C'est ce qui pilote l'étape 7 — et c'est aussi là que tu découvres les CPT métier du site : si `post_types` de la config ne les contient pas tous, complète-le et relance `list-urls.sh` avec la liste complète, sinon des pages entières échapperont aux étapes 3 à 6.

## Étape 2 — Cookies (plugin werocket tools)

Vérifie que le plugin est actif (`wp plugin list`, nom contenant `werocket`). Active-le s'il est installé mais inactif. Puis vérifie que le module Cookies est activé — la procédure exacte est dans `references/werocket-tools.md`. Si cette référence ne documente pas encore l'option, découvre-la : `wp option list --search='*werocket*'`, puis lecture du code du plugin (`wp plugin path <slug> --dir` puis `ssh … grep -rn "cookie" <path> --include=*.php | head`). Documente ce que tu trouves dans la référence pour la prochaine fois. Si le plugin n'est pas installé, c'est un bloquant : signale-le et continue.

## Étape 3 — Médiathèque : alt, légende, description

Priorise les images **utilisées** sur le site (champ `used_in` non vide dans `media.json`), puis le reste. Pour chaque image : regarde-la réellement (télécharge-la via son URL, ouvre-la), puis rédige les trois champs selon `references/redaction-seo-local-geo.md` (section « Médias »). En bref : alt descriptif et factuel, ≤ 125 caractères, mention de la ville seulement quand elle est pertinente pour cette image, zéro « image de », zéro bourrage de mots-clés ; légende courte ; description en une ou deux phrases. Les images décoratives (formes, textures, séparateurs) reçoivent un alt vide explicite (`""`), pas un alt inventé.

Écris tes propositions dans `.we-finalise/media-updates.json` (`[{ "id": 12, "alt": "…", "caption": "…", "description": "…" }]`) puis applique : `scripts/apply-media.sh @staging .we-finalise/media-updates.json`. Ne réécris pas un alt existant qui est déjà correct — le script n'écrase que les champs présents dans ton JSON.

## Étape 4 — Audit front (responsive, alt rendus, liens sociaux, favicon)

```bash
node scripts/front-audit.mjs --urls .we-finalise/urls.json --media .we-finalise/media.json --out .we-finalise/front
# staging protégé par htaccess : --auth user:pass · une seule page : --only slug
```

Prérequis : `npm i -D playwright && npx playwright install chromium` dans le projet (une fois). Le script visite chaque URL à 375, 768, 1024 et 1440 px, capture des screenshots pleine page, et écrit `.we-finalise/front/report.json` contenant par page :

- `overflow` : largeur de scroll > largeur de viewport (débordement horizontal) par breakpoint ;
- `images` : chaque `<img>` rendu avec `src`, `alt`, et `alt_mismatch: true` quand l'alt rendu diffère de celui de la médiathèque (Breakdance a un alt personnalisé ou vide sur l'élément) ;
- `social_links` : tous les liens vers facebook/instagram/linkedin/tiktok/youtube/x, avec leur emplacement (header/footer/contenu) ;
- `favicon` : présence de `<link rel="icon">` et code HTTP de son URL ;
- `head` : title et meta description rendus.

Ensuite : **regarde les screenshots**, breakpoint par breakpoint, pas seulement le JSON. Un `overflow: false` n'exclut pas un texte tronqué, une image écrasée ou un menu illisible. Note chaque problème visuel avec page + breakpoint + description précise. Ces corrections se font dans Breakdance par un humain : elles vont dans le rapport, section « À corriger dans Breakdance », pas dans une modification automatique.

Liens sociaux : compare chaque lien trouvé à `client.social`. Un lien vide, un `#`, un lien vers le profil d'un autre client ou vers le réseau générique (`https://facebook.com`) est une erreur à corriger. Si le lien est dans le header/footer (donc un template Breakdance), il est corrigeable via `scripts/bd-text-replace.sh` sur le template concerné — sinon rapport.

Alt : pour chaque `alt_mismatch`, corrige l'élément Breakdance pour qu'il reprenne l'alt de la médiathèque. Si l'alt personnalisé Breakdance est meilleur que celui de la médiathèque, c'est la médiathèque qu'il faut aligner (étape 3), pas l'inverse : la médiathèque est la source de vérité.

## Étape 5 — SEO local et GEO : metas et contenus

Lis d'abord `references/redaction-seo-local-geo.md` en entier. Puis, pour chaque URL publique de `urls.json` :

1. Récupère le contenu textuel rendu (déjà dans `front/report.json` → `text`) et la meta actuelle (`wp post meta get <ID> rank_math_title`, `rank_math_description`, `rank_math_focus_keyword`).
2. Rédige meta title (≤ 60 caractères, mot-clé + ville, marque en fin), meta description (140–155 caractères, bénéfice + localité + appel à l'action) et focus keyword. Écris-les dans `.we-finalise/seo-meta.json` et applique : `scripts/apply-seo-meta.sh @staging .we-finalise/seo-meta.json`. Les metas s'appliquent partout, pages clés incluses.
3. Analyse le texte de la page contre la grille GEO de la référence (réponse directe en tête, Hn qui posent la question de l'utilisateur, entités locales nommées, FAQ, données concrètes). Rédige tes modifications de texte comme des paires exactes `{ "post_id": 12, "from": "texte exact actuel", "to": "nouveau texte" }` dans `.we-finalise/content-updates.json`.
4. Teste puis applique : `scripts/bd-text-replace.sh @staging .we-finalise/content-updates.json --dry-run`, puis sans `--dry-run`. Le script écrit dans l'arbre Breakdance quand la page en a un, dans `post_content` sinon (articles Gutenberg) — **sauf** pour les `key_pages` et `legal_pages` : ces paires ne vont pas dans `content-updates.json` mais dans le rapport, section « Propositions de contenu à valider ».

Le script refuse une paire dont le `from` n'est pas trouvé exactement une fois dans le post ciblé : vérifie alors ton texte source (guillemets typographiques, espaces insécables, apostrophes ’ vs ') et, au besoin, dump les chaînes de l'arbre (voir `references/breakdance.md`).

Après toute écriture dans `breakdance_data` : `wp @staging cache flush` puis purge du plugin de cache s'il y en a un (`wp plugin list` te dira lequel ; voir `references/breakdance.md`).

## Étape 6 — Modules Rank Math

Suis `references/rankmath.md` pas à pas. Résumé :

- Mode avancé activé (sinon les modules n'apparaissent pas).
- Module **Schema (Structured Data)** activé — slug `rich-snippet`. L'activation et la configuration par CPT sont l'étape 7 ; ici, vérifie seulement qu'il est dans `rank_math_modules`.
- Module **Local SEO** activé et fiche remplie depuis `client` : type d'organisation, nom, logo, adresse, téléphone, email, horaires, zone. Le champ `local_business_type` sort de la table secteur → type de `references/schema.md` §1 : c'est lui qui décide des rich results, `LocalBusiness` générique n'en active presque aucun.
- **robots.txt** : contenu vérifié via Rank Math, avec `Sitemap:` pointant sur le sitemap Rank Math, sans `Disallow: /` résiduel de la phase de dev.
- Module **LLMs.txt** activé, post types publics inclus, puis `curl -s <site_url>/llms.txt` pour vérifier que le fichier répond et liste les bonnes pages.
- Vérifier aussi que `blog_public` vaut `1` (`wp option get blog_public`) — un site laissé en « demander aux moteurs de ne pas indexer » est l'erreur de mise en ligne la plus fréquente et la plus coûteuse.

## Étape 7 — Données structurées (Schema)

Lis `references/schema.md` avant d'écrire quoi que ce soit : `rankmath.md` §4 donne la mécanique,
`schema.md` donne le seul choix qui compte — quel type sur quel contenu. Trois principes y
gouvernent tout : une entité d'entreprise unique et bien sous-typée, un type juste par CPT, et
**aucune donnée inventée**. Un `aggregateRating` sans avis réels ou des horaires approximatifs sont
la seule partie de cette procédure qui peut coûter une pénalité manuelle au client : une propriété
absente est neutre, une propriété fausse est un risque.

1. **Audit** : `scripts/schema-audit.sh @staging` → `.we-finalise/schema.json`. Tu obtiens l'état du
   module, le `local_business_type` en place, les post types **réellement présents** avec leur
   nombre de contenus publiés et leur schema par défaut actuel, les schemas déjà posés à la main, et
   les noms de clés d'options tels qu'ils existent dans cette version du plugin.
2. **Sous-type de l'entreprise** : compare `local_business_type` à la table secteur → type de
   `schema.md` §1. `LocalBusiness` générique ou un type absent alors que le secteur en a un précis,
   c'est la correction la plus rentable de l'étape (elle se fait via la fiche Local SEO, étape 6).
   Cas particuliers traités dans la référence : praticien seul sous son nom, service sans zone
   physique.
3. **Mapping CPT → schema** : pour chaque post type de l'audit (`published > 0`), choisis le type
   avec la table de `schema.md` §2. Le réflexe à casser est le `article` que Rank Math met par
   défaut partout : sur une page de service ou un CPT « réalisations », c'est un contresens. `off`
   vaut toujours mieux qu'un type approximatif. Les CPT techniques (slider, popup, formulaire)
   passent en `off` et doivent être `noindex`.
4. **Pages prestation** : ce sont elles qui rapportent. Un `Service` par page, avec `provider.@id`
   pointant sur l'entité de l'accueil, `areaServed` tiré de `client.zone`, et pas d'`offers` sans
   tarifs affichés (`schema.md` §3). Récupère le `@id` réellement émis dans le JSON-LD de l'accueil
   avant de le recopier : il varie selon la version de Rank Math.
5. **Plan et application** : écris `.we-finalise/schema-plan.json` (format dans `rankmath.md` §4),
   puis `scripts/apply-schema.sh @staging .we-finalise/schema-plan.json --dry-run` — **lis le diff
   `from` → `to`** — et enfin sans `--dry-run`. Le plan active le module `rich-snippet` (`"module":
   true`) et le fil d'Ariane, écrit les défauts par post type et pose les schemas par contenu.
   `replace: true` efface les `rank_math_schema_*` existants du contenu : ne l'utilise que si
   `existing_schemas` de l'audit ne contient rien d'écrit à la main par un humain.
6. **Validation sur le JSON-LD rendu**, pas sur la config : `wp @staging cache flush`, purge du
   plugin de cache, puis la grille de `schema.md` §5 sur l'accueil et un exemplaire de chaque CPT.
   Une seule entité d'entreprise, `BreadcrumbList` sur les pages profondes, aucun `%placeholder%`
   non résolu, aucun `Article` sur une page de service, aucun graphe concurrent émis par le thème
   ou WooCommerce.

Reporte le sous-type retenu, le mapping CPT → schema appliqué, et surtout ce que tu as
**volontairement laissé de côté** faute de données (avis, tarifs, horaires) : cette liste est ce
que le client doit fournir pour aller plus loin.

## Étape 8 — Favicon

Si `front/report.json` indique un favicon présent et fonctionnel (HTTP 200) et que `site_icon` est défini dans `urls.json → _meta`, passe. Sinon : `scripts/make-favicon.sh @staging <logo_source>` génère `.we-finalise/favicon-512.png` à partir du logo. **Regarde le PNG.** S'il est lisible, relance avec `--apply` : import en médiathèque et `site_icon`. Le logo source vient de `logo_source` dans la config, sinon de `wp theme mod get custom_logo`, sinon de l'image du header Breakdance (voir `references/breakdance.md`). Un logotype horizontal réduit en carré est illisible : dans ce cas, isole le symbole si le logo en a un, sinon signale dans le rapport qu'un favicon dédié est à demander au client — ne mets pas un favicon moche en prod.

## Étape 9 — Independent Analytics

`wp plugin list` : si `independent-analytics` est absent, `wp plugin install independent-analytics --activate` ; s'il est inactif, active-le. Puis autorise le rôle Éditeur à voir les statistiques : l'option est dans les réglages du plugin ; découvre sa clé avec `wp option list --search='iawp*'` (cherche une option contenant `role`, `permission` ou `access`), lis sa valeur, ajoute `editor`, réécris-la. Documente la clé trouvée dans `references/independent-analytics.md`.

## Étape 10 — Rapport

Génère `we-finalise-report.md` à la racine du projet à partir de `references/report-template.md`. Le rapport est le livrable : il liste ce qui a été vérifié, ce qui a été modifié (avec compte et exemples), ce qui reste à faire par un humain dans Breakdance, et les propositions de contenu en attente de validation client. Termine par les bloquants éventuels (plugin manquant, `blog_public` à 0, favicon à demander).

## Ordre et interruptions

Les étapes 1 → 10 s'enchaînent ; l'étape 4 dépend de 1 et 3 (comparaison des alt), et l'étape 7 de l'inventaire des post types (étape 1) et de la fiche Local SEO (étape 6). Si l'utilisateur demande une seule étape (« fais juste les metas Rank Math »), fais l'intake minimal nécessaire (config + backup), l'inventaire, l'étape demandée, et un rapport réduit à cette étape. Ne saute jamais le backup.

Quand une commande échoue en SSH (timeout, permissions), ne contourne pas en supposant le résultat : signale, propose la correction d'accès, et marque l'étape « non vérifiée » dans le rapport.
