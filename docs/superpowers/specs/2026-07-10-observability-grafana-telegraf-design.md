# Observability Pivot: Grafana CE + Telegraf (retire HyperDX) — Design

Date: 2026-07-10
Branch: `akvorado`
Status: Design — awaiting review
Supersedes: the SNMP + HyperDX portions of `2026-07-09-clickstack-akvorado-merge-design.md`
(shared ClickHouse, Akvorado stack, data preservation still hold).

## Goal

Replace HyperDX (found unsuitable as a UI) with Grafana CE, and replace the
alpha OTEL `snmpreceiver` with Telegraf. Keep everything on the shared ClickHouse
so Grafana, Akvorado's console, and Telegraf all read/write one database. No flow
data loss; Akvorado stack untouched.

## Decisions (locked)

1. **Remove HyperDX `app` + `db` (mongo).** HyperDX UI is dropped.
2. **Replace the HyperDX `otel-collector` with a standalone `otel/opentelemetry-collector-contrib`**
   running a static OTLP→ClickHouse config. Reason: the HyperDX collector is
   OpAMP-supervised — it fetches its pipeline config from `app:4320` and cannot
   run without the app. The standalone collector preserves OTLP ingestion
   (4317/4318) for future apps, decoupled from HyperDX. (Nothing currently sends
   external OTLP; this is forward-looking capacity.)
3. **Replace `snmp-collector` (alpha OTEL snmpreceiver) with Telegraf.**
   - `inputs.snmp` — active, polling the MikroTik exporter(s) discovered from
     `default.exporters`; community from `${SNMP_COMMUNITY}`.
   - `inputs.gnmi` — **commented template** (example subscription paths + TLS/cred
     wiring) ready to enable per-device when gNMI-capable gear (Cisco/Juniper/
     Arista/Nokia) is added. The MikroTik/RouterOS does NOT support gNMI, so SNMP
     stays required for it.
   - `outputs.sql` with the `clickhouse` driver →
     `data_source_name = clickhouse://ch-server:9000/default` → a wide table
     `default.snmp` (Telegraf auto-creates it). gNMI, when enabled, lands in its
     own table (e.g. `default.gnmi`).
4. **Add Grafana CE** (`grafana/grafana-oss`):
   - Published `127.0.0.1:3000`; `GF_SERVER_ROOT_URL=https://grafana.example.com`.
   - `GF_INSTALL_PLUGINS=grafana-clickhouse-datasource`.
   - Admin user `admin`, password from gitignored secrets (`GRAFANA_ADMIN_PASSWORD`).
   - Provisioned ClickHouse datasource + a starter SNMP-interface dashboard
     (queries `default.snmp`).
5. **Access:** new Cloudflare tunnel ingress `grafana.example.com → http://localhost:3000`
   (user edits `/etc/cloudflared/config.yml` + Cloudflare DNS). The freed
   `hyperdx.example.com` ingress can be retired.
