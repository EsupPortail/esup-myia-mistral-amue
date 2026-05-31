# MyIA — Clone souverain de ChatGPT (Mistral AMUE)

Stack open source pour déployer une interface type ChatGPT souveraine dans l'enseignement supérieur français, basée sur l'offre Mistral AMUE.

> **Basé sur les Ateliers IA Esup 2025/2026** — Université de Rennes / Université de Strasbourg

---

## Stack

| Composant | Rôle |
|---|---|
| [Mistral AI](https://mistral.ai) | Modèles LLM souverains (via clé AMUE) |
| [LiteLLM](https://litellm.ai) | Passerelle API, quotas, load-balancing |
| [OpenWebUI](https://github.com/open-webui/open-webui) | Interface utilisateur type ChatGPT |
| PostgreSQL | Persistance logs, budgets, clés |
| Redis | Cache des requêtes |

---

## Démarrage rapide

```bash
# 1. Cloner le repo
git clone https://github.com/votre-org/esup-myia-mistral-amue.git
cd esup-myia-mistral-amue

# 2. Copier et remplir le fichier de config
cp .env.example .env
# Éditer .env avec vos secrets (voir section Configuration)

# 3. Créer les dossiers de données
mkdir -p data/{postgres,redis,openwebui}

# 4. Lancer
docker compose up -d

# 5. Ouvrir l'interface
open http://localhost:3000
```

> ⚠️ Pour macOS Apple Silicon (M2/M3), voir [README-macos.md](README-macos.md)

---

## Configuration

### Secrets obligatoires (`.env`)

Copier `.env.example` en `.env` et remplir toutes les valeurs `changez-moi`.

Pour générer des secrets robustes :
```bash
for i in 1 2 3 4 5 6; do python3 -c "import secrets; print(secrets.token_urlsafe(32))"; done
```

> ⚠️ Ces secrets doivent être définis **avant** le premier `docker compose up`. PostgreSQL grave le mot de passe en base à l'initialisation — le changer après nécessite un `docker compose down -v`.

### Clé Mistral AMUE

La clé est fournie par votre référent numérique via le dispositif AMUE. La renseigner dans `.env` :
```
MISTRAL_API_KEY=sk-votre-vraie-cle-ici
```

### Premier compte admin

Au premier lancement, mettre temporairement dans `.env` ou `docker-compose.yml` :
```yaml
ENABLE_SIGNUP: "true"
```
Créer le compte admin sur http://localhost:3000, puis remettre à `"false"` et relancer :
```bash
docker compose up -d openwebui
```

### Modèle local Ollama (sans clé AMUE)

Pour tester sans clé AMUE, décommenter la section Ollama dans `config/litellm_config.yaml` et installer Ollama :
```bash
# Linux
curl -fsSL https://ollama.com/install.sh | sh
ollama serve
ollama pull qwen3:0.6b
```

---

## Structure du projet

```
.
├── docker-compose.yml          # Linux / serveur
├── docker-compose.macos.yml    # macOS Apple Silicon
├── config/
│   └── litellm_config.yaml     # modèles et paramètres LiteLLM
├── .env.example                # template de configuration
├── .gitignore
├── CHANGELOG.md
├── README.md                   # ce fichier (Linux)
└── README-macos.md             # tutoriel macOS M2
```

---

## Prérequis

- Linux Ubuntu 22.04+ ou Debian 11+
- Docker Engine 24+ et Docker Compose v2
- Accès HTTPS sortant vers `api.mistral.ai`
- Clé API Mistral AMUE

---

## Documentation complète

- [Tutoriel Linux complet](README.md)
- [Tutoriel macOS M2](README-macos.md)
- [Wiki GT IA Esup](https://www.esup-portail.org/wiki/x/BgDLVg)
- [Canal RocketChat GT IA](https://rocket.esup-portail.org/channel/GT-IA)

---

## Historique

| Date | Auteur | Modification |
|---|---|---|
| 16 avr. 2026 | Nicolas Truchaud (Univ. Strasbourg) | Version initiale |
| 20 mai 2026 | David Boit | OS Debian, doc Docker officielle, ports/volumes dans .env, réseau nw_llm |
| 30 mai 2026 | Nicolas Truchaud | Corrections config LiteLLM, healthchecks, ENABLE_SIGNUP, support Ollama, tutoriel macOS M2 |

---

## Licence

Ce projet est publié sous licence MIT. Les modèles Mistral sont soumis aux conditions d'utilisation de Mistral AI et de l'accord AMUE.
