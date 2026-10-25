# dsh-asi1-orchestrator

Mode **orchestrateur ASI:One** sélectionnable à l'initiation d'un chat dans DeepSeek Harness (DSH) : les appels API du modèle coordinateur portent nativement `planner_mode: true` (corps) et l'en-tête `x-session-id`, pour que ASI1 délègue le travail à ses sous-agents au lieu de l'exécuter lui-même.

## Le problème

Dans DSH, ASI1 est servi par un provider OpenAI-compatible (`asi-cloud`) via la lib interne pi-ai. Trois obstacles empêchent aujourd'hui le mode orchestrateur natif :

1. `planner_mode` et `x-session-id` ne sont émis ni par DSH ni par pi-ai (aucun champ générique de passage de corps).
2. `app.asar` est scellé par `ElectronAsarIntegrity` : patcher le bundle casse le lancement et demanderait un re-codesign à chaque mise à jour.
3. ASI1 raisonne à profondeur maximale par défaut sur cette voie : sans réglage dédié, l'orchestrateur tend à faire lui-même le travail au lieu de déléguer.

## L'approche (zéro patch binaire)

- **Route clonée** `asi-cloud-planner` dans le patch profil (`llm-pi-ai.providers`) : `baseURL: http://127.0.0.1:<port>/v1`, `compat.supportsLongCacheRetention: true`, `cacheRetention: long`, en-tête marqueur `x-asi-planner: 1`.
- **Proxy local** (Node, bibliothèque standard uniquement) : pour les requêtes marquées, réécrit le corps (`planner_mode: true`, `x-session-id` dérivé de `prompt_cache_key`, knobs orchestrateur), relaie le streaming vers l'upstream avec l'`Authorization` d'origine, journalise des métadonnées uniquement.
- **Hébergement** : LaunchAgent macOS.
- **Sélection du mode** (v1) : choix de la route `asi-cloud-planner` dans le picker du composer avant le premier tour du chat.

Le contrat complet, les exigences de streaming, la politique de journalisation et les knobs sont détaillés dans [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).

## État du projet

**En conception — à implémenter.** La conception a été revue en lecture seule le 2026-10-25 (revue indépendante, verdict : viable, conditions intégrées dans le document d'architecture). Les lots d'implémentation sont décrits dans les issues :

- Proxy local Node (contrat de streaming et sécurité)
- Route clonée `cordis.patch.yml` + preset orchestrateur
- LaunchAgent et scripts d'installation
- Vérification de bout en bout
- Packaging en plugin DSH (après validation de la v1)

Le template d'instructions du mode orchestrateur est fourni dans [templates/asi1-orchestrator.template.md](templates/asi1-orchestrator.template.md).

## Conventions

- Branche de travail : `dev` ; intégration vers `main` par MR. Pas de commit direct sur `main` après l'amorçage.
- Répondre en français, identifiants techniques en anglais.

## Avertissements

- Projet communautaire d'exploration, **non affilié** à DeepSeek, Fetch.ai, Cudos ni à aucun autre fournisseur ; sans garantie.
- La clé `ASI_API_KEY` reste dans l'environnement DSH de l'opérateur : **jamais** dans un dépôt, un log ou un exemple de configuration.
- Le proxy n'est pas un dispositif d'authentification : `x-asi-planner: 1` est un marqueur de routage local, pas un secret.
