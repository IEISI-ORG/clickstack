# Host metrics via a native OpenTelemetry Collector

The in-stack `otel-collector` container receives OTLP on `4317`/`4318` and writes to
ClickHouse, but nothing sends the **host's own** metrics (CPU, memory, disk, network).
Collecting those from *inside* a container would need `/proc` and `/sys` bind-mounted;
this deployment intentionally does not do that.

Instead, run a **native** OpenTelemetry Collector on the host. It reads the real
system directly and forwards OTLP to the containerized collector on `localhost:4317`
(published by Docker). No `/proc`/`/sys` mounts, and no ClickHouse credentials on the
host.

```
host: otelcol-contrib (hostmetrics) --OTLP--> localhost:4317
                                                   │
                                                   ▼
                              [container] otel-collector --clickhouse--> default.otel_metrics_*
                                                                              │
                                                                    Grafana (system.* panels)
```

> This runs **outside** Docker Compose, so it is documentation only — not a service in
> `docker-compose.yml`.

## 1. Install (Debian/Ubuntu)

Match the version to the in-stack collector (`docker-compose.yml` pins
`otel/opentelemetry-collector-contrib:0.119.0`) to avoid metric-schema drift:

```bash
VER=0.119.0
curl -fsSLo /tmp/otelcol-contrib.deb \
  "https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v${VER}/otelcol-contrib_${VER}_linux_amd64.deb"
sudo dpkg -i /tmp/otelcol-contrib.deb   # installs the binary + a systemd unit: otelcol-contrib
```

## 2. Configure — `/etc/otelcol-contrib/config.yaml`

```yaml
receivers:
  hostmetrics:
    collection_interval: 30s
    root_path: /
    scrapers:
      cpu: {}
      load: {}
      memory: {}
      disk: {}
      filesystem: {}
      network: {}
      paging: {}
      processes: {}

processors:
  resourcedetection:
    detectors: [system]
    system:
      hostname_sources: [os]
  batch: {}

exporters:
  # Forward to the containerized collector (published on the host loopback).
  otlp:
    endpoint: localhost:4317
    tls:
      insecure: true

service:
  pipelines:
    metrics:
      receivers: [hostmetrics]
      processors: [resourcedetection, batch]
      exporters: [otlp]
  telemetry:
    logs:
      level: info
```

Start and enable it:

```bash
sudo systemctl restart otelcol-contrib
sudo systemctl enable otelcol-contrib
sudo systemctl status otelcol-contrib --no-pager
```

## 3. Verify it reaches ClickHouse

```bash
docker exec akvorado-clickhouse-1 clickhouse-client -q "
SELECT MetricName, count() AS points, max(TimeUnix) AS latest
FROM default.otel_metrics_sum
WHERE MetricName LIKE 'system.%' AND TimeUnix > now() - INTERVAL 5 MINUTE
GROUP BY MetricName ORDER BY MetricName"
```

Expect `system.cpu.time`, `system.memory.usage`, `system.disk.io`,
`system.network.io`, `system.paging.*`, etc. (gauges land in
`default.otel_metrics_gauge`, sums/counters in `default.otel_metrics_sum`).

Quick end-to-end smoke test without installing anything (proves the receive path):

```bash
curl -s -X POST http://localhost:4318/v1/metrics -H 'Content-Type: application/json' \
  -d '{"resourceMetrics":[{"scopeMetrics":[{"metrics":[{"name":"test.host.ping",
       "gauge":{"dataPoints":[{"asDouble":1,"timeUnixNano":"'"$(date +%s)000000000"'"}]}}]}]}]}'
docker exec akvorado-clickhouse-1 clickhouse-client -q \
  "SELECT * FROM default.otel_metrics_gauge WHERE MetricName='test.host.ping' LIMIT 1"
```

## 4. Chart it in Grafana

Query `default.otel_metrics_sum` / `_gauge` for `system.*`. Example — CPU time by state:

```sql
SELECT $__timeInterval(TimeUnix) AS time,
       Attributes['cpu.state'] AS state,
       avg(Value) AS value
FROM default.otel_metrics_sum
WHERE MetricName = 'system.cpu.time' AND $__timeFilter(TimeUnix)
GROUP BY time, state
ORDER BY time
```

Memory usage:

```sql
SELECT $__timeInterval(TimeUnix) AS time,
       Attributes['state'] AS state,
       avg(Value) AS bytes
FROM default.otel_metrics_gauge
WHERE MetricName = 'system.memory.usage' AND $__timeFilter(TimeUnix)
GROUP BY time, state ORDER BY time
```

The `host.name` resource attribute (from `resourcedetection`) is in
`ResourceAttributes['host.name']` — use it to tell multiple hosts apart.

## Notes

- **Multiple hosts:** install the same native collector on each; they all forward to
  the one containerized collector's `4317`. If the host is remote, point `endpoint`
  at the Docker host's address (and secure the link — the container's OTLP receiver
  is plaintext by default and bound to `0.0.0.0`).
- **Security:** `otel-collector`'s `4317`/`4318` are published on `0.0.0.0`. If you
  onboard remote senders, front them with TLS/auth or restrict the published bind to
  the loopback / a firewall.
- **Version pinning** keeps the host collector's metric schema aligned with the
  container collector's ClickHouse tables.
