# Tâche — we-finalise : données structurées Rank Math

## Objectif
Activer le module Schema (Structured Data) de Rank Math dans la procédure we-finalise
et modéliser les données structurées en fonction des CPT présents et du type de site.

## Plan

- [x] 1. `scripts/php/schema-audit.php` + `scripts/schema-audit.sh`
      Découvre les post types publics (slug, label, nb publiés, has_archive, taxonomies)
      et lit la config schema Rank Math en place (module actif, défauts par post type,
      postmeta schema par contenu). → `.we-finalise/schema.json`
- [x] 2. `scripts/php/apply-schema.php` + `scripts/apply-schema.sh`
      Active le module `rich-snippet`, écrit les défauts par post type, applique les
      schemas par contenu (postmeta `rank_math_schema_*`), `--dry-run` obligatoire d'abord.
- [x] 3. `references/schema.md` (nouveau)
      Doctrine de modélisation : arbre schema d'un site vitrine local, mapping
      secteur → sous-type LocalBusiness, mapping CPT → type schema, imbrication @id,
      FAQPage / Service / Person / Article, règles anti-invention, validation.
- [x] 4. `references/rankmath.md`
      Nouvelle section « Données structurées » : clés d'options, breadcrumbs, renumérotation.
- [x] 5. `SKILL.md`
      Étape 1 : découverte des CPT. Étape 6 : sous-étape données structurées.
      Frontmatter `description` : mentionner schema / données structurées (déclenchement).
- [x] 6. Re-packager `we-finalise.skill` (zip) + synchroniser le `SKILL.md` à côté
- [x] 7. Vérification : bash -n sur les scripts, php -l sur les PHP, jq sur les JSON d'exemple,
      cohérence des chemins référencés dans SKILL.md vs contenu du bundle

## Bilan

Tout appliqué et vérifié.

**Nouveaux fichiers dans le bundle**
- `scripts/schema-audit.sh` + `scripts/php/schema-audit.php` — découverte des post types
  réellement présents (publiés, archives, taxonomies) + état de la config Schema Rank Math,
  y compris les noms de clés d'options tels qu'ils existent dans la version installée.
- `scripts/apply-schema.sh` + `scripts/php/apply-schema.php` — active `rich-snippet`,
  écrit les défauts par post type sans écraser les clés voisines de l'option sérialisée,
  pose les schemas par contenu (`rank_math_schema_<Type>` avec le bloc `metadata` requis),
  `--dry-run` renvoyant les couples `from` → `to`.
- `references/schema.md` — doctrine : table secteur → sous-type `LocalBusiness` (~25 secteurs),
  table CPT → type de schema, `Service` sur les pages prestation avec `provider.@id`,
  les quatre pièges (avis inventés, FAQPage non visible, doublons, horaires faux), validation.

**Fichiers modifiés**
- `references/rankmath.md` — section 4 « Données structurées », renumérotation 4→5 … 7→8.
- `references/report-template.md` — ligne de synthèse + section « Données structurées »
  + contrôle du JSON-LD sur le domaine de prod.
- `SKILL.md` — étape 1 : audit schema et complétion des CPT dans la config ; étape 6 :
  module Schema + `local_business_type` issu de la table ; **nouvelle étape 7** dédiée ;
  renumérotation 7→8 (favicon), 8→9 (analytics), 9→10 (rapport) ; frontmatter enrichi
  pour le déclenchement sur « données structurées / schema / rich results ».

**Vérifications faites**
- `bash -n` sur les 8 scripts shell, `php -l` sur les 7 scripts PHP : OK.
- Garde jq de `apply-schema.sh` testée sur un plan valide (accepté) et un schema sans
  `@type` (rejeté avant envoi) ; injection de `dry_run` vérifiée.
- Les 16 fichiers `scripts/*` et `references/*` cités dans `SKILL.md` et les références
  existent tous dans le bundle.
- Frontmatter YAML parsé, `description` à 943 caractères (sous la limite).
- Bundle re-zippé puis ré-extrait : identique à la source de travail, et le `SKILL.md`
  du zip est identique à celui posé à côté.

**Non vérifié — et pourquoi**
Aucun site WordPress n'était joignable dans cette session : les scripts n'ont pas été
exécutés contre un Rank Math réel. Les clés d'options (`pt_<pt>_default_rich_snippet`,
`rank_math_schema_<Type>`, `metadata.isPrimary`) varient selon la version du plugin — c'est
exactement pourquoi `schema-audit.sh` renvoie la liste des clés réellement présentes et
pourquoi `apply-schema.sh` impose un `--dry-run` lisible avant écriture. À confirmer au
premier passage sur un site client.
