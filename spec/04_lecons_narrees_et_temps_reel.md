# 04 — Leçons Narrées, Mode Live Temps Réel, Enregistrement & Export

Ce document spécifie le fonctionnement des deux modes pédagogiques multimodaux d'InkLM :
1. **Le mode Leçon Narrée (`lesson`)** orchestré par [`planLesson`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/tools.ts#L244-L281), [`presentLessonStep`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/tools.ts#L283-L395), le moteur de direction [`lesson/direction.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/lesson/direction.ts) et le lecteur client [`LessonPlayer`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/apps/web/src/lib/lesson-player.ts).
2. **Le mode Live Temps Réel (`realtime`)** combinant Gemini Live WebSocket ([`live.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/live.ts)) et le chef d'orchestre client [`TeachingConductor`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/apps/web/src/lib/live/teaching-conductor.ts).
3. **Les fonctionnalités de capitalisation** : enregistrement cloud des leçons ([`recordings.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/apps/functions/src/recordings.ts)) et export PowerPoint `.pptx` ([`lesson/pptx.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/lesson/pptx.ts)).

---

## 1. Flux Utilisateurs (User Flows)

### 1.1 Leçon Narrée Étape par Étape (Mode `lesson`)
1. **Planification Initiale (`planLesson`)** :
   - Lorsque l'utilisateur pose une question en mode `lesson`, le prompt [`LESSON_SYSTEM_PROMPT`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/prompts.ts#L345-L426) oblige l'agent à appeler d'abord `planLesson({ title, steps })` pour déclarer entre **2 et 6 étapes pédagogiques (*beats*)** avec leurs budgets visuels (`maxElements`, `maxWords`).
2. **Exécution Séquentielle des Étapes (`presentLessonStep`)** :
   - Pour chaque étape $i$, l'agent appelle `presentLessonStep` avec :
     - Le script parlé (`spokenText`),
     - Les nouveaux éléments à dessiner (`elements`),
     - Les directives scéniques (`direction` : pointeur laser, emphase couleur, révélation pas-à-pas, ou mutations d'une structure de données existante).
   - Côté serveur, [`presentBeat`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/tools.ts#L103-L160) :
     1. Exécute [`applyLessonDirection`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/lesson/direction.ts#L450-L539) et [`applyDrawElements`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/adapters/draw-stream.ts#L81-L157) pour valider la géométrie.
     2. Calcule la timeline temporelle de l'étape via [`buildLessonStepTimeline`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/lesson/timeline.ts).
     3. Appelle en parallèle [`synthesizeAgentSpeech`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/speech.ts#L194-L280) pour générer l'audio TTS en base64 (WAV/MP3).
3. **Lecture Synchronisée Client (`LessonPlayer`)** :
   - Au fil de la réception des étapes (`LessonStepPayload`), [`LessonPlayer`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/apps/web/src/lib/lesson-player.ts) les met en file d'attente.
   - Pour chaque étape, il synchronise :
     - La lecture de la piste audio TTS (ou une horloge synthétique de secours si le TTS a échoué),
     - Le cadrage caméra sur la zone active (`cameraTargetIds`),
     - Le tracé progressif des éléments (`SvgStage`),
     - Le déplacement du curseur enseignant (`TeacherCursor`) vers les éléments ou sous-parties d'équations (`id#partName`) cités dans la voix.
   - L'utilisateur dispose de contrôles de lecture : **Pause / Lecture**, **Étape précédente / suivante**, **Réglage de vitesse** et **Export PPTX**.

### 1.2 Session Vocale Temps Réel (Mode `realtime`)
1. **Initialisation de la Session Live** :
   - L'utilisateur active le microphone en mode Live via [`useRealtimeTeaching`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/apps/web/src/hooks/use-realtime-teaching.ts).
   - Le client appelle `POST /v1/core/agent/live-token` ([`apps/functions/src/index.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/apps/functions/src/index.ts#L586-L654)) qui génère un jeton éphémère Google GenAI configuré avec le modèle `gemini-3.8-live` et l'outil vocal `start_visual_lesson`.
2. **Dialogue Bidirectionnel & Déclenchement Visuel** :
   - Le navigateur établit une connexion WebSocket directe avec Gemini Live ([`live-teaching-session.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/apps/web/src/lib/live/live-teaching-session.ts)) pour l'échange audio temps réel.
   - Dès que la question de l'élève nécessite une explication au tableau, le modèle vocal invoque `start_visual_lesson({ topic, visualGoals })`.
   - [`TeachingConductor`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/apps/web/src/lib/live/teaching-conductor.ts) appelle alors le endpoint SSE `/v1/core/agent/plan-lesson` ([`apps/functions/src/index.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/apps/functions/src/index.ts#L656-L742)) qui streame des **Teaching Cues** (`TeachingCue` contenant les éléments normalisés à dessiner et les points clés à énoncer) pendant que le modèle Live les commente à la voix.

### 1.3 Enregistrement Cloud & Export PowerPoint (`.pptx`)
1. **Enregistrement de Leçon (`recordingsRoutes`)** :
   - Durant une leçon, le client téléverse chaque segment audio et les métadonnées de l'étape via `PUT /v1/recordings/:recordingId/segments/:index` ([`apps/functions/src/recordings.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/apps/functions/src/recordings.ts#L85-L170)) dans Cloudflare R2, puis finalise le manifeste via `PUT /v1/recordings/:recordingId/manifest`.
   - L'utilisateur peut réécouter la leçon complète hors-ligne ou depuis un autre appareil.
2. **Export PowerPoint (`exportLessonPptx`)** :
   - L'utilisateur clique sur *Export PPTX* dans la barre du lecteur de leçon.
   - [`buildLessonPresentationSlides`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/lesson/pptx.ts) convertit chaque étape de la leçon en une diapositive 16:9 contenant le rendu visuel du canevas à cet instant, le titre de l'étape et les notes de l'orateur (`spokenText`).

---

## 2. Règles Métiers Actuelles (Business Rules)

### 2.1 Contrats de Planification & Budgets Visuels ([`tools.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/tools.ts#L244-L395))
- **Appel Préalable Obligatoire** : Tout appel à `presentLessonStep` sans appel préalable à `planLesson` (ou hors de la plage d'index `0 .. steps.length - 1`) est rejeté ou auto-corrigé.
- **Tonalité Vocale Non Répétitive** : L'historique des tons vocaux (`lessonTones`) est suivi d'une étape à l'autre pour éviter la monotonie vocale (`warm`, `curious`, `emphatic`, `calm`, `excited`).
- **Continuité des Identifiants (`lessonIds`)** : Les identifiants d'éléments créés aux étapes précédentes sont conservés dans `lessonIds`, permettant aux étapes suivantes de les pointer (`pointerTargetIds`), de les mettre en valeur (`emphasisIds`) ou de les muter sur place (`structureMutations`).

### 2.2 Primitives de Direction Pédagogique ([`lesson/direction.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/lesson/direction.ts))
Le moteur `applyLessonDirection` supporte quatre mécanismes d'animation pédagogique :
1. **Pointage Sous-Élément (`pointerTargetIds`)** : Supporte le pointage d'un élément complet (`"eq1"`) ou d'un terme nommé à l'intérieur d'une équation LaTeX (`"eq1#discriminant"`) grâce aux ancres `\part{name}{...}` extraites par [`math/svg.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/math/svg.ts).
2. **Emphase Visuelle (`emphasisIds`)** : Applique temporairement ou durablement une couleur d'accentuation et un contour renforcé aux éléments clés du beat.
3. **Mutations de Structures de Données (`structureMutations`)** : Permet d'animer sur une structure existante (`array`, `stack`, `queue`, `tree`, `graph`) des opérations algorithmiques (`highlight`, `swap`, `setValue`, `push`, `pop`, `markVisited`, `movePointer`) sans recréer le diagramme.
4. **Révélation Séquentielle (`stepReveal`)** : Synchronise l'apparition de chaque groupe d'éléments avec la progression temporelle de la phrase parlée correspondante ([`lesson/timeline.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/lesson/timeline.ts)).

### 2.3 Politique de Synthèse Vocale Multi-Fournisseurs ([`speech.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/speech.ts))
- Le texte parlé est nettoyé du balisage Markdown/LaTeX avant synthèse ([`cleanSpokenScript`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/speech.ts#L43-L79)).
- Ordre de priorité des fournisseurs TTS dans [`synthesizeAgentSpeech`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/speech.ts#L194-L280) :
  1. **OpenRouter (`heygen/heygen-voice-1-turbo:free`)** si `OPENROUTER_API_KEY` est configurée.
  2. **Google Gemini TTS (`gemini-3.8-flash-lite-tts`)** via l'API `/v1beta/interactions` en fallback (conversion PCM 24kHz → WAV encapsulé).
  3. **Dégradation Gracieuse** : Si aucun fournisseur TTS ne répond dans le délai imparti, `presentLessonStep` réussit quand même avec `speech: null` et le client utilise une durée de lecture calculée à partir du nombre de mots.

---

## 3. Entités du Domaine (Domain Entities)

| Entité / Type | Fichier Source | Rôle & Attributs Clés |
| :--- | :--- | :--- |
| [`LessonPlan`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/lesson/direction.ts#L15-L34) | `packages/core/src/lesson/direction.ts` | Plan directeur d'une leçon : `title`, `steps` (`Array<{ index, title, goal, visualKind, maxElements, maxWords }>`). |
| [`LessonDirection`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/lesson/direction.ts#L64-L98) | `packages/core/src/lesson/direction.ts` | Directives scéniques d'un beat : `cameraTargetIds`, `pointerTargetIds`, `emphasisIds`, `stepReveal`, `structureMutations`, `callout`. |
| [`LessonStepPayload`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/lesson/timeline.ts) | `packages/core/src/lesson/timeline.ts` | Charge utile complète d'une étape envoyée au `LessonPlayer` : `stepIndex`, `title`, `spokenText`, `speech` (`audioBase64`, `mimeType`, `durationMs`), `patch` (`DrawingPatch`), `timeline`, `direction`. |
| [`TeachingCue`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/live.ts#L31-L42) | `packages/core/src/agent/live.ts` | Unité de synchronisation du mode Live temps réel : `id`, `title`, `talkingPoints` (`string[]`), `elements` (`ExcalidrawElementInput[]`), `focusElementIds`. |
| [`RecordingManifest`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/apps/functions/src/recordings.ts#L19-L36) | `apps/functions/src/recordings.ts` | Manifeste persisté dans R2 pour rejouer une leçon enregistrée : `recordingId`, `canvasId`, `title`, `createdAt`, `segments` (clés R2 audio + timestamps + états de scène). |
