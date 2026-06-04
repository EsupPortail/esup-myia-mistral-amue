# Déployer votre IA souveraine dans le cadre de l'expérimentation Mistral AMUE
## Version maquette locale — macOS Apple Silicon (M2)

> ✅🍎 **VERSION VALIDÉE SUR MACOS**
> Ce document est une adaptation de la version Linux pour une installation de maquette locale sur MacBook Air M2. Il ne couvre pas un déploiement en production.

---

> **Contexte** : Ce tutoriel permet d'installer la stack MyIA souveraine (Mistral AMUE via ILaaS + LiteLLM + OpenWebUI) sur un MacBook Air M2 pour expérimentation locale. L'objectif est d'avoir une instance fonctionnelle sur votre poste en moins d'une heure, sans serveur Linux.
>
> **Modèle disponible via l'accord AMUE** : Mistral Medium 3 (`mistral-medium-250523`), hébergé sur l'infrastructure souveraine ILaaS du CINES. Température recommandée : 0.1 ou inférieure.
>
> **RAG / embeddings** : non disponibles via ILaaS — configurer Ollama + `nomic-embed-text` dans OpenWebUI (voir étape 6b).

---

| Date | Auteur | Modification | Validation |
|---|---|---|---|
| 29 mai 2026 | | Version initiale macOS M2 | ✅ |

---

## Sommaire

