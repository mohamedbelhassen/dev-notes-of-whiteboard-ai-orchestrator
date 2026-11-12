# Procès-Verbal & Rapport Stratégique Détaillé des Réunions

**Projet :** Plateforme E-Learning Adaptative & Orchestrateur Multi-Agents pour Tableaux Blancs  
**Participants :** Mohamed Belhassen (Encadreur, Stratège, Product Owner & Leader) et l'Étudiant (Chef de Projet Technique & Développeur Principal)  
**Nature du document :** Synthèse exhaustive, structurée et détaillée des deux séances d'enregistrement (audio 1 et audio 2).

---

## Préambule & Cadre Général de la Collaboration

Les deux séances de travail consignées dans ce document définissent les fondations techniques, pédagogiques, organisationnelles et humaines d'un partenariat de co-création. L'objectif partagé est de concevoir et de déployer une solution innovante combinant un **Projet de Fin d'Études (PFE)** de haut niveau et un **produit commercialisable prêt pour le marché** (principe de la formule « 2-en-1 »).

Le projet s'articule autour de deux axes complémentaires :
1. Un **orchestrateur d'intelligence artificielle multi-agents** pour bibliothèques de tableaux blancs interactifs (*whiteboards*).
2. Une **plateforme d'apprentissage adaptatif (e-learning)** ciblant en priorité le cycle primaire (concours de la 6ème année) et secondaire, reposant sur une pédagogie d'accompagnement par indices (*hints*), l'alignement avec les programmes officiels et une stratégie d'optimisation garantissant un coût d'inférence minimal.

---

# Partie 1 : Projet d'Orchestrateur Basé sur l'IA pour Librairies de Whiteboard & Architecture Développée

### 1.1 Genèse, Contexte et Limites des Approches Précédentes
L'initiative est née de la volonté de l'étudiant de concevoir un système d'apprentissage visuel et interactif, provisoirement dénommé *CanvasLM* ou *InkLM*, inspiré de *Google NotebookLM*. Alors que NotebookLM excelle dans la synthèse textuelle et la génération de discussions audio statiques, il lui manque la dimension visuelle dynamique essentielle à l'assimilation de raisonnements scientifiques (mathématiques, géométrie, physique). L'objectif est donc de matérialiser l'explication sous forme de tracés animés étape par étape sur un tableau blanc numérique.

Cependant, les premiers développements de l'étudiant reposaient sur une liaison étroite avec une bibliothèque graphique spécifique, soulevant des risques techniques et concurrentiels majeurs.

### 1.2 Évaluation Critique des Librairies Existantes & Contraintes de Licences
L'examen des technologies candidates a permis d'identifier plusieurs écueils opérationnels et juridiques :

* **TLDraw** : Bien que dotée de fonctionnalités avancées, cette bibliothèque impose des restrictions de licence importantes pour les usages commerciaux, des obligations de mention de crédits et d'éventuelles redevances. De plus, son éditeur développe ses propres fonctionnalités génératives, ce qui expose une solution tierce dépendante au risque d'être contournée ou écrasée par la plateforme hôte.
* **Excalidraw** : Solution open source solide, bénéficiant d'un rendu « dessiné à la main » très adapté sur le plan pédagogique, mais dont les formats de données restent spécifiques à son écosystème.
* **Mermaid.js** : Particulièrement adaptée aux diagrammes de flux ou d'architecture par simple saisie textuelle déclarative, mais rigide pour des tracés libres et des animations spatiales progressives.

Face à ces constats, Mohamed Belhassen a fixé une directive fondamentale : le système doit être impérativement **« Platform-Agnostic » (indépendant de toute bibliothèque tierce)**.

