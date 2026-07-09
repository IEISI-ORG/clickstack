# ClickStack + Akvorado Combined Stack Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Merge the ClickStack (HyperDX) and Akvorado (flow analytics) Docker Compose deployments into one `docker compose up` stack that shares Akvorado's existing ClickHouse and adds an SNMP metrics collector, without losing the 38.4M flow rows already ingested.

**Architecture:** One authored root `docker-compose.yml` (project name `akvorado`) inlines all services with pinned images (no `extends:`). Akvorado's live ClickHouse (25.8) becomes the shared DB via a `ch-server` network alias; HyperDX and otel-collector point at it. A dedicated `snmp-collector` (otelcol-contrib) polls devices discovered from `default.exporters` and exports OTLP to `otel-collector`. Real config/credentials stay gitignored; sanitized `.example` files are committed.

**Tech Stack:** Docker Compose, ClickHouse 25.8, HyperDX 2.19, Akvorado 2.0.1, Kafka/Valkey/Traefik, OpenTelemetry Collector Contrib (snmpreceiver).

## Global Constraints

- Shared ClickHouse image: `clickhouse/clickhouse-server:25.8` (Akvorado supports only 25.x LTS). Never bump to 26.x in this stack.
- Compose project name: `akvorado` (set `COMPOSE_PROJECT_NAME=akvorado` in `.env`) — required so top-level volume `akvorado-clickhouse` maps to the existing `akvorado_akvorado-clickhouse`.
- HyperDX app host port: `1800` (never 8080 — reserved by Traefik).
- Public fork: no community strings, device IPs, IPinfo tokens, or site topology committed. Real files gitignored; only `*.example.*` committed.
- Preserve data: never `docker volume rm` or `docker compose down -v` on `akvorado_*` volumes.
- Akvorado images pinned verbatim from `/home/terry/akvorado/docker/versions.yml`: kafka `apache/kafka:4.1.0`, redis `valkey/valkey:7.2`, akvorado `ghcr.io/akvorado/akvorado:2.0.1`, traefik `traefik:v3.6.1`, kafka-ui `ghcr.io/kafbat/kafka-ui:v1.3.0`, geoip `ghcr.io/akvorado/ipinfo-geoipupdate:latest`.
- All work on branch `akvorado`. Commit after each task.
- **Restore flag:** any restore of this data MUST pass `--allow_suspicious_low_cardinality_types=1` (the `exporters` table uses `LowCardinality(IPv6)`, blocked on CREATE by default).

---

## Task 0: Backup current Akvorado state — COMPLETED 2026-07-09

Done before execution as a cutover safety net. Hot ClickHouse native `BACKUP`
(no downtime), verified restorable (restored `default.exporters` → `RESTORED`).

**Location:** `/home/terry/akvorado-backups/20260709-124313/`
- `clickhouse-default-*.zip` (958 M) — full `default` DB (flows, rollups, exporters, dictionaries)
- `console-db-*.tar.gz` — console SQLite volume
- `akvorado-config-*.tar.gz` — `/home/terry/akvorado` {config, docker, .env}
- `SHA256SUMS`, `RESTORE.md` (runbook, incl. the low-cardinality flag)

This is the rollback source if Task 6 cutover goes wrong. Do not delete it until
the combined stack is confirmed stable.

---

## File Structure

- `.gitignore` — add rules keeping real config/secrets out of git.
- `config/secrets/secrets.env` — real `SNMP_COMMUNITY`, `IPINFO_TOKEN` (gitignored).
- `config/secrets/secrets.env.example` — placeholders (committed).
- `config/akvorado/*.yaml` — real Akvorado app config, copied from host (gitignored).
- `config/akvorado/*.example.yaml` — sanitized templates (committed).
- `docker/clickhouse/akvorado/{observability.xml,server.xml}` — copied CH fragments (committed; no secrets).
- `docker/otel/snmp-collector.example.yaml` — SNMP config template (committed).
- `docker/otel/snmp-collector.yaml` — generated per-device config (gitignored).
- `docker/otel/gen-snmp-targets.sh` — target generator: exporters table ∪ extra file (committed).
- `docker/otel/snmp-extra-targets.example.txt` — manual extra-targets template (committed).
- `docker/otel/snmp-extra-targets.txt` — real manual targets, optional (gitignored).
- `.env` — merged environment, no secrets (committed).
- `docker-compose.yml` — combined stack (committed; overwrites the ClickStack-only one).

---

## Task 1: Secrets scaffolding and .gitignore

**Files:**
- Modify: `.gitignore`
- Create: `config/secrets/secrets.env.example`
- Create: `config/secrets/secrets.env` (gitignored, real values)

**Interfaces:**
- Produces: gitignored `config/secrets/secrets.env` providing container env vars `SNMP_COMMUNITY`, `IPINFO_TOKEN`; committed example alongside.

- [ ] **Step 1: Append ignore rules to `.gitignore`**

Append these lines to the end of `.gitignore`:

```gitignore
# ClickStack+Akvorado merge: keep real config/secrets out of the public fork
config/akvorado/*.yaml
!config/akvorado/*.example.yaml
docker/otel/snmp-collector.yaml
docker/otel/snmp-extra-targets.txt
config/secrets/
!config/secrets/*.example
```

- [ ] **Step 2: Create the committed secrets template**

Create `config/secrets/secrets.env.example`:

```dotenv
# SNMP community used by snmp-collector (reuse Akvorado's existing community).
SNMP_COMMUNITY=changeme
# IPinfo token for the geoip updater (https://ipinfo.io/).
IPINFO_TOKEN=changeme
```

