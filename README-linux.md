# Déployer votre IA souveraine dans le cadre de l'expérimentation Mistral AMUE

> ⚠️ **VERSION DE TRAVAIL — contenu non encore validé**
> Le présent document est un document de travail en cours de test au sein des équipes d'Esup.
> N'hésitez pas à nous faire remonter vos questions ou remarques dans les [issues github](https://github.com/EsupPortail/esup-myia-mistral-amue/issues) associées à ce projet.

| Date | Auteur | Modification | Validation |
|---|---|---|---|
| 16 avr. 2026 | Nicolas Truchaud | Version initiale | |
| 20 mai 2026 | David Boit | Debian dans tableau OS, lien doc officielle Docker, ports et volumes externalisés dans le .env avec valeur par défaut (`:-`), réseau Docker renommé `nw_llm`. | |

---

> **Contexte** : Ce tutoriel s'appuie sur les Ateliers IA Esup 2025/2026 (Université de Rennes / Université de Strasbourg) pour assembler une stack complète type ChatGPT souverain. Le choix architectural retenu ici est d'utiliser l'API Mistral via l'infrastructure ILaaS négociée par l'AMUE pour l'enseignement supérieur français — ce qui permet de démarrer sans infrastructure GPU et avec un haut niveau de souveraineté (données hébergées en Europe, opérateur français). Si votre établissement dispose de GPU et souhaite une indépendance totale vis-à-vis de toute API externe, la brique Mistral API peut être remplacée par un moteur vLLM local — la configuration est décrite dans l'Atelier 2 Esup.
>
> **Modèle disponible via l'accord AMUE** : Mistral Medium 3 (`mistral-medium-latest`), hébergé sur l'infrastructure souveraine ILaaS du CINES. Température recommandée : 0.1 ou inférieure.
>
> **RAG / embeddings** : non disponibles via ILaaS — configurer Ollama + `nomic-embed-text` dans OpenWebUI (Admin → Settings → Documents).
>
> Si vous disposez d'une clé Mistral standard (hors accord AMUE/ILaaS), vous pouvez l'utiliser : les blocs commentés dans `config/litellm_config.yaml` et `.env.example` vous permettent de basculer facilement — décommentez `MISTRAL_API_KEY` et les modèles avec `api_base: https://api.mistral.ai/v1`.

---

## Sommaire