### 1.3 Vision Architecturale : Découplage Total & Couche d'Abstraction
Pour garantir la pérennité technologique du projet :
* **Couche d'Abstraction Intermédiaire (Bridge / Abstraction Layer)** : L'orchestrateur produit une représentation neutre, universelle et déclarative en JSON. Cette structure décrit les primitives géométriques, coordonnées relatives, connecteurs, blocs de texte, formules mathématiques et transitions de caméra.
* **Moteurs de Rendu Découplés (Drivers / Adaptateurs)** : Des modules de traduction spécifiques convertissent ce JSON standard vers la syntaxe propriétaire de la bibliothèque cible (Excalidraw, TLDraw, Mermaid, ou ultérieurement des moteurs 3D comme Three.js ou Blender).
* **Architecture API & Serveurs MCP** : Le moteur d'orchestration est hébergé sur un backend autonome, accessible via des API REST et projeté sous forme de serveur MCP (*Model Context Protocol*), rendant le service consommable par n'importe quelle interface (web, mobile, plugin).

### 1.4 Architecture Multi-Agents en Python & Pipeline d'Exécution
L'orchestration repose sur une division modulaire des responsabilités entre plusieurs agents spécialisés :

1. **Agent Planificateur / Scénariste (Planner Agent)** :
   * Traite la requête pédagogique de l'utilisateur (ex. : « expliquer la surface d'un disque » ou « illustrer la photosynthèse »).
   * Établit le plan logique et découpe la démonstration en jalons séquentiels (*milestones* ou diapositives logiques).
   * **Arbitrage sur le pipeline d'inférence** : Mohamed a recadré la proposition de l'étudiant, qui prévoyait une boucle itérative multi-requêtes vers le LLM. Mohamed préconise un fonctionnement en **une seule passe majeure** (ou deux passes maximum) générant le script JSON complet, afin d'éliminer la latence excessive, les dérives de raisonnement et l'explosion des coûts d'API.
2. **Agents Métiers Spécialisés (Domain-Specific Agents)** :
   * Agents dotés d'un outillage (*tooling*) spécifique à chaque discipline : géométrie, algèbre, physique-chimie, algorithmique.
   * Chaque agent manipule des primitives dédiées (formes mathématiques, tracés vectoriels, repères cartésiens, blocs de code syntaxique).
3. **Agent Vérificateur et Linter Visuel (Visual Validator / Inspector)** :
   * Réalise un contrôle spatial statique sur le JSON généré avant rendu à l'écran.
   * Corrige automatiquement les anomalies d'affichage identifiées lors des tests : chevauchement de textes, étiquettes masquées par des formes géométriques, flèches inversées, éléments tronqués hors du champ de vision (*viewport*) ou cadrages de caméra incohérents.

### 1.5 Optimisation des Fenêtres de Contexte & Stratégie « Zéro-Token »
L'injection de la totalité des outils et des règles graphiques dans un prompt unique provoquerait une surcharge massive (plusieurs dizaines de milliers de tokens par requête), générant des coûts prohibitifs et des ralentissements incompatibles avec l'expérience utilisateur.

* **Classifieurs Sémantiques Légers** : Intégration en amont de modèles de classification rapides et économiques (ex. modèles de classification textuelle ou classifieurs Jina) pour déterminer immédiatement la matière et le type d'exercice.
* **Activation Dynamique du Tooling** : Seul l'outillage strictement nécessaire à la tâche identifiée est chargé dans le contexte du modèle, évitant toute consommation superflue de tokens.

### 1.6 Modèle Économique de l'Orchestrateur & Sécurisation de Marque
* **Commercialisation en « Result-as-a-Service » (RaaS)** : Facturation du service non pas sur la base des tokens consommés en entrée, mais sur la valeur du livrable généré (nombre de scènes, complexité des animations et précision des schémas).
* **Analogie avec la Facturation Télécom et CDN (Burstable 95th/96th Percentile)** : Mohamed a rappelé le principe de tolérance aux pics de trafic des fournisseurs d'accès (où les 4 à 5 % de pics d'utilisation extrême sont déduits de la facture mensuelle), proposant un système d'abonnement flexible pour absorber les révisions intensives des élèves à l'approche des épreuves.
* **Pipeline d'Exportation & Protection par Filigrane (Watermarking)** :
  * Fonctionnalité d'exportation vers des formats standards : présentations PowerPoint (PPTX) et vidéos explicatives destinées aux réseaux sociaux ou à YouTube.
  * **Application systématique d'un filigrane visuel et sonore** : Cette protection évite le pillage de contenu par des tiers et permet, en cas de rediffusion non autorisée sur YouTube, d'activer les outils de détection de droits d'auteur (*Content ID*) pour récupérer automatiquement la monétisation générée.
* **Option « Bring Your Own Key » (BYOK)** : Possibilité offerte aux utilisateurs professionnels et établissements scolaires d'utiliser leurs propres clés d'API (OpenAI, Gemini, Anthropic), isolant la plateforme des coûts d'inférence sous-jacents.

---

# Partie 2 : Plateforme E-Learning Dédiée aux Élèves de Primaire / Secondaire (MVP, Brainstorming & Concepts)

### 2.1 Contexte et Retours d'Expérience du Prototype « Sokratix »
Mohamed Belhassen s'est appuyé sur les acquis de sa propre plateforme expérimentale, *Sokratix* :
* Conçue en une semaine de développement intensif pour une compétition internationale organisée par Meta (regroupant 5 400 candidats), Sokratix a franchi les sélections avec succès et remporté un prix d'innovation.
* Le prototype initial a été partagé sur GitHub sous licence open source GPL v3. Mohamed a toutefois souligné que la future plateforme commerciale devra être développée sur un dépôt privé sous architecture modulaire pour garantir la protection industrielle de ses composants clés.

### 2.2 Pédagogie par Compétences (Skill-Based Mapping & Micro-Compétences)
La conception éducative repose sur une formalisation précise des savoirs scolaires :
* **Numérisation Intégrale des Manuels Scolaires Tunisiens** : Exploitation des manuels officiels édités par le ministère de l'Éducation, en intégrant impérativement les **livres du maître (guides pédagogiques de l'enseignant)**. Ces guides fournissent la structure exacte des compétences visées, les prérequis et les grilles d'évaluation.
* **Découpage en Micro-Compétences** : Chaque chapitre est décomposé en unités d'apprentissage atomiques (ex. : « savoir additionner deux entiers à deux chiffres », « appliquer le théorème de Pythagore dans un triangle rectangle », « identifier un circuit en série »).
* **Matrice de Couverture (Coverage Matrix)** : L'IA analyse les énoncés d'exercices et les travaux pratiques pour évaluer leur adéquation avec les leçons théoriques. Elle identifie automatiquement les compétences orphelines (enseignées mais non testées) ou les exercices sollicitant des prérequis non encore étudiés.
* **Harmonisation et Déduplication Taxonomique** : Rapprochement sémantique des concepts similaires (ex. : unifier « notions de classes en Java » et « programmation par classes ») afin d'éviter la dispersion des indicateurs de progression de l'élève.

### 2.3 Pédagogie Active : Le Principe Fondateur des « Hints » (Indices Sans Solution Brute)
Une consigne pédagogique majeure a été arrêtée : **le modèle ne doit en aucun cas donner la réponse directement à l'élève**.
* **Accompagnement Maïeutique par Échelons** : L'IA identifie l'erreur de l'enfant et lui soumet un indice contextuel (*hint*), en le renvoyant précisément au paragraphe, à l'exemple ou à la page du manuel officiel où se trouve la règle applicable.
* **Few-Shot Prompting par Discipline** : Le comportement de l'assistant s'adapte à la matière. En mathématiques ou en programmation, il valide la justesse du raisonnement logique sans sanctionner immédiatement les coquilles mineures, encourageant ainsi l'effort intellectuel de l'élève.
* **Garde-fous Éducatifs Stricts (Guardrails)** : Confinement rigoureux des réponses au programme scolaire de l'élève. Toute question hors-programme (ex. physique nucléaire ou contenus inappropriés demandés par un élève de 6ème primaire) est systématiquement refusée avec bienveillance, recentrant l'échange sur les révisions en cours.

### 2.4 Ciblage de Marché & Distinction « Client Acheteur vs Élève Utilisateur »
Les arbitrages de marché retenus définissent une stratégie commerciale ciblée :
* **Priorité Absolue du MVP : Les Concours Nationaux** : Mohamed a exclu le secteur universitaire, marqué par un manque d'assiduité des étudiants (seuls 10 à 12 étudiants sur une promotion de 110 assistent régulièrement aux cours magistraux). Le MVP se concentre exclusivement sur les **concours nationaux : 6ème année primaire** (concours d'entrée aux collèges pilotes), puis 9ème année et Baccalauréat.
* **Dissociation Client / Utilisateur** :
  * Le **parent est le client** : C'est lui qui ressent l'anxiété de la réussite de son enfant, apprécie la valeur pédagogique et paie l'abonnement financier.
  * L'**élève est l'utilisateur final** : Il requiert un environnement stimulant, ergonomique, sans friction technique ni sentiment d'échec.
* **Étude Comparative du Leader Existant (*Atoum Academy*)** :
  * Analyse de l'abonnement souscrit par Mohamed pour son propre fils sur Atoum Academy (dont le tarif est passé de 90 TND à 450 TND par an).
  * Constat : Atoum produit d'excellentes vidéos en haute définition présentées par des enseignants qualifiés, mais son modèle demeure unilatéral et passif. Il n'offre aucune interactivité adaptative, aucun diagnostic en temps réel et ne prend pas en compte les profils cognitifs variés des enfants (visuel, auditif ou kinesthésique).

### 2.5 Stratégie « Coût IA Tendant vers Zéro » (Zero-AI-Cost Strategy)
Pour éviter qu'une explosion des requêtes vers des LLM commerciaux ne vienne détruire la marge opérationnelle :
* **Base de Connaissances Pré-Calculée et Validée (Knowledge Base & Cache Sémantique)** : Les explications, animations vectorielles et étapes de résolution de la majorité des exercices des manuels sont générées, contrôlées et stockées en amont.
* **Routage Déterministe** : Lorsqu'un élève pose une question sur un exercice standard, la plateforme affiche directement l'explication validée issue du cache sans solliciter de nouvelle inférence LLM payante.
* **Validation Humaine (« Human-in-the-Loop »)** : Les nouvelles fiches d'exercices et réponses automatisées complexes sont soumises à la relecture d'enseignants partenaires avant leur mise à disposition générale.

### 2.6 Parcours Utilisateur, Gamification & Tableaux de Bord
* **Diagnostic Initial d'Onboarding** : À l'inscription, un test de positionnement court et interactif cartographie les acquis de l'élève et détecte ses points de blocage spécifiques.
* **Gamification Inspirée des Mécanismes de Duolingo** :
  * Maintien d'un rythme régulier par séries de révision quotidiennes (*streaks*).
  * Attribution de badges numériques à chaque palier de compétence validé.
  * Points d'expérience (XP) et classements stimulants au niveau de la classe, de l'école ou de la région.
* **Tableaux de Bord Métiers** :
  * **Dashboard Enseignant** : Visualisation en direct de l'état d'avancement de la classe (pastilles vertes pour les étapes réussies, alertes rouges instantanées si plus de 30 % des élèves achoppent sur la même notion).
  * **Dashboard Parent** : Suivi lisible du taux de couverture des compétences requises pour l'examen national, exempt de toute complexité technique.

---

# Partie 3 : Tous les Autres Points Abordés, Anecdotes Stratégiques, Aspects Juridiques & Organisationnels

### 3.1 Cadre de Collaboration du PFE & Contrat Moral Entre les Parties
* **Alignement « 2-en-1 »** : L'étudiant réalise son stage et sa soutenance académique de PFE tout en construisant les fondations techniques du produit e-learning, valorisant ainsi son travail universitaire en opportunité entrepreneuriale directe.
* **Répartition des Responsabilités** :
  * **Mohamed Belhassen** : Encadreur, Mentor, Product Owner et Stratège. Il pilote la vision d'ensemble, l'architecture pédagogique et logicielle, les relations administratives, les partenariats d'affaires et la stratégie légale.
  * **L'Étudiant** : Chef de Projet Technique et Développeur Principal. Il est chargé de l'implémentation du code, de l'expérimentation des agents, de la maintenance du dépôt et de la tenue des délais techniques.
* **Méthodologie Opérationnelle** :
  * Gestion et suivi des livrables via un tableau partagé sur **Trello**.
  * Synchronisations quotidiennes lors de la phase de cadrage, relayées par un point hebdomadaire lors du développement.
  * Progression par étapes validées : prototype initial ($V_0$), architecture découplée ($V_1$), puis version de test terrain (MVP).
  * Possibilité de déléguer certains modules secondaires (ex. interfaces de paiement, tableaux de bord de modération) à d'autres étudiants sous forme de mini-stages encadrés par l'étudiant.

