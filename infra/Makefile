SERVICES := postgres redis garage garage-webui

-include .env
SHARED_NETWORK ?= shared-net
GARAGE_BUCKET  ?= default-bucket

# Optional: limit a target to one service, e.g. `make up SVC=redis`.
SVC ?=

COMPOSE := docker compose
# psql inside the postgres container as the superuser from .env. Extra args are passed through.
PSQL := $(COMPOSE) exec -T postgres sh -c 'psql -v ON_ERROR_STOP=1 -U "$$POSTGRES_USER" -d "$$POSTGRES_DB" "$$@"' sh

.DEFAULT_GOAL := help
.PHONY: help init network up down restart ps logs pull config \
        psql db-create pg-monitor-user redis-cli garage-status garage-bucket garage-webui-auth $(addprefix up-,$(SERVICES)) \
        $(addprefix down-,$(SERVICES)) $(addprefix restart-,$(SERVICES)) \
        $(addprefix logs-,$(SERVICES))

help:
	@echo "Services: $(SERVICES)"
	@echo ""
	@echo "  make init                 create .env with random secrets (never overwrites)"
	@echo "  make up                   start all services   (make up SVC=redis for one)"
	@echo "  make down                 stop + remove all containers (data is kept)"
	@echo "  make restart              restart all          (SVC=... for one)"
	@echo "  make up-<svc> | down-<svc> | restart-<svc> | logs-<svc>"
	@echo "  make ps | logs | pull | config"
	@echo "  make network              create external network '$(SHARED_NETWORK)' if missing"
	@echo "  make psql                 psql shell in postgres"
	@echo "  make db-create NAME=x     create postgres user + database x with a random password"
	@echo "  make pg-monitor-user      create read-only role 'exporter' for postgres-exporter"
	@echo "  make redis-cli            redis-cli shell"
	@echo "  make garage-status        garage cluster status"
	@echo "  make garage-bucket NAME=x create bucket x and grant the default key access"
	@echo "  make garage-webui-auth    set a random web UI login in .env (prints the password once)"

init:
	@test ! -f .env || { echo ".env already exists, not overwriting"; exit 1; }
	@cp .env.example .env
	@set -e; \
	gen() { openssl rand -hex "$$1"; }; \
	sed -i \
	  -e "s|^POSTGRES_PASSWORD=.*|POSTGRES_PASSWORD=$$(gen 16)|" \
	  -e "s|^REDIS_PASSWORD=.*|REDIS_PASSWORD=$$(gen 16)|" \
	  -e "s|^GARAGE_ACCESS_KEY=.*|GARAGE_ACCESS_KEY=GK$$(gen 16)|" \
	  -e "s|^GARAGE_SECRET_KEY=.*|GARAGE_SECRET_KEY=$$(gen 32)|" \
	  -e "s|^GARAGE_RPC_SECRET=.*|GARAGE_RPC_SECRET=$$(gen 32)|" \
	  -e "s|^GARAGE_ADMIN_TOKEN=.*|GARAGE_ADMIN_TOKEN=$$(gen 32)|" \
	  -e "s|^GARAGE_METRICS_TOKEN=.*|GARAGE_METRICS_TOKEN=$$(gen 32)|" \
	  .env
	@chmod 600 .env
	@echo ".env created with generated secrets. Run 'make garage-webui-auth' to set the web UI login."

network:
	@docker network inspect $(SHARED_NETWORK) >/dev/null 2>&1 \
	  || docker network create $(SHARED_NETWORK)

up: network
	$(COMPOSE) up -d $(SVC)

# `down` with SVC removes only that service's container; without, all of them.
# The external network is never removed, other apps may be using it.
down:
ifeq ($(SVC),)
	$(COMPOSE) down
else
	$(COMPOSE) rm -sf $(SVC)
endif

restart: network
	$(COMPOSE) restart $(SVC)

ps:
	$(COMPOSE) ps

logs:
	$(COMPOSE) logs -f --tail=100 $(SVC)

pull:
	$(COMPOSE) pull $(SVC)

config:
	$(COMPOSE) config

$(addprefix up-,$(SERVICES)): up-%:
	@$(MAKE) --no-print-directory up SVC=$*
$(addprefix down-,$(SERVICES)): down-%:
	@$(MAKE) --no-print-directory down SVC=$*
$(addprefix restart-,$(SERVICES)): restart-%:
	@$(MAKE) --no-print-directory restart SVC=$*
$(addprefix logs-,$(SERVICES)): logs-%:
	@$(MAKE) --no-print-directory logs SVC=$*

psql:
	$(COMPOSE) exec postgres sh -c 'psql -U "$$POSTGRES_USER" -d "$$POSTGRES_DB"'

