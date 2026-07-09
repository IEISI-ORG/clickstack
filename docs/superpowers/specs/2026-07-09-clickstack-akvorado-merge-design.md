# ClickStack + Akvorado Combined Stack — Design

Date: 2026-07-09
Branch: `akvorado` (fork of `ClickHouse/ClickStack`)
Status: Design — awaiting review

## Goal

Combine the ClickStack (HyperDX observability) and Akvorado (network flow
analytics) Docker Compose deployments into a single `docker compose up` stack,
sharing one ClickHouse instance so flow data and observability data live in the
same database.

## Decisions (locked)

1. **Single shared ClickHouse.** Akvorado writes flows into ClickStack's
   `ch-server`; HyperDX queries the same instance. Akvorado's own `clickhouse`
   service is dropped.
2. **ClickHouse version 25.8-alpine.** Akvorado 2.0.1 supports only the 25.x LTS
   line (`clickhouse-server:25.8`); ClickStack was on 26.1. The shared instance
   runs `clickhouse/clickhouse-server:25.8-alpine` (small downgrade for
   ClickStack, backward-compatible for HyperDX).
3. **Single authored compose + copied config.** One root `docker-compose.yml`
   with all services inlined (Akvorado images pinned directly, no `extends:` /
   `versions.yml` indirection). Akvorado config and needed ClickHouse fragments
   are copied into the ClickStack repo.

## Source stacks (as-is)

### ClickStack (`/home/terry/ClickStack`)
- `db` — mongo:5.0.32-focal (HyperDX metadata), internal only.
- `otel-collector` — OTLP receiver; host ports 13133, 24225, 4317, 4318, 8888.
- `app` — HyperDX UI; host ports `HYPERDX_API_PORT=8000`, `HYPERDX_APP_PORT=8080`, OPAMP 4320.
- `ch-server` — clickhouse-server:26.1-alpine; internal only; mounts
  `config.xml` + `users.xml`; `default` user, empty password.
- Network: single `internal` bridge. Volumes: bind mounts under `.volumes/`.

### Akvorado (`/home/terry/akvorado`)
- `kafka`, `kafka-ui`, `redis`.
- `akvorado-orchestrator`, `akvorado-console`, `akvorado-inlet`
  (UDP 2055/4739/6343), `akvorado-outlet` (10179/tcp), `akvorado-conntrack-fixer`
  (host network).
- `clickhouse` — clickhouse-server:25.8; mounts `observability.xml`, `server.xml`.
- `traefik` — reverse proxy; host `127.0.0.1:8080` (private) + `8081` (public).
- IPinfo geoip updater (default enrichment, no license key).
- Network: `default` bridge with IPv6 + fixed subnet
  (`247.16.14.0/24`, `fd1c:8ce3:6fb:1::/64`), bridge name `br-akvorado`.
- Config: `extends:` from `versions.yml`; app config under `../config`.
- Volumes: named (`akvorado-kafka`, `akvorado-geoip`, `akvorado-clickhouse`,
  `akvorado-run`, `akvorado-console-db`).

## Combined architecture

### Services
Kept from ClickStack: `db`, `otel-collector`, `app`, `ch-server` (→ 25.8-alpine,
aliased `clickhouse`).

Kept from Akvorado: `kafka`, `kafka-ui`, `redis`, `akvorado-orchestrator`,
`akvorado-console`, `akvorado-inlet`, `akvorado-outlet`,
`akvorado-conntrack-fixer`, `traefik`, IPinfo geoip updater.

Dropped: Akvorado `clickhouse` service and `akvorado-clickhouse` volume.

### Shared ClickHouse wiring
- `ch-server` image → `clickhouse/clickhouse-server:25.8-alpine`.
- Add a **network alias** `clickhouse` to `ch-server` so Akvorado's config
  (`clickhousedb.servers: clickhouse:9000`, console/outlet deps) resolves it
  with zero Akvorado config edits.
- HyperDX keeps referencing `ch-server` (its `DEFAULT_CONNECTIONS`).
- ClickHouse must reach `kafka:9092` (Kafka table engine) — satisfied by the
  shared network.

### Schema ownership (no collision)
- Akvorado orchestrator auto-creates flow tables + Kafka-engine consumer +
  dictionaries in `default` on startup.
- HyperDX / otel-collector auto-creates `otel_logs`, `otel_traces`,
  `otel_metrics_*`, `hyperdx_sessions` in `default`
  (`HYPERDX_OTEL_EXPORTER_CREATE_LEGACY_SCHEMA=true`).
- Different table names in the same `default` database. No collision.

### ClickHouse config reconciliation
- Keep ClickStack `config.xml` (prometheus on 9363, `debug` logger,
  `default` database, remote_servers) as authoritative base.
