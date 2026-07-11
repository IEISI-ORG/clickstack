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

Install an OpenTelemetry Collector distribution that includes the **`hostmetrics`**
receiver (the `-contrib` build, or any distro package built from it):

```bash
VER=0.119.0   # any recent version is fine — see the version note below
curl -fsSLo /tmp/otelcol-contrib.deb \
  "https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v${VER}/otelcol-contrib_${VER}_linux_amd64.deb"
sudo dpkg -i /tmp/otelcol-contrib.deb
```

The package name, systemd unit, and config path vary by distribution — commonly
**`otelcol-contrib`** → `/etc/otelcol-contrib/config.yaml`, or **`otelcol`** →
`/etc/otelcol/config.yaml`. Check with `systemctl cat <unit>` (look at `ExecStart`
/ `--config`). Confirm your build has the receiver/processors you need:
`otelcol components | grep -E 'hostmetrics|resource'`.

**Version note:** the host collector only *forwards* OTLP — it does not write to
ClickHouse — so its version does **not** need to match the container collector
(OTLP is stable across versions; the container's exporter owns the ClickHouse schema).

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
  # Label metrics with the host name. The built-in `resource` processor works in
  # every build; contrib builds also offer `resourcedetection` (detectors: [system])
  # to fill host.name automatically.
  resource:
    attributes:
      - key: host.name
        value: my-host          # set to this host's name
        action: upsert
  batch: {}

exporters:
  # Forward to the containerized collector (published on the host loopback).
  otlp:
    endpoint: localhost:4317
    tls:
      insecure: true

service:
  # No listening receivers/extensions and self-metrics disabled -> this collector
  # opens NO ports, so it can't clash with the container's published 4317/4318/13133.
  telemetry:
    metrics:
      level: none
  pipelines:
    metrics:
      receivers: [hostmetrics]
      processors: [resource, batch]
      exporters: [otlp]
```

> **Port-clash warning:** the distro's *default* config usually enables OTLP/jaeger/
> zipkin receivers on `0.0.0.0:4317/4318/...`, which collide with the container
> collector and leave the service `failed`. Replace it entirely with the config above
> (hostmetrics in, OTLP out, nothing listening). Verify: `ss -ltnp | grep otelcol`
> should return nothing.

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
- **No port clash by design:** this config has no listening receivers/extensions and
  disables self-metrics, so it coexists with the container collector. The container
  keeps owning `4317`/`4318`; the host collector only makes an outbound OTLP connection.
