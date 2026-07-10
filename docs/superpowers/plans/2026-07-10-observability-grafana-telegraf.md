# Observability Pivot (Grafana CE + Telegraf) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax.

**Goal:** Replace HyperDX with Grafana CE, replace the alpha OTEL SNMP receiver with Telegraf (→ ClickHouse `default.snmp`), and swap the OpAMP-coupled HyperDX otel-collector for a standalone contrib collector — all on the live combined stack, without losing flow data or disrupting Akvorado.

**Architecture:** Telegraf `inputs.snmp` → `outputs.sql`(clickhouse) → `default.snmp`; standalone `otelcol-contrib` OTLP→`default.otel_*`; Grafana CE with `grafana-clickhouse-datasource` reads `default.*`. Single query surface = ClickHouse `default`.

**Tech Stack:** Docker Compose, ClickHouse 25.8, Telegraf, otelcol-contrib, Grafana OSS, Cloudflare Tunnel.

## Global Constraints

- Do NOT touch ClickHouse flow data or any Akvorado service. Never `docker compose down -v` / `docker volume rm` on `akvorado_*` volumes.
- Shared ClickHouse reached as `ch-server` (alias) — native `9000`, http `8123`, user `default`, empty password, database `default`.
- Public-fork hygiene: no secrets in committed files. `SNMP_COMMUNITY` and new `GRAFANA_ADMIN_PASSWORD` live only in gitignored `config/secrets/secrets.env`; committed configs use `${ENV}`/placeholders. Generated `telegraf.d/agents.conf` is gitignored.
- Safe order: add new (Grafana, Telegraf) and verify BEFORE removing old (snmp-collector, HyperDX). Something works at every step.
- Live stack is running under compose project `akvorado`. Use `docker compose` from `/home/terry/ClickStack`. Read-only docker for verification; no destructive ops outside the explicit removal steps.
- Branch `akvorado`. Commit after each task. (History scrub + fork/push remain the separate, held Task 7 of the merge plan.)
- Pinned images: `telegraf:1.32-alpine`, `grafana/grafana-oss:11.3.1`, `otel/opentelemetry-collector-contrib:0.119.0`.

---

## File Structure

- `config/secrets/secrets.env(.example)` — add `GRAFANA_ADMIN_PASSWORD`.
- `.gitignore` — add `docker/telegraf/telegraf.d/agents.conf`.
- `docker/grafana/provisioning/datasources/clickhouse.yaml` — ClickHouse datasource (committed).
- `docker/grafana/provisioning/dashboards/default.yaml` — dashboard provider (committed).
- `docker/grafana/dashboards/snmp-interfaces.json` — starter dashboard (committed).
- `docker/telegraf/telegraf.conf` — Telegraf base config: snmp + commented gnmi + outputs.sql (committed).
- `docker/telegraf/telegraf.d/agents.conf` — generated `agents=[...]` (gitignored).
- `docker/telegraf/gen-telegraf-agents.sh` — target generator (committed; replaces gen-snmp-targets.sh role).
- `docker/otel/otel-collector.yaml` — standalone collector config (committed).
- `docker-compose.yml` — add `grafana`, `telegraf`; replace `otel-collector`; remove `snmp-collector`, `app`, `db`.
- `.env` — prune HyperDX vars.
- Remove: `docker/otel/snmp-collector.example.yaml`, `docker/otel/gen-snmp-targets.sh`, `docker/otel/snmp-collector.yaml` (gitignored, on disk).

---

## Task 1: Secrets + gitignore for Grafana/Telegraf

**Files:** Modify `.gitignore`, `config/secrets/secrets.env`, `config/secrets/secrets.env.example`.

- [ ] **Step 1: Add the gitignore rule for the generated Telegraf agents file**

Append to `.gitignore`:
```gitignore
docker/telegraf/telegraf.d/agents.conf
```

- [ ] **Step 2: Add the Grafana admin password to the committed example**

Append to `config/secrets/secrets.env.example` (named exactly as Grafana's env var so it flows straight through `env_file`):
```dotenv
# Grafana admin password (grafana service reads GF_SECURITY_ADMIN_PASSWORD directly).
GF_SECURITY_ADMIN_PASSWORD=changeme
```

- [ ] **Step 3: Add the real value to the gitignored secrets file**

Append to `config/secrets/secrets.env` (NOT committed) — ask the user for the value; do not invent one. Placeholder line to be replaced by the user:
```dotenv
GF_SECURITY_ADMIN_PASSWORD=<set by user>
```

- [ ] **Step 4: Verify**
```bash
git check-ignore docker/telegraf/telegraf.d/agents.conf config/secrets/secrets.env && echo IGNORED_OK
grep -q GF_SECURITY_ADMIN_PASSWORD config/secrets/secrets.env.example && echo EXAMPLE_OK
```
Expected: `IGNORED_OK`, `EXAMPLE_OK`.

- [ ] **Step 5: Commit (example + .gitignore only)**
```bash
git add .gitignore config/secrets/secrets.env.example
git commit -m "chore: gitignore telegraf agents + grafana admin password template"
```

---

## Task 2: Grafana service + ClickHouse datasource + dashboard scaffolding

**Files:** Create `docker/grafana/provisioning/datasources/clickhouse.yaml`, `docker/grafana/provisioning/dashboards/default.yaml`, `docker/grafana/dashboards/snmp-interfaces.json`; Modify `docker-compose.yml`.

**Interfaces:** Produces the `grafana` service on `127.0.0.1:3000` reading ClickHouse `default`.

- [ ] **Step 1: ClickHouse datasource provisioning**

Create `docker/grafana/provisioning/datasources/clickhouse.yaml`:
```yaml
apiVersion: 1
datasources:
  - name: ClickHouse
    uid: clickhouse
    type: grafana-clickhouse-datasource
    access: proxy
    isDefault: true
    jsonData:
      host: ch-server
      port: 9000
      protocol: native
      secure: false
      username: default
      defaultDatabase: default
    secureJsonData:
      password: ""
```

- [ ] **Step 2: Dashboard provider**

Create `docker/grafana/provisioning/dashboards/default.yaml`:
```yaml
apiVersion: 1
providers:
  - name: default
    orgId: 1
    folder: ""
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    allowUiUpdates: true
    options:
      path: /var/lib/grafana/dashboards
      foldersFromFilesStructure: false
```

- [ ] **Step 3: Starter SNMP dashboard**

Create `docker/grafana/dashboards/snmp-interfaces.json` (per-interface counters; refine to bps later). This is a minimal valid dashboard with one ClickHouse time-series panel grouped by `ifName`:
```json
{
  "uid": "snmp-interfaces",
  "title": "SNMP Interfaces",
  "tags": ["snmp"],
  "timezone": "browser",
  "schemaVersion": 39,
  "version": 1,
  "time": {"from": "now-6h", "to": "now"},
  "panels": [
    {
      "id": 1,
      "type": "timeseries",
      "title": "Interface in-octets by interface",
      "gridPos": {"h": 10, "w": 24, "x": 0, "y": 0},
      "datasource": {"type": "grafana-clickhouse-datasource", "uid": "clickhouse"},
      "fieldConfig": {"defaults": {"custom": {"drawStyle": "line"}, "unit": "bytes"}, "overrides": []},
      "targets": [
        {
          "refId": "A",
          "datasource": {"type": "grafana-clickhouse-datasource", "uid": "clickhouse"},
          "editorType": "sql",
          "rawSql": "SELECT $__timeInterval(time) AS time, ifName, max(ifHCInOctets) AS in_octets FROM default.snmp WHERE $__timeFilter(time) GROUP BY time, ifName ORDER BY time",
          "format": 1
        }
      ]
    }
  ]
}
```

- [ ] **Step 4: Add the `grafana` service to `docker-compose.yml`** (before `snmp-collector`):
```yaml
  grafana:
    image: grafana/grafana-oss:11.3.1
    restart: unless-stopped
    env_file:
      - config/secrets/secrets.env
    environment:
      GF_SERVER_ROOT_URL: https://grafana.example.com
      GF_INSTALL_PLUGINS: grafana-clickhouse-datasource
      GF_SECURITY_ADMIN_USER: admin
      GF_USERS_ALLOW_SIGN_UP: "false"
    ports:
      - 127.0.0.1:3000:3000
    volumes:
      - grafana-data:/var/lib/grafana
      - ./docker/grafana/provisioning:/etc/grafana/provisioning:ro
      - ./docker/grafana/dashboards:/var/lib/grafana/dashboards:ro
    depends_on:
      - clickhouse
```
Add `grafana-data:` under the top-level `volumes:` block.

Note: `GF_SECURITY_ADMIN_PASSWORD` is supplied to the container via `env_file: config/secrets/secrets.env` (Task 1) — Grafana reads that env var directly, so it never appears in the committed compose or `.env`.

- [ ] **Step 5: Validate + bring up Grafana**
```bash
docker compose config >/dev/null && echo COMPOSE_OK
docker compose up -d grafana
sleep 25
docker compose logs --tail=20 grafana | grep -iE "plugin|clickhouse|http server|error" | tail
```
Expected: `COMPOSE_OK`; logs show the clickhouse plugin installed and HTTP server started.

- [ ] **Step 6: Verify Grafana up + datasource queries existing data**
```bash
curl -s -o /dev/null -w "grafana :3000 -> %{http_code}\n" http://127.0.0.1:3000/api/health
# datasource health via Grafana API (admin:password from secrets)
PW=$(grep -E '^GF_SECURITY_ADMIN_PASSWORD=' config/secrets/secrets.env | cut -d= -f2-)
curl -s -u "admin:$PW" "http://127.0.0.1:3000/api/datasources/uid/clickhouse/health" ; echo
```
Expected: health `200`; datasource health JSON `"status":"OK"`.

- [ ] **Step 7: Commit**
```bash
git add docker/grafana docker-compose.yml
git commit -m "feat: add Grafana CE + ClickHouse datasource + SNMP dashboard scaffold"
```

---

## Task 3: Telegraf (SNMP → ClickHouse default.snmp) + agent generator

**Files:** Create `docker/telegraf/telegraf.conf`, `docker/telegraf/gen-telegraf-agents.sh`; Modify `docker-compose.yml`.

**Interfaces:** Produces `telegraf` writing `default.snmp`; consumes `default.exporters` for agents; `SNMP_COMMUNITY` from secrets env.

- [ ] **Step 1: Telegraf base config**

Create `docker/telegraf/telegraf.conf` (numeric OIDs → no MIB files needed):
```toml
[agent]
  interval = "60s"
  round_interval = true
  flush_interval = "60s"
  omit_hostname = true

# Real agent list is generated into telegraf.d/agents.conf (gitignored) and
# merged via --config-directory. This base file defines fields/output.

[[inputs.snmp]]
  ## agents = [...] is supplied by telegraf.d/agents.conf (generated)
  version = 2
  community = "${SNMP_COMMUNITY}"
  agent_host_tag = "source"
  timeout = "5s"
  retries = 1
  [[inputs.snmp.table]]
    name = "snmp"
    [[inputs.snmp.table.field]]
      oid = "1.3.6.1.2.1.31.1.1.1.1"
      name = "ifName"
      is_tag = true
    [[inputs.snmp.table.field]]
      oid = "1.3.6.1.2.1.2.2.1.1"
      name = "ifIndex"
      is_tag = true
    [[inputs.snmp.table.field]]
      oid = "1.3.6.1.2.1.31.1.1.1.6"
      name = "ifHCInOctets"
    [[inputs.snmp.table.field]]
      oid = "1.3.6.1.2.1.31.1.1.1.10"
      name = "ifHCOutOctets"
    [[inputs.snmp.table.field]]
      oid = "1.3.6.1.2.1.2.2.1.14"
      name = "ifInErrors"
    [[inputs.snmp.table.field]]
      oid = "1.3.6.1.2.1.2.2.1.20"
      name = "ifOutErrors"
    [[inputs.snmp.table.field]]
      oid = "1.3.6.1.2.1.2.2.1.8"
      name = "ifOperStatus"

# ---- gNMI streaming telemetry (TEMPLATE — enable when you have gNMI-capable gear) ----
# [[inputs.gnmi]]
#   addresses = ["10.0.0.1:57400"]
#   username = "${GNMI_USERNAME}"
#   password = "${GNMI_PASSWORD}"
#   # enable_tls = true
#   # tls_ca = "/etc/telegraf/certs/ca.pem"
#   [[inputs.gnmi.subscription]]
#     name = "ifcounters"
#     origin = "openconfig"
#     path = "/interfaces/interface/state/counters"
#     subscription_mode = "sample"
#     sample_interval = "10s"

[[outputs.sql]]
  driver = "clickhouse"
  data_source_name = "clickhouse://default:@ch-server:9000/default"
  ## ClickHouse needs an explicit engine — the plugin default lacks one.
  table_template = "CREATE TABLE {TABLE}({COLUMNS}) ENGINE = MergeTree() ORDER BY ({TAG_COLUMN_NAMES}, {TIMESTAMP_COLUMN_NAME})"
  [outputs.sql.convert]
    conversion_style = "literal"
    integer   = "Int64"
    unsigned  = "UInt64"
    real      = "Float64"
    text      = "String"
    bool      = "UInt8"
    timestamp = "DateTime"
    defaultvalue = "String"
```

- [ ] **Step 2: Agent generator**

Create `docker/telegraf/gen-telegraf-agents.sh` (exporters table ∪ optional extra file → `agents = [...]`):
```bash
#!/usr/bin/env bash
set -euo pipefail
CH_CONTAINER="${CH_CONTAINER:-akvorado-clickhouse-1}"
EXTRA_FILE="${EXTRA_FILE:-$(dirname "$0")/snmp-extra-targets.txt}"
OUT="$(dirname "$0")/telegraf.d/agents.conf"
mkdir -p "$(dirname "$OUT")"

mapfile -t ADDRS < <(docker exec -i "$CH_CONTAINER" clickhouse-client -q \
  "SELECT DISTINCT replaceRegexpOne(toString(ExporterAddress),'^::ffff:','') FROM default.exporters ORDER BY 1" 2>/dev/null || true)
if [ -f "$EXTRA_FILE" ]; then
  while IFS= read -r line; do line="${line%%#*}"; line="$(echo "$line" | tr -d '[:space:]')"; [ -n "$line" ] && ADDRS+=("$line"); done < "$EXTRA_FILE"
fi
mapfile -t ADDRS < <(printf '%s\n' "${ADDRS[@]}" | sed '/^$/d' | sort -u)
if [ "${#ADDRS[@]}" -eq 0 ]; then echo "No SNMP targets found." >&2; exit 1; fi

{
  echo "# GENERATED by gen-telegraf-agents.sh — do not edit; do not commit."
  echo "[[inputs.snmp]]"
  printf "  agents = ["
  printf '%s' "$(printf '"udp://%s:161",' "${ADDRS[@]}" | sed 's/,$//')"
  echo "]"
} > "$OUT"
echo "Wrote $OUT with ${#ADDRS[@]} agent(s): ${ADDRS[*]}"
```
Note: both the base `telegraf.conf` and `agents.conf` define `[[inputs.snmp]]`; Telegraf merges configs from a directory, and the `agents` array from `agents.conf` augments the snmp input. If the running Telegraf version does NOT merge two `[[inputs.snmp]]` blocks into one, fall back to putting the full `[[inputs.snmp]]` (including fields) in the generated file and leave only `[agent]`/`[[outputs.sql]]` in the base — verify in Step 4 and adjust.

- [ ] **Step 3: Add the `telegraf` service to `docker-compose.yml`**:
```yaml
  telegraf:
    image: telegraf:1.32-alpine
    restart: unless-stopped
    env_file:
      - config/secrets/secrets.env
    command: ["telegraf", "--config", "/etc/telegraf/telegraf.conf", "--config-directory", "/etc/telegraf/telegraf.d"]
    volumes:
      - ./docker/telegraf/telegraf.conf:/etc/telegraf/telegraf.conf:ro
      - ./docker/telegraf/telegraf.d:/etc/telegraf/telegraf.d:ro
    depends_on:
      - clickhouse
```

- [ ] **Step 4: Generate agents, validate config, bring up**
```bash
chmod +x docker/telegraf/gen-telegraf-agents.sh
./docker/telegraf/gen-telegraf-agents.sh          # expect: 192.168.88.1
docker compose config >/dev/null && echo COMPOSE_OK
docker compose up -d telegraf
sleep 70
docker compose logs --tail=30 telegraf | grep -iE "error|snmp|clickhouse|written|E!|W!" | tail -15
```
Expected: agents file written; `COMPOSE_OK`; telegraf logs show no fatal errors (a first-scrape timeout may appear then recover).

- [ ] **Step 5: Verify `default.snmp` fills with per-interface rows**
```bash
docker exec akvorado-clickhouse-1 clickhouse-client -q "SHOW TABLES FROM default LIKE 'snmp'"
docker exec akvorado-clickhouse-1 clickhouse-client -q \
  "SELECT ifName, ifIndex, max(ifHCInOctets) FROM default.snmp WHERE time > now() - INTERVAL 5 MINUTE GROUP BY ifName, ifIndex ORDER BY ifName"
```
Expected: table `snmp` exists; rows for `ether1..8`, `bridge`, `lo`, `sfp-sfpplus1`. If the table wasn't created, apply the Step-2 fallback (full snmp input in generated file) or add `ENGINE` to `table_template` (already present) and check telegraf logs for the exact SQL error.

- [ ] **Step 6: Commit (generated agents.conf stays gitignored)**
```bash
git add docker/telegraf/telegraf.conf docker/telegraf/gen-telegraf-agents.sh docker-compose.yml
git status   # confirm docker/telegraf/telegraf.d/agents.conf NOT staged
git commit -m "feat: Telegraf SNMP -> ClickHouse default.snmp (+ gNMI template, agent generator)"
```

---

## Task 4: Point the dashboard at real data; verify per-interface breakdown

**Files:** none (data verification; dashboard already provisioned in Task 2).

- [ ] **Step 1: Reload Grafana dashboards (provisioning picks up changes automatically; force if needed)**
```bash
docker compose restart grafana ; sleep 15
```

- [ ] **Step 2: Verify the dashboard's query returns per-interface series (mirrors the panel SQL)**
```bash
docker exec akvorado-clickhouse-1 clickhouse-client -q \
  "SELECT ifName, count() pts, max(ifHCInOctets) FROM default.snmp WHERE time > now() - INTERVAL 15 MINUTE GROUP BY ifName ORDER BY ifName"
```
Expected: one row per interface with growing counters — i.e., the timeseries panel will render one line per interface.

- [ ] **Step 3: (If browser available) confirm in UI** — open `http://127.0.0.1:3000` (or `https://grafana.example.com`), dashboard **SNMP Interfaces**, panel shows one series per `ifName`. Otherwise the Step-2 query is the accepted evidence.

No commit (no file changes).

---

## Task 5: Remove the old OTEL snmp-collector

**Files:** Modify `docker-compose.yml`; delete `docker/otel/snmp-collector.example.yaml`, `docker/otel/gen-snmp-targets.sh`; remove gitignored `docker/otel/snmp-collector.yaml`.

- [ ] **Step 1: Confirm Telegraf is the live SNMP source (data fresh) before removing the old collector**
```bash
docker exec akvorado-clickhouse-1 clickhouse-client -q \
  "SELECT max(time) FROM default.snmp"
```
Expected: within the last ~2 minutes.

- [ ] **Step 2: Stop + remove the snmp-collector service**
```bash
docker compose stop snmp-collector
docker compose rm -f snmp-collector
```

- [ ] **Step 3: Delete the snmp-collector service block from `docker-compose.yml`** (the `snmp-collector:` service and its `depends_on`/volumes lines).

- [ ] **Step 4: Remove the obsolete OTEL SNMP files**
```bash
git rm docker/otel/snmp-collector.example.yaml docker/otel/gen-snmp-targets.sh
rm -f docker/otel/snmp-collector.yaml   # gitignored, on disk
# keep docker/otel/snmp-extra-targets.example.txt (still used by telegraf generator) — leave it
```
Move the extra-targets example to telegraf if desired: `git mv docker/otel/snmp-extra-targets.example.txt docker/telegraf/snmp-extra-targets.example.txt` and update the generator's `EXTRA_FILE` default path if you do (already points at `$(dirname)/snmp-extra-targets.txt`, i.e. `docker/telegraf/`).

- [ ] **Step 5: Validate + verify SNMP still flowing (now only Telegraf)**
```bash
docker compose config >/dev/null && echo COMPOSE_OK
docker compose ps | grep -q snmp-collector && echo "STILL PRESENT" || echo "REMOVED_OK"
sleep 65
docker exec akvorado-clickhouse-1 clickhouse-client -q "SELECT max(time) FROM default.snmp"
```
Expected: `COMPOSE_OK`, `REMOVED_OK`, snmp `max(time)` still fresh.

- [ ] **Step 6: Commit**
```bash
git add -A docker/otel docker/telegraf docker-compose.yml
git commit -m "refactor: remove alpha OTEL snmp-collector (Telegraf is now the SNMP source)"
```

---

## Task 6: Swap HyperDX otel-collector → standalone otelcol-contrib

**Files:** Create `docker/otel/otel-collector.yaml`; Modify `docker-compose.yml`.

**Interfaces:** Produces OTLP ingestion on 4317/4318 → `default.otel_*`, decoupled from HyperDX.

- [ ] **Step 1: Standalone collector config**

Create `docker/otel/otel-collector.yaml`:
```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
processors:
  batch: {}
exporters:
  clickhouse:
    endpoint: tcp://ch-server:9000?dial_timeout=10s
    database: default
    username: default
    password: ""
    create_schema: true
    ttl: 720h
extensions:
  health_check:
    endpoint: 0.0.0.0:13133
service:
  extensions: [health_check]
  pipelines:
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [clickhouse]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [clickhouse]
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [clickhouse]
```

- [ ] **Step 2: Replace the `otel-collector` service in `docker-compose.yml`** with:
```yaml
  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.119.0
    restart: unless-stopped
    command: ["--config=/etc/otelcol/config.yaml"]
    volumes:
      - ./docker/otel/otel-collector.yaml:/etc/otelcol/config.yaml:ro
    ports:
      - "4317:4317"
      - "4318:4318"
      - "13133:13133"
    depends_on:
      - clickhouse
```
Remove the old HyperDX otel-collector env (CLICKHOUSE_ENDPOINT/OPAMP_SERVER_URL/etc.) and its `24225`/`8888` port lines unless you still want them (drop them).

- [ ] **Step 3: Validate + recreate**
```bash
docker run --rm -v "$PWD/docker/otel/otel-collector.yaml:/c.yaml:ro" otel/opentelemetry-collector-contrib:0.119.0 validate --config=/c.yaml && echo VALIDATE_OK
docker compose up -d --force-recreate otel-collector
sleep 15
docker compose logs --tail=20 otel-collector | grep -iE "started|error|4317" | tail
```
Expected: `VALIDATE_OK`; "Everything is ready" in logs; no OpAMP errors (that dependency is gone).

- [ ] **Step 4: Verify OTLP ingestion works end-to-end**
```bash
# is 4317 listening now?
docker exec akvorado-otel-collector-1 sh -c 'for p in 4317 4318; do (echo > /dev/tcp/127.0.0.1/$p) 2>/dev/null && echo "$p OPEN" || echo "$p CLOSED"; done' 2>&1 || \
  ss -ltn | grep -E ':4317|:4318'
```
Expected: 4317/4318 OPEN (the HyperDX collector left them CLOSED — this proves the fix).

- [ ] **Step 5: Commit**
```bash
git add docker/otel/otel-collector.yaml docker-compose.yml
git commit -m "feat: standalone otelcol-contrib for OTLP ingestion (drop OpAMP/app coupling)"
```

---

## Task 7: Remove HyperDX (app + mongo) and prune .env

**Files:** Modify `docker-compose.yml`, `.env`.

- [ ] **Step 1: Stop + remove the HyperDX services**
```bash
docker compose stop app db
docker compose rm -f app db
```
(Mongo data in `.volumes/db` is HyperDX metadata — intentionally abandoned. Do NOT remove any `akvorado_*` volume.)

- [ ] **Step 2: Delete the `app` and `db` service blocks from `docker-compose.yml`.** Also drop any remaining references (none should point at them; `otel-collector` no longer references `app`).

- [ ] **Step 3: Prune now-unused HyperDX vars from `.env`** — remove: `HDX_IMAGE_REPO`, `IMAGE_NAME_DOCKERHUB`, `NEXT_OTEL_COLLECTOR_IMAGE_NAME_DOCKERHUB`, `OTEL_COLLECTOR_IMAGE_NAME_DOCKERHUB`, `IMAGE_VERSION`, `HYPERDX_API_PORT`, `HYPERDX_APP_PORT`, `HYPERDX_APP_URL`, `HYPERDX_FRONTEND_URL`, `HYPERDX_LOG_LEVEL`, `HYPERDX_OPAMP_PORT`, `HYPERDX_BASE_PATH`, `HYPERDX_OTEL_EXPORTER_CLICKHOUSE_DATABASE`, `CH_IMAGE_REPO`, `CODE_VERSION`. Keep `COMPOSE_PROJECT_NAME=akvorado`.

- [ ] **Step 4: Validate + confirm final service set**
```bash
docker compose config >/dev/null && echo COMPOSE_OK
docker compose config --services | sort | tr '\n' ' '; echo
```
Expected: `COMPOSE_OK`; services = akvorado-* , clickhouse, grafana, kafka, kafka-ui, otel-collector, redis, telegraf, traefik (NO app/db/snmp-collector).

- [ ] **Step 5: Verify the whole stack healthy + data intact**
```bash
docker compose ps --format '{{.Name}}\t{{.Status}}' | sort
docker exec akvorado-clickhouse-1 clickhouse-client -q "SELECT count() FROM default.flows"   # still ~40M, growing
docker exec akvorado-clickhouse-1 clickhouse-client -q "SELECT max(time) FROM default.snmp"  # fresh
curl -s -o /dev/null -w "grafana -> %{http_code}\n" http://127.0.0.1:3000/api/health
curl -s -o /dev/null -w "akvorado console -> %{http_code}\n" http://127.0.0.1:8081/
```
Expected: all up; flows preserved + growing; SNMP fresh; Grafana 200; console 200.

- [ ] **Step 6: Commit**
```bash
git add docker-compose.yml .env
git commit -m "refactor: retire HyperDX (remove app+mongo); Grafana is the UI"
```

---

## Task 8: Expose grafana.example.com (user infra) + verify

**Files:** none in-repo (user edits `/etc/cloudflared/config.yml` + Cloudflare DNS).

- [ ] **Step 1: Add the tunnel ingress** — user adds to `/etc/cloudflared/config.yml` ingress (above the `http_status:404` catch-all):
```yaml
  - hostname: grafana.example.com
    service: http://localhost:3000
```
Then: create the Cloudflare DNS route for `grafana.example.com` (Cloudflare dashboard or `cloudflared tunnel route dns <tunnel> grafana.example.com`) and `sudo systemctl restart cloudflared` (or the tunnel's run unit).

- [ ] **Step 2: Verify end-to-end**
```bash
curl -s -o /dev/null -w "https://grafana.example.com -> %{http_code}\n" --max-time 20 https://grafana.example.com/api/health
```
Expected: `200`. Log into `https://grafana.example.com` (admin / the secret password), open **SNMP Interfaces**.

- [ ] **Step 3: (optional) Retire the old hyperdx.example.com ingress** — remove that ingress line from `/etc/cloudflared/config.yml` and its DNS record; restart cloudflared.

---

## Self-Review

**Spec coverage:**
- Remove HyperDX app+mongo → Task 7. ✅
- Standalone otelcol-contrib (OTLP) → Task 6. ✅
- Telegraf inputs.snmp active + gnmi template → outputs.sql/default.snmp → Task 3. ✅
- Exporters-table target discovery feeding Telegraf → Task 3 generator. ✅
- Grafana CE + clickhouse datasource + SNMP dashboard → Tasks 2, 4. ✅
- grafana.example.com via cloudflared → Task 8. ✅
- Secrets hygiene (GRAFANA admin pw, gitignored agents) → Task 1. ✅
- Safe sequencing (add+verify before remove) → Task order 2,3,4 (add) → 5,6,7 (remove/replace). ✅
- Data safety (no akvorado volume touched) → Global Constraints + Task 7 note. ✅

**Placeholder scan:** Real config throughout; `<set by user>` in Task 1 Step 3 is deliberate (secret entered by user, per rules); dashboard JSON is a complete minimal panel.

**Type/name consistency:** service names (`grafana`, `telegraf`, `otel-collector`, `clickhouse`), datasource uid `clickhouse`, table `default.snmp`, tags `ifName`/`ifIndex`, secret `GF_SECURITY_ADMIN_PASSWORD` (per Task 2 Step 4 simplification — update Task 1 to name it `GF_SECURITY_ADMIN_PASSWORD`), and `ch-server` alias are used consistently.

**Known build-time risks (flagged inline):** Telegraf two-`[[inputs.snmp]]`-block merge (Task 3 Step 2 fallback); ClickHouse `table_template` engine (handled); Grafana datasource native vs http (spec risk).
