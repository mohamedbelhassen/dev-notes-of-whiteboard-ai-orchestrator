# 02 — Gestion des Utilisateurs, Contrôle d'Accès, Crédits & BYOK

Ce document décrit les flux fonctionnels et règles métiers gouvernant l'authentification, la liste d'attente bêta (*Waitlist*), le système de quotas de crédits journaliers et la gestion des clés API personnelles (*Bring Your Own Key — BYOK*).

---

## 1. Flux Utilisateurs (User Flows)

### 1.1 Inscription sur la Liste d'Attente (Waitlist) & Accès Bêta
1. **Demande d'invitation** :
   - Sur la page d'accueil ou d'accès restreint, un visiteur saisit son adresse e-mail, son rôle et son cas d'usage.
   - L'appel `POST /v1/waitlist` ([`apps/functions/src/crud.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/apps/functions/src/crud.ts#L121-L163)) normalise l'e-mail en minuscules, l'enregistre dans la table Supabase `waitlist` et envoie un e-mail de confirmation via l'API Resend.
2. **Connexion & Vérification d'Approbation** :
   - L'utilisateur se connecte via **Magic Link** (`POST /v1/auth/magic-link` dans [`crud.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/apps/functions/src/crud.ts#L165-L214)) ou via **OAuth Supabase** (Google/GitHub).
   - Lors du chargement du profil via `GET /v1/me` ([`crud.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/apps/functions/src/crud.ts#L216-L240)), le backend vérifie si l'utilisateur est approuvé dans la table `waitlist` (`approved = true`), s'il figure dans la liste d'exceptions `WAITLIST_BYPASS_EMAILS`, ou si `WAITLIST_OPEN = "true"`.
   - Tant que `waitlistApproved` vaut `false`, toutes les routes d'intelligence artificielle (`/v1/core/agent/*`) refusent l'exécution avec un statut HTTP `403 Forbidden (waitlist_pending)`.

### 1.2 Consommation des Crédits Journaliers
1. **Consultation du Solde** :
   - Le client interroge `GET /v1/credits` ([`apps/functions/src/index.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/apps/functions/src/index.ts#L1087-L1101)) qui appelle la fonction RPC Supabase `credit_balance` via [`creditsRepo.getStatus`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/db/src/index.ts#L397-L444).
2. **Vérification Pré-Exécution** :
   - Avant chaque requête IA utilisant les clés de la plateforme (Chat, Planification, Recherche, TTS, Live), le backend vérifie le solde journalier restant (`remaining > 0`).
   - Si le quota journalier est épuisé (`remaining <= 0`), le backend renvoie une erreur `402 Payment Required` (`code: "credits_exhausted"`), invitant l'utilisateur à attendre la réinitialisation à minuit UTC ou à brancher sa propre clé API (BYOK).
3. **Décompte Post-Exécution** :
   - À la fin du flux de génération, [`creditsRepo.record`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/db/src/index.ts#L446-L468) insère une ligne dans `credit_usage` avec le nombre réel de tokens consommés (`inputTokens + outputTokens`) ou le forfait fixe de l'opération.

### 1.3 Configuration et Utilisation d'une Clé API Personnelle (BYOK)
1. **Validation & Enregistrement (`POST /v1/keys`)** :
   - L'utilisateur colle une clé API (`openai`, `anthropic`, `google`, `meta`, `openrouter` ou `vercel`) dans les paramètres.
   - [`keysRoutes`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/apps/functions/src/keys.ts#L60-L120) valide d'abord le format syntaxique via `provider.keyPattern`, puis interroge en direct l'endpoint `/models` du fournisseur via [`checkProviderKey`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/providers.ts#L78-L111) pour récupérer la liste des modèles de chat accessibles.
   - La clé est chiffrée en **AES-256-GCM** via [`sealProviderKey`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/db/src/index.ts#L557-L579) à l'aide du secret serveur `USER_KEYS_ENCRYPTION_KEY` avant d'être stockée dans la table `provider_keys` (seuls les 4 derniers caractères sont conservés en clair dans `key_hint`).
2. **Exécution Débridée (Sans Quotas)** :
   - Lorsqu'un appel `/v1/core/agent/chat` cible un modèle `key:<provider>:<model>`, [`resolveUserModelKey`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/apps/functions/src/index.ts#L497-L534) déchiffre la clé en mémoire pour la durée de la requête. Aucun crédit plateforme n'est vérifié ni débité.

---

## 2. Règles Métiers Actuelles (Business Rules)

### 2.1 Barème et Quotas de Crédits ([`packages/db/src/index.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/db/src/index.ts#L387-L468) & [`apps/functions/src/index.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/apps/functions/src/index.ts#L462-L495))
| Paramètre / Opération | Valeur / Coût | Règle d'Application |
| :--- | :--- | :--- |
| **Quota Standard (`DAILY_CREDIT_LIMIT`)** | **100 000 crédits / jour** | Réinitialisé automatiquement chaque jour à `00:00:00 UTC`. |
| **Quota Comptes Privilégiés (`UNLIMITED_CREDIT_LIMIT`)** | **5 000 000 crédits / jour** | Appliqué aux e-mails listés dans `UNLIMITED_CREDITS_EMAILS` ou codés dans [`UNLIMITED_EMAILS`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/apps/functions/src/index.ts#L468-L473). |
| **Chat & Leçons (`chat`, `lesson`)** | `inputTokens + outputTokens` (min. `500`) | Si le flux s'interrompt avant que le fournisseur ne remonte `totalUsage`, un forfait minimal de `500` crédits est débité. |
| **Planification Live (`plan-lesson`)** | Forfait de **1 500 crédits** | Débité lors de la génération d'un plan de leçon temps réel. |
| **Recherche Documentaire / Web (`research`)** | `totalTokens` (min. `1 000`) | Débité lors d'un appel à `/v1/core/agent/research`. |
| **Synthèse Vocale (`speech`)** | Forfait de **500 crédits** | Débité par segment audio généré via `/v1/core/agent/speech`. |
| **Session Vocale Temps Réel (`live`)** | Forfait de **5 000 crédits** | Débité à l'émission du jeton éphémère Gemini Live (`/v1/core/agent/live-token`). |

### 2.2 Règles de Sécurité & Rate-Limiting ([`packages/auth/src/index.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/auth/src/index.ts))
- **Authentification JWT** :
  - Toute requête protégée doit présenter un jeton `Bearer` Supabase vérifié cryptographiquement auprès du endpoint JWKS (`/.well-known/jwks.json`) du projet Supabase ([`verifySupabaseToken`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/auth/src/index.ts#L41-L75)).
- **Filtrage des Modèles BYOK ([`providers.ts`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/core/src/agent/providers.ts#L72-L132))** :
  - Lors de l'inspection des modèles d'une clé API, les modèles non conversationnels ou dépourvus de support d'outils (`embed`, `tts`, `whisper`, `dall-e`, `imagen`, `veo`, `moderation`, `live`, etc.) sont exclus via l'expression régulière `NOT_CHAT`. Pour OpenRouter, la présence de `"tools"` dans `supported_parameters` est exigée.

---

## 3. Entités du Domaine (Domain Entities)

| Entité / Schéma | Fichier Source | Attributs & Structure |
| :--- | :--- | :--- |
| [`ProfileRecord`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/db/src/index.ts#L10-L16) | `packages/db/src/index.ts` | `id` (UUID Supabase Auth), `email`, `display_name`, `avatar_url`, `created_at`. |
| [`CreditStatus`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/db/src/index.ts#L390-L395) | `packages/db/src/index.ts` | `used` (crédits consommés aujourd'hui), `limit` (plafond journalier), `remaining`, `resetsAt` (timestamp ISO du prochain minuit UTC). |
| [`ProviderKeyRecord`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/db/src/index.ts#L494-L504) | `packages/db/src/index.ts` | `user_id`, `provider` (`KeyProviderId`), `encrypted_key` (format `v1.<iv_b64url>.<cipher_b64url>`), `key_hint`, `selected_model`, `models` (`ProviderModel[]`), `updated_at`. |
| [`ProviderKeyStatus`](file:///c:/dev/_current_apps/whiteboard-ai-orchestrator/inklm/packages/db/src/index.ts#L506-L514) | `packages/db/src/index.ts` | Vue publique sans secret renvoyée au client : `provider`, `hint`, `selectedModel`, `models`, `updatedAt`. |