redis-cli:
	$(COMPOSE) exec redis redis-cli

garage-status:
	$(COMPOSE) exec garage /garage status

garage-bucket:
	@test -n "$(NAME)" || { echo "usage: make garage-bucket NAME=<bucket>"; exit 1; }
	@printf '%s' "$(NAME)" | grep -Eq '^[a-z0-9][a-z0-9.-]{1,61}[a-z0-9]$$' \
	  || { echo "NAME must be 3-63 chars: lowercase letters, digits, '.' and '-', starting and ending with a letter or digit"; exit 1; }
	$(COMPOSE) exec garage /garage bucket create $(NAME)
	$(COMPOSE) exec garage /garage bucket allow --read --write --owner $(NAME) --key $(GARAGE_ACCESS_KEY)

# Writes GARAGE_WEBUI_AUTH (user admin, bcrypt) into .env and prints the password once.
# `$` is doubled because compose interpolates .env values.
garage-webui-auth:
	@command -v htpasswd >/dev/null || { echo "htpasswd not found (apt install apache2-utils)"; exit 1; }
	@test -f .env || { echo ".env missing, run 'make init' first"; exit 1; }
	@set -e; umask 077; \
	pw=$$(openssl rand -hex 12); \
	hash=$$(htpasswd -nbBC 10 admin "$$pw" | tr -d '\n' | sed 's/\$$/$$$$/g'); \
	grep -v '^GARAGE_WEBUI_AUTH=' .env > .env.tmp || true; \
	printf 'GARAGE_WEBUI_AUTH=%s\n' "$$hash" >> .env.tmp; \
	mv .env.tmp .env; chmod 600 .env; \
	echo "Web UI login saved to .env. Save the password now, it is not stored anywhere:"; \
	echo ""; \
	echo "  user:     admin"; \
	echo "  password: $$pw"; \
	echo ""; \
	echo "Apply: make up-garage-webui"

# Creates a dedicated user + database named NAME, other users cannot connect to it.
# The password is printed once and stored nowhere.
db-create:
	@test "$(origin NAME)" = "command line" || { echo "usage: make db-create NAME=<name>"; exit 1; }
	@printf '%s' "$$NAME" | grep -Eq '^[a-z][a-z0-9_]{0,62}$$' \
	  || { echo "NAME must match [a-z][a-z0-9_]{0,62} (lowercase letters, digits, underscore)"; exit 1; }
	@set -e; \
	exists=$$($(PSQL) -tAc "SELECT 1 FROM pg_roles WHERE rolname='$$NAME' UNION ALL SELECT 1 FROM pg_database WHERE datname='$$NAME'"); \
	[ -z "$$exists" ] || { echo "user or database '$$NAME' already exists, nothing changed"; exit 1; }; \
	pw=$$(openssl rand -hex 16); \
	printf '%s\n' \
	  "CREATE USER $$NAME WITH PASSWORD '$$pw';" \
	  "CREATE DATABASE $$NAME OWNER $$NAME;" \
	  "REVOKE CONNECT ON DATABASE $$NAME FROM PUBLIC;" \
	  | $(PSQL) -q >/dev/null; \
	port=$(or $(POSTGRES_PORT),5432); \
	echo "Created database and user '$$NAME'. Save the password now, it is not stored anywhere:"; \
	echo ""; \
	echo "  user:     $$NAME"; \
	echo "  password: $$pw"; \
	echo "  database: $$NAME"; \
	echo "  host (native on VPS):       127.0.0.1:$$port"; \
	echo "  host (container, $(SHARED_NETWORK)): shared-postgres:5432"; \
	echo ""; \
	echo "  postgresql://$$NAME:$$pw@127.0.0.1:$$port/$$NAME"

# Read-only role for postgres-exporter. pg_monitor only grants access to statistics
# views; the role is not a superuser and owns nothing. The password is printed once.
pg-monitor-user:
	@set -e; \
	exists=$$($(PSQL) -tAc "SELECT 1 FROM pg_roles WHERE rolname='exporter'"); \
	[ -z "$$exists" ] || { echo "role 'exporter' already exists, nothing changed (reset: make psql, then ALTER USER exporter WITH PASSWORD '...';)"; exit 1; }; \
	pw=$$(openssl rand -hex 16); \
	printf '%s\n' \
	  "CREATE USER exporter WITH PASSWORD '$$pw';" \
	  "GRANT pg_monitor TO exporter;" \
	  | $(PSQL) -q >/dev/null; \
	echo "Created read-only role 'exporter'. Save the password now, it is not stored anywhere:"; \
	echo ""; \
	echo "  POSTGRES_EXPORTER_PASSWORD=$$pw"; \
	echo ""; \
	echo "Put it in monitoring/.env"