- Keep ClickStack `users.xml` (`default`/empty, `api`, `worker`).
- **Add** Akvorado `server.xml` as a `config.d` fragment (system-log TTLs =
  30-day retention; purely additive).
- **Skip** Akvorado `observability.xml` — its `<prometheus>` (no port) and
  `fatal`/JSON logger conflict with ClickStack's `config.xml`. ClickStack
  already exposes Prometheus metrics on 9363.

### Network
- Single network adopting Akvorado's `default` config: IPv6 enabled, fixed
  subnets, bridge name `br-akvorado`. All ClickStack services join it.
- `ch-server` carries alias `clickhouse` on this network.
- `akvorado-conntrack-fixer` keeps `network_mode: host` (unchanged).

### Ports (final host map)
| Service | Port(s) | Change |
|---|---|---|
| HyperDX app | **8090** | moved from 8080 to avoid Traefik collision |
| HyperDX API | 8000 | unchanged |
| HyperDX OPAMP | 4320 | unchanged |
| otel-collector | 13133, 24225, 4317, 4318, 8888 | unchanged |
| Akvorado Traefik | 127.0.0.1:8080, 8081 | unchanged (documented convention) |
| Akvorado inlet | 2055/udp, 4739/udp, 6343/udp | unchanged |
| Akvorado outlet | 10179/tcp | unchanged |

`HYPERDX_APP_PORT`, `HYPERDX_APP_URL`, and `FRONTEND_URL` updated for 8090.

### Volumes
- Keep ClickStack bind mounts: `.volumes/ch_data`, `.volumes/ch_logs`,
  `.volumes/db`.
- Keep Akvorado named volumes: `akvorado-kafka`, `akvorado-geoip`,
  `akvorado-run`, `akvorado-console-db`.
- Remove `akvorado-clickhouse` (shared ch uses `.volumes/ch_data`).

### Environment / `.env`
Merge both `.env` files into one root `.env`:
- ClickStack image repo/version vars, HyperDX ports (APP port → 8090),
  `HYPERDX_OTEL_EXPORTER_CLICKHOUSE_DATABASE=default`.
- Akvorado has minimal env; its multi-file `COMPOSE_FILE` chain is not carried
  over (single authored compose replaces it). Akvorado image tag
  (`ghcr.io/akvorado/akvorado:2.0.1`) and support images pinned inline.

## File layout (target)
```
ClickStack/
  docker-compose.yml          # combined, authored, no `extends`
  .env                        # merged
  config/akvorado/            # copied: akvorado.yaml, console.yaml, inlet.yaml,
                              #         outlet.yaml, demo.yaml
  docker/clickhouse/local/    # existing config.xml, users.xml, init-db.sh
  docker/clickhouse/akvorado-server.xml   # copied Akvorado server.xml (TTLs)
  docker/akvorado/            # copied: geoip Dockerfile/script if IPinfo used
```

## Deferred / out of scope
- Akvorado optional profiles: Prometheus, Loki, Grafana, demo exporters, TLS,
  Maxmind geoip. Not carried into the core combined file (can be added later).
- Putting HyperDX behind Akvorado's Traefik. HyperDX stays on its own host port.
- Cross-querying flows from HyperDX beyond what the shared `default` database
  makes available (HyperDX can add the flow tables as a source manually later).
- Cluster/keeper ClickHouse (Akvorado's cluster compose). Standalone only.

## Risks
- **Version downgrade of ClickStack ch-server (26.1 → 25.8).** If existing
  `.volumes/ch_data` was written by 26.1, ClickHouse may refuse to start on
  25.8 (no downgrade of on-disk format). Mitigation: fresh `.volumes/ch_data`
  for the combined stack, or verify current data dir version first.
- **Akvorado migrations on shared instance.** Orchestrator must complete schema
  creation against the shared ch before console/outlet become healthy.
  Mitigation: keep `depends_on` health conditions; verify orchestrator logs.
- **Prometheus endpoint overlap.** ClickStack exposes CH metrics on 9363;
  Akvorado's Traefik `metrics.port=8123` label referenced its own clickhouse.
  Low impact (metrics scraping only); note during verification.

## Verification plan
1. `docker compose config` parses without error.
2. `docker compose up -d`; all services reach healthy/running.
3. ClickHouse: `SHOW TABLES FROM default` shows both `otel_*` and Akvorado flow
   tables; `SELECT 1` as `default` user succeeds.
4. HyperDX UI reachable on `:8090`, connected to `ch-server`.
5. Akvorado console reachable via Traefik `:8081`; ingest test flow to inlet
   (2055/udp) and confirm rows land in the flow table.
6. `git` diff review: only intended files added/changed.
