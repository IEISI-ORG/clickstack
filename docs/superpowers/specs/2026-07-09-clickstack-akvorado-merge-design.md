# ClickStack + Akvorado Combined Stack — Design

Date: 2026-07-09
Branch: `akvorado` (fork of `ClickHouse/ClickStack`)
Status: Design — awaiting review

## Goal

Combine the ClickStack (HyperDX observability) and Akvorado (network flow
analytics) Docker Compose deployments into a single `docker compose up` stack,
sharing one ClickHouse instance so flow data and observability data live in the
same database — **while preserving the ~38.4M flow rows already ingested by the
running Akvorado deployment.**

## Decisions (locked)

1. **Single shared ClickHouse = Akvorado's existing instance.** The live
   Akvorado ClickHouse (backed by volume `akvorado_akvorado-clickhouse`, ~780
   MiB / 38.4M rows across `flows*`, `exporters`, dictionaries) becomes the
   shared database. HyperDX and otel-collector point at it; it gains a network
   alias `ch-server`. ClickStack's own `ch-server` (empty `default` DB) is
   dropped.
2. **ClickHouse version 25.8-alpine (unchanged).** Akvorado 2.0.1 supports only
   the 25.x LTS line, and the existing data was written by 25.8. Keeping the
   Akvorado instance means no downgrade and no data reset.
3. **Single authored compose + copied config.** One root `docker-compose.yml`
   with all services inlined (images pinned directly, no `extends:` /
   `versions.yml`). Akvorado app config and ClickHouse fragments are copied into
   the ClickStack repo.
4. **HyperDX app on host port 1800** (moved from 8080 to avoid Traefik).
5. **Combined compose project name = `akvorado`** so it re-attaches the existing
   `akvorado_*` named volumes instead of creating empty ones.

## Current runtime facts (verified 2026-07-09)

- ClickStack ClickHouse (`.volumes/ch_data`): `default` DB **empty** (no
  `otel_*` tables); 89M is only `system.*` logs. Nothing to preserve.
- Akvorado stack is **running** (up 2 days): `akvorado-clickhouse-1`
  (`clickhouse/clickhouse-server:25.8`, healthy) holds:
  - `default.flows` — 780.72 MiB, 38,420,480 rows
  - `default.flows_5m0s` / `flows_1h0m0s` / `flows_1m0s` — rollups
  - `default.exporters` + dictionaries (`asns`, `networks`, `protocols`, ...)
- Existing named volumes: `akvorado_akvorado-clickhouse`,
  `akvorado_akvorado-kafka`, `akvorado_akvorado-console-db`,
  `akvorado_akvorado-geoip`, `akvorado_akvorado-run`.

## Combined architecture

### Services
From ClickStack: `db` (mongo:5.0.32-focal), `otel-collector`, `app` (HyperDX).

From Akvorado (images pinned inline from `versions.yml`): `kafka`
(apache/kafka:4.1.0), `kafka-ui`, `redis` (valkey/valkey:7.2),
`akvorado-orchestrator`, `akvorado-console`, `akvorado-inlet`,
`akvorado-outlet`, `akvorado-conntrack-fixer`, `traefik` (traefik:v3.6.1),
IPinfo geoip updater (default enrichment, no license key).

**Shared:** `clickhouse` (clickhouse/clickhouse-server:25.8) — Akvorado's
existing service, now also aliased `ch-server`.

**Dropped:** ClickStack's `ch-server` service and its empty `.volumes/ch_data`.

### Shared ClickHouse wiring
- Service `clickhouse` keeps volume `akvorado_akvorado-clickhouse` (declared so
  the combined stack re-attaches it — via project name `akvorado` or an
  `external`/explicit `name:` volume mapping).
- **Network aliases:** `clickhouse` (Akvorado's config references
  `clickhouse:9000`) **and** `ch-server` (HyperDX `DEFAULT_CONNECTIONS` uses
  `http://ch-server:8123`; otel-collector uses `tcp://ch-server:9000`).
- Akvorado config files need **no edits**.
- ClickHouse reaches `kafka:9092` (Kafka table engine) via the shared network.

### Schema ownership (no collision)
- Existing Akvorado tables (`flows*`, `exporters`, dictionaries) stay in
  `default`.
- HyperDX / otel-collector create `otel_logs`, `otel_traces`,
  `otel_metrics_*`, `hyperdx_sessions` in `default`
  (`HYPERDX_OTEL_EXPORTER_CREATE_LEGACY_SCHEMA=true`).
- Distinct table names in the same `default` DB. No collision with existing data.

### ClickHouse config
- Keep Akvorado's ClickHouse config fragments as the container already uses
  them: `observability.xml` (Prometheus `/metrics` + JSON logger) and
  `server.xml` → `config.d/akvorado.xml` (system-log TTLs).
- `CLICKHOUSE_SKIP_USER_SETUP=1` (Akvorado default): `default` user, empty
  password, full access — satisfies HyperDX's default connection.
