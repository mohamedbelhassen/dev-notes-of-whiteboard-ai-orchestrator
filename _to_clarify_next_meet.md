# (expliquer ces choix) dans "Limites d'Appels d'Outils Mutatifs (buildStopConditions)" :
- **Limites d'Appels d'Outils Mutatifs ([`buildStopConditions`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/policy.ts#L133-L158))** :
  - En mode `read`, un tour s'arrête dès que **`MAX_MUTATING_CALLS_PER_REQUEST = 3`** appels réussis à `drawDiagram` ou `clearBoard` ont été appliqués, ou après **6 tentatives** (incluant les rejets de validation géométrique).
  - Le nombre d'inspections de groupes via `readCanvasGroup` est plafonné à **`MAX_READ_CALLS_PER_REQUEST = 8`** ; au-delà, seuls les outils mutatifs restent autorisés ([`buildFollowupToolPolicy`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/policy.ts#L118-L131)).

- **Détection d'Intention par Expressions Régulières ([`policy.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/policy.ts#L26-L84))** :
  - Des heuristiques lexicales analysent le dernier message utilisateur (en excluant les négations comme *"don't"*, *"never"*, *"without"*) pour détecter :
    - `hasExplicitDeleteIntent` / `hasExplicitClearIntent` (autorisant la suppression d'éléments ou l'effacement complet du tableau).
    - `hasExplicitNewTopicIntent` / `hasExplicitEditIntent`.
    - `hasChartOrTableIntent` et `hasComparisonIntent` (injectant dynamiquement des directives prioritaires dans le prompt système).
# --> si la recherche est statique (basé sur expression régulière), ça peut ne va pas fonctionner si l'utilisateur utilise une autre langue. Laissons ceci à un classifieur comme Jev/etc.