- [ ] **Step 3: Create the real gitignored secrets file**

Create `config/secrets/secrets.env` (NOT committed):

```dotenv
SNMP_COMMUNITY=CHANGEME
IPINFO_TOKEN=CHANGEME
```

- [ ] **Step 4: Verify the real secrets file is ignored, the example is not**

Run:
```bash
git check-ignore config/secrets/secrets.env && echo IGNORED
git check-ignore config/secrets/secrets.env.example || echo TRACKED_OK
```
Expected: prints `IGNORED` then `TRACKED_OK`.

- [ ] **Step 5: Commit (example + .gitignore only)**

```bash
git add .gitignore config/secrets/secrets.env.example
git status   # confirm config/secrets/secrets.env is NOT staged
git commit -m "chore: gitignore rules and secrets template for akvorado merge"
```

---

## Task 2: Copy and sanitize Akvorado config and ClickHouse fragments

**Files:**
- Create (gitignored): `config/akvorado/{akvorado,inlet,outlet,console,demo}.yaml`
- Create (committed): `config/akvorado/{akvorado,inlet,outlet,console,demo}.example.yaml`
- Create (committed): `docker/clickhouse/akvorado/observability.xml`, `docker/clickhouse/akvorado/server.xml`

**Interfaces:**
- Produces: `./config/akvorado` mounted read-only into Akvorado services at `/etc/akvorado`; `./docker/clickhouse/akvorado/*.xml` mounted into the `clickhouse` service `config.d`.

- [ ] **Step 1: Copy real Akvorado app config (gitignored)**

```bash
mkdir -p config/akvorado docker/clickhouse/akvorado docker/otel
cp /home/terry/akvorado/config/akvorado.yaml config/akvorado/akvorado.yaml
cp /home/terry/akvorado/config/inlet.yaml     config/akvorado/inlet.yaml
cp /home/terry/akvorado/config/outlet.yaml    config/akvorado/outlet.yaml
cp /home/terry/akvorado/config/console.yaml   config/akvorado/console.yaml
cp /home/terry/akvorado/config/demo.yaml      config/akvorado/demo.yaml
```

- [ ] **Step 2: Copy ClickHouse config fragments (committed, no secrets)**

```bash
cp /home/terry/akvorado/docker/clickhouse/observability.xml docker/clickhouse/akvorado/observability.xml
cp /home/terry/akvorado/docker/clickhouse/server.xml        docker/clickhouse/akvorado/server.xml
```

- [ ] **Step 3: Create sanitized `.example` templates**

Create each `config/akvorado/<name>.example.yaml` as a copy of the real file with secrets/topology replaced by placeholders. Concretely:

- `config/akvorado/inlet.example.yaml` and `config/akvorado/console.example.yaml`: identical to the real files (they contain no secrets — verify with `grep -i community`).
- `config/akvorado/outlet.example.yaml`: copy of `outlet.yaml` with the SNMP community line changed:

```yaml
      credentials:
        ::/0:
          communities:
            - "CHANGEME"          # real community lives only in the gitignored outlet.yaml
```

- `config/akvorado/akvorado.example.yaml`: copy of `akvorado.yaml` with the `clickhouse.networks:` and `clickhouse.asns:` blocks reduced to the shipped example values (they already use RFC-5737 example ranges and `ACME Corporation`, so a verbatim copy is safe — confirm no private ranges before committing).
- `config/akvorado/demo.example.yaml`: verbatim copy of `demo.yaml` (demo exporters only, no secrets).

Generate them:
```bash
for f in inlet console demo; do cp config/akvorado/$f.yaml config/akvorado/$f.example.yaml; done
cp config/akvorado/akvorado.yaml config/akvorado/akvorado.example.yaml
sed 's/"CHANGEME"/"CHANGEME"/' config/akvorado/outlet.yaml > config/akvorado/outlet.example.yaml
```

- [ ] **Step 4: Verify no secret leaked into committed files**

Run:
```bash
grep -R "CHANGEME" config/akvorado/*.example.yaml docker/clickhouse/akvorado/ && echo "LEAK" || echo "CLEAN"
git check-ignore config/akvorado/outlet.yaml && echo REAL_IGNORED
```
Expected: `CLEAN` then `REAL_IGNORED`.

- [ ] **Step 5: Commit (examples + xml fragments only)**

```bash
git add config/akvorado/*.example.yaml docker/clickhouse/akvorado/observability.xml docker/clickhouse/akvorado/server.xml
git status   # confirm no real config/akvorado/*.yaml staged
git commit -m "chore: add sanitized akvorado config templates and CH fragments"
```

---

## Task 3: Merged `.env`

**Files:**
- Modify: `.env` (overwrite with merged content)

**Interfaces:**
- Produces: compose-interpolation variables consumed by `docker-compose.yml` in Task 4 — notably `HYPERDX_APP_PORT=1800`, `COMPOSE_PROJECT_NAME=akvorado`, image repo/version vars.

- [ ] **Step 1: Overwrite `.env` with merged, secret-free content**

Replace the entire contents of `.env` with:

