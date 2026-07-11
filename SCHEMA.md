# Data Schema — finding data in ClickHouse with Grafana

All data lives in the single ClickHouse database **`default`**, reached inside the
stack as **`ch-server`** (native `9000`, HTTP `8123`, user `default`, no password).
Grafana ships with one provisioned datasource, **ClickHouse** (uid `clickhouse`),
that points at it — every query below runs against that datasource.

## Tables at a glance

| Table | Written by | Engine | What it holds |
|---|---|---|---|
| `flows` | Akvorado outlet | MergeTree | Raw NetFlow/IPFIX/sFlow records (one row per flow). **15-day TTL.** |
| `flows_1m0s` / `flows_5m0s` / `flows_1h0m0s` | Akvorado | SummingMergeTree | Pre-aggregated rollups of `flows` for wide time ranges. Longer retention. |
| `exporters` | Akvorado | ReplacingMergeTree | Per-exporter + per-interface metadata (name, role, site, speed…). |
| `snmp` | Telegraf | MergeTree | SNMP interface counters (one row per interface per poll). |
| `otel_metrics_sum` / `_gauge` / `_histogram` … | otel-collector | MergeTree | OTLP metrics — only populated if apps send OTLP to `:4317/:4318`. |
| `otel_logs` / `otel_traces` | otel-collector | MergeTree | OTLP logs/traces (empty unless apps send them). |
| `asns`, `networks`, `protocols`, `tcp`, `udp`, `icmp` | Akvorado | Dictionary | Enrichment lookups (e.g. `dictGet('protocols', 'name', Proto)`). |

> **Prefer the Akvorado console** (Traefik `:8081`, or your tunnel hostname) for
> interactive flow exploration — it's purpose-built. Use Grafana/ClickHouse when you want custom
> panels, to join flows with SNMP, or to build alerting.

## `flows` — network flow records

Key columns (full list is ~60; these are the ones you'll query most):

- **Time:** `TimeReceived` (DateTime) — the time column for `$__timeFilter`.
- **Volume:** `Bytes`, `Packets` (UInt64). **`SamplingRate` (UInt64)** — flows are
  sampled; multiply by it for real volume: `sum(Bytes * SamplingRate)`.
- **Exporter (the router):** `ExporterAddress` (IPv6), `ExporterName`, `ExporterGroup`,
  `ExporterRole`, `ExporterSite`, `ExporterRegion`.
- **Source / Dest addressing:** `SrcAddr` / `DstAddr` (IPv6 — IPv4 is v4-mapped),
  `SrcPort` / `DstPort` (UInt16), `Proto` (UInt32, IP proto number), `EType` (ethertype).
- **Routing / ASN:** `SrcAS` / `DstAS` (UInt32), `DstASPath` (Array), `Dst1stAS`…`Dst3rdAS`.
- **Geo/network naming (enriched):** `SrcCountry` / `DstCountry` (2-char), `SrcNetName` /
  `DstNetName`, `SrcNetRole`, `SrcGeoCity`, etc.
- **Interfaces:** `InIfName` / `OutIfName`, `InIfDescription` / `OutIfDescription`,
  `InIfSpeed` / `OutIfSpeed`, `InIfBoundary` / `OutIfBoundary` (`external`/`internal`).

Proto/port numbers are human-readable via dictionaries:
`dictGet('protocols', 'name', Proto)` → e.g. `TCP`.

### Example: top talkers (by real bytes)
```sql
SELECT SrcAddr, sum(Bytes * SamplingRate) AS bytes
FROM default.flows
WHERE $__timeFilter(TimeReceived)
GROUP BY SrcAddr
ORDER BY bytes DESC
LIMIT 20
```

### Example: traffic bitrate over time, by destination country
```sql
SELECT $__timeInterval(TimeReceived) AS time,
       DstCountry,
       sum(Bytes * SamplingRate) * 8 AS bits
FROM default.flows
WHERE $__timeFilter(TimeReceived)
GROUP BY time, DstCountry
ORDER BY time
```
(Divide `bits` by your bucket size for bps, or use a rollup table below.)

### Example: top applications (by dest port + proto)
```sql
SELECT dictGet('protocols','name',Proto) AS proto, DstPort,
       sum(Bytes * SamplingRate) AS bytes
FROM default.flows
WHERE $__timeFilter(TimeReceived) AND InIfBoundary = 'external'
GROUP BY proto, DstPort ORDER BY bytes DESC LIMIT 20
```

## Rollup tables — use these for wide time ranges

`flows_1m0s`, `flows_5m0s`, `flows_1h0m0s` are `SummingMergeTree` aggregates with the
same dimension columns as `flows` (minus per-flow fields like `SrcAddr`/`SrcPort`).
For a dashboard spanning days, query `flows_1h0m0s`; for hours, `flows_5m0s`. Same
`TimeReceived` time column and `Bytes`/`Packets`/`SamplingRate` semantics.

## `snmp` — Telegraf interface counters

Columns: `time` (DateTime), `source` (agent IP, tag), `ifName` (tag, e.g. `ether1`),
`ifIndex` (tag), `ifHCInOctets` / `ifHCOutOctets` (UInt64 **counters**),
`ifInErrors` / `ifOutErrors` (UInt64 counters), `ifOperStatus` (Int64; 1=up, 2=down).
`ifAlias` (String) is the interface description / RouterOS comment (e.g. "TRANSIT: NBN", "Internet Gateway") — added as an SNMP tag; use `argMax(ifAlias, time)` to get the current value.

Octet columns are **monotonic counters** — compute a rate (delta ÷ time) for bandwidth.

### Example: per-interface inbound bitrate (bps)
Use a window-function delta between consecutive polls — NOT `(max-min)` per
`$__timeInterval` bucket, which returns 0 (a flat line) whenever a bucket holds a
single poll (SNMP is polled every 60s):
```sql
SELECT time, iface, in_bps FROM (
  SELECT time, concat(source, ' / ', ifName) AS iface,
    (ifHCInOctets - lagInFrame(ifHCInOctets) OVER (PARTITION BY source, ifName ORDER BY time))
      / greatest(toUnixTimestamp(time) - toUnixTimestamp(lagInFrame(time)
        OVER (PARTITION BY source, ifName ORDER BY time)), 1) * 8 AS in_bps
  FROM default.snmp
  WHERE $__timeFilter(time)
) WHERE in_bps >= 0 ORDER BY time
```

### Example: interface up/down status
```sql
SELECT ifName, argMax(ifOperStatus, time) AS status  -- 1=up, 2=down
FROM default.snmp
WHERE $__timeFilter(time)
GROUP BY ifName ORDER BY ifName
```

SNMP poll targets are generated from the `exporters` table into a gitignored
`docker/telegraf/telegraf.d/agents.conf` by `docker/telegraf/gen-telegraf-agents.sh`
(re-run it when you add devices; add non-flow devices to `docker/telegraf/snmp-extra-targets.txt`).

## Grafana query cheatsheet

The ClickHouse datasource supports these macros (use raw-SQL editor):

- `$__timeFilter(col)` → `col >= <from> AND col <= <to>`
- `$__timeInterval(col)` → rounds `col` to the panel's bucket (for time series)
- `$__fromTime` / `$__toTime` → the dashboard range bounds

Time-series panels expect a column named/aliased `time` first, then a series label
column (e.g. `ifName`, `DstCountry`), then value column(s).

Provisioned dashboard **SNMP Interfaces** (`docker/grafana/dashboards/snmp-interfaces.json`)
is a working starting point — clone its panels and adapt the SQL above.
