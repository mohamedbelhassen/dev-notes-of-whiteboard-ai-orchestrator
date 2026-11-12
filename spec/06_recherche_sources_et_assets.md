# 06 — Ingestion Documentaire, Recherche Web & Bibliothèque d'Assets

Ce document spécifie les mécanismes permettant à InkLM d'ancrer ses explications et ses schémas dans des sources externes ou des ressources visuelles :
1. **L'ingestion et l'interrogation de documents de cours** ([`research-tools.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/research-tools.ts) & `/v1/core/research/ingest` dans [`apps/functions/src/index.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/apps/functions/src/index.ts#L955-L1024)).
2. **La recherche web vérifiée avec citations** ([`sources.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/sources.ts)).
3. **La bibliothèque de composants vectoriels Excalidraw (`searchLibraryAssets`) et la recherche d'images éducatives (`searchImages`)** ([`tools.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/tools.ts#L469-L671)).

---

## 1. Flux Utilisateurs (User Flows)

### 1.1 Ingestion et Analyse de Documents de Cours (PDF / arXiv / Texte)
1. **Ajout d'une Source par l'Utilisateur** :
   - L'utilisateur dépose un fichier PDF, colle une URL (dont les liens `arxiv.org/abs/...` ou `arxiv.org/pdf/...`) ou saisit des notes de cours dans le panneau *Sources*.
   - Le client appelle `POST /v1/core/research/ingest` ([`apps/functions/src/index.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/apps/functions/src/index.ts#L955-L1024)).
2. **Extraction, Découpage et Indexation dans R2** :
   - Le backend extrait le texte (et la table des matières / sections), découpe le document en passages numérotés et stocke le document structuré ainsi que le manifeste du projet (`ResearchManifest`) dans le bucket Cloudflare R2 (`ASSETS`).
3. **Consultation Outillée par l'Agent pendant le Chat** :
   - Lorsqu'un `researchManifest` est attaché à la session, l'orchestrateur active 5 outils de recherche documentaire ([`createResearchTools`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/research-tools.ts#L215-L398)) :
     - `listResearchSources` : liste les documents disponibles et leur structure.
     - `getResearchSourceOutline` : inspecte les titres et sections d'un document.
     - `searchResearchSources` : recherche par mots-clés (scoring lexical TF/IDF simplifié) à travers tous les passages.
     - `readResearchPassage` : lit le contenu complet d'un passage spécifique et des passages adjacents.
     - `searchSources` : effectue une recherche web externe si les documents fournis ne suffisent pas.

### 1.2 Recherche Web Vérifiée avec Citations (`searchSources`)
1. Lorsque la question porte sur des faits récents, des données cliniques ou que l'utilisateur active la recherche web, l'agent appelle `searchSources({ query })` (ou le client appelle `/v1/core/agent/research`).
2. [`researchSources`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/sources.ts#L156-L246) interroge Gemini avec l'outil natif `googleSearch({})`, extrait les sources d'ancrage (`groundingChunks` / URLs vérifiées) et renvoie une synthèse structurée accompagnée de la liste des citations (`CitationSource[]`) affichées sous forme de badges cliquables dans le chat.

### 1.3 Enrichissement Visuel via Bibliothèque Vectorielle & Images
1. **Symboles Vectoriels Excalidraw (`searchLibraryAssets`)** :
   - Pour illustrer une architecture informatique ou réseau (serveur, routeur, pare-feu, base de données, utilisateur, cloud), l'agent appelle `searchLibraryAssets({ query })` ([`tools.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/tools.ts#L469-L539)).
   - Le backend interroge la table Supabase `library_items` via la fonction RPC `search_library_items` ([`libraryRepo.search`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/db/src/index.ts#L470-L492)).
   - L'outil retourne les éléments vectoriels Excalidraw pré-groupés que l'agent peut placer directement dans `drawDiagram` ou `presentLessonStep`.
2. **Images Éducatives & Schémas Réels (`searchImages`)** :
   - Pour des sujets anatomiques, biologiques, historiques ou géographiques qu'un tracé géométrique ne peut pas représenter fidèlement, l'agent appelle `searchImages({ query, source })` ([`tools.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/tools.ts#L541-L671)).
   - L'outil recherche soit dans **Wikimedia Commons** (`source: "commons"`), soit dans les pages liées par l'étudiant (`source: "resources"`), télécharge l'image, l'encode en `dataURL` dans le dictionnaire `files` de la scène et renvoie un identifiant (`imageId`) utilisable dans un élément `image` ou un template `callouts`.

---

## 2. Règles Métiers Actuelles (Business Rules)

### 2.1 Règles d'Ingestion et de Lecture Documentaire ([`research-tools.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/research-tools.ts))
- **Ancrage Prioritaire sur les Sources de l'Élève** : Lorsqu'un `researchManifest` contient des documents actifs, le prompt système ([`buildResearchContextBlock`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/research-tools.ts#L400-L435)) impose à l'agent de chercher d'abord dans les sources de l'utilisateur avant d'utiliser ses connaissances générales.
- **Pagination et Plafonds de Lecture** :
  - `searchResearchSources` retourne au maximum les **8 meilleurs passages** avec un extrait court pour économiser les tokens.
  - `readResearchPassage` permet de lire une fenêtre de passages contigus (plafonnée à 5 passages par appel).

### 2.2 Règles de Sélection et Sécurité des Images (`searchImages`)
- **Filtrage de Qualité Wikimedia Commons** :
  - Seuls les formats supportés par le canevas (`image/png`, `image/jpeg`, `image/webp`, `image/svg+xml`) et les images de taille raisonnable sont retenus.
  - Chaque image sélectionnée est mise en cache dans la map de session `imageCandidates` afin que `drawDiagram` / `presentLessonStep` puisse injecter automatiquement son binaire (`BinaryFileData`) dans le patch Excalidraw dès que l'élément `image` correspondant est dessiné.

---

## 3. Entités du Domaine (Domain Entities)

| Entité / Type | Fichier Source | Rôle & Attributs Clés |
| :--- | :--- | :--- |
| [`ResearchManifest`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/research-tools.ts#L15-L29) | `packages/core/src/agent/research-tools.ts` | Catalogue des sources documentaires d'un projet : `projectId`, `updatedAt`, `sources` (`Array<{ id, title, kind, url, passageCount, summary }>`). |
| [`ResearchSourceDoc`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/research-tools.ts#L31-L52) | `packages/core/src/agent/research-tools.ts` | Document découpé stocké dans R2 : `id`, `title`, `sections`, `passages` (`Array<{ id, index, sectionTitle, text, pageNumber }>`). |
| [`CitationSource`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/sources.ts#L10-L18) | `packages/core/src/agent/sources.ts` | Référence bibliographique ou web vérifiée : `id`, `title`, `url`, `domain`, `snippet`. |
| [`LibraryItemHit`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/db/src/index.ts#L470-L479) | `packages/db/src/index.ts` | Composant vectoriel issu de `library_items` : `id`, `name`, `keywords`, `elements` (tableau d'éléments Excalidraw), `width`, `height`. |
| [`ImageCandidate`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/tools.ts#L67-L78) | `packages/core/src/agent/tools.ts` | Image prête à être insérée sur le canevas : `imageId`, `fileId`, `title`, `url`, `dataURL`, `mimeType`, `width`, `height`, `attribution`. |
