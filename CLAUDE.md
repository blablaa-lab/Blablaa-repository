# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## Orchestration des Workflows

### 1. Mode Planification par Défaut
- Passer en mode planification pour TOUTE tâche non triviale (3+ étapes ou décisions architecturales)
- Si quelque chose déraille, STOPPER et replanifier immédiatement — ne pas s'acharner
- Utiliser le mode planification pour les étapes de vérification, pas uniquement pour la construction
- Rédiger des specs détaillées en amont pour réduire les ambiguïtés

### 2. Stratégie de Sous-agents
- Utiliser les sous-agents généreusement pour garder la fenêtre de contexte principale propre
- Déléguer la recherche, l'exploration et les analyses parallèles aux sous-agents
- Pour les problèmes complexes, augmenter la puissance de calcul via les sous-agents
- Une approche par sous-agent pour une exécution ciblée

### 3. Boucle d'Auto-amélioration
- Après TOUTE correction de l'utilisateur : mettre à jour `_tasks/lessons.md` avec le pattern identifié
- Écrire des règles pour soi-même qui empêchent la même erreur de se reproduire
- Itérer sans relâche sur ces leçons jusqu'à ce que le taux d'erreur diminue
- Réviser les leçons en début de session pour chaque projet concerné

### 4. Vérification Avant Clôture
- Ne jamais marquer une tâche comme terminée sans en prouver le bon fonctionnement
- Comparer le comportement entre la version principale et les modifications apportées si pertinent
- Se demander : *"Un développeur senior validerait-il ce code ?"*
- Exécuter les tests, vérifier les logs, démontrer la correction

### 5. Exiger l'Élégance (avec mesure)
- Pour les modifications non triviales : marquer une pause et se demander *"existe-t-il une approche plus élégante ?"*
- Si un correctif semble bricolé : *"En tenant compte de tout ce que je sais, implémenter la solution élégante"*
- Passer outre pour les corrections simples et évidentes — ne pas sur-ingéniérer
- Challenger son propre travail avant de le présenter

### 6. Correction Autonome des Bugs
- Face à un rapport de bug : le corriger, point. Ne pas attendre qu'on vous prenne par la main
- Pointer les logs, erreurs et tests en échec — puis les résoudre
- Zéro changement de contexte requis de la part de l'utilisateur
- Corriger les tests CI en échec sans attendre d'instructions

---

## Gestion des Tâches

1. **Planifier d'abord** : Écrire le plan dans `_tasks/todo.md` avec des éléments cochables
2. **Valider le plan** : Faire un point avant de démarrer l'implémentation
3. **Suivre l'avancement** : Marquer les éléments comme terminés au fur et à mesure
4. **Expliquer les changements** : Résumé de haut niveau à chaque étape
5. **Documenter les résultats** : Ajouter une section de bilan dans `_tasks/todo.md`
6. **Capitaliser les leçons** : Mettre à jour `_tasks/lessons.md` après chaque correction

---

## Principes Fondamentaux

- **Simplicité avant tout** : Rendre chaque changement aussi simple que possible. Impacter le moins de code possible.
- **Pas de paresse** : Trouver les causes profondes. Pas de correctifs temporaires. Standards développeur senior.
- **Impact minimal** : Les modifications ne doivent toucher que le strict nécessaire. Éviter d'introduire de nouveaux bugs.