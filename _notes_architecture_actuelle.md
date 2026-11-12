# dans "Boucle d'Exécution Multi-Étapes (Tool Calling)", pour la recherche documentaire, on doit tenir compte que la documentation peut être fournie depuis l'extérieur (RAG/ ou comme on a fait dans socratix). donc, on doit être capable de controler les sources d'information disponibles à l'orchestrateur (documentation fixée/recherche web)

# dans "Assemblage du Contexte par l'Orchestrateur", le fait de résumer l'historique de messages lorsqu'on dépasse 12 msg va faire perdre des détails qui peuvent être inmportants pour la compréhension des dernières intéractions.
## approche proposée: fenêtres glissante des X derniers messages inclus integralement et on résume les messages qui dépassent X (plus anciens que X messages)

# Selon la section "Choix du Profil de Tuteur Spécialisé", on a plusieurs profils de tuteurs ((General, IT, Math, Physics, Medical, etc). Donc, dans l'orchestrateur, on aura besoin d'un agent spécialisé dans la gestion de tuteurs:
## la plateforme externe (inklm) instructe l'orchestrateur de créer un profil de tuteur donné à partir d'un nom ou une description donnée et l'orchstrateur doit être capable de stocker ce profil dans son backend et l'invoquer lorsque nécessaire.
## Puis, le routage des requêtes utilisateurs peut être fixé par (inklm) ou laisser l'orchestrateur faire un dispatching autmatique via un classifieur (jev/etc). 

# Il faut avoir un agent ervant comme parefeux contre le prompt injection qui sera utilisé pour se protéger nativement contre tous ces attaques en analysant tous les inputs (prompts) avant de les traiter

# (toDo by Belhassen) pour tous les appels de IA ça doit être fait via un gateway centralisé. comme ça, on peut faire le tracking de l'utilisation/prise en charge de nouveau fournisseurs de LLM (openrouter/gemini/deepInfre/meta AI/etc) d'une manière facile. la gestion de ces modèles (activation (on/off) /clé api/ etc) va se faire via une table app_configs. les clés à stocker d'une manière sécurisée (e.g Vault)

# (toDo by Belhassen) l'envoi d'emails se fait via une gateway API centralisée pour faciliter le tracking d'usage et le re-envoi d'emails en cas de pannes.

# La gestion de canvas doit être faite exclusivement via un agent dédié "Canvas Manager" comme ça, la gestion