1. [Les ingrédients](#1-les-ingrédients)
2. [Architecture de la solution](#2-architecture-de-la-solution)
3. [Mise en place pas à pas](#3-mise-en-place-pas-à-pas)
4. [Vérifier que tout fonctionne](#4-vérifier-que-tout-fonctionne)
5. [Et ensuite ?](#5-et-ensuite-)

---

## 1. Les ingrédients

### Matériel & Infrastructure

| Composant | Minimum | Recommandé |
|---|---|---|
| CPU | 4 vCPU | 8 vCPU |
| RAM | 8 Go | 16 Go |
| Disque | 20 Go SSD | 50 Go SSD |
| OS | Ubuntu 22.04 LTS ou Debian 12 | Ubuntu 24.04 LTS ou Debian 12 |
| Réseau | Accès HTTPS sortant vers llm.ilaas.fr | + nom de domaine propre |

> Pas de GPU requis : les inférences sont déléguées à l'API ILaaS (cloud souverain français).

### Logiciels requis

- **Docker Engine** 24+ et **Docker Compose** v2
- **Git**
- **Python 3.11+** (optionnel, pour les tests)

### Accès & Clés

- **Clé API ILaaS (Mistral AMUE)** : fournie par votre référent numérique via le dispositif AMUE.
  Forme attendue : `sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`
  Endpoint de base : `https://llm.ilaas.fr/v1` — [documentation ILaaS](https://www.ilaas.fr/services-inference/)
  Modèle disponible : **Mistral Medium 3** (`mistral-medium-latest`)

- **Option Mistral standard** : si vous n'avez pas de clé ILaaS, une clé Mistral classique (mistral.ai) fonctionne aussi. Voir les blocs commentés dans `config/litellm_config.yaml` et `.env.example`.

- **Accès SSH** au serveur cible
- **Droits sudo** sur le serveur

### Compétences requises

- Administration Linux (niveau intermédiaire)
- Bases de Docker / Docker Compose
- Notions de LLM et d'API REST (cf. Ateliers 1 & 2 Esup)

---

### RGPD

> Avant de démarrer : les prompts que vous envoyez transitent vers les serveurs de Mistral AI via l'infrastructure ILaaS (CINES) pour traitement. Mistral est un opérateur français, les données sont hébergées en Europe, et l'accord AMUE prévoit la non-utilisation de vos données pour le réentraînement des modèles — mais vérifiez ce point avec votre DPO avant tout usage impliquant des données sensibles. Ne faites jamais transiter de données personnelles d'étudiants ou d'agents via cet outil sans analyse préalable.

---

## 2. Architecture de la solution

```
                            ┌─────────────────────────────────┐
                            │        Établissement             │
  ┌─────────┐   HTTPS       │  ┌──────────────────────────┐   │
  │  Users  │──────────────▶│  │   Nginx / Traefik         │   │
  └─────────┘               │  │   (reverse proxy + TLS)   │   │
                            │  └────────────┬─────────────┘   │
                            │               │ :3000            │
                            │  ┌────────────▼─────────────┐   │
                            │  │   OpenWebUI              │   │
                            │  │   (interface ChatGPT-like)│   │
                            │  └────────────┬─────────────┘   │
                            │               │ :4000 (OpenAI API)│
                            │  ┌────────────▼─────────────┐   │
                            │  │   LiteLLM Proxy          │   │
                            │  │   (gateway + quotas)     │   │
                            │  └────────────┬─────────────┘   │
                            │               │                  │
                            │  ┌────────────▼─────────────┐   │
                            │  │  PostgreSQL + Redis       │   │
                            │  │  (logs, budgets, users)   │   │
                            │  └──────────────────────────┘   │
                            └──────────────┬──────────────────┘
                                           │ HTTPS
                                           ▼
                            ┌──────────────────────────────┐
                            │  API ILaaS — llm.ilaas.fr     │
                            │  Mistral Medium 3 (AMUE)      │
                            └──────────────────────────────┘
```

**Pourquoi cette pile ?**

- **Mistral AI via ILaaS (CINES)** : modèles français, données hébergées en Europe, conformité RGPD facilitée
- **LiteLLM** : passerelle open source qui uniformise l'accès, gère les quotas par utilisateur/équipe, et permet d'ajouter d'autres modèles sans changer l'interface
- **OpenWebUI** : interface open source type ChatGPT, supporte RAG, SSO, outils, gestion des espaces de travail
- **PostgreSQL + Redis** : persistance des logs, budgets, sessions — aucune donnée ne sort du périmètre établissement

---

## 3. Mise en place pas à pas

> Tous les fichiers de configuration sont dans le repo cloné. Le répertoire de travail est `~/myia`.

### Étape 1 — Préparer le serveur

```bash
# Mise à jour du système
sudo apt update && sudo apt upgrade -y

# Installation de Docker
# Méthode recommandée : suivre la documentation officielle selon votre distribution
# https://docs.docker.com/engine/install/
# Pour Ubuntu/Debian, le script automatique fonctionne aussi :
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker

# Vérification
docker --version          # Docker version 24+
docker compose version    # Docker Compose version v2+
```

---

### Étape 2 — Cloner le repo

```bash
git clone https://github.com/EsupPortail/esup-myia-mistral-amue.git ~/myia
cd ~/myia

# Créer les répertoires de données
mkdir -p data/{postgres,redis,openwebui}
```

Le repo contient déjà tous les fichiers nécessaires :
- `docker-compose.yml` — stack Linux / serveur
- `config/litellm_config.yaml` — modèles et paramètres LiteLLM (préconfiguré pour ILaaS)
- `.env.example` — template de configuration à copier

---

### Étape 3 — Fichier d'environnement

Copier le template et remplir les secrets :

```bash
cp .env.example .env
chmod 600 .env
```

Ouvrir `.env` dans un éditeur et remplacer **toutes** les valeurs de substitution :

| Variable | Rôle | Valeur |
|---|---|---|
| `ILAAS_API_KEY` | Clé API ILaaS fournie par l'AMUE | Votre vraie clé `sk-xxx...` |
| `LITELLM_MASTER_KEY` | Clé admin LiteLLM | `sk-` + 32 caractères aléatoires |
| `LITELLM_SECRET_KEY` | Secret JWT interne | Chaîne aléatoire longue |
| `POSTGRES_PASSWORD` | Mot de passe base de données | Chaîne aléatoire |
| `REDIS_PASSWORD` | Mot de passe cache Redis | Chaîne aléatoire |
| `WEBUI_SECRET_KEY` | Secret session OpenWebUI | Chaîne aléatoire |
| `WEBUI_NAME` | Nom affiché dans l'interface | Ex: `MyIA — Université de Strasbourg` |
| `WEBUI_URL` | URL publique future de l'instance | Ex: `https://chat.votre-etablissement.fr` |

Pour générer des valeurs robustes directement dans le Terminal :

```bash
for i in 1 2 3 4 5 6; do python3 -c "import secrets; print(secrets.token_urlsafe(32))"; done
```

> ⚠️ Remplir le `.env` **avant** `docker compose up -d`. Une fois PostgreSQL démarré une première fois, les mots de passe sont gravés en base et ne peuvent plus être changés sans réinitialiser les données.

> **Option Mistral standard** : si vous utilisez une clé Mistral classique (hors accord AMUE/ILaaS), décommentez la ligne `# MISTRAL_API_KEY` dans le `.env` et adaptez `config/litellm_config.yaml` (voir les blocs commentés dans ce fichier).

---

### Étape 4 — Vérifier la configuration LiteLLM

Le fichier `config/litellm_config.yaml` est préconfiguré pour l'endpoint ILaaS :

- **Endpoint actif** : `https://llm.ilaas.fr/v1`
- **Variable de clé** : `ILAAS_API_KEY`
- **Modèle disponible via ILaaS AMUE** : `mistral-medium` (Mistral Medium 3 — `mistral-medium-latest`)
- **Température** : 0.1 (recommandé par ILaaS)
- **RAG / embeddings** : non disponibles via ILaaS — configurer Ollama + `nomic-embed-text` dans OpenWebUI

Aucune modification n'est nécessaire si vous avez une clé ILaaS. Pour basculer sur une clé Mistral standard (accès à `mistral-large`, `codestral`, `mistral-embed`...), commentez le bloc ILaaS et décommentez les blocs `MISTRAL_API_KEY / https://api.mistral.ai/v1` dans ce fichier.

---

### Étape 5 — Premier démarrage

```bash
# Se placer dans le répertoire du projet
cd ~/myia

# Lancer tous les services
docker compose up -d

# Surveiller le démarrage (attendre ~60s la première fois)
docker compose ps

# Logs en temps réel si besoin
docker compose logs -f litellm
docker compose logs -f openwebui

# Vérifier la santé des services
docker compose ps --format "table {{.Name}}\t{{.Status}}\t{{.Ports}}"
```

Résultat attendu — tous les services doivent afficher `healthy` :

```
NAME                 STATUS                    PORTS
myia-postgres        Up X minutes (healthy)    5432/tcp
myia-redis           Up X minutes (healthy)    6379/tcp
myia-litellm         Up X minutes (healthy)    0.0.0.0:4000->4000/tcp
myia-openwebui       Up X minutes (healthy)    0.0.0.0:3000->8080/tcp
```

---

### Étape 6 — Configuration initiale OpenWebUI

1. Ouvrir http://localhost:3000
2. Créer le **compte administrateur** (premier compte = admin automatiquement)
3. Aller dans **Paramètres → Connexions** et vérifier que LiteLLM est détecté
4. Aller dans **Admin → Utilisateurs** pour créer les comptes

---

## 4. Vérifier que tout fonctionne

On vérifie couche par couche, du bas vers le haut : d'abord l'infrastructure, puis l'API, puis l'interface utilisateur.

---

### 4.1 — Les conteneurs sont-ils tous en vie ?

```bash
docker compose ps --format "table {{.Name}}\t{{.Status}}\t{{.Ports}}"
```

Tous les services doivent afficher `healthy`. Si un service est en `starting` ou `unhealthy`, consulter ses logs :

```bash
docker compose logs --tail=50 litellm
docker compose logs --tail=50 openwebui
```

---

### 4.2 — LiteLLM reçoit-il bien la clé ILaaS ?

```bash
curl -s \
  -H "Authorization: Bearer sk-master-CHANGEZ-MOI-en-prod" \
  http://localhost:4000/health | python3 -m json.tool
```

La réponse doit contenir un statut `healthy` pour le modèle `mistral-medium`. Si le modèle remonte `unhealthy`, c'est généralement la clé ILaaS qui est incorrecte ou l'endpoint qui ne répond pas — vérifier la valeur de `ILAAS_API_KEY` dans le `.env`.

---

### 4.3 — Le modèle Mistral est-il bien exposé ?

```bash
curl -s \
  -H "Authorization: Bearer sk-master-CHANGEZ-MOI-en-prod" \
  http://localhost:4000/models | python3 -m json.tool
```

On doit voir le modèle déclaré dans `config/litellm_config.yaml` (config ILaaS par défaut) :

```json
{
  "data": [
    {"id": "mistral-medium", "object": "model"}
  ]
}
```

> Avec une clé Mistral standard, vous verriez également `mistral-large`, `codestral` et `mistral-embed`.

---

### 4.4 — Le chat fonctionne-t-il de bout en bout ?

Ce script Python vérifie les cas essentiels en une seule passe.

```python
# recette.py
import openai, sys

BASE_URL = "http://localhost:4000/v1"
API_KEY  = "sk-master-CHANGEZ-MOI-en-prod"

client = openai.OpenAI(api_key=API_KEY, base_url=BASE_URL)
ok = True

def check(label, fn):
    global ok
    try:
        fn()
        print(f"✅  {label}")
    except Exception as e:
        print(f"❌  {label} → {e}")
        ok = False

def test_chat():
    rep = client.chat.completions.create(
        model="mistral-medium",
        messages=[
            {"role": "system", "content": "Tu es un assistant de l'ESR français."},
            {"role": "user",   "content": "Dis bonjour en une phrase."}
        ],
        max_tokens=50
    )
    assert rep.choices[0].message.content.strip(), "Réponse vide"
    assert rep.usage.completion_tokens > 0, "Pas de tokens comptabilisés"

def test_streaming():
    tokens = []
    stream = client.chat.completions.create(
        model="mistral-medium",
        messages=[{"role": "user", "content": "Compte de 1 à 3."}],
        stream=True
    )
    for chunk in stream:
        delta = chunk.choices[0].delta.content
        if delta:
            tokens.append(delta)
    assert tokens, "Aucun token reçu en streaming"

print("=== Recette MyIA AMUE ===\n")
check("Chat simple (mistral-medium)",    test_chat)
check("Streaming (mistral-medium)",      test_streaming)

print(f"\n{'✅ Recette réussie — passage en production possible.' if ok else '❌ Des tests ont échoué — ne pas passer en production.'}")
sys.exit(0 if ok else 1)
```

```bash
pip install openai
python3 recette.py
```

---

### 4.5 — L'interface OpenWebUI est-elle fonctionnelle ?

Ouvrir http://localhost:3000 et dérouler les vérifications suivantes dans l'ordre :

**Connexion et accès**
- [ ] La page de connexion s'affiche sans erreur
- [ ] La création du compte administrateur (premier lancement) fonctionne
- [ ] Après connexion, le modèle `mistral-medium` apparaît dans le menu déroulant

**Conversation**
- [ ] Envoyer *"Bonjour, qui es-tu ?"* → la réponse s'affiche en streaming, en français
- [ ] Le compteur de tokens apparaît sous la réponse (activer dans Paramètres si besoin)

**RAG — Retrieval Augmented Generation**

> Les embeddings ne sont pas disponibles via ILaaS. Pour activer le RAG, configurer Ollama + `nomic-embed-text` dans OpenWebUI :
> - Installer Ollama et lancer `ollama pull nomic-embed-text`
> - OpenWebUI : **Admin → Settings → Documents** → Embedding Engine : `Ollama` → URL : `http://host.docker.internal:11434` (Linux : IP hôte, ex: `172.17.0.1`)

- [ ] Uploader un PDF (ex: le règlement intérieur de votre établissement) via l'icône trombone
- [ ] Poser une question dont la réponse se trouve dans ce document
- [ ] Vérifier que la réponse cite correctement le document et n'hallucine pas

**Administration**
- [ ] Aller dans **Admin → Utilisateurs** et créer un second compte de test
- [ ] Se connecter avec ce compte : vérifier qu'il voit les modèles mais pas le panneau admin
- [ ] Retourner en admin et vérifier que la conversation du compte de test apparaît dans les logs

---

### 4.6 — Que faire si quelque chose ne va pas ?

| Symptôme | Cause probable | Action |
|---|---|---|
| Conteneur `litellm` en `unhealthy` | PostgreSQL pas encore prêt | Attendre 30s, relancer `docker compose up -d` |
| Erreur 401 sur `/health` | Mauvaise valeur de `LITELLM_MASTER_KEY` | Vérifier le `.env` et redémarrer |
| Erreur 429 ou `quota exceeded` | Clé AMUE épuisée ou trop de requêtes | Contacter votre référent AMUE |
| Modèles absents dans OpenWebUI | OpenWebUI ne joint pas LiteLLM | Vérifier les logs OpenWebUI, tester `curl http://litellm:4000/models` depuis le conteneur |
| RAG sans résultat pertinent | Modèle d'embedding non configuré | Configurer Ollama + nomic-embed-text dans Admin → Settings → Documents |
| Streaming coupé au bout de 30s | Timeout nginx trop court | Augmenter `proxy_read_timeout` à 600s dans la config Nginx |
| Impossible de créer le compte admin | `ENABLE_SIGNUP: "false"` | Passer à `"true"` le temps de créer le compte, puis remettre à `"false"` |

---

## 5. Et ensuite ?

Votre instance MyIA est maintenant fonctionnelle en local. Vous pouvez vous connecter sur http://localhost:3000, dialoguer avec Mistral Medium 3, et tester le RAG avec vos propres documents.

C'est suffisant pour expérimenter, former vos collègues et valider l'intérêt de la solution dans votre contexte.

Pour déployer cette stack dans votre établissement et l'ouvrir à vos utilisateurs, les étapes suivantes feront l'objet d'un second tutoriel :

- Exposition publique sécurisée avec Nginx et certificat TLS
- Authentification via le SSO de votre établissement (Shibboleth / Keycloak)
- Gestion des quotas et budgets par équipe ou département
- Monitoring, alerting et sauvegardes automatisées

---

*Tutoriel rédigé dans le cadre des Ateliers IA Esup 2025/2026 — basé sur les briques open source LiteLLM, OpenWebUI et l'offre Mistral AMUE via ILaaS (CINES).*