```dotenv
# ---- Compose ----
COMPOSE_PROJECT_NAME=akvorado

# ---- HyperDX / ClickStack images ----
HDX_IMAGE_REPO=docker.hyperdx.io
CH_IMAGE_REPO=docker.clickhouse.com
IMAGE_NAME_DOCKERHUB=hyperdx/hyperdx
NEXT_OTEL_COLLECTOR_IMAGE_NAME_DOCKERHUB=clickhouse/clickstack-otel-collector
OTEL_COLLECTOR_IMAGE_NAME_DOCKERHUB=hyperdx/hyperdx-otel-collector
CODE_VERSION=2.19.0
IMAGE_VERSION=2

# ---- HyperDX ports/URLs (app moved 8080 -> 1800) ----
HYPERDX_API_PORT=8000
HYPERDX_APP_PORT=1800
HYPERDX_APP_URL=http://localhost
HYPERDX_LOG_LEVEL=debug
HYPERDX_OPAMP_PORT=4320
HYPERDX_BASE_PATH=

# ---- Otel/ClickHouse ----
HYPERDX_OTEL_EXPORTER_CLICKHOUSE_DATABASE=default
```

- [ ] **Step 2: Verify interpolation resolves and no secrets present**

Run:
```bash
grep -Ei "CHANGEME|IPINFO_TOKEN|SNMP_COMMUNITY" .env && echo "LEAK" || echo "CLEAN"
```
Expected: `CLEAN`.

- [ ] **Step 3: Commit**

```bash
git add .env
git commit -m "chore: merged .env for combined stack (project akvorado, app port 1800)"
```

---

## Task 4: Combined `docker-compose.yml`

**Files:**
- Modify: `docker-compose.yml` (overwrite the ClickStack-only file)

**Interfaces:**
- Consumes: `.env` (Task 3), `config/akvorado/*` (Task 2), `docker/clickhouse/akvorado/*.xml` (Task 2), `config/secrets/secrets.env` (Task 1), `docker/otel/snmp-collector.yaml` (Task 5).
- Produces: services `db`, `otel-collector`, `app`, `clickhouse` (alias `ch-server`), `kafka`, `kafka-ui`, `redis`, `akvorado-orchestrator`, `akvorado-console`, `akvorado-inlet`, `akvorado-outlet`, `akvorado-conntrack-fixer`, `traefik`, `geoip`, `snmp-collector`.

- [ ] **Step 1: Overwrite `docker-compose.yml`**

Replace the entire file with the following. For the two long HyperDX strings marked `<<COPY ...>>`, paste the value **verbatim** from the original ClickStack compose preserved in git at `git show 463e1ab:docker-compose.yml` (lines 67–74) — they are unchanged.

