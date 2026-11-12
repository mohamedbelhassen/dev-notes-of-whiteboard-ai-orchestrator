# 01 — Orchestrateur Core, Profils de Tuteurs & Contexte

Ce document spécifie le fonctionnement fonctionnel du cœur d'orchestration conversationnelle d'InkLM ([`AgentOrchestrator`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/orchestrator.ts#L131-L280)), la spécialisation par profil de tuteur ([`tutors.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/tutors.ts)), la construction du contexte spatial/historique ([`context.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/context.ts)) et les politiques de contrôle de boucle ([`policy.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/policy.ts)).

---

## 1. Flux Utilisateurs (User Flows)

### 1.1 Soumission d'une Requête Utilisateur (Tour de Chat)
1. **Saisie & Préparation Client** :
   - L'utilisateur saisit une question dans le panneau latéral de chat ou sélectionne des éléments sur le canevas avant d'envoyer son message via [`useAgentSession.sendUserMessage`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/apps/web/src/hooks/use-agent-session.ts#L533-L619).
   - Si des éléments sont sélectionnés sur le tableau, [`buildCanvasSelectionAttachment`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/apps/web/src/hooks/use-agent-session.ts#L220-L312) capture un aperçu de la sélection (`selectionElements`) et l'attache aux métadonnées du message.
   - Si le tableau porte encore un nom par défaut (*Untitled*), [`nameBlankBoard`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/apps/web/src/hooks/use-agent-session.ts#L172-L218) déclenche en parallèle l'auto-nommage du projet/tableau.
   - Le hook enregistre un point de sauvegarde d'annulation ([`DrawingTurnCheckpoint`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/adapters/draw-stream.ts#L45-L53)) permettant d'annuler uniquement les modifications de ce tour.
2. **Assemblage du Contexte par l'Orchestrateur** :
   - Le backend reçoit la requête sur `/v1/core/agent/chat` ([`apps/functions/src/index.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/apps/functions/src/index.ts#L789-L953)) et invoque [`AgentOrchestrator.streamChat`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/orchestrator.ts#L139-L279).
   - L'orchestrateur résume les anciens messages si l'historique dépasse 12 messages ([`prepareConversationHistory`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/context.ts#L31-L97)).
   - Il compresse l'état actuel du tableau en **groupes logiques de proximité** (`canvasGroups`) afin de ne pas saturer la fenêtre de contexte du LLM avec l'intégralité du JSON Excalidraw ([`buildCanvasContext`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/context.ts#L218-L296)).
3. **Boucle d'Exécution Multi-Étapes (Tool Calling)** :
   - Le modèle exécute séquentiellement ses outils (recherche documentaire, inspection d'un groupe existant via `readCanvasGroup`, dessin via `drawDiagram` ou `presentLessonStep`).
   - En cas d'erreur de validation spatiale renvoyée par un outil de dessin (`success: false`), le modèle reçoit le diagnostic géométrique précis et tente de corriger son schéma lors de l'étape suivante.
4. **Clôture & Réponse Textuelle** :
   - Une fois les appels d'outils terminés (ou les plafonds `stopWhen` atteints), le modèle rédige ou conclut sa réponse pédagogique dans le flux SSE consommé par le client.

### 1.2 Choix du Profil de Tuteur Spécialisé
1. L'utilisateur choisit un tuteur dans le sélecteur de l'interface (`General`, `IT`, `Math`, `Physics`, `Medical`).
2. Ce choix est transmis dans le corps de la requête (`tutor: TutorId`) et injecté dans le prompt système via [`tutorPromptBlock`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/tutors.ts#L65-L68), modifiant la grammaire visuelle et pédagogique exigée du modèle.

### 1.3 Choix du Modèle IA (Catalogue Plateforme ou BYOK)
1. Par défaut, l'utilisateur utilise l'un des modèles hébergés définis dans [`CHAT_MODELS`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/models.ts#L6-L10) (`muse-spark-1.3-contributor`, `gemini-3.8-flash`, `gemini-3.5-flash-lite`).
2. S'il a configuré une clé API personnelle, il peut sélectionner un modèle préfixé par `key:<provider>:<model>` ([`parseKeyModelId`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/providers.ts#L47-L55)), routant l'exécution directement vers le fournisseur choisi sans consommer ses crédits plateforme.

---

## 2. Règles Métiers Actuelles (Business Rules)

### 2.1 Règles de Spécialisation par Tuteur ([`tutors.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/tutors.ts#L17-L59))
| ID Tuteur | Domaine | Règles Pédagogiques & Visuelles Imposées |
| :--- | :--- | :--- |
| **`general`** | Polyvalent (Défaut) | Détecte automatiquement la discipline de la question et applique les règles du spécialiste correspondant (`IT`, `MATH`, `PHYSICS`, `MEDICAL`) ou utilise des timelines/cartes conceptuelles pour les sciences humaines. |
| **`it`** | Informatique, Réseaux, Sécurité | Impose `mermaid` (`sequenceDiagram`, `flowchart`, `stateDiagram-v2`) pour les protocoles/états, les templates `layers` (OSI/TCP), `packet` (RFC bit grid), `structure` (structures de données), et la recherche de symboles réseau via `searchLibraryAssets`. Limite les blocs de code à 6 lignes en police monospace (`fontFamily: 3`). |
| **`math`** | Algèbre, Analyse, Géométrie, Probas | Toute formule doit être un élément `math` en LaTeX (jamais de texte brut). Dérivations alignées verticalement (espacement 24–32 px) avec balises `\part{name}{...}` pour le pointage. Impose les templates `plot`, `geometry`, `solid`, `axes3d` avec `roughness: 0`. |
| **`physics`** | Mécanique, Ondes, Élec, Optique | Dessine d'abord le montage physique via les templates dédiés (`spring`, `pendulum`, `incline`, `freebody`, `circuit`, `lens`, `wave`, `field`), puis annote les vecteurs forces/vitesses sur place. Impose de muter la même figure d'une étape à l'autre (`update`) plutôt que de la redessiner. |
| **`medical`** | Anatomie, Physiologie, Pharmacologie | Privilégie une image réelle issue de Wikimedia (`searchImages` + `callouts`) pour l'anatomie, des flowcharts causaux pour la physiologie, cite des sources cliniques et interdit tout conseil médical ou posologique individualisé. |

### 2.2 Gouvernance de la Boucle d'Exécution ([`policy.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/policy.ts#L86-L157) & [`orchestrator.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/orchestrator.ts#L160-L182))
- **Plafond d'Étapes (`maxSteps`)** :
  - Mode `read` sans sources documentaires : **10 étapes** maximum.
  - Mode `read` avec sources documentaires (`needsResearch`) : **14 étapes** maximum.
  - Mode `lesson` sans sources : **18 étapes** maximum.
  - Mode `lesson` avec sources : **22 étapes** maximum.
- **Limites d'Appels d'Outils Mutatifs ([`buildStopConditions`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/policy.ts#L133-L158))** :
  - En mode `read`, un tour s'arrête dès que **`MAX_MUTATING_CALLS_PER_REQUEST = 3`** appels réussis à `drawDiagram` ou `clearBoard` ont été appliqués, ou après **6 tentatives** (incluant les rejets de validation géométrique).
  - Le nombre d'inspections de groupes via `readCanvasGroup` est plafonné à **`MAX_READ_CALLS_PER_REQUEST = 8`** ; au-delà, seuls les outils mutatifs restent autorisés ([`buildFollowupToolPolicy`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/policy.ts#L118-L131)).
- **Filtrage Contextuel des Outils** :
  - En mode `lesson`, les outils `drawDiagram` et `clearBoard` sont **retirés** du jeu d'outils exposé au LLM pour le forcer à passer par `planLesson` et `presentLessonStep` ([`orchestrator.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/orchestrator.ts#L171)).
  - En mode `read`, les outils `planLesson` et `presentLessonStep` sont retirés ([`orchestrator.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/orchestrator.ts#L172)).
- **Détection d'Intention par Expressions Régulières ([`policy.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/policy.ts#L26-L84))** :
  - Des heuristiques lexicales analysent le dernier message utilisateur (en excluant les négations comme *"don't"*, *"never"*, *"without"*) pour détecter :
    - `hasExplicitDeleteIntent` / `hasExplicitClearIntent` (autorisant la suppression d'éléments ou l'effacement complet du tableau).
    - `hasExplicitNewTopicIntent` / `hasExplicitEditIntent`.
    - `hasChartOrTableIntent` et `hasComparisonIntent` (injectant dynamiquement des directives prioritaires dans le prompt système).

### 2.3 Gestion de la Fenêtre de Contexte ([`context.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/context.ts))
- **Compression de l'Historique Conversationnel** :
  - Au-delà de **`RECENT_MESSAGE_WINDOW = 12`** messages, les messages plus anciens sont condensés dans un bloc mémoire textuel (`Conversation memory (older turns summarized)`) plafonné à **`MAX_SUMMARY_CHARS = 3200`** caractères.
- **Partitionnement Spatial du Canevas en Groupes** :
  - Les éléments présents sur le tableau sont regroupés par composantes connexes et proximité spatiale ([`buildCanvasContext`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/context.ts#L218-L296)).
  - Le prompt système ne reçoit que l'index résumé des groupes (`groupId`, boîte englobante `bounds`, compteurs par type, aperçu des labels textuels) ainsi que le détail complet du **dernier groupe actif** (jusqu'à `MAX_GROUP_ELEMENTS_CARRY_FORWARD = 80` éléments). Pour modifier un autre groupe existant, l'agent doit appeler `readCanvasGroup(groupId)`.

---

## 3. Entités du Domaine (Domain Entities)

| Entité / Type | Fichier Source | Rôle & Attributs Clés |
| :--- | :--- | :--- |
| [`AgentChatInput`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/orchestrator.ts#L33-L55) | `packages/core/src/agent/orchestrator.ts` | Contrat d'entrée de l'orchestrateur : `messages` (`UIMessage[]`), `canvas` (éléments Excalidraw courants), `selectionElements`, `viewportBounds`, `presentation` (`"read" \| "lesson"`), `tutor` (`TutorId`), `model`, `apiKey`, `providerBaseURL`, `researchManifest`, `searchAssetLibrary`. |
| [`TutorProfile`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/tutors.ts#L8-L15) | `packages/core/src/agent/tutors.ts` | Définition d'un profil de tuteur : `id` (`"general" \| "it" \| "math" \| "physics" \| "medical"`), `label`, `description`, `prompt` (règles injectées dans le prompt système). |
| [`KeyProvider`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/providers.ts#L6-L19) | `packages/core/src/agent/providers.ts` | Configuration d'un fournisseur compatible BYOK (`openai`, `anthropic`, `google`, `meta`, `openrouter`, `vercel`) : `transport` (`"openai" \| "google"`), `baseURL`, `keyPattern`. |
| [`CanvasGroupSummary`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/context.ts#L101-L114) | `packages/core/src/agent/context.ts` | Résumé spatial d'un cluster d'éléments sur le tableau : `groupId`, `elementCount`, `bounds` (`minX`, `minY`, `maxX`, `maxY`, `width`, `height`), `typeCounts`, `sampleText`, `elementIds`. |
| [`CanvasContext`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/context.ts#L116-L125) | `packages/core/src/agent/context.ts` | État spatial synthétique injecté au début du tour : `totalElements`, `groups` (`CanvasGroupSummary[]`), `latestGroupId`, `latestGroupElements`, `groupElementsById`. |
