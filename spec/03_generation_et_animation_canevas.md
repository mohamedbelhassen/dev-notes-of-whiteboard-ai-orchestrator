# 03 — Génération Visuelle, Templates Métiers & Animation du Canevas

Ce document spécifie le pipeline fonctionnel de création graphique d'InkLM : la génération de schémas via [`drawDiagram`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/tools.ts#L186-L242), l'expansion des **26 templates métiers (`predefinedElement`)** ([`expand.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/parser/domain/predefined/expand.ts)), le placement et routage automatique ([`visual/`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/visual/index.ts)), les règles de validation géométrique ([`layout-check.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/visual/layout-check.ts)) et l'animation progressive sur le canevas ([`drawing-reveal.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/adapters/drawing-reveal.ts) & [`svg-stage.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/apps/web/src/lib/stage/svg-stage.ts)).

---

## 1. Flux Utilisateurs (User Flows)

### 1.1 Génération et Dessin Progressif d'un Diagramme (`drawDiagram`)
1. **Émission par l'Agent** :
   - L'agent appelle l'outil `drawDiagram({ elements, deleteElementIds })` en fournissant un tableau mêlant des primitives Excalidraw (`rectangle`, `ellipse`, `diamond`, `arrow`, `line`, `text`, `image`) et des éléments haut-niveau (`predefinedElement`, `math`).