```yaml
name: akvorado

networks:
  default:
    enable_ipv6: true
    ipam:
      config:
        - subnet: 247.16.14.0/24
        - subnet: fd1c:8ce3:6fb:1::/64
    driver: bridge
    driver_opts:
      com.docker.network.bridge.name: br-akvorado

volumes:
  akvorado-kafka:
  akvorado-geoip:
  akvorado-clickhouse:
  akvorado-run:
  akvorado-console-db:

services:
  # ===================== ClickStack / HyperDX =====================
  db:
    image: mongo:5.0.32-focal
    restart: unless-stopped
    volumes:
      - .volumes/db:/data/db

  otel-collector:
    image: ${CH_IMAGE_REPO}/${NEXT_OTEL_COLLECTOR_IMAGE_NAME_DOCKERHUB}:${IMAGE_VERSION}
    environment:
      CLICKHOUSE_ENDPOINT: "tcp://ch-server:9000?dial_timeout=10s"
      HYPERDX_OTEL_EXPORTER_CLICKHOUSE_DATABASE: ${HYPERDX_OTEL_EXPORTER_CLICKHOUSE_DATABASE}
      HYPERDX_LOG_LEVEL: ${HYPERDX_LOG_LEVEL}
      OPAMP_SERVER_URL: "http://app:${HYPERDX_OPAMP_PORT}"
      HYPERDX_OTEL_EXPORTER_CREATE_LEGACY_SCHEMA: "true"
    ports:
      - "13133:13133"
      - "24225:24225"
      - "4317:4317"
      - "4318:4318"
      - "8888:8888"
    restart: always
    depends_on:
      - clickhouse

  app:
    image: ${HDX_IMAGE_REPO}/${IMAGE_NAME_DOCKERHUB}:${IMAGE_VERSION}
    ports:
      - ${HYPERDX_API_PORT}:${HYPERDX_API_PORT}
      - ${HYPERDX_APP_PORT}:${HYPERDX_APP_PORT}
    environment:
      FRONTEND_URL: ${HYPERDX_APP_URL}:${HYPERDX_APP_PORT}
      HYPERDX_API_KEY: ${HYPERDX_API_KEY}
      HYPERDX_API_PORT: ${HYPERDX_API_PORT}
      HYPERDX_APP_PORT: ${HYPERDX_APP_PORT}
      HYPERDX_APP_URL: ${HYPERDX_APP_URL}
      HYPERDX_LOG_LEVEL: ${HYPERDX_LOG_LEVEL}
      MINER_API_URL: "http://miner:5123"
      MONGO_URI: "mongodb://db:27017/hyperdx"
      SERVER_URL: http://127.0.0.1:${HYPERDX_API_PORT}
      OPAMP_PORT: ${HYPERDX_OPAMP_PORT}
      OTEL_EXPORTER_OTLP_ENDPOINT: "http://otel-collector:4318"
      OTEL_SERVICE_NAME: "hdx-oss-app"
      USAGE_STATS_ENABLED: ${USAGE_STATS_ENABLED:-true}
      DEFAULT_CONNECTIONS: <<COPY DEFAULT_CONNECTIONS value verbatim from git show 463e1ab:docker-compose.yml lines 67-68>>
      DEFAULT_SOURCES: <<COPY DEFAULT_SOURCES value verbatim from git show 463e1ab:docker-compose.yml lines 69-74>>
    restart: unless-stopped
    depends_on:
      - clickhouse
      - db

  # ===================== Shared ClickHouse (Akvorado's live 25.8) =====================
  clickhouse:
    image: clickhouse/clickhouse-server:25.8
    restart: unless-stopped
    networks:
      default:
        aliases:
          - ch-server
    volumes:
      - akvorado-clickhouse:/var/lib/clickhouse
      - ./docker/clickhouse/akvorado/observability.xml:/etc/clickhouse-server/config.d/observability.xml
      - ./docker/clickhouse/akvorado/server.xml:/etc/clickhouse-server/config.d/akvorado.xml
    environment:
      CLICKHOUSE_INIT_TIMEOUT: 60
      CLICKHOUSE_SKIP_USER_SETUP: 1
    cap_add:
      - SYS_NICE
    stop_grace_period: 30s
    healthcheck:
      interval: 20s
      test: ["CMD", "wget", "-T", "1", "--spider", "--no-proxy", "http://127.0.0.1:8123/ping"]
    labels:
      - traefik.enable=true
      - traefik.http.routers.clickhouse.entrypoints=private
      - traefik.http.routers.clickhouse.rule=PathPrefix(`/clickhouse`)
      - traefik.http.routers.clickhouse.middlewares=clickhouse-strip
      - traefik.http.middlewares.clickhouse-strip.stripprefix.prefixes=/clickhouse
      - metrics.port=8123

  # ===================== Akvorado =====================
  kafka:
    image: apache/kafka:4.1.0
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: controller,broker
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_LISTENERS: CLIENT://:9092,CONTROLLER://:9093
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CLIENT:PLAINTEXT,CONTROLLER:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: CLIENT://kafka:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_INTER_BROKER_LISTENER_NAME: CLIENT
      KAFKA_DELETE_TOPIC_ENABLE: "true"
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_SHARE_COORDINATOR_STATE_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_SHARE_COORDINATOR_STATE_TOPIC_MIN_ISR: 1
      KAFKA_LOG_DIRS: /var/lib/kafka/data
    restart: unless-stopped
    volumes:
      - akvorado-kafka:/var/lib/kafka/data
    healthcheck:
      interval: 20s
      test: ["CMD", "/opt/kafka/bin/kafka-topics.sh", "--list", "--bootstrap-server", "kafka:9092"]

  kafka-ui:
    image: ghcr.io/kafbat/kafka-ui:v1.3.0
    restart: unless-stopped
    depends_on:
      - kafka
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
      KAFKA_CLUSTERS_0_READONLY: true
      SERVER_SERVLET_CONTEXT_PATH: /kafka-ui
    labels:
      - traefik.enable=true
      - traefik.http.routers.kafka-ui.rule=PathPrefix(`/kafka-ui`)

  redis:
    image: valkey/valkey:7.2
    restart: unless-stopped
    healthcheck:
      interval: 20s
      test: ["CMD-SHELL", "timeout 3 redis-cli ping | grep -q PONG"]

  akvorado-orchestrator:
    image: ghcr.io/akvorado/akvorado:2.0.1
    restart: unless-stopped
    depends_on:
      kafka:
        condition: service_healthy
    command: orchestrator /etc/akvorado/akvorado.yaml
    volumes:
      - ./config/akvorado:/etc/akvorado:ro
      - akvorado-geoip:/usr/share/GeoIP:ro
    labels:
      - traefik.enable=true
      - traefik.http.routers.akvorado-orchestrator-metrics.rule=PathPrefix(`/api/v0/orchestrator/metrics`)
      - traefik.http.routers.akvorado-orchestrator-metrics.service=akvorado-orchestrator
      - traefik.http.routers.akvorado-orchestrator-metrics.observability.accesslogs=false
      - traefik.http.routers.akvorado-orchestrator.entrypoints=private
      - traefik.http.routers.akvorado-orchestrator.rule=PathPrefix(`/api/v0/orchestrator`)
      - traefik.http.services.akvorado-orchestrator.loadbalancer.server.port=8080
      - metrics.port=8080
      - metrics.path=/api/v0/metrics

  akvorado-console:
    image: ghcr.io/akvorado/akvorado:2.0.1
    restart: unless-stopped
    depends_on:
      akvorado-orchestrator:
        condition: service_healthy
      redis:
        condition: service_healthy
      clickhouse:
        condition: service_healthy
    command: console http://akvorado-orchestrator:8080
    volumes:
      - akvorado-console-db:/run/akvorado
    environment:
      AKVORADO_CFG_CONSOLE_DATABASE_DSN: /run/akvorado/console.sqlite
      AKVORADO_CFG_CONSOLE_BRANDING: ${AKVORADO_CFG_CONSOLE_BRANDING-false}
    healthcheck:
      disable: ${CONSOLE_HEALTHCHECK_DISABLED-false}
    labels:
      - traefik.enable=true
      - traefik.http.routers.akvorado-console-debug.rule=PathPrefix(`/debug`)
      - traefik.http.routers.akvorado-console-debug.entrypoints=private
      - traefik.http.routers.akvorado-console-debug.service=akvorado-console
      - traefik.http.routers.akvorado-console-metrics.rule=PathPrefix(`/api/v0/console/metrics`)
      - traefik.http.routers.akvorado-console-metrics.service=akvorado-console
      - traefik.http.routers.akvorado-console-metrics.observability.accesslogs=false
      - "traefik.http.routers.akvorado-console.rule=!PathPrefix(`/debug`)"
      - traefik.http.routers.akvorado-console.priority=1
      - traefik.http.routers.akvorado-console.middlewares=console-auth
      - traefik.http.services.akvorado-console.loadbalancer.server.port=8080
      - traefik.http.middlewares.console-auth.headers.customrequestheaders.Remote-User=alfred
      - traefik.http.middlewares.console-auth.headers.customrequestheaders.Remote-Name=Alfred Pennyworth
      - traefik.http.middlewares.console-auth.headers.customrequestheaders.Remote-Email=alfred@example.com
      - metrics.port=8080
      - metrics.path=/api/v0/metrics

  akvorado-inlet:
    image: ghcr.io/akvorado/akvorado:2.0.1
    ports:
      - 2055:2055/udp
      - 4739:4739/udp
      - 6343:6343/udp
    restart: unless-stopped
    depends_on:
      akvorado-orchestrator:
        condition: service_healthy
      kafka:
        condition: service_healthy
    command: inlet http://akvorado-orchestrator:8080
    volumes:
      - akvorado-run:/run/akvorado
    labels:
      - traefik.enable=true
      - traefik.http.routers.akvorado-inlet-metrics.rule=PathPrefix(`/api/v0/inlet/metrics`)
      - traefik.http.routers.akvorado-inlet-metrics.service=akvorado-inlet
      - traefik.http.routers.akvorado-inlet-metrics.observability.accesslogs=false
      - traefik.http.routers.akvorado-inlet.entrypoints=private
      - traefik.http.routers.akvorado-inlet.rule=PathPrefix(`/api/v0/inlet`)
      - traefik.http.services.akvorado-inlet.loadbalancer.server.port=8080
      - akvorado.conntrack.fix=true
      - metrics.port=8080
      - metrics.path=/api/v0/metrics

  akvorado-outlet:
    image: ghcr.io/akvorado/akvorado:2.0.1
    ports:
      - 10179:10179/tcp
    restart: unless-stopped
    depends_on:
      akvorado-orchestrator:
        condition: service_healthy
      kafka:
        condition: service_healthy
      clickhouse:
        condition: service_healthy
    command: outlet http://akvorado-orchestrator:8080
    volumes:
      - akvorado-run:/run/akvorado
    environment:
      AKVORADO_CFG_OUTLET_METADATA_CACHEPERSISTFILE: /run/akvorado/metadata.cache
    labels:
      - traefik.enable=true
      - traefik.http.routers.akvorado-outlet-metrics.rule=PathPrefix(`/api/v0/outlet/metrics`)
      - traefik.http.routers.akvorado-outlet-metrics.service=akvorado-outlet
      - traefik.http.routers.akvorado-outlet-metrics.observability.accesslogs=false
      - traefik.http.routers.akvorado-outlet.entrypoints=private
      - traefik.http.routers.akvorado-outlet.rule=PathPrefix(`/api/v0/outlet`)
      - traefik.http.services.akvorado-outlet.loadbalancer.server.port=8080
      - metrics.port=8080
      - metrics.path=/api/v0/metrics

  akvorado-conntrack-fixer:
    image: ghcr.io/akvorado/akvorado:2.0.1
    cap_add:
      - NET_ADMIN
    command: conntrack-fixer
    restart: unless-stopped
    network_mode: host
    healthcheck:
      disable: true
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro

  traefik:
    image: traefik:v3.6.1
    restart: unless-stopped
    environment:
      TRAEFIK_API: "true"
      TRAEFIK_API_BASEPATH: "/traefik"
      TRAEFIK_METRICS_PROMETHEUS: "true"
      TRAEFIK_METRICS_PROMETHEUS_MANUALROUTING: "true"
      TRAEFIK_METRICS_PROMETHEUS_ADDROUTERSLABELS: "true"
      TRAEFIK_PROVIDERS_DOCKER: "true"
      TRAEFIK_PROVIDERS_DOCKER_EXPOSEDBYDEFAULT: "false"
      TRAEFIK_ENTRYPOINTS_private_ADDRESS: ":8080"
      TRAEFIK_ENTRYPOINTS_private_HTTP_MIDDLEWARES: compress@docker
      TRAEFIK_ENTRYPOINTS_public_ADDRESS: ":8081"
      TRAEFIK_ENTRYPOINTS_public_HTTP_MIDDLEWARES: compress@docker
      TRAEFIK_ACCESSLOG: "true"
    labels:
      - traefik.enable=true
      - "traefik.http.routers.traefik.rule=PathPrefix(`/traefik`) && !PathPrefix(`/traefik/debug`)"
      - traefik.http.routers.traefik.service=api@internal
      - traefik.http.routers.traefik-metrics.rule=PathPrefix(`/traefik/metrics`)
      - traefik.http.routers.traefik-metrics.priority=200
      - traefik.http.routers.traefik-metrics.service=prometheus@internal
      - traefik.http.middlewares.compress.compress=true
    expose:
      - 8080/tcp
    ports:
      - 127.0.0.1:8080:8080/tcp
      - 8081:8081/tcp
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro

  geoip:
    image: ghcr.io/akvorado/ipinfo-geoipupdate:latest
    restart: unless-stopped
    env_file:
      - config/secrets/secrets.env
    environment:
      IPINFO_DATABASES: country asn
      UPDATE_FREQUENCY: 48h
    volumes:
      - akvorado-geoip:/data

  # ===================== SNMP metrics collector =====================
  snmp-collector:
    image: otel/opentelemetry-collector-contrib:0.119.0
    restart: unless-stopped
    env_file:
      - config/secrets/secrets.env
    command: ["--config=/etc/otelcol/config.yaml"]
    volumes:
      - ./docker/otel/snmp-collector.yaml:/etc/otelcol/config.yaml:ro
    depends_on:
      - otel-collector
```