- ClickStack's `config.xml` / `users.xml` are **not** used (their reconciliation
  is unnecessary under this design).

### Network
- Single network from Akvorado's `default`: IPv6 enabled, fixed subnets
  (`247.16.14.0/24`, `fd1c:8ce3:6fb:1::/64`), bridge name `br-akvorado`.
- All ClickStack services (`db`, `otel-collector`, `app`) join it.
- `clickhouse` service carries alias `ch-server`.
- `akvorado-conntrack-fixer` keeps `network_mode: host`.

### Ports (final host map)
| Service | Port(s) | Change |
|---|---|---|
| HyperDX app | **1800** | moved from 8080 |
| HyperDX API | 8000 | unchanged |
| HyperDX OPAMP | 4320 | unchanged |
| otel-collector | 13133, 24225, 4317, 4318, 8888 | unchanged |
| Akvorado Traefik | 127.0.0.1:8080, 8081 | unchanged |
| Akvorado inlet | 2055/udp, 4739/udp, 6343/udp | unchanged |
| Akvorado outlet | 10179/tcp | unchanged |

`HYPERDX_APP_PORT=1800`, `HYPERDX_APP_URL`, and `FRONTEND_URL` updated for 1800.

### Volumes
- Re-attach existing Akvorado named volumes:
  `akvorado_akvorado-clickhouse` (flow data), `akvorado_akvorado-kafka`,
  `akvorado_akvorado-console-db`, `akvorado_akvorado-geoip`,
  `akvorado_akvorado-run`.
- Keep ClickStack bind mount for mongo: `.volumes/db`.
- ClickStack's `.volumes/ch_data`, `.volumes/ch_logs` are no longer used.

### Environment / `.env`
Merge into one root `.env`:
- ClickStack image repo/version vars; `HYPERDX_APP_PORT=1800` and matching URLs;
  `HYPERDX_OTEL_EXPORTER_CLICKHOUSE_DATABASE=default`.
- Set `COMPOSE_PROJECT_NAME=akvorado` (re-attaches existing volumes).
- Akvorado's multi-file `COMPOSE_FILE` chain is replaced by the single authored
  compose. Akvorado image tag (`ghcr.io/akvorado/akvorado:2.0.1`) and support
  images pinned inline.

## File layout (target)
```
ClickStack/
  docker-compose.yml          # combined, authored, no `extends`
  .env                        # merged; COMPOSE_PROJECT_NAME=akvorado
  config/akvorado/            # copied: akvorado.yaml, console.yaml, inlet.yaml,
                              #         outlet.yaml, demo.yaml
  docker/clickhouse/akvorado/ # copied: observability.xml, server.xml
  docker/akvorado/            # copied geoip Dockerfile/script (IPinfo)
```

## Cutover procedure (operational)
The combined stack reuses Akvorado's host ports and named volumes, so it cannot
run alongside the existing `/home/terry/akvorado` deployment.
1. `cd /home/terry/akvorado && docker compose down` (stops containers; **named
   volumes are retained**, so flow data is preserved).
2. `cd /home/terry/ClickStack && docker compose up -d` (project `akvorado`
   re-attaches `akvorado_akvorado-clickhouse` etc.).
3. Verify flow rows are intact (see Verification).

## Deferred / out of scope
- Akvorado optional profiles: Prometheus, Loki, Grafana, demo exporters, TLS,
  Maxmind geoip. Not in the core combined file (addable later).
- Putting HyperDX behind Akvorado's Traefik. HyperDX stays on host port 1800.
- ClickHouse cluster/keeper (Akvorado cluster compose). Standalone only.

## Risks
- **HyperDX access-management.** Akvorado's `SKIP_USER_SETUP` may leave
  `access_management=0`. If HyperDX needs to run access-control SQL, add a small
  `config.d` fragment enabling `<access_management>1</access_management>` for the
  `default` user. Verify during bring-up.
- **Volume re-attach depends on project name.** If `COMPOSE_PROJECT_NAME` is not
  `akvorado` (or the volume isn't declared `external`), the stack creates empty
  volumes and the flow data appears "lost" (it isn't — it stays in
  `akvorado_akvorado-clickhouse`). Fix by correcting the project name/volume ref.
- **Prometheus endpoint expectations.** Akvorado's Traefik `metrics.port=8123`
  label targeted its clickhouse; unchanged here. Low impact.
- **Both stacks running at once.** Would collide on ports/volumes — follow the
  cutover procedure.

## Verification plan
1. `docker compose config` parses without error.
2. After cutover, all services reach healthy/running.
3. ClickHouse: `SELECT count() FROM default.flows` still ≈ 38.4M (data preserved).
4. `SHOW TABLES FROM default` shows Akvorado flow tables **and** new `otel_*`.
5. HyperDX UI reachable on `:1800`, connected via `ch-server` alias.
6. Akvorado console reachable via Traefik `:8081`; send a test flow to inlet
   (2055/udp) and confirm new rows land in `default.flows`.
7. `git` diff review: only intended files added/changed.
