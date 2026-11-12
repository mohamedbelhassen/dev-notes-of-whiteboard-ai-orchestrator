# Prompt : Analyse et Cartographie pour Refactoring Architectural

## Rôle et Contexte
Agis en tant qu'**Architecte Logiciel Principal et Expert en Rétro-ingénierie (Reverse Engineering)**. 
Ton objectif est d'analyser l'intégralité du code source du répertoire "inklm" pour cartographier son fonctionnement actuel. Cette cartographie exhaustive servira de ligne de base (baseline) pour préparer un refactoring architectural majeur.

## Objectif Cible du Futur Refactoring (À garder en contexte)
Le système doit évoluer vers une architecture où l'orchestrateur d'IA et le moteur de rendu visuel atteignent une quadruple indépendance : 
- **Application-Agnostic**
- **Tool-Agnostic**
- **LLM-Agnostic**
- **Renderer-Agnostic**

Cette cible reposera sur un pipeline multi-agents (Agent Planificateur, Agents Spécialisés, Linter Spatial) et une couche d'abstraction (Universal Scene Graph).

---

## Démarche Stratégique & Tâches à Exécuter

### Étape 1 : Analyse Globale (Read-only)
- Explore silencieusement l'ensemble des répertoires, fichiers de configuration, code frontend/backend, routes, et l'intégration actuelle des modèles IA et librairies graphiques. 
- Identifie les composants logiques et techniques distincts.

### Étape 2 : Création de la Cartographie (Spécifications Fonctionnelles)
Dans le dossier `inklm/spec/` (crée-le si nécessaire) :
1. Crée un fichier `00_index_specs.md` qui liste et décrit brièvement tous les composants fonctionnels identifiés.
2. Génère un fichier `.md` séparé pour chaque composant/processus majeur (ex: `01_orchestrateur_core.md`, `02_gestion_utilisateurs.md`, `03_generation_animation.md`, etc.).
3. **Contenu de chaque fichier :** Décris les flux utilisateurs (User Flows), les règles métiers actuelles, et les entités du domaine propres à ce composant.

### Étape 3 : Création des Plans d'Implémentation et d'Architecture
Dans le dossier `inklm/implementation_plans/` (crée-le si nécessaire) :
1. Crée un fichier `00_index_architecture.md` listant les modules techniques.
2. Génère un fichier `.md` séparé pour chaque couche ou module technique (ex: `01_api_websockets.md`, `02_pipeline_llm.md`, `03_canvas_renderer.md`, `04_state_management.md`).
3. **Contenu de chaque fichier :**
   - Détaille les flux d'exécution techniques (comment la donnée circule).
   - Intègre des diagrammes de séquence ou d'architecture en syntaxe Mermaid.js.
   - **Crucial :** Isole et mets en évidence (via des balises `> [!WARNING] Point de Couplage`) les endroits exacts du code où la logique applicative, le moteur IA et le rendu graphique sont fortement couplés (Vendor Lock-in).

---

## Contraintes d'Exécution
- **Pas de monolithisme :** Ne génère pas de fichiers monolithiques. Le découpage doit être granulaire et logique.
- **Qualité :** Rédige tous les documents en Markdown professionnel, structuré et clair.
- **Méthodologie :** Procède étape par étape. Affiche un résumé de ton plan de fichiers avant de commencer à les écrire sur le disque pour validation.