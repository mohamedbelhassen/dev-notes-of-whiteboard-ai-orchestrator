# Prompt : Audit de Performance, Sécurité et Architecture (Inklm)

## Rôle et Contexte

Agis en tant qu'**Architecte Logiciel Senior, Expert en Performance IA/Graphique et Auditeur de Sécurité** (spécialiste Next.js, Pipelines Multi-Agents LLM, WebSockets et Moteurs de Rendu Canvas/WebGL). 

Ta mission est d'effectuer une analyse statique exhaustive de la plateforme "inklm" pour détecter les goulots d'étranglement de performance, les failles de sécurité, et les incohérences de conception par rapport à l'architecture cible.

## Périmètre de l'Audit

Au lieu d'explorer le code à l'aveugle, base ton analyse sur l'architecture documentée :

1. **Dossier `inklm/spec/`** : Analyse les spécifications fonctionnelles (flux utilisateurs, règles métiers).
2. **Dossier `inklm/implementation_plans/`** : Analyse les plans d'architecture technique (pipeline multi-agents, Universal Scene Graph, etc.).
3. **Code source `inklm`** : Audite le code réel (frontend, orchestrateur IA, rendu visuel) à la lumière des documents ci-dessus.

## Objectifs de l'Analyse

### 1. Analyse de Performance & Optimisation (IA et Rendu)
* **Pipeline LLM & Orchestrateur :** Identifie les latences évitables, la mauvaise gestion de la concurrence entre agents, les requêtes redondantes, et le gaspillage de tokens.
* **Moteur de Rendu (Canvas/UI) :** Traque les fuites de mémoire, les chutes de framerate (FPS), les rendus React inutiles, et la surcharge de l'Universal Scene Graph.
* **Réseau & WebSockets :** Détecte les surcharges de messages en temps réel et les états désynchronisés.

### 2. Cohérence Architecture vs Code (Design Conformity)
* **Alignement strict :** Vérifie l'alignement entre les plans définis dans `implementation_plans/` (ex: découpage des agents, indépendance des composants) et le code actuel.
* **Fuites de logique :** Identifie les "fuites" de logique (ex: logique métier couplée à l'UI) et le non-respect de l'agnosticisme (LLM-Agnostic, Renderer-Agnostic).

### 3. Audit de Sécurité Spécifique IA & Web
* **Sécurité LLM :** Identifie les vulnérabilités aux injections de prompts, les fuites de contexte sensible, et la mauvaise gestion des clés API.
* **Flux de données :** Vérifie la validation des payloads WebSockets et la protection contre la manipulation de l'Universal Scene Graph par l'utilisateur.

## Format de Sortie Exigé

**Ne réponds pas directement dans le chat.** Rédige tes conclusions de manière exhaustive et structurée dans des fichiers Markdown (`.md`) distincts. Crée ces fichiers dans le répertoire cible : `_dev_/audit_perf_sec/`.

Je souhaite la création de l'arborescence de fichiers suivante :

* `01_executive_summary.md` : Bilan global de la santé du projet (sécurité IA, performance de rendu, dette technique).
* `02_performance_bottlenecks.md` : Liste des problèmes de performance (LLM, WebSockets, Moteur de rendu) avec optimisations proposées.
* `03_architecture_code_consistency.md` : Rapport détaillé sur les écarts entre les dossiers `spec`/`implementation_plans` et l'implémentation réelle.
* `04_security_vulnerabilities.md` : Failles identifiées (Injections de prompt, auth, WebSockets, validation des entrées).
* `05_action_plan.md` : Un plan d'action priorisé étape par étape pour corriger les problèmes.

### Structure requise pour les anomalies
Pour chaque problème ou vulnérabilité identifié dans les fichiers `02`, `03` et `04`, utilise strictement la structure suivante :

* **Titre du problème / Type de faille**
* **Domaine** (Performance IA / Rendu / Sécurité / Conception)
* **Niveau de sévérité** (Critique, Élevé, Moyen, Faible)
* **Localisation** (Fichiers précis et lignes de code)
* **Description de l'impact** (Risque de plantage, de piratage, ou de surcoût d'API LLM)
* **Recommandation de remédiation** (Explication claire accompagnée du code corrigé ou de l'architecture cible)

***

**Déclencheur d'action :** Commence l'audit maintenant en lisant d'abord les dossiers de spécifications avant de plonger dans le code.