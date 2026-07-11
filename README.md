# ClickStack — network observability on ClickHouse

A self-hosted network-observability stack that unifies **flow analytics** and
**SNMP metrics** on a single ClickHouse backend, visualized with **Grafana** and the
**Akvorado** console.

- **Flows** — NetFlow / IPFIX / sFlow ingested and enriched by [Akvorado](https://akvorado.net)
  (GeoIP, ASN, interface metadata), stored in ClickHouse.
- **SNMP** — interface counters polled by [Telegraf](https://github.com/influxdata/telegraf)
  (`inputs.snmp`) straight into ClickHouse. gNMI-ready for future streaming telemetry.
- **Metrics/OTLP** — a standalone OpenTelemetry Collector accepts OTLP (4317/4318) for
  any app logs/traces/metrics you want alongside the network data. To collect the
  **host's own** CPU/mem/disk/net, run a native collector on the host — see
  [docs/host-metrics.md](./docs/host-metrics.md).
- **Visualize** — Grafana (ClickHouse datasource) for custom dashboards; the Akvorado
  console for interactive flow exploration.

Everything shares one ClickHouse database, so flows, SNMP, and app telemetry are
cross-queryable. See **[SCHEMA.md](./SCHEMA.md)** for the data model and example queries.

## Architecture

```
 routers ──NetFlow/IPFIX/sFlow──▶ akvorado-inlet ─▶ kafka ─▶ akvorado-outlet ─┐
 devices ──────SNMP (UDP 161)───▶ telegraf ───────────────────────────────┐  │
 apps    ──────OTLP (4317/4318)─▶ otel-collector (contrib) ─────────────┐  │  │
                                                                        ▼  ▼  ▼
                                                         ClickHouse (default DB, 25.8)
                                                                        ▲     ▲
                                    Grafana (grafana-clickhouse-ds) ────┘     │
                                    Akvorado console (flow explorer) ─────────┘
        Traefik + Cloudflare Tunnel front the console/Grafana on your domains.
```

## Services

| Service | Purpose |
|---|---|
| `clickhouse` | Shared datastore (ClickHouse 25.8), aliased `ch-server`. |
| `akvorado-{inlet,outlet,orchestrator,console,conntrack-fixer}` | Flow ingest, processing, UI. |
| `kafka`, `redis` | Akvorado's flow buffer + console cache. |
| `telegraf` | SNMP polling → `default.snmp`. |
| `otel-collector` | Standalone OTLP receiver → `default.otel_*`. |
| `grafana` | Dashboards over ClickHouse (`127.0.0.1:3000`). |
| `traefik`, `kafka-ui`, `geoip` | Reverse proxy, Kafka admin, GeoIP updater. |

## Quick start

```bash
git clone git@github.com:IEISI-ORG/clickstack.git
cd clickstack

# 1. Provide secrets (never committed — see config/secrets/secrets.env.example)
cp config/secrets/secrets.env.example config/secrets/secrets.env
$EDITOR config/secrets/secrets.env      # SNMP_COMMUNITY, IPINFO_TOKEN,
                                        # GF_SECURITY_ADMIN_PASSWORD, GF_SERVER_ROOT_URL

# 2. Provide Akvorado app config (also gitignored)
cp config/akvorado/akvorado.example.yaml config/akvorado/akvorado.yaml   # and the others
#    ...customize networks/exporters/community for your site.

# 3. Bring it up
docker compose up -d

# 4. Generate SNMP poll targets from the flow exporters, then (re)start Telegraf
./docker/telegraf/gen-telegraf-agents.sh
docker compose up -d telegraf
```

Send flows to the inlet on `2055/udp` (NetFlow), `4739/udp` (IPFIX), `6343/udp` (sFlow).

## Configuration & secrets

Real credentials and site config are **gitignored**; only sanitized `*.example`
templates are tracked.

| Real (gitignored) | Committed template |
|---|---|
| `config/secrets/secrets.env` | `config/secrets/secrets.env.example` |
| `config/akvorado/*.yaml` | `config/akvorado/*.example.yaml` |
| `docker/telegraf/telegraf.d/agents.conf` (generated) | `docker/telegraf/gen-telegraf-agents.sh` |

- **SNMP targets** are generated from Akvorado's `exporters` table by
  `gen-telegraf-agents.sh`; add non-flow devices to `docker/telegraf/snmp-extra-targets.txt`.
- **gNMI** streaming telemetry: a commented `[[inputs.gnmi]]` template is in
  `docker/telegraf/telegraf.conf`, ready for Cisco/Juniper/Arista/Nokia gear.

## Access

- **Grafana:** `http://127.0.0.1:3000` (admin + your password), or your public URL via
  the Cloudflare tunnel. Dashboard **SNMP Interfaces** ships provisioned.
- **Akvorado console:** Traefik public entrypoint `:8081` (or your tunnel hostname).
- Flow UDP ports: `2055`, `4739`, `6343`. OTLP: `4317` (gRPC), `4318` (HTTP).

## Data

One ClickHouse `default` database holds `flows` (+ rollups), `snmp`, `otel_*`, and
Akvorado enrichment dictionaries. **[SCHEMA.md](./SCHEMA.md)** documents every table and
gives copy-paste Grafana queries (top talkers, per-interface bitrate, traffic by country…).

## Origins

Bootstrapped from [ClickHouse/ClickStack](https://github.com/ClickHouse/ClickStack)'s
compose, then reworked into a network-observability deployment: Akvorado for flows,
Telegraf for SNMP, Grafana for visualization, HyperDX removed. Built on
[ClickHouse](https://clickhouse.com), [Akvorado](https://akvorado.net),
[Telegraf](https://www.influxdata.com/time-series-platform/telegraf/),
[Grafana](https://grafana.com), and the [OpenTelemetry Collector](https://opentelemetry.io).
