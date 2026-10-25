# Architecture — mode orchestrateur ASI:One dans DeepSeek Harness

Cible vérifiée : DSH `0.2.1-rc.2` (app Electron macOS). Les numéros de ligne cités le sont dans la version extraite de ce build ; à revérifier à chaque mise à niveau.

## 1. Contrat ASI:One (confirmé côté opérateur)

- Mode orchestrateur = champ de corps `planner_mode: true` + en-tête `x-session-id` (array `agents` optionnel).
- Les instructions orchestrateur doivent être en `role: "system"` dans `messages[0]`.
- `reasoning_effort: "low"` sur les appels orchestrateur (ASI1 raisonne toujours ; profondeur maximale par défaut → symptôme « l'orchestrateur fait lui-même le travail »).
- Un seul outil de délégation ; les résultats des travailleurs reviennent en `role: "tool"` avec le `tool_call_id` correspondant.
- `use_reasoning_history: true` (replay verbatim du `reasoning_content`) ; température basse (0.2–0.3).

## 2. Contraintes DSH 0.2.1-rc.2 (faits vérifiés dans le code)

1. `Info.plist` scelle `Resources/app.asar` (`ElectronAsarIntegrity`, SHA-256). Toute modification de l'asar casse le lancement sans re-codesign ad-hoc, à refaire à chaque mise à jour → patch asar écarté (plan B seulement).
2. Les profils provider (`cordis.patch.yml`, dict `llm-pi-ai.providers`) sont clonables : `resolveProfiles` propage les champs inconnus (`...rest`), `assertValidHeaders` accepte tout en-tête Fetch valide, `compat.supportsLongCacheRetention` et profil `cacheRetention: "long"` sont reconnus et copiés vers pi-ai.
3. pi-ai `buildParams` émet `prompt_cache_key = clampOpenAIPromptCacheKey(options?.sessionId)` quand `cacheRetention === "long" && compat.supportsLongCacheRetention` ; clamp = identité jusqu'à 64 caractères ; `options.sessionId` = identifiant de session DSH, stable sur la durée du chat. `reasoning_effort` n'est jamais émis sur cette voie ; `onPayload` existe comme hook corps mais n'est pas branché par DSH.
4. La création de session ne porte pas de modèle ; le picker du composer → RPC `session/selectModel`, qui persiste aussi le choix comme **défaut global** (effet de bord accepté en v1).

## 3. Architecture

```
Chat (route asi-cloud-planner choisie au picker)
  └─ pi-ai (openai-completions) ── POST http://127.0.0.1:<port>/v1/chat/completions
       en-tête x-asi-planner: 1 ; corps avec prompt_cache_key = sessionId
       └─ Proxy local Node (stdlib uniquement)
            ├─ marqueur présent ? → réécriture : planner_mode: true,
            │   x-session-id = prompt_cache_key, retrait prompt_cache_key/stream_options,
            │   injection reasoning_effort + temperature
            ├─ relais streaming (SSE) → https://inference.asicloud.cudos.org/v1/chat/completions
            │   avec l'Authorization d'origine
            └─ journal JSONL : métadonnées uniquement
```

- Route normale `asi-cloud` inchangée (jamais détournée).
- Catalogue de la route planner limité au modèle `asi1`.
- Instructions orchestrateur (`role: "system"` dans `messages[0]`) et **un seul outil de délégation** : portés par le preset/plugin, pas par le proxy — le proxy valide le payload final mais n'écrit pas le prompt.

## 4. Contrat du proxy (exigences de la revue indépendante, 2026-10-25)

### 4.1 Relais et streaming

- Relais **octet par octet** avec backpressure : ne jamais supposer qu'un chunk réseau correspond à un événement SSE.
- Recalcul de `Content-Length` après réécriture du corps ; suppression des en-têtes hop-by-hop et de ceux nommés dans `Connection` ; `Host` upstream correct.
- Compression cohérente : ne jamais relayer un corps décompressé avec les anciens `Content-Encoding`/`Content-Length`.
- Délais distincts : connexion, réception des en-têtes, inactivité, durée totale ; annulation de la requête upstream à la déconnexion du client.
- Statuts d'erreur conservés avant ouverture du flux ; après ouverture, interruption explicite **sans faux `[DONE]`**.
- **Aucun retry** d'une inférence susceptible d'avoir déjà déclenché une délégation. `maxRetries: 0` est déjà posé dans le profil asi-cloud ; vérifier les retries éventuels des autres couches.

### 4.2 Sécurité et journalisation

- Journal JSONL en **métadonnées seules** : statut, durées, compteurs, identifiant de corrélation opaque. Jamais le corps des requêtes/réponses, jamais `Authorization`, jamais les erreurs qui recopieraient la requête.
- Listener lié à `127.0.0.1` uniquement ; upstream HTTPS fixe (`inference.asicloud.cudos.org`), chemin autorisé unique (`/v1/chat/completions`) ; ne pas suivre une redirection transportant l'autorisation.
- `x-asi-planner: 1` est un marqueur de routage local, pas une authentification.
- Journal : permissions restrictives, rotation, rétention bornée.

### 4.3 Identité de session (`prompt_cache_key` → `x-session-id`)

- La passerelle est une **dépendance versionnée** (émission conditionnée à deux réglages ; clamp garanti seulement ≤ 64 caractères).
- Refuser localement (erreur claire, sans appel upstream) toute requête planner sans `prompt_cache_key` utilisable.
- **Jamais** de valeur globale ni d'identifiant aléatoire par appel.
- Écraser tout `x-session-id` entrant avec la valeur validée, puis retirer `prompt_cache_key` du corps.
- Ne pas imposer d'UUID sans preuve du contrat ASI:One ; si un format exigeait une transformation, passer par une correspondance déterministe (namespace stable) ou une table durable.

## 5. Knobs orchestrateur

- `reasoning_effort: "low"` et `temperature: 0.25` (dans l'intervalle 0.2–0.3) : **valeurs initiales à mesurer**, pas des acquis. Critère : l'orchestrateur délègue au lieu d'exécuter, sans perte de qualité de coordination.
- Pas de `tool_choice` forcé ; pas de réduction de `max_tokens` (une délégation peut être longue).

## 6. Hébergement — LaunchAgent

- Clés : `RunAtLoad`, `KeepAlive` explicite, `ThrottleInterval`, logs vers un chemin dédié. **Pas de `WatchPaths`** (relance au moindre changement de fichier : indésirable).
- Pas besoin de `ASI_API_KEY` dans le plist : le proxy relaie l'`Authorization` d'origine.
- Pièges à traiter : port occupé (échec net avec log lisible), boucle de crash (ThrottleInterval), mise à jour de DSH (le proxy ne dépend pas du bundle).
- Le plist et les scripts d'installation sont des artefacts vérifiés ; leur pose sur la machine est une opération opérateur.

## 7. Sélection du mode

- **v1** : picker du composer sur la route `asi-cloud-planner` avant le premier tour. Caveat accepté : le défaut global bascule sur la route planner après un chat orchestrateur (`session/selectModel` persiste le choix).
- **v2 (piste sans patch)** : plugin DSH qui route par session sur l'événement `agent/request` ; à explorer seulement après validation de la v1.

## 8. Vérifications attendues

1. Faux upstream produisant du SSE fragmenté, UTF-8 coupé, compression, client lent, silence prolongé, 401/429/500, coupure après un premier événement → comportement conforme au §4.1.
2. Secrets factices injectés dans chaque champ puis erreurs provoquées → aucun secret dans les logs.
3. Chat orchestrateur → journal prouvant `planner_mode: true` et `x-session-id` stable par chat ; instructions en `role: "system"` dans `messages[0]` ; un seul outil de délégation.
4. Chat sur la route normale `asi-cloud` → aucun passage par le proxy, comportement inchangé.
5. Continuité : plusieurs tours d'un même chat partagent le même `x-session-id` ; deux chats simultanés ont des identifiants distincts.

## 9. Plan B (conservé pour mémoire)

Patch micro de l'asar (`dsh-llm-pi-ai/lib/index.js` : injection `planner_mode`/en-têtes au point d'appel pi-ai ~:1983-1990) suivi d'un re-codesign ad-hoc. Écarté : le scellement `ElectronAsarIntegrity` fait perdre le patch à chaque mise à jour de l'app.
