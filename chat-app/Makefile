# Open WebUI on top of Command Code's OpenAI-compatible API.
# Every setting lives in .env (see .env.example). Run `make help` for targets.

# .env holds generated hex secrets; it is also read by `docker compose`.
-include .env

DATA_DIR ?= ./data
WEBUI_URL ?= http://127.0.0.1:$(or $(OPEN_WEBUI_HOST_PORT),3080)

# The superuser of the shared Postgres belongs to ../infra/.env, not this file.
PG_USER := $(or $(shell sed -n 's/^POSTGRES_USER=//p' ../infra/.env 2>/dev/null),postgres)
PG      := docker exec shared-postgres psql -v ON_ERROR_STOP=1 -U $(PG_USER)

COMPOSE    := docker compose
# Reaches the container from the host while nginx is not in front of it.
LOCAL_URL  := http://127.0.0.1:$(or $(OPEN_WEBUI_HOST_PORT),3080)

.DEFAULT_GOAL := help
.PHONY: help init db up down restart ps logs pull config urls health shell models chat psql

help:
	@echo "  make init      create .env with random secrets + data dir (refuses to overwrite)"
	@echo "  make db        create the pgvector role/database in shared-postgres (idempotent)"
	@echo "  make up        start Open WebUI"
	@echo "  make down      stop it (data/ is kept)"
	@echo "  make restart   restart it"
	@echo "  make ps        container status"
	@echo "  make logs      follow logs"
	@echo "  make pull      pull the pinned image"
	@echo "  make config    validate compose + .env interpolation"
	@echo "  make urls      print the web UI and API URLs"
	@echo "  make health    container health + /health from the host"
	@echo "  make models    list models the provider serves (needs OPENAI_API_KEY)"
	@echo "  make chat      one non-streaming completion against the provider"
	@echo "  make psql      psql shell on the vector database"
	@echo "  make shell     shell inside the container"

# Creates .env from .env.example and replaces every `change-me` with a secret.
# Refuses to overwrite: .env holds WEBUI_SECRET_KEY, and rotating it invalidates
# every login.
init:
	@test ! -f .env || { echo ".env already exists, not overwriting"; exit 1; }
	@cp .env.example .env
	@set -e; \
	sed -i \
	  -e "s|^WEBUI_SECRET_KEY=.*|WEBUI_SECRET_KEY=$$(openssl rand -hex 32)|" \
	  -e "s|^WEBUI_ADMIN_PASSWORD=.*|WEBUI_ADMIN_PASSWORD=$$(openssl rand -hex 16)|" \
	  .env
	@chmod 600 .env
	@mkdir -p $(DATA_DIR)
	@echo ".env created. Set WEBUI_URL and OPENAI_API_KEY, then 'make up'."
	@echo "Admin login: $$(sed -n 's/^WEBUI_ADMIN_EMAIL=//p' .env) / $$(sed -n 's/^WEBUI_ADMIN_PASSWORD=//p' .env)"

