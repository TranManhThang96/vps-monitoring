SERVICES := prometheus grafana mysqld-exporter node-exporter nginx-exporter postgres-exporter redis-exporter

# Only SHARED_NETWORK is read from .env. Not `-include .env`: make would parse the
# passwords, and a `#` or `$` in MYSQL_PASSWORD would become a comment or a variable.
SHARED_NETWORK ?= $(or $(shell sed -n 's/^SHARED_NETWORK=//p' .env 2>/dev/null),shared-net)
INFRA_ENV := ../infra/.env

# Optional: limit a target to one service, e.g. `make up SVC=grafana`.
SVC ?=

COMPOSE := docker compose
PROM := $(COMPOSE) exec -T prometheus

.DEFAULT_GOAL := help
.PHONY: help init secrets check network up down restart ps logs pull config reload targets \
        $(addprefix up-,$(SERVICES)) $(addprefix down-,$(SERVICES)) \
        $(addprefix restart-,$(SERVICES)) $(addprefix logs-,$(SERVICES))

help:
	@echo "Services: $(SERVICES)"
	@echo ""
	@echo "  make init                 create .env (Redis password copied from ../infra/.env) and secrets/"
	@echo "  make secrets              write secrets/garage_metrics_token from ../infra/.env"
	@echo "  make up                   start all services   (make up SVC=grafana for one)"
	@echo "  make down                 stop + remove all containers (volumes and network are kept)"
	@echo "  make restart              restart all          (SVC=... for one)"
	@echo "  make up-<svc> | down-<svc> | restart-<svc> | logs-<svc>"
	@echo "  make ps | logs | pull | config"
	@echo "  make network              create external network '$(SHARED_NETWORK)' if missing"
	@echo "  make reload               reload prometheus.yml without restarting"
	@echo "  make targets              scrape target health (job, health, last error)"

# Creates .env from .env.example. MYSQL_PASSWORD and POSTGRES_EXPORTER_PASSWORD
# cannot be generated here, they belong to accounts created elsewhere.
init:
	@test ! -f .env || { echo ".env already exists, not overwriting"; exit 1; }
	@test -f $(INFRA_ENV) || { echo "$(INFRA_ENV) not found: run 'make init' in ../infra first"; exit 1; }
	@redis=$$(grep -E '^REDIS_PASSWORD=' $(INFRA_ENV) | cut -d= -f2-); \
	[ -n "$$redis" ] || { echo "REDIS_PASSWORD missing in $(INFRA_ENV)"; exit 1; }; \
	REDIS_PW="$$redis" awk '/^REDIS_PASSWORD=/ { print "REDIS_PASSWORD=" ENVIRON["REDIS_PW"]; next } { print }' \
	  .env.example > .env
	@chmod 600 .env
	@$(MAKE) --no-print-directory secrets
	@echo ".env created. Still to fill in by hand:"
	@grep -n '=CHANGE_ME$$' .env | sed 's/^/  line /'
	@echo "  MYSQL_PASSWORD: password of the MySQL 'exporter' user (see README.md)"
	@echo "  POSTGRES_EXPORTER_PASSWORD: printed by 'make pg-monitor-user' in ../infra"

# Prometheus runs as user nobody in the container, so the token must be world-readable.
secrets:
	@test -f $(INFRA_ENV) || { echo "$(INFRA_ENV) not found: run 'make init' in ../infra first"; exit 1; }
	@tok=$$(grep -E '^GARAGE_METRICS_TOKEN=' $(INFRA_ENV) | cut -d= -f2-); \
	{ [ -n "$$tok" ] && [ "$$tok" != CHANGE_ME ]; } || { echo "GARAGE_METRICS_TOKEN missing in $(INFRA_ENV)"; exit 1; }; \
	mkdir -p secrets; \
	printf '%s' "$$tok" > secrets/garage_metrics_token; \
	chmod 644 secrets/garage_metrics_token
	@echo "secrets/garage_metrics_token written"

# Compose does not notice a missing token file or a placeholder password: the
# bind mount fails or the exporter just logs auth errors. Catch both before `up`.
check:
	@test -f .env || { echo ".env missing: run 'make init'"; exit 1; }
	@! grep -q '=CHANGE_ME$$' .env || { echo "placeholders left in .env:"; grep -n '=CHANGE_ME$$' .env; exit 1; }
	@test -s secrets/garage_metrics_token || { echo "secrets/garage_metrics_token missing: run 'make secrets'"; exit 1; }

network:
	@docker network inspect $(SHARED_NETWORK) >/dev/null 2>&1 \
	  || { echo "network '$(SHARED_NETWORK)' not found: run 'make up' in ../infra first (it creates it)"; exit 1; }

up: network check
	$(COMPOSE) up -d $(SVC)

# `down` with SVC removes only that service's container; without, all of them.
# Volumes (prometheus_data, grafana-data) and the external network are never removed.
down:
ifeq ($(SVC),)
	$(COMPOSE) down
else
	$(COMPOSE) rm -sf $(SVC)
endif

restart: network check
	$(COMPOSE) restart $(SVC)

ps:
	$(COMPOSE) ps

logs:
	$(COMPOSE) logs -f --tail=100 $(SVC)

pull:
	$(COMPOSE) pull $(SVC)

config:
	$(COMPOSE) config

reload:
	@$(PROM) wget -qO- --post-data='' http://localhost:9090/-/reload
	@echo "prometheus config reloaded"

# Parsed with sed: no jq dependency. Splits the API response into one line per target.
targets:
	@$(PROM) wget -qO- 'http://localhost:9090/api/v1/targets?state=active' \
	  | sed 's/{"discoveredLabels"/\n&/g' \
	  | sed -n -E 's/.*"labels":\{[^}]*"job":"([^"]*)".*"lastError":"((\\.|[^"\\])*)".*"health":"([^"]*)".*/\1\t\4\t\2/p' \
	  | sed 's/\\"/"/g' \
	  | sort

$(addprefix up-,$(SERVICES)): up-%:
	@$(MAKE) --no-print-directory up SVC=$*
$(addprefix down-,$(SERVICES)): down-%:
	@$(MAKE) --no-print-directory down SVC=$*
$(addprefix restart-,$(SERVICES)): restart-%:
	@$(MAKE) --no-print-directory restart SVC=$*
$(addprefix logs-,$(SERVICES)): logs-%:
	@$(MAKE) --no-print-directory logs SVC=$*
