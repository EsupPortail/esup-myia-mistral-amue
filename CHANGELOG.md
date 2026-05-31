# Changelog

## [Unreleased]

## [0.2.0] - 2026-05-30
### Ajouté
- Tutoriel macOS Apple Silicon (M2) — `README-macos.md`
- `docker-compose.macos.yml` avec `platform: linux/arm64` et timeouts adaptés
- Support modèle local Ollama (qwen3:0.6b) en commentaire dans `litellm_config.yaml`
- Section RAG avec `nomic-embed-text` via Ollama pour tests sans clé AMUE

### Corrigé
- Suppression `master_key` et `database_url` du `litellm_config.yaml` (doublons avec variables d'environnement)
- Correction `password: os.environ/REDIS_PASSWORD` (était un placeholder littéral)
- Suppression `version: '3.8'` du `docker-compose.yml` (attribut obsolète)
- Suppression du healthcheck LiteLLM (bloquait le démarrage sans clé valide)
- `depends_on` OpenWebUI : `service_healthy` → `service_started`
- `start_period` PostgreSQL : ajout `60s`
- Note `ENABLE_SIGNUP: "true"` obligatoire pour créer le premier compte admin

### Modifié
- Réseau Docker renommé `internal` → `nw_llm`
- Ports et volumes externalisés dans le `.env` avec valeur par défaut (`:-`)

## [0.1.0] - 2026-04-16
### Ajouté
- Version initiale générée sur Claude.ai — Nicolas Truchaud (Université de la Polynésie Française)
- Tutoriel Linux Ubuntu/Debian — `README.md`
- `docker-compose.yml` avec PostgreSQL, Redis, LiteLLM, OpenWebUI
- `config/litellm_config.yaml` avec modèles Mistral AMUE
- `.env.example`

### Modifié - 2026-05-20
- Ajout Debian 11/12 dans le tableau OS — David Boit
- Lien documentation officielle Docker
- Ports et volumes dans le `.env`
