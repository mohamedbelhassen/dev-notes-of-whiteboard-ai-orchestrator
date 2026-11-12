# 05 — Projets, Canevas, Persistance & Synchronisation

Ce document spécifie la gestion du cycle de vie des espaces de travail utilisateurs (Projets, Canevas, Fils de discussion), la stratégie de persistance locale hors-ligne (IndexedDB / `localStorage`), le moteur de synchronisation asynchrone ([`SyncEngine`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/apps/web/src/lib/sync/sync-engine.ts)) et les mécanismes serveur de contrôle de concurrence et de protection contre la perte de données ([`crud.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/apps/functions/src/crud.ts) & [`packages/db/src/index.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/db/src/index.ts)).

---

## 1. Flux Utilisateurs (User Flows)

### 1.1 Organisation en Projets et Tableaux (Canvases)
1. **Création & Navigation** :
   - Un utilisateur organise son travail en **Projets** (`projects`), chaque projet pouvant contenir un ou plusieurs **Tableaux** (`canvases`) et **Fils de discussion** (`chat_threads`).
   - À l'ouverture d'un projet sans tableau, un canevas initial est créé automatiquement.
2. **Auto-Nommage Intelligent (`POST /v1/projects/:projectId/auto-name`)** :
   - Dès le premier échange significatif sur un projet intitulé *Untitled*, le client déclenche [`nameBlankBoard`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/apps/web/src/hooks/use-agent-session.ts#L172-L218) qui appelle `/v1/projects/:projectId/auto-name` ([`apps/functions/src/index.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/apps/functions/src/index.ts#L1026-L1085)).
   - Un appel LLM léger génère un titre court (2 à 5 mots) reflétant le sujet étudié et met à jour le nom du projet et du canevas uniquement s'ils portent encore leur nom par défaut.

### 1.2 Sauvegarde Locale Immédiate & Synchronisation Cloud (`SyncEngine`)
1. **Écriture Locale Prioritaire (Offline-First)** :
   - Chaque modification du canevas (dessin manuel de l'utilisateur ou patch appliqué par l'IA) est sauvegardée localement avec un debounce court via [`useCanvasPageController`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/apps/web/src/hooks/use-canvas-page-controller.ts) dans IndexedDB (Dexie) et `localStorage`.
2. **Mise en File d'Attente & Synchronisation Distante** :
   - [`SyncEngine`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/apps/web/src/lib/sync/sync-engine.ts) dépile en arrière-plan les mutations en attente (`canvas_save`, `thread_messages`, `asset_upload`) vers l'API Cloudflare Worker.
   - Les fichiers binaires intégrés au tableau (images uploadées, SVG mathématiques, enregistrements) sont téléversés vers Cloudflare R2 via `/v1/storage/upload` ([`apps/functions/src/index.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/apps/functions/src/index.ts#L298-L410)).
   - En cas de perte de connexion réseau ou d'erreur serveur transitoire (`5xx`, `429`), l'entrée reste persistée dans la file locale avec backoff exponentiel jusqu'au retour de la connectivité.

### 1.3 Versionnement, Détection de Conflits & Restauration
1. **Création de Versions (`POST /v1/canvases/:canvasId/versions`)** :
   - À chaque point de sauvegarde distant, le client envoie l'état de la scène (`scene`), son hash SHA-256 (`contentHash`) et l'identifiant de la version parente connue (`expectedHeadVersionId`).
2. **Résolution de Conflit (`409 Conflict`)** :
   - Si un autre onglet ou appareil a mis à jour le tableau entre-temps (`head_version_id !== expectedHeadVersionId`), le serveur renvoie `409 Conflict` avec la version distante actuelle afin que le client puisse réconcilier la scène sans écrasement silencieux.

---

## 2. Règles Métiers Actuelles (Business Rules)

### 2.1 Garde-fou Anti-Perte de Données (`checkBoardSave`)
Implémentée dans [`apps/functions/src/crud.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/apps/functions/src/crud.ts#L50-L92), la fonction `checkBoardSave` protège les tableaux contre les régressions accidentelles (ex. écrasement d'un grand tableau par un état vide au rechargement d'un onglet) :
- **Rejet d'Effacement Accidentel (`accidental_clear`)** : Si le tableau existant compte au moins **4 éléments** et que la nouvelle sauvegarde contient **0 élément** sans que le client n'ait explicitement passé `intent: "clear"`, la sauvegarde est bloquée avec HTTP `409 Conflict`.
- **Rejet de Réduction Massive (`suspicious_shrink`)** : Si le tableau existant compte au moins **10 éléments** et que la nouvelle sauvegarde perd **plus de 50 % des éléments** (`nextCount < prevCount * 0.5`) sans `intent: "clear"` ni `intent: "undo"`, la sauvegarde est bloquée avec HTTP `409 Conflict`.

### 2.2 Versionnement Atomique Compare-And-Swap (CAS)
Implémenté dans [`canvasVersionsRepo.create`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/db/src/index.ts#L261-L355) et la fonction RPC PostgreSQL `insert_canvas_version_if_head` :
- **Déduplication par Hash** : Si le `content_hash` de la nouvelle scène est identique à celui de la version de tête actuelle (`head_version_id`), aucune nouvelle ligne n'est créée dans `canvas_versions` (`deduplicated: true`).
- **Atomicité CAS** : La création de la nouvelle version et la mise à jour de `canvases.head_version_id` s'exécutent dans une transaction atomique vérifiant `expected_head_version_id`.

### 2.3 Politiques de Stockage Cloudflare R2 ([`packages/storage/src/index.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/storage/src/index.ts))
- **Cloisonnement Hiérarchique des Clés** : Tout objet R2 suit strictement le schéma `user/{userId}/project/{projectId}/canvas/{canvasId}/{kind}/{filename}` ([`buildAssetObjectKey`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/storage/src/index.ts#L19-L36)). Toute lecture ou écriture vérifie que le préfixe appartient bien au `userId` authentifié.
- **Limites de Taille et Types MIME ([`validateUploadPolicy`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/storage/src/index.ts#L47-L69))** :
  - Types autorisés : `image/png`, `image/jpeg`, `image/webp`, `image/svg+xml`, `application/json`, `application/pdf`, `application/zip`.
  - Taille maximale par fichier : **25 Mo** (`MAX_UPLOAD_BYTES = 25 * 1024 * 1024`).

---

## 3. Entités du Domaine (Domain Entities)

| Entité / Table | Fichier Source | Attributs & Rôle |
| :--- | :--- | :--- |
| [`Project`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/contracts/src/index.ts#L155-L163) | `packages/contracts/src/index.ts` | `id`, `userId`, `name`, `description`, `archivedAt`, `createdAt`, `updatedAt`. |
| [`Canvas`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/contracts/src/index.ts#L165-L175) | `packages/contracts/src/index.ts` | `id`, `projectId`, `userId`, `name`, `scene` (état JSON Excalidraw), `headVersionId`, `versionNumber`, `createdAt`, `updatedAt`. |
| [`CanvasVersion`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/contracts/src/index.ts#L204-L216) | `packages/contracts/src/index.ts` | `id`, `canvasId`, `versionNumber`, `parentVersionId`, `source` (`"autosave" \| "manual" \| "agent" \| "restore"`), `contentHash`, `summary`, `scene`, `createdAt`. |
| [`ChatThread`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/contracts/src/index.ts#L177-L185) & [`ChatMessage`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/contracts/src/index.ts#L187-L195) | `packages/contracts/src/index.ts` | Fils de discussion associés à un canevas et leurs messages (`role`, `content`, `metadata` incluant les appels d'outils et checkpoints). |
| [`Asset`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/contracts/src/index.ts#L197-L202) | `packages/contracts/src/index.ts` | Métadonnées d'un fichier stocké dans R2 : `id`, `projectId`, `canvasId`, `objectKey`, `mimeType`, `sizeBytes`. |