- [ ] **Step 2: Provide a placeholder SNMP config so compose can parse before Task 5**

`docker-compose.yml` mounts `docker/otel/snmp-collector.yaml`, which does not exist yet. Create a temporary stub so `docker compose config` works (Task 5 replaces it):

```bash
printf 'receivers: {}\nexporters: {}\nservice:\n  pipelines: {}\n' > docker/otel/snmp-collector.yaml
```

- [ ] **Step 3: Validate compose parses and interpolates**

Run:
```bash
docker compose config >/dev/null && echo "COMPOSE_OK"
```
Expected: `COMPOSE_OK` with no `variable is not set` warnings. If `DEFAULT_CONNECTIONS`/`DEFAULT_SOURCES` still contain the `<<COPY ...>>` markers, replace them now with the verbatim values from `git show 463e1ab:docker-compose.yml`.

- [ ] **Step 4: Confirm the shared-CH alias and volume mapping are correct**

Run:
```bash
docker compose config | grep -A2 "aliases" | grep ch-server && echo "ALIAS_OK"
docker compose config --volumes | grep -x "akvorado-clickhouse" && echo "VOL_OK"
```
Expected: `ALIAS_OK` and `VOL_OK` (the volume resolves to `akvorado_akvorado-clickhouse` under project `akvorado`).