1. [Les ingrédients](#1-les-ingrédients)
2. [Architecture de la solution](#2-architecture-de-la-solution)
3. [Mise en place pas à pas](#3-mise-en-place-pas-à-pas)
4. [Vérifier que tout fonctionne](#4-vérifier-que-tout-fonctionne)
5. [Et ensuite ?](#5-et-ensuite-)

---

## 1. Les ingrédients

### Matériel

| Composant | Votre config | Suffisant ? |
|---|---|---|
| Puce | Apple M2 (ARM64) | ✅ |
| RAM | 16 Go | ✅ (8 Go alloués à Docker, largement suffisant) |
| Disque | 20 Go libres recommandés | ✅ |
| OS | macOS Sequoia 15.x | ✅ |

> Pas de GPU requis : les inférences sont déléguées à l'API ILaaS (cloud souverain français).

### Logiciels à installer

- **Docker Desktop for Mac** (ARM64) — obligatoire, remplace Docker Engine sur Mac
- **Homebrew** — gestionnaire de paquets macOS
- **Python 3.11+** — optionnel, pour les tests

### Accès & Clés

- **Clé API ILaaS (Mistral AMUE)** : fournie par votre référent numérique via le dispositif AMUE.
  Forme attendue : `sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`
  Endpoint de base : `https://llm.ilaas.fr/v1` — [documentation ILaaS](https://www.ilaas.fr/services-inference/)
  Modèle disponible : **Mistral Medium 3** (`mistral-medium-250523`)

- **Option Mistral standard** : si vous n'avez pas encore de clé ILaaS, une clé Mistral classique (mistral.ai) fonctionne aussi. Voir les blocs commentés dans `config/litellm_config.yaml` et `.env.example`.

- **Sans clé du tout** : l'étape 6b décrit comment utiliser Ollama (modèle local) en attendant votre clé ILaaS.

### Compétences requises

- Bases du Terminal macOS
- Notions de Docker (lancer des conteneurs)
- Notions de LLM et d'API REST (cf. Ateliers 1 & 2 Esup)

---

### RGPD

> Avant de démarrer : les prompts que vous envoyez transitent vers les serveurs de Mistral AI via l'infrastructure ILaaS (CINES) pour traitement. Mistral est un opérateur français, les données sont hébergées en Europe, et l'accord AMUE prévoit la non-utilisation de vos données pour le réentraînement des modèles — mais vérifiez ce point avec votre DPO avant tout usage impliquant des données sensibles. Ne faites jamais transiter de données personnelles d'étudiants ou d'agents via cet outil sans analyse préalable.

---

## 2. Architecture de la solution

Identique à la version Linux — tout tourne dans des conteneurs Docker Desktop :

```
  ┌──────────────────────────────────────────┐
  │        MacBook Air M2                     │
  │                                          │
  │  http://localhost:3000                   │
  │  ┌───────────────────────────────────┐   │
  │  │   OpenWebUI (interface ChatGPT)   │   │
  │  └─────────────────┬─────────────────┘   │
  │                    │ :4000               │
  │  ┌─────────────────▼─────────────────┐   │
  │  │   LiteLLM Proxy (gateway)         │   │
  │  └─────────────────┬─────────────────┘   │
  │                    │                     │
  │  ┌─────────────────▼─────────────────┐   │
  │  │   PostgreSQL + Redis              │   │
  │  └───────────────────────────────────┘   │
  └──────────────────────┬───────────────────┘
                         │ HTTPS
                         ▼
          API ILaaS — llm.ilaas.fr
          Mistral Medium 3 (accord AMUE)
```

---

## 3. Mise en place pas à pas

> Tous les fichiers de configuration sont dans le repo cloné. Le répertoire de travail est `~/myia`. Ouvrir le Terminal et ne plus le quitter.

---

### Étape 1 — Installer les prérequis

**Docker Desktop for Mac**

Télécharger et installer la version Apple Silicon (ARM64) depuis le site officiel :
https://docs.docker.com/desktop/install/mac-install/

Une fois installé, lancer Docker Desktop depuis le Launchpad. Attendre que la baleine dans la barre de menu soit stable (pas animée).

> **Réglage mémoire important** : par défaut Docker Desktop alloue la moitié de votre RAM. Avec 16 Go, il alloue 8 Go — c'est suffisant. Pour vérifier ou ajuster : Docker Desktop → Settings → Resources → Memory.

Vérifier l'installation dans le Terminal :

```bash
docker --version          # Docker version 24+
docker compose version    # Docker Compose version v2+
```

**Homebrew et Python** (si pas déjà installés)

```bash
# Installer Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Installer Python
brew install python@3.11

# Vérification
python3 --version         # Python 3.11+
```

---

### Étape 2 — Cloner le repo

```bash
git clone https://github.com/nicktruch/esup-myia-mistral-amue.git ~/myia
cd ~/myia

# Créer les répertoires de données
mkdir -p data/{postgres,redis,openwebui}
```

Le repo contient déjà tous les fichiers nécessaires :
- `docker-compose.macos.yml` — stack optimisée Apple Silicon
- `config/litellm_config.yaml` — modèles et paramètres LiteLLM (préconfiguré pour ILaaS)
- `.env.example` — template de configuration à copier

---

### Étape 3 — Fichier d'environnement

Copier le template et remplir les secrets :

```bash
cp .env.example .env
chmod 600 .env
```

Ouvrir le `.env` dans un éditeur :

```bash
open -e ~/myia/.env
```

Remplacer **toutes** les valeurs de substitution :

| Variable | Rôle | Valeur |
|---|---|---|
| `ILAAS_API_KEY` | Clé API ILaaS fournie par l'AMUE | Votre vraie clé `sk-xxx...` |
| `LITELLM_MASTER_KEY` | Clé admin LiteLLM | `sk-` + 32 caractères aléatoires |
| `LITELLM_SECRET_KEY` | Secret JWT interne | Chaîne aléatoire longue |
| `POSTGRES_PASSWORD` | Mot de passe base de données | Chaîne aléatoire |
| `REDIS_PASSWORD` | Mot de passe cache Redis | Chaîne aléatoire |
| `WEBUI_SECRET_KEY` | Secret session OpenWebUI | Chaîne aléatoire |

Pour générer des valeurs robustes directement dans le Terminal :

```bash
for i in 1 2 3 4 5 6; do python3 -c "import secrets; print(secrets.token_urlsafe(32))"; done
```

Sauvegarder le fichier avant de passer à l'étape suivante.

> ⚠️ Remplir le `.env` **avant** `docker compose up -d`. Une fois PostgreSQL démarré une première fois, les mots de passe sont gravés en base et ne peuvent plus être changés sans réinitialiser les données.

> **Option Mistral standard** : si vous utilisez une clé Mistral classique (hors accord AMUE/ILaaS), décommentez la ligne `# MISTRAL_API_KEY` dans le `.env` et adaptez `config/litellm_config.yaml` (voir les blocs commentés dans ce fichier).

---

### Étape 4 — Vérifier la configuration LiteLLM

Le fichier `config/litellm_config.yaml` est préconfiguré pour l'endpoint ILaaS :

- **Endpoint actif** : `https://llm.ilaas.fr/v1`
- **Variable de clé** : `ILAAS_API_KEY`
- **Modèle disponible via ILaaS AMUE** : `mistral-medium` (Mistral Medium 3 — `mistral-medium-250523`)
- **Température** : 0.1 (recommandé par ILaaS)
- **RAG / embeddings** : non disponibles via ILaaS — voir étape 6b pour Ollama

Aucune modification n'est nécessaire si vous avez une clé ILaaS. Pour basculer sur une clé Mistral standard (accès à `mistral-large`, `codestral`, `mistral-embed`...), commentez le bloc ILaaS et décommentez les blocs `MISTRAL_API_KEY / https://api.mistral.ai/v1` dans ce fichier.

---

### Étape 5 — Premier démarrage

```bash
cd ~/myia

# Lancer tous les services (version macOS Apple Silicon)
docker compose -f docker-compose.macos.yml up -d

# Le premier démarrage télécharge les images (~1-2 Go) — patienter
# Surveiller le démarrage
docker compose -f docker-compose.macos.yml ps

# Logs si besoin
docker compose -f docker-compose.macos.yml logs -f litellm
docker compose -f docker-compose.macos.yml logs -f openwebui
```

Résultat attendu après ~60-90 secondes :

```
NAME             STATUS                    PORTS
myia-postgres    Up X minutes (healthy)    5432/tcp
myia-redis       Up X minutes (healthy)    6379/tcp
myia-litellm     Up X minutes (healthy)    0.0.0.0:4000->4000/tcp
myia-openwebui   Up X minutes (healthy)    0.0.0.0:3000->8080/tcp
```

> **Note timeouts** : les `start_period` sont volontairement plus longs que dans la version Linux. Docker Desktop sur Mac ajoute une couche de virtualisation qui ralentit les démarrages — les migrations Prisma de LiteLLM prennent jusqu'à 3 minutes sur M2.

---

### Étape 6 — Configuration initiale OpenWebUI

1. Ouvrir http://localhost:3000 dans Safari ou Chrome
2. Créer le **compte administrateur** (premier compte = admin automatiquement)
3. Aller dans **Paramètres → Connexions** et vérifier que LiteLLM est détecté
4. Aller dans **Admin → Utilisateurs** pour créer d'autres comptes si besoin

---

### Étape 6b — Tester avec Ollama en attendant la clé ILaaS

Si tu n'as pas encore ta clé ILaaS, tu peux utiliser un modèle local via Ollama. Ollama est aussi nécessaire pour les **embeddings RAG** (non disponibles via ILaaS).

**Installer Ollama**
```bash
brew install ollama
```

**Démarrer Ollama** — laisser ce terminal ouvert :
```bash
ollama serve
```

**Télécharger un modèle léger** dans un second terminal :
```bash
ollama pull qwen3:0.6b
```

**Activer le modèle dans LiteLLM** — décommenter la section Ollama dans `config/litellm_config.yaml` :
```yaml
  - model_name: qwen3
    litellm_params:
      model: ollama/qwen3:0.6b
      api_base: http://host.docker.internal:11434
    model_info:
      description: "Qwen3 0.6b — modèle local via Ollama"
```

**Redémarrer LiteLLM** :
```bash
docker compose -f docker-compose.macos.yml restart litellm
```

Le modèle `qwen3` apparaît dans le menu déroulant d'OpenWebUI.

**Pour activer le RAG avec Ollama** (embeddings) :

```bash
ollama pull nomic-embed-text
```

Puis dans OpenWebUI : **Admin → Settings → Documents**
- Embedding Model Engine → `Ollama`
- Embedding Model → `nomic-embed-text`
- Ollama URL → `http://host.docker.internal:11434` (ne pas mettre `localhost` ou `127.0.0.1`)
- Sauvegarder

> **Note mémoire M2** : `qwen3:0.6b` (500 Mo) et `nomic-embed-text` (292 Mo) coexistent sans problème sur M2 16 Go. Éviter `qwen3:1.7b` — trop gourmand en parallèle.

> **Note** : `host.docker.internal` est l'adresse spéciale Docker Desktop sur Mac pour joindre un service qui tourne sur l'hôte (ton Mac) depuis un conteneur. Ollama doit être lancé avec `ollama serve` avant de démarrer la stack.

---

## 4. Vérifier que tout fonctionne

---

### 4.1 — Les conteneurs sont-ils tous en vie ?

```bash
docker compose -f docker-compose.macos.yml ps --format "table {{.Name}}\t{{.Status}}\t{{.Ports}}"
```

Tous les services doivent afficher `healthy`. Si un service est en `starting` ou `unhealthy` :

```bash
docker compose -f docker-compose.macos.yml logs --tail=50 litellm
docker compose -f docker-compose.macos.yml logs --tail=50 openwebui
```

---

### 4.2 — LiteLLM reçoit-il bien la clé ILaaS ?

```bash
curl -s \
  -H "Authorization: Bearer sk-master-CHANGEZ-MOI" \
  http://localhost:4000/health | python3 -m json.tool
```

---

### 4.3 — Le modèle est-il bien exposé ?

```bash
curl -s \
  -H "Authorization: Bearer sk-master-CHANGEZ-MOI" \
  http://localhost:4000/models | python3 -m json.tool
```

On doit voir `mistral-medium` (config ILaaS par défaut).

---

### 4.4 — Le chat fonctionne-t-il de bout en bout ?

```python
# recette.py
import openai, sys

BASE_URL = "http://localhost:4000/v1"
API_KEY  = "sk-master-CHANGEZ-MOI"

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

print("=== Recette MyIA AMUE — macOS M2 ===\n")
check("Chat simple (mistral-medium)",   test_chat)
check("Streaming (mistral-medium)",     test_streaming)

print(f"\n{'✅ Recette réussie.' if ok else '❌ Des tests ont échoué.'}")
sys.exit(0 if ok else 1)
```

```bash
pip3 install openai
python3 recette.py
```

---

### 4.5 — L'interface OpenWebUI est-elle fonctionnelle ?

Ouvrir http://localhost:3000 et dérouler les vérifications suivantes :

**Connexion et accès**
- [ ] La page de connexion s'affiche sans erreur
- [ ] La création du compte administrateur fonctionne
- [ ] Le modèle `mistral-medium` apparaît dans le menu déroulant

**Conversation**
- [ ] Envoyer *"Bonjour, qui es-tu ?"* → réponse en streaming, en français
- [ ] Changer de modèle si Ollama est activé et constater que la réponse change de style

**RAG — test** (nécessite Ollama + nomic-embed-text configuré — voir étape 6b)
- [ ] Créer un petit PDF de test si nécessaire (voir commande ci-dessous)
- [ ] Uploader le PDF via l'icône trombone dans le chat
- [ ] Poser une question sur le contenu de ce document
- [ ] Vérifier que la réponse cite le document sans halluciner

Pour créer un PDF de test minimal :
```bash
cat > /tmp/test.txt << 'TXTEOF'
L'université de Strasbourg a été fondée en 1621.
Elle compte environ 52 000 étudiants.
TXTEOF
cupsfilter /tmp/test.txt > /tmp/test.pdf 2>/dev/null
```

> **Note** : commencer par un petit PDF (1-2 pages) pour valider le pipeline. Les PDFs scannés (images) ne sont pas supportés sans OCR — utiliser des PDFs natifs avec texte sélectionnable.

---

### 4.6 — Que faire si quelque chose ne va pas ?

| Symptôme | Cause probable | Action |
|---|---|---|
| Conteneur `litellm` en `unhealthy` | PostgreSQL pas encore prêt | Attendre 30s, relancer `docker compose up -d` |
| Erreur 401 sur `/health` | Mauvaise `LITELLM_MASTER_KEY` | Vérifier le `.env` et redémarrer |
| Erreur 429 ou `quota exceeded` | Clé ILaaS épuisée | Contacter votre référent AMUE |
| Modèles absents dans OpenWebUI | OpenWebUI ne joint pas LiteLLM | Vérifier les logs OpenWebUI |
| RAG sans résultat | Modèle d'embedding non configuré | Configurer Ollama + nomic-embed-text dans Admin → Settings → Documents |
| Images lentes au démarrage | Émulation ARM/x86 | Vérifier le tag `platform: linux/arm64` dans `docker-compose.macos.yml` |
| Impossible de créer le compte admin | `ENABLE_SIGNUP: "false"` | Passer à `"true"` le temps de créer le compte, puis remettre à `"false"` |
| Modèle Ollama inaccessible | Ollama non démarré ou mauvais endpoint | Vérifier que `ollama serve` tourne, et que `api_base` est bien `http://host.docker.internal:11434` |
| Ollama embed error EOF | URL incorrecte dans OpenWebUI | Admin → Settings → Documents → URL Ollama → `http://host.docker.internal:11434` (pas `127.0.0.1`) |
| Ollama embed error 500 | Mémoire insuffisante ou PDF trop gros | Stopper les autres modèles Ollama (`ollama stop`), tester avec un petit PDF d'abord |
| Docker Desktop ne démarre pas | Ressources insuffisantes | Vérifier dans Settings → Resources que la mémoire est ≥ 6 Go |

---

### 4.7 — Arrêter et reprendre la maquette

```bash
# Arrêter sans perdre les données
docker compose -f docker-compose.macos.yml stop

# Reprendre
docker compose -f docker-compose.macos.yml start

# Tout supprimer (données comprises) — pour repartir de zéro
docker compose -f docker-compose.macos.yml down -v
```

---

## 5. Et ensuite ?

Votre maquette tourne sur http://localhost:3000. Vous pouvez expérimenter, tester le RAG avec Ollama, former vos collègues localement.

Pour passer à une instance partagée dans votre établissement, consultez le tutoriel de déploiement en production (version Linux/serveur) qui couvre :

- Exposition publique sécurisée avec Nginx et certificat TLS
- Authentification via le SSO de votre établissement (Shibboleth / Keycloak)
- Gestion des quotas et budgets par équipe ou département
- Monitoring, alerting et sauvegardes automatisées

---

*Adaptation macOS M2 du tutoriel rédigé dans le cadre des Ateliers IA Esup 2025/2026 — basé sur les briques open source LiteLLM, OpenWebUI et l'offre Mistral AMUE via ILaaS (CINES).*