2. **Streaming & Aperçu Temps Réel (Pendant la Génération JSON)** :
   - Côté client, [`useAgentSession`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/apps/web/src/hooks/use-agent-session.ts#L334-L380) intercepte les fragments d'arguments de l'outil (`tool-input-available` / streaming) et invoque [`buildDrawingPreview`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/adapters/draw-stream.ts#L287-L333) via [`DrawingDirector`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/director/drawing-director.ts).
   - Le parseur tolérant aux JSON incomplets ([`parseElementStream`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/parser/json-stream.ts)) extrait les éléments déjà fermés pour les pré-visualiser immédiatement.
3. **Expansion, Normalisation & Validation (Serveur)** :
   - L'outil [`drawDiagram`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/tools.ts#L186-L242) exécute [`applyDrawElements`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/adapters/draw-stream.ts#L81-L157) qui :
     1. Déploie chaque `predefinedElement` et chaque formule `math` en primitives normalisées.
     2. Calcule la mise en page automatique des graphes (`dagre`), positionne la nouvelle section dans un espace libre à proximité du viewport de l'utilisateur et route les flèches en évitant les obstacles (algorithme A*).
     3. Vérifie la lisibilité du résultat via [`assertReadableDrawing`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/visual/layout-check.ts#L316-L364). Si un chevauchement critique ou un débordement est détecté, le patch est rejeté et l'erreur détaillée est retournée au LLM pour auto-correction.
4. **Animation par Traits (`SvgStage`) & Commit Excalidraw (Client)** :
   - Lorsque le patch validé (`DrawingPatch`) arrive au client, [`SvgStage`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/apps/web/src/lib/stage/svg-stage.ts) découpe le schéma en **unités de révélation** (`RevealUnit`) et **étapes d'animation** (`RevealStage`).
   - Une surcouche SVG anime le tracé progressif des contours (`strokeDasharray` / `strokeDashoffset`) et l'apparition en fondu des remplissages/textes tout en déplaçant fluidement la caméra Excalidraw.
   - Une fois l'animation terminée, les éléments sont fusionnés dans la scène Excalidraw définitive via [`mergePatchIntoScene`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/adapters/scene.ts#L98-L163).

### 1.2 Inspection et Édition Ciblée d'une Section Existante
1. Pour modifier un diagramme existant qui ne fait pas partie du dernier groupe actif, l'agent appelle `readCanvasGroup({ groupId })` ([`tools.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/tools.ts#L397-L444)) afin de charger ses éléments exacts (IDs, coordonnées, textes).
2. Il émet ensuite un appel `drawDiagram` réutilisant les mêmes `id` pour mettre à jour ou déplacer les éléments existants, ou renseigne `deleteElementIds` (uniquement si l'utilisateur a exprimé une intention explicite d'édition ou de suppression).

### 1.3 Annulation d'un Tour de Dessin IA (`undoDrawingTurn`)
1. L'utilisateur clique sur l'action d'annulation associée à un message de l'agent.
2. Le client exécute [`undoDrawingTurn`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/adapters/draw-stream.ts#L335-L382) à partir du [`DrawingTurnCheckpoint`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/adapters/draw-stream.ts#L45-L53) capturé avant ce tour : seuls les éléments créés ou modifiés par l'agent durant ce tour sont retirés ou restaurés à leur état antérieur, en préservant toute modification manuelle faite par l'utilisateur entre-temps.

---

## 2. Règles Métiers Actuelles (Business Rules)

### 2.1 Catalogue des 26 Templates Métiers (`predefinedElement`)
Définis dans [`packages/core/src/parser/domain/predefined/expand.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/parser/domain/predefined/expand.ts#L45-L185), ces templates transforment une spécification déclarative JSON en un ensemble cohérent de formes, textes et connecteurs :

| Catégorie | Identifiants (`identifier`) | Rôle Fonctionnel & Rendu Généré |
| :--- | :--- | :--- |
| **Graphes & Diagrammes** | `graph`, `mermaid` | Graphes orientés/non-orientés et syntaxe Mermaid (`flowchart`, `sequenceDiagram`, `stateDiagram-v2`, `classDiagram`) convertis en nœuds et flèches routées. |
| **Données & Hiérarchies** | `tree`, `table`, `chart`, `matrix`, `structure`, `venn` | Arbres hiérarchiques, tableaux de données, graphiques (`bar`, `line`, `pie`, `scatter`), grilles matricielles, structures de données (`array`, `linked-list`, `stack`, `queue`, `hashmap`, `heap`, `bst`) et diagrammes de Venn. |
| **Mathématiques & Géométrie** | `math` *(type natif)*, `plot`, `geometry`, `solid`, `axes3d` | Équations LaTeX rendues en SVG via KaTeX ([`math/svg.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/math/svg.ts)), courbes de fonctions $f(x)$ avec racines/asymptotes/aires, figures géométriques cotées, solides 3D et repères 3D. |
| **Physique & Ingénierie** | `circuit`, `freebody`, `spring`, `pendulum`, `incline`, `lens`, `wave`, `field` | Schémas électriques normalisés, diagrammes de corps libre, systèmes masse-ressort, pendules, plans inclinés, bancs d'optique géométrique, ondes et champs vectoriels. |
| **Informatique & Réseaux** | `layers`, `packet` | Piles protocolaires (OSI/TCP-IP avec encapsulation) et grilles d'en-têtes de paquets binaires RFC. |
| **Pédagogie Générale & Sciences** | `timeline`, `cycle`, `comparison`, `callouts`, `anatomy` | Frises chronologiques, cycles causaux, cartes comparatives côte-à-côte, étiquettes fléchées sur image et schémas anatomiques imbriqués. |

### 2.2 Garde-fous Géométriques & Lisibilité ([`layout-check.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/visual/layout-check.ts))
La fonction [`assertReadableDrawing`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/visual/layout-check.ts#L316-L364) rejette tout dessin violant l'une des règles suivantes :
1. **Chevauchement de Formes (`shape-overlap`)** : Deux conteneurs non imbriqués intentionnellement ne peuvent pas se superposer au-delà d'un seuil de surface.
2. **Collision de Textes (`text-overlap`)** : Deux labels textuels indépendants ne doivent jamais entrer en collision.
3. **Débordement de Texte (`text-overflow`)** : Le texte lié à un conteneur doit tenir dans les dimensions internes de la forme (vérifié via les métriques de police [`font-metrics.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/core/font-metrics.ts)).
4. **Flèches Orphelines ou Croisements Excessifs** : Les connecteurs doivent relier des éléments valides et sont re-routés orthogonalement ou par A* ([`routing.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/visual/routing.ts)) autour des obstacles.
5. **Protection Contre la Suppression Non Autorisée** : Toute utilisation de `deleteElementIds` ou `clearBoard` sans intention explicite de l'utilisateur dans son message (`hasExplicitDeleteIntent` / `hasExplicitClearIntent`) est bloquée ou ignorée ([`tools.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/tools.ts#L194-L202)).

### 2.3 Règles de Découpage d'Animation ([`drawing-reveal.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/adapters/drawing-reveal.ts))
- Un conteneur et son texte lié (`containerId`) forment une **unité atomique (`RevealUnit`)** afin que la boîte et son label apparaissent toujours ensemble.
- Les unités sont ordonnées topologiquement (du haut vers le bas, de gauche à droite, et les nœuds avant les flèches qui les relient) et regroupées en **`RevealStage`** pour rythmer le tracé sans lenteur excessive sur les grands schémas.

---

## 3. Entités du Domaine (Domain Entities)

| Entité / Type | Fichier Source | Rôle & Attributs Clés |
| :--- | :--- | :--- |
| [`ExcalidrawElementInput`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/core/types.ts#L58-L157) | `packages/core/src/core/types.ts` | Élément brut produit par le LLM : `id`, `type`, `x`, `y`, `width`, `height`, `text`, `strokeColor`, `backgroundColor`, `fillStyle`, `roughness`, `startBinding`, `endBinding`, ou `identifier` + `params` pour un `predefinedElement`. |
| [`NormalizedElement`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/core/types.ts#L159-L214) | `packages/core/src/core/types.ts` | Primitive Excalidraw normalisée après expansion des templates et calcul des boîtes englobantes, prête à être fusionnée dans la scène. |
| [`DrawingPatch`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/core/types.ts#L228-L237) | `packages/core/src/core/types.ts` | Delta atomique appliqué au tableau : `elements` (`NormalizedElement[]`), `files` (SVG mathématiques ou images encodées), `deletedIds` (`string[]`). |
| [`RevealUnit`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/adapters/drawing-reveal.ts#L15-L22) | `packages/core/src/adapters/drawing-reveal.ts` | Groupe indivisible d'éléments animés simultanément (ex. une forme + son label + ses flèches entrantes). |
| [`DrawingTurnCheckpoint`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/adapters/draw-stream.ts#L45-L53) | `packages/core/src/adapters/draw-stream.ts` | Snapshot différentiel d'avant-tour (`beforeById`, `touchedIds`, `deletedIds`) permettant un `undoDrawingTurn` chirurgical. |