- [ ] **Step 5: Commit**

```bash
git add docker-compose.yml
git commit -m "feat: combined ClickStack+Akvorado docker-compose (shared ClickHouse, port 1800)"
```

---

## Task 5: SNMP collector config template and target generator

**Files:**
- Create (committed): `docker/otel/snmp-collector.example.yaml`
- Create (committed): `docker/otel/gen-snmp-targets.sh`
- Overwrite (gitignored): `docker/otel/snmp-collector.yaml`

**Interfaces:**
- Consumes: running `clickhouse` container (`akvorado-clickhouse-1`) with `default.exporters`; `SNMP_COMMUNITY` from container env at runtime.
- Produces: `docker/otel/snmp-collector.yaml` mounted by the `snmp-collector` service (Task 4).

- [ ] **Step 1: Create the committed template `docker/otel/snmp-collector.example.yaml`**

This is a single-device reference the generator clones per exporter. Community is resolved at runtime via `${env:SNMP_COMMUNITY}` so no secret is stored.

```yaml
# Template — the real docker/otel/snmp-collector.yaml is generated by
# gen-snmp-targets.sh from default.exporters and is gitignored.
receivers:
  snmp/example:
    collection_interval: 60s
    endpoint: udp://192.0.2.1:161      # placeholder device
    version: v2c
    community: ${env:SNMP_COMMUNITY}
    resource_attributes:
      snmp.interface.index:
        oid: "1.3.6.1.2.1.2.2.1.1"     # ifIndex
    metrics:
      snmp.interface.in.octets:
        unit: By
        sum:
          aggregation: cumulative
          monotonic: true
          value_type: int
        column_oid: "1.3.6.1.2.1.31.1.1.1.6"  # ifHCInOctets
        attributes: [snmp.interface.index]
      snmp.interface.out.octets:
        unit: By
        sum:
          aggregation: cumulative
          monotonic: true
          value_type: int
        column_oid: "1.3.6.1.2.1.31.1.1.1.10" # ifHCOutOctets
        attributes: [snmp.interface.index]

processors:
  batch: {}

exporters:
  otlp:
    endpoint: otel-collector:4317
    tls:
      insecure: true

service:
  pipelines:
    metrics:
      receivers: [snmp/example]
      processors: [batch]
      exporters: [otlp]
```

- [ ] **Step 1b: Create the committed extra-targets example**

