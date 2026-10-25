# dsh-asi1-orchestrator

Plugin DSH portant les options de proxying du mode orchestrateur ASI1 (ASI:One `planner_mode`). Version plugin uniquement : ce dépôt ne contient pas l'intégration opérationnelle (proxy autonome, route clonée, LaunchAgent, e2e), réalisée dans [heraisys-sas/reprise-harness](https://github.com/heraisys-sas/reprise-harness).

## Options de proxying à porter par le plugin

1. **Marquage orchestrateur** : détecter le header `x-asi-planner` sur les requêtes du profil ASI1 ; réécrire le corps avec `planner_mode: true`. Requêtes sans marqueur : non traitées par le chemin planner.
2. **Identité de session** : `x-session-id` dérivé du `prompt_cache_key` émis par pi-ai (`clampOpenAIPromptCacheKey(sessionId)`, ≤64 chars, stable par chat) ; écraser tout `x-session-id` entrant ; supprimer `prompt_cache_key` avant envoi upstream.
3. **Knobs orchestrateur** : `reasoning_effort: "low"`, `temperature: 0.25` (plage 0.2–0.3) ; pas de `tool_choice` forcé ; pas de réduction de `max_tokens`.
4. **Contrat ASI:One** : instructions orchestrateur en `role: "system"` dans `messages[0]` ; outil de délégation unique ; résultats des workers en `role: "tool"` avec le tool_call id correspondant ; `use_reasoning_history: true` (replay verbatim de `reasoning_content`).
5. **Streaming & sécurité** : relais binaire avec backpressure ; hop-by-hop strippés, Host réécrit, pas de suivi de redirection portant `authorization` ; timeouts distincts connect 2 s / headers 10 s / idle 120 s / total 600 s ; annulation upstream à la déconnexion client ; statuts d'erreur préservés ; pas de faux `[DONE]` ; **aucun retry** (`maxRetries: 0` conservé) ; bind 127.0.0.1 uniquement ; logs JSONL métadonnées seules (0o600, jamais de corps ni de secret).

Détail complet du contrat : [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).

## Statut

Spécification. Implémentation plugin : issue #5. Intégration opérationnelle : dépôt reprise-harness.