6. **Unchanged:** ClickHouse (shared, 25.8), all Akvorado services, flow data,
   the exporters-table target discovery (now feeding Telegraf's `agents` list).

## Architecture (after pivot)

```
Network devices --SNMP--> telegraf (inputs.snmp) --outputs.sql--> ClickHouse default.snmp
(future gNMI)   --gNMI--> telegraf (inputs.gnmi)  --outputs.sql--> ClickHouse default.gnmi
apps (future)   --OTLP--> otel-collector (contrib, static) ------> ClickHouse default.otel_*
network flows   --------> akvorado (kafka/outlet) --------------> ClickHouse default.flows*
                                                                        |
                    Grafana CE (grafana-clickhouse-datasource) --------/  (dashboards)
                    Akvorado console -----------------------------------/  (flow explorer)
```

Single query surface: ClickHouse `default`.

## Component detail

### Telegraf (`telegraf`)
- Image: `telegraf` (pin a current stable, e.g. `telegraf:1.32-alpine`).
- `env_file: config/secrets/secrets.env` (provides `SNMP_COMMUNITY`).
- Config `docker/telegraf/telegraf.conf` (committed template) + a generated
  `docker/telegraf/telegraf.d/agents.conf` (gitignored) with the real agent IPs
  from `default.exporters`.
- `[[inputs.snmp]]`: `agents` (generated), `version = 2`, `community = "${SNMP_COMMUNITY}"`,
  IF-MIB interface `field`/`table` (ifName, ifHCInOctets, ifHCOutOctets, ifInErrors,
  ifOutErrors, ifOperStatus). Extendable to HOST-RESOURCES CPU/mem later.
- `[[outputs.sql]]`: `driver = "clickhouse"`,
  `data_source_name = "clickhouse://ch-server:9000/default"`,
  `table_template`/`init_sql` as needed for ClickHouse (MergeTree, `time` order key).
- Commented `[[inputs.gnmi]]` block: `addresses`, `port`, `username`/`password`
  or TLS, `[[inputs.gnmi.subscription]]` example paths.
- Generator: repurpose `gen-snmp-targets.sh` → emit Telegraf `agents = [...]` list
  (exporters table ∪ optional extra file), writing the gitignored
  `telegraf.d/agents.conf`.

### Standalone OTEL collector (`otel-collector`)
- Image: `otel/opentelemetry-collector-contrib:0.119.0`.
- Config `docker/otel/otel-collector.yaml` (committed): `receivers.otlp` (grpc
  4317, http 4318) → `exporters.clickhouse` (`tcp://ch-server:9000`, db `default`,
  `create_schema: true`, ttl 720h) → `service.pipelines` for logs/metrics/traces.
- Ports: 4317, 4318 (+ optional 13133 health). Drops HyperDX-specific
  fluentforward/opamp.

### Grafana (`grafana`)
- `docker/grafana/provisioning/datasources/clickhouse.yaml`: type
  `grafana-clickhouse-datasource`, host `ch-server`, port 9000, protocol native,
  username `default`, empty password, default database `default`.
- `docker/grafana/provisioning/dashboards/default.yaml`: file provider →
  `/var/lib/grafana/dashboards`.
- `docker/grafana/dashboards/snmp-interfaces.json`: time-series of interface
  in/out octet **rates** from `default.snmp`, grouped by interface name; plus a
  status/table panel. (Flows panel optional/deferred — Akvorado console already
  covers flow exploration.)
- Volume `grafana-data:/var/lib/grafana`.

### Removals
- Services `app`, `db`; volume dependency on mongo bind `.volumes/db` (mongo data
  is HyperDX metadata, intentionally abandoned).
- `.env`: prune now-unused HyperDX vars (APP_URL, FRONTEND_URL, APP_PORT,
  API_PORT, OPAMP_PORT, MONGO, DEFAULT_CONNECTIONS/SOURCES). Add
  `GRAFANA_ADMIN_PASSWORD` handling via secrets.

## Secrets (public-fork hygiene)
- `GRAFANA_ADMIN_PASSWORD` → gitignored `config/secrets/secrets.env` (+ `.example`
  placeholder). Loaded into the `grafana` container via `env_file`.
- `SNMP_COMMUNITY` reused (already gitignored).
- Committed configs use `${ENV}` refs / placeholders only.

## Safe sequencing (no visualization/collection gap)
1. Add `grafana` + ClickHouse datasource; verify it queries existing
   `default.flows` and `default.otel_metrics_*`.
2. Add `telegraf`; verify `default.snmp` fills with per-interface data; build the
   Grafana SNMP dashboard against it.
3. Remove the old `snmp-collector` (OTEL).
4. Swap `otel-collector` → standalone contrib; verify 4317 accepts OTLP and
   writes `default.otel_*`.
5. Remove HyperDX `app` + `db`; prune `.env`.
6. User adds cloudflared `grafana.example.com` ingress + DNS; retire
   `hyperdx.example.com`.

## Verification
- Grafana reachable (`:3000`, then `https://grafana.example.com`); ClickHouse
  datasource "Save & test" OK.
- `default.snmp` has fresh rows with `ifName` in {ether1..8, bridge, lo,
  sfp-sfpplus1}; dashboard shows per-interface series.
- Standalone collector: `:4317` open; a test OTLP metric lands in `default.otel_*`.
- Akvorado: `flows` count still increasing; console `:8081` and
  `akvorado.ieisi.org` OK.
- No secret in committed files (`git grep`); generated agent/config files
  gitignored.

## Risks
- **Telegraf ClickHouse output (`outputs.sql` + clickhouse driver)**: confirm the
  DSN/driver and table auto-creation against the pinned Telegraf version during
  build; adjust `init_sql`/engine if needed (MergeTree, `ORDER BY (time, ...)`).
- **Grafana ClickHouse datasource native vs http**: if native 9000 has issues,
  fall back to http 8123.
- **Removing mongo** drops HyperDX saved views/sources — intentional (abandoning
  HyperDX); no flow/SNMP data affected.
- **Access/DNS**: `grafana.example.com` needs the tunnel ingress + Cloudflare DNS
  (user action) before the public URL works; local `:3000` works meanwhile.

## Out of scope (for now)
- gNMI live onboarding (template only until gNMI-capable devices exist).
- Grafana Cloudflare Access / SSO (Grafana login used; can add later).
- Flows dashboards in Grafana (Akvorado console covers this; can add later).
- HOST-RESOURCES CPU/memory SNMP fields (easy Telegraf extension later).