### 3.2 L'Expérience Traumatisante Passée & Nécessité de l'Exit Strategy
Mohamed a expliqué l'origine de son exigence d'un contrat moral strict en partageant une épreuve vécue avec un ancien étudiant :
* **L'Incident de l'Ancien Étudiant** : Après avoir encadré un étudiant qui a brillamment obtenu 18/20 à son PFE, Mohamed a continué à l'accompagner en lui versant une aide financière de 300 DT par mois à perte. Le projet ayant été retenu pour une compétition en Algérie, l'étudiant s'est vu refuser la sortie du territoire à la frontière la veille du concours, en application de la loi tunisienne exigeant une autorisation parentale pour les moins de 35 ans. L'étudiant et sa mère ont alors accusé l'établissement et l'épouse de Mohamed de harcèlement, déposé des plaintes administratives et judiciaires injustifiées, saisi des ligues de droits de l'homme, refusé de rendre le code source développé et occasionné une convocation de Mohamed devant la justice.
* **Directives Fondatrices Tirées de cet Incident** :
  * **Résilience Structurelle (« Le projet ne s'arrête avec personne »)** : Aucune personne ne doit pouvoir bloquer le développement. L'ensemble des architectures, documentations, dépôts et identifiants doivent être documentés et partagés.
  * **Stratégie de Sortie (« Exit Strategy »)** : Définition préalable des droits en cas de séparation ou de départ à l'étranger (l'étudiant conserve les droits d'exploitation commerciale en Mauritanie, Mohamed en Tunisie, et partage contractuel des revenus sur les marchés internationaux).
  * **Transparence Absolue et Loyauté** : Respect scrupuleux de la parole donnée et primauté de la confiance mutuelle sur les contentieux juridiques formels.

### 3.3 Cadre Startup, Statut Étudiant-Entrepreneur & Fiscalité
* **Dispositif « Étudiant-Entrepreneur » (PEES)** :
  * Cursus structuré en trois niveaux : *Initiateur* (phase de maturation de l'idée), *Innovateur* (élaboration du Business Model Canvas - BMC et étude de marché) et *Promoteur* (création formelle de l'entreprise).
  * Mohamed a recommandé à l'étudiant de briguer le statut d'Innovateur et a annoncé sa participation à un séminaire dédié à Djerba pour maîtriser le cadre d'application du Startup Act.
* **Avantages Clés du Startup Act Tunisien** :
  * Obtention de la **Carte Technologique Internationale**, autorisant jusqu'à 100 000 TND (100 millions de millimes) d'achats annuels en devises pour payer les infrastructures cloud et les API d'IA (AWS, OpenAI, etc.).
  * Prise en charge par l'État de la rémunération de **trois fondateurs** à raison de 1 000 DT par mois chacun pendant deux ans.
  * Exonération complète de l'impôt sur les sociétés et franchises douanières sur les équipements informatiques importés.
* **Contraintes Fiscales & Choix d'une Structure Hôte au Démarrage** :
  * Mohamed a détaillé les prélèvements fiscaux en Tunisie : TVA à 19 % et impôt sur les sociétés (IS) entre 25 % et 35 %. Avec les retenues à la source et les charges sociales, les taxes peuvent atteindre 45 % à 50 % de la marge brute.
  * Mention de la plateforme *TUNEPS* (marchés publics tunisiens) comme modèle de segmentation d'abonnements limités à 5 rubriques clés par entreprise pour verrouiller le partage de compte.
  * Répercussions de la loi de finances 2026 : généralisation de la retenue à la source sur le commerce en ligne (illustrée par le cas de la plateforme Jumia en Tunisie, déstabilisée par l'obligation de tracer fiscalement ses vendeurs informels).
  * **Choix opérationnel** : Ne pas enregistrer prématurément de société juridique pour le MVP (ce qui génère des frais comptables et des obligations déclaratives mensuelles immédiates). Le projet s'adossra temporairement à une entreprise légale du réseau de Mohamed afin de facturer en toute conformité et d'intégrer des passerelles de paiement bancaire (Konnect, Flouci).
* **Préservation des Subventions Cloud** : Ne pas consommer prématurément les crédits d'hébergement gratuits offerts par les grands fournisseurs cloud (packages de 600 $ à 1 000 $ chez AWS) pendant la phase de conception, afin de les préserver pour la montée en charge réelle.

### 3.4 Opportunités et Réalités du Marché Mauritanien
* **Potentiel Démographique** : Un marché de 6 millions d'habitants à forte croissance démographique et forte jeunesse, où les besoins en modernisation éducative sont cruciaux.
* **Infrastructures de Paiement** : Forte pénétration du paiement mobile en Mauritanie avec plus de dix applications spécialisées (ex. *Bankily*). Néanmoins, l'absence d'outils de reversement international automatisé (*payouts*) exige de prévoir une structure de perception financière sur place.

### 3.5 Stratégie de Déploiement Terrain & Alliances Institutionnelles
Afin d'éviter les coûts exorbitants de publicité digitale (budgets de 10 000 DT sur Facebook ou les médias traditionnels sans garantie de conversion) :
* Signature d'un partenariat avec une **association scientifique reconnue**.
* Conclusion d'une convention formelle avec la **Délégation Régionale de l'Éducation (Mendoubiyya)** permettant d'accéder légalement aux collèges, lycées et écoles primaires pilotes pour expérimenter le système auprès des enseignants et des élèves.

### 3.6 Retours d'Expérience Techniques & Anecdotes Concrètes Partagées par Mohamed
* **Posture « CyberOps / Full-Stack Security »** :
  * Sécurité intégrée dès l'architecture (*Security by Design*) : protection hermétique contre l'injection de prompts (*prompt injection*) et sanctuarisation des données personnelles des élèves mineurs.
  * Intégration de tests d'analyse statique du code lors de chaque validation (*commit*) afin de prévenir les failles de sécurité avant tout déploiement en production.
* **Défaillance Réelle sur Vercel et Tâches Planifiées (Cron Jobs)** : Récit d'un incident survenu sur un projet hébergé sur le plan gratuit de Vercel, où une tâche planifiée de maintenance s'est arrêtée silencieusement pendant une semaine complète. Mohamed préconise d'installer des alertes proactives indépendantes sur smartphone (via *Ntfy* ou alertes push).
* **Environnement Outillé (Antigravity IDE & Serveurs MCP)** : Démonstration par Mohamed de son flux de travail combinant Antigravity, VS Code et les serveurs MCP (*Model Context Protocol*) interfacés avec Supabase, automatisant la mise à jour des bases de données et évitant les erreurs manuelles de saisie.
* **Projet Déployé d'Affichage Dynamique (« Campus TV / Info Tablette »)** : Démonstration de la plateforme développée par Mohamed pour diffuser des informations universitaires sur des écrans via des tablettes connectées : filtrage des annonces par département et date de fin de validité, intégration de QR codes dynamiques pointant vers le site de l'établissement, bandeaux défilants d'alerte (bandeau rouge d'urgence ou bleu informatif), informations météo et vidéos.
* **Application d'Aide aux Déficients Visuels** : Récit d'une consultation menée par Mohamed auprès d'un imam malvoyant pour identifier ses besoins réels. L'échange a fait émerger trois attentes concrètes : la mise en sourdine automatique du smartphone lors des appels à la prière (Adhan), l'ajout d'un capteur de luminosité à l'application *Be My Eyes* pour savoir si la lumière est allumée avant de contacter un bénévole, et un classifieur d'images permettant d'identifier les taxis officiels et d'éviter les malentendus avec des automobilistes particuliers.
* **L'Anecdote du Juge et de la Conférence sur les Droits de l'Homme (1997)** : Souvenir d'enfance de Mohamed accompagnant son père, alors magistrat, dans une conférence sur les droits de l'homme près d'El Metouia. Alors que le magistrat évoquait le droit à l'information et la révolution d'Internet, un habitant lui a fait remarquer qu'il manquait de nourriture, de vêtements et de transport pour envoyer son enfant à l'école. Cette anecdote résume la philosophie de Mohamed : **répondre impérativement à des besoins réels et concrets plutôt que de développer des solutions techniques déconnectées du terrain**.
* **Soft Skills, Pitch Commercial & Leadership** :
  * *Vente de la Solution E-Commerce (Ayas)* : Pour commercialiser une plateforme de 3 000 produits, Mohamed s'est abstenu d'utiliser le moindre jargon technique (pas de mention de VPS, Supabase ou Next.js), démontrant simplement au client un score Google PageSpeed de 96 % et un temps de chargement inférieur à la seconde.
  * *La Réussite de Jack Ma (Alibaba)* : Rappel que Jack Ma, initialement professeur d'anglais sans bagage informatique, a bâti son entreprise sur l'art de convaincre les autorités, de négocier et de faire collaborer des experts techniques autour d'une vision unifiée.
  * *La Commande Arduino de Chine* : Expérience d'un achat de microcontrôleur Arduino commandé en Chine pour 0,98 $ port compris, mais taxé de 5 TND de frais postaux à son arrivée en Tunisie, rappelant l'impact déterminant des réalités douanières et réglementaires locales.

---

## Synthèse Opérationnelle des Décisions Validées

| Domaine | Décision Stratégique & Arbitrage | Porteur |
| :--- | :--- | :--- |
| **Gouvernance & Vision** | Pilotage général, Product Ownership, validation pédagogique et partenariats | Mohamed Belhassen |
| **Architecture Logicielle** | Développement de l'orchestrateur découplé, linter visuel et intégration API | L'Étudiant |
| **Gestion Opérationnelle** | Planification sur Trello, jalons $V_0$, $V_1$, MVP et synchronisations régulières | Équipe conjointe |
| **Périmètre du MVP** | 6ème année primaire (concours collèges pilotes) - Mathématiques & Sciences | Équipe conjointe |
| **Garantie Pédagogique** | Pédagogie par indices (*hints*), validation Human-in-the-Loop et guides du maître | Mohamed Belhassen |
| **Valorisation PFE** | Alignement des livrables techniques du MVP avec le mémoire académique | Équipe conjointe |
| **Sécurisation Commerciale**| Adossement initial à une entreprise partenaire et convention associative locale | Mohamed Belhassen |