# Creates the pgvector role + database in shared-postgres and enables the
# extension. Idempotent: a rerun re-syncs the password from .env and re-asserts
# the extension. A missing DB_PASSWORD is generated once and written to .env.
db:
	@test -f .env || { echo ".env missing, run 'make init' first"; exit 1; }
	@printf '%s' "$(DB_USER)" | grep -Eq '^[a-z_][a-z0-9_]*$$' \
	  || { echo "DB_USER must match [a-z_][a-z0-9_]*"; exit 1; }
	@printf '%s' "$(DB_NAME)" | grep -Eq '^[a-z_][a-z0-9_]*$$' \
	  || { echo "DB_NAME must match [a-z_][a-z0-9_]*"; exit 1; }
	@docker inspect shared-postgres >/dev/null 2>&1 \
	  || { echo "shared-postgres is not running: cd ../infra && make up"; exit 1; }
	@if grep -q '^DB_PASSWORD=change-me$$' .env; then \
	  sed -i "s|^DB_PASSWORD=.*|DB_PASSWORD=$$(openssl rand -hex 16)|" .env; \
	  echo "generated DB_PASSWORD in .env (make init does not know the database yet)"; \
	fi
	@set -e; \
	pw=$$(sed -n 's/^DB_PASSWORD=//p' .env); \
	role=$$($(PG) -tAc "SELECT 1 FROM pg_roles WHERE rolname='$(DB_USER)'"); \
	if [ -z "$$role" ]; then \
	  $(PG) -d postgres -c "CREATE ROLE \"$(DB_USER)\" LOGIN PASSWORD '$$pw'"; \
	else \
	  $(PG) -d postgres -c "ALTER ROLE \"$(DB_USER)\" WITH LOGIN PASSWORD '$$pw'"; \
	fi; \
	db=$$($(PG) -tAc "SELECT 1 FROM pg_database WHERE datname='$(DB_NAME)'"); \
	if [ -z "$$db" ]; then \
	  $(PG) -d postgres -c "CREATE DATABASE \"$(DB_NAME)\" OWNER \"$(DB_USER)\""; \
	fi; \
	$(PG) -d $(DB_NAME) -c "CREATE EXTENSION IF NOT EXISTS vector"; \
	echo "vector store ready: postgresql://$(DB_USER):***@$(or $(SHARED_POSTGRES_HOST),shared-postgres):5432/$(DB_NAME)"
	@# The column dimension is fixed at the first index. OWUI pads shorter
	@# embeddings and silently truncates longer ones, so a mismatch degrades
	@# retrieval without an error; surface it here instead.
	@dim=$$($(PG) -d $(DB_NAME) -tAc "SELECT atttypmod FROM pg_attribute WHERE attrelid = 'document_chunk'::regclass AND attname = 'vector'" 2>/dev/null || true); \
	if [ -n "$$dim" ] && [ "$$dim" != "-1" ] && [ "$$dim" != "$(PGVECTOR_DIM)" ]; then \
	  echo "WARNING: document_chunk.vector is vector($$dim) but PGVECTOR_DIM=$(PGVECTOR_DIM)."; \
	  echo "         Embeddings are padded or truncated to the column size without an error:"; \
	  echo "         set PGVECTOR_DIM=$$dim again, or DROP TABLE document_chunk; and re-index."; \
	fi

psql:
	@test -f .env || { echo ".env missing, run 'make init' first"; exit 1; }
	docker exec -it shared-postgres psql -U $(PG_USER) -d $(DB_NAME)

up: db
	@test -f .env || { echo ".env missing, run 'make init' first"; exit 1; }
	$(COMPOSE) up -d

down:
	$(COMPOSE) down

restart:
	$(COMPOSE) restart

ps:
	$(COMPOSE) ps

logs:
	$(COMPOSE) logs -f --tail=100 open-webui

pull:
	$(COMPOSE) pull open-webui

config:
	$(COMPOSE) config --quiet && echo "compose config OK"

urls:
	@echo "local:  $(LOCAL_URL)"
	@echo "public: $(WEBUI_URL)   (via host nginx; see README)"
	@echo "API:    $(WEBUI_URL)/api/chat/completions"

health:
	@$(COMPOSE) ps --format '{{.Name}} {{.State}} {{.Status}}'
	@curl -fsS -m 5 $(LOCAL_URL)/health && echo

# Discovery is public on Command Code (any key returns 200), so this only proves
# the URL, not the key.
models:
	@curl -sS -m 20 $(OPENAI_API_BASE_URL)/models \
	  | python3 -c 'import json,sys; d=json.load(sys.stdin)["data"]; print(len(d), "models"); [print(" ", m["id"]) for m in d[:10]]'

chat:
	@curl -sS -m 120 $(OPENAI_API_BASE_URL)/chat/completions \
	  -H "Authorization: Bearer $(OPENAI_API_KEY)" \
	  -H "Content-Type: application/json" \
	  -d '{"model":"deepseek/deepseek-v4.1-flash","messages":[{"role":"user","content":"say ok"}],"max_tokens":64}' \
	  | python3 -c 'import json,sys; d=json.load(sys.stdin); m=d["choices"][0]["message"]; print(m.get("reasoning_content") or "", m.get("content") or ""); print("usage:", d["usage"])'

shell:
	$(COMPOSE) exec open-webui bash