Non-flow devices (switches, hosts that don't export NetFlow) are added here later.
Create `docker/otel/snmp-extra-targets.example.txt`:

```text
# Extra SNMP targets merged with devices discovered from default.exporters.
# One IP per line; '#' starts a comment. Copy to snmp-extra-targets.txt (gitignored) and edit.
# 10.0.0.2
# 10.0.0.3
```

The real `docker/otel/snmp-extra-targets.txt` is gitignored (Task 1) and optional;
the generator merges it with the exporters-table addresses.

- [ ] **Step 2: Create the generator `docker/otel/gen-snmp-targets.sh`**

```bash
#!/usr/bin/env bash
# Generate docker/otel/snmp-collector.yaml from default.exporters.
# Community is NOT written here; it is resolved at runtime via ${env:SNMP_COMMUNITY}.
set -euo pipefail

CH_CONTAINER="${CH_CONTAINER:-akvorado-clickhouse-1}"
EXTRA_FILE="${EXTRA_FILE:-$(dirname "$0")/snmp-extra-targets.txt}"
OUT="$(dirname "$0")/snmp-collector.yaml"

# Source 1: exporters discovered from the flow database.
mapfile -t ADDRS < <(docker exec -i "$CH_CONTAINER" clickhouse-client -q \
  "SELECT DISTINCT replaceRegexpOne(toString(ExporterAddress),'^::ffff:','') FROM default.exporters ORDER BY 1" 2>/dev/null || true)

# Source 2: optional manually-maintained targets (gitignored), one IP per line, '#' comments allowed.
if [ -f "$EXTRA_FILE" ]; then
  while IFS= read -r line; do
    line="${line%%#*}"; line="$(echo "$line" | tr -d '[:space:]')"
    [ -n "$line" ] && ADDRS+=("$line")
  done < "$EXTRA_FILE"
fi

# Union: dedup, stable sorted order.
mapfile -t ADDRS < <(printf '%s\n' "${ADDRS[@]}" | sort -u)

if [ "${#ADDRS[@]}" -eq 0 ]; then
  echo "No SNMP targets: default.exporters is empty and no $EXTRA_FILE provided." >&2
  exit 1
fi

{
  echo "# GENERATED by gen-snmp-targets.sh — do not edit; do not commit."
  echo "receivers:"
  for ip in "${ADDRS[@]}"; do
    key="${ip//./_}"
    cat <<YAML
  snmp/${key}:
    collection_interval: 60s
    endpoint: udp://${ip}:161
    version: v2c
    community: \${env:SNMP_COMMUNITY}
    resource_attributes:
      snmp.interface.index:
        oid: "1.3.6.1.2.1.2.2.1.1"
    metrics:
      snmp.interface.in.octets:
        unit: By
        sum: {aggregation: cumulative, monotonic: true, value_type: int}
        column_oid: "1.3.6.1.2.1.31.1.1.1.6"
        attributes: [snmp.interface.index]
      snmp.interface.out.octets:
        unit: By
        sum: {aggregation: cumulative, monotonic: true, value_type: int}
        column_oid: "1.3.6.1.2.1.31.1.1.1.10"
        attributes: [snmp.interface.index]
YAML
  done
  echo "processors:"
  echo "  batch: {}"
  echo "exporters:"
  echo "  otlp:"
  echo "    endpoint: otel-collector:4317"
  echo "    tls:"
  echo "      insecure: true"
  echo "service:"
  echo "  pipelines:"
  echo "    metrics:"
  printf "      receivers: ["
  printf '%s' "$(IFS=,; echo "${ADDRS[*]/#/snmp/}" | sed 's/\./_/g')"
  echo "]"
  echo "      processors: [batch]"
  echo "      exporters: [otlp]"
} > "$OUT"

echo "Wrote $OUT with ${#ADDRS[@]} device(s): ${ADDRS[*]}"
chmod +x "$0"
```

- [ ] **Step 3: Make the generator executable and run it against the live ClickHouse**

The current Akvorado ClickHouse is still running as `akvorado-clickhouse-1`.

```bash
chmod +x docker/otel/gen-snmp-targets.sh
./docker/otel/gen-snmp-targets.sh
```
Expected: `Wrote docker/otel/snmp-collector.yaml with 1 device(s): 192.168.88.1`.

- [ ] **Step 4: Validate the generated collector config**

```bash
docker run --rm -e SNMP_COMMUNITY=dummy \
  -v "$PWD/docker/otel/snmp-collector.yaml:/etc/otelcol/config.yaml:ro" \
  otel/opentelemetry-collector-contrib:0.119.0 validate --config=/etc/otelcol/config.yaml \
  && echo "OTELCOL_OK"
```
Expected: `OTELCOL_OK`. If validation reports a schema mismatch for `snmpreceiver` (it is alpha and the field layout shifts between releases), consult the receiver README for tag `v0.119.0` at
`https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/v0.119.0/receiver/snmpreceiver`
and adjust the metric/attribute keys in both `snmp-collector.example.yaml` and the generator, then re-run Steps 3–4.

- [ ] **Step 5: Verify the generated file is gitignored; commit template + generator only**

```bash
git check-ignore docker/otel/snmp-collector.yaml && echo REAL_IGNORED
git add docker/otel/snmp-collector.example.yaml docker/otel/gen-snmp-targets.sh docker/otel/snmp-extra-targets.example.txt
git status   # confirm snmp-collector.yaml and snmp-extra-targets.txt NOT staged
git commit -m "feat: SNMP collector template + generator (exporters table union extra targets)"
```

---

## Task 6: Cutover and live verification

**Files:** none (operational). Do NOT run `docker compose down -v` anywhere.

**Interfaces:**
- Consumes: all prior tasks. Requires Docker access to the running Akvorado stack at `/home/terry/akvorado`.

- [ ] **Step 1: Snapshot current flow-row count (baseline for data-preservation check)**

```bash
docker exec akvorado-clickhouse-1 clickhouse-client -q "SELECT count() FROM default.flows"
```
Record the number (≈ 38,420,480 or higher).

> **Rollback:** if anything below goes wrong, the verified backup from Task 0
> (`/home/terry/akvorado-backups/20260709-124313/`) restores full state — see its
> `RESTORE.md` (remember `--allow_suspicious_low_cardinality_types=1`).

- [ ] **Step 2: Stop the old Akvorado stack (retain volumes)**

```bash
cd /home/terry/akvorado
docker compose down          # NEVER add -v
docker volume ls | grep akvorado_akvorado-clickhouse && echo "VOLUME_RETAINED"
```
Expected: `VOLUME_RETAINED`.

- [ ] **Step 3: Bring up the combined stack**

```bash
cd /home/terry/ClickStack
docker compose up -d
```

- [ ] **Step 4: Wait for health and confirm all services running**

```bash
sleep 30
docker compose ps
```
Expected: `clickhouse`, `kafka`, `redis` healthy; `app`, `otel-collector`, `akvorado-*`, `traefik`, `snmp-collector` up.

- [ ] **Step 5: Verify flow data preserved**

```bash
docker exec akvorado-clickhouse-1 clickhouse-client -q "SELECT count() FROM default.flows"
```
Expected: ≥ the Step 1 baseline (data intact; ingest may have added rows).

- [ ] **Step 6: Verify shared DB has both schemas**

```bash
docker exec akvorado-clickhouse-1 clickhouse-client -q \
  "SELECT name FROM system.tables WHERE database='default' AND (name='flows' OR name LIKE 'otel_%') ORDER BY name"
```
Expected: `flows` present immediately; `otel_*` appear once HyperDX/otel-collector initialize (send a test OTLP log if needed).

- [ ] **Step 7: Verify HyperDX reachable on 1800 and Akvorado console on 8081**

```bash
curl -sf -o /dev/null -w "hyperdx:%{http_code}\n" http://localhost:1800/
curl -sf -o /dev/null -w "console:%{http_code}\n" http://localhost:8081/
```
Expected: HTTP `200` (or `3xx` redirect) for both.

- [ ] **Step 8: (Re)generate SNMP targets against the combined stack and restart collector**

```bash
./docker/otel/gen-snmp-targets.sh
docker compose up -d snmp-collector
sleep 20
docker compose logs --tail=40 snmp-collector
```
Expected: logs show SNMP scrape of `192.168.88.1` without auth/timeout errors. Wrong community or unreachable device shows here — fix `config/secrets/secrets.env` or device reachability and restart.

- [ ] **Step 9: Confirm SNMP metrics reached ClickHouse via HyperDX pipeline**

```bash
docker exec akvorado-clickhouse-1 clickhouse-client -q \
  "SELECT count() FROM default.otel_metrics_sum WHERE MetricName LIKE 'snmp.interface%'"
```
Expected: count > 0 within a couple of collection intervals. In the HyperDX UI (`:1800`) the `snmp.interface.in.octets` metric is queryable.

---

## Task 7: Fork and push (secret-safety gated)

**Files:** none (git/remote).

**Interfaces:**
- Consumes: committed branch `akvorado`. Requires user go-ahead (forking is an outward action).

- [ ] **Step 1: Final secret-safety audit before any push**

```bash
git check-ignore config/akvorado/outlet.yaml docker/otel/snmp-collector.yaml config/secrets/secrets.env
git ls-files | grep -E "config/akvorado/.*\.yaml$" | grep -v ".example.yaml$" && echo "LEAK_REAL_CONFIG" || echo "NO_REAL_CONFIG"
git grep -nI "CHANGEME\|CHANGEME" $(git rev-list --all) -- . 2>/dev/null && echo "LEAK_IN_HISTORY" || echo "HISTORY_CLEAN"
```
Expected: all three real paths print (ignored); `NO_REAL_CONFIG`; `HISTORY_CLEAN`. **Stop and remediate if any leak is reported.**

- [ ] **Step 2: Create the fork (only after user confirms)**

```bash
gh repo fork ClickHouse/ClickStack --remote=false --clone=false
```
This creates `<your-user>/ClickStack`. Add it as a remote:
```bash
gh repo view --json owner -q .owner.login   # note your username
git remote add fork git@github.com:<your-user>/ClickStack.git
```

- [ ] **Step 3: Push the branch to the fork**

```bash
git push -u fork akvorado
```
Expected: branch `akvorado` published to your fork.

- [ ] **Step 4: Confirm the pushed tree contains no secrets**

```bash
git ls-tree -r --name-only fork/akvorado | grep -E "secrets\.env$|outlet\.yaml$|snmp-collector\.yaml$" && echo "LEAK" || echo "REMOTE_CLEAN"
```
Expected: `REMOTE_CLEAN` (only `.example` variants exist on the remote).

---

## Self-Review

**Spec coverage:**
- Shared ClickHouse = Akvorado's live instance, alias `ch-server` → Task 4 (clickhouse service, aliases) + Task 6 (data preserved). ✅
- ClickHouse 25.8 → Global Constraints + Task 4 image. ✅
- Single authored compose, no `extends` → Task 4. ✅
- Project name `akvorado` / volume re-attach → Task 3 (.env) + Task 4 Step 4. ✅
- HyperDX port 1800 → Task 3 + Task 4 (app) + Task 6 Step 7. ✅
- SNMP collector via otelcol-contrib → Task 4 (service) + Task 5 (config). ✅
- SNMP targets from `default.exporters` → Task 5 generator. ✅
- Community reuse via runtime env, no secret in files → Task 1 + Task 5. ✅
- Public-fork secrets split (.example vs gitignored) → Tasks 1, 2, 5, 7. ✅
- CH config fragments (observability.xml, server.xml), skip ClickStack config → Task 2 + Task 4 (clickhouse mounts). ✅
- IPinfo geoip updater (token via secrets) → Task 4 (geoip) + Task 1. ✅
- Cutover procedure (down without -v, up combined) → Task 6. ✅
- Verification plan (flow count, both schemas, ports, SNMP metrics, secret check) → Task 6 + Task 7. ✅

**Placeholder scan:** Only intentional template placeholders remain (`CHANGEME`, `192.0.2.1`, `<your-user>`) and the explicit verbatim-copy markers for the two unchanged HyperDX JSON strings (sourced from `git show 463e1ab:docker-compose.yml`). No "TODO"/"handle edge cases" gaps.

**Type/name consistency:** service names (`clickhouse`, `otel-collector`, `snmp-collector`, `app`), alias `ch-server`, volume `akvorado-clickhouse`, secret vars `SNMP_COMMUNITY`/`IPINFO_TOKEN`, and generated file path `docker/otel/snmp-collector.yaml` are used identically across Tasks 1–7.
