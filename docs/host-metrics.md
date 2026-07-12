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

The `host.name` resource attribute — set by the `resource` processor in §2 (or
auto-filled by `resourcedetection` on contrib builds) — lands in
`ResourceAttributes['host.name']`; use it to tell multiple hosts apart.

A provisioned **Host Metrics** dashboard
(`docker/grafana/dashboards/host-metrics.json`) already does this: every panel
splits its series by `ResourceAttributes['host.name']`, and a **`Host` dropdown**
(`$host` template variable) lets you focus on one machine or view **All**. The
dropdown is populated from `SELECT DISTINCT ResourceAttributes['host.name']`, so a
newly onboarded host appears automatically — no dashboard edit needed.

## 5. Onboarding other hosts (remote OTLP senders)

Every extra host runs the **same** native collector from §1–§2. Because it only
*forwards* OTLP, onboarding a host is two edits to the §2 config — no change to the
container collector, its exporter, or the ClickHouse schema.

The in-stack receiver is **already reachable from the LAN**: Docker publishes
`4317`/`4318` with the short `"4317:4317"` syntax, which binds `0.0.0.0` on the Docker
host (confirm with `ss -ltn | grep -E ':4317|:4318'`). So there is **no
`docker-compose.yml` change** to make — only the remote config and a firewall rule.

**On each remote host**, take the §2 config and change exactly two things:

```yaml
processors:
  resource:
    attributes:
      - key: host.name
        value: web-01            # ← distinct name per host; this is how Grafana tells them apart
        action: upsert

exporters:
  otlp:
    endpoint: <docker-host-ip>:4317   # ← the Docker host's LAN address, not localhost
    tls:
      insecure: true                  # plaintext; fine on a trusted LAN, see Security below
```

> **Set `host.name` correctly *before* the first start.** The name is written into
> every row as `ResourceAttributes['host.name']`. If a host starts with the wrong
> name (e.g. a copy-paste of another host's config), its metrics are mislabelled at
> write time — and if the name collides with a host that is *also* running, the two
> machines' rows are interleaved under one label with nothing in the row to tell them
> apart. Untangling that after the fact is only possible by luck (e.g. the two
> collectors happening to emit at different sub-second timestamps) and needs manual
> `ALTER TABLE … UPDATE` mutations. Renaming later is cheap; unmixing is not.

Restart, then verify from the **Docker host** that the new `host.name` is landing:

```bash
docker exec akvorado-clickhouse-1 clickhouse-client -q "
SELECT ResourceAttributes['host.name'] AS host, count() AS points, max(TimeUnix) AS latest
FROM default.otel_metrics_sum
WHERE TimeUnix > now() - INTERVAL 5 MINUTE
GROUP BY host ORDER BY host"
```

In Grafana, split series per machine with `ResourceAttributes['host.name']` (add it to
`GROUP BY` and as a series label) — the same panels from §4 then cover every host.

### Security — the receiver is open and plaintext

`4317`/`4318` accept OTLP from **anyone who can reach the port**, with no auth and no
encryption. Pick a posture before onboarding senders:

- **Trusted LAN (recommended here): restrict the port with a firewall.** Allow only
  known senders and drop the rest. Example (iptables; adapt to `ufw`/`nftables`):

  ```bash
  LAN=192.168.0.0/24    # ← your trusted subnet
  # allow the local LAN, drop everyone else on the OTLP ports
  sudo iptables -A INPUT -p tcp -s "$LAN" --dport 4317 -j ACCEPT
  sudo iptables -A INPUT -p tcp -s "$LAN" --dport 4318 -j ACCEPT
  sudo iptables -A INPUT -p tcp --dport 4317 -j DROP
  sudo iptables -A INPUT -p tcp --dport 4318 -j DROP
  ```

- **Crossing an untrusted network: add TLS + a bearer token** on the container
  collector (`docker/otel/otel-collector.yaml`) and match it on the sender. This
  requires editing the in-stack collector, so it is a real config change, not just docs:

  ```yaml
  # container collector — docker/otel/otel-collector.yaml
  extensions:
    bearertokenauth:
      token: ${env:OTLP_INGEST_TOKEN}     # keep the value in config/secrets, not in git
  receivers:
    otlp:
      protocols:
        grpc: { endpoint: 0.0.0.0:4317, auth: { authenticator: bearertokenauth } }
        http: { endpoint: 0.0.0.0:4318, auth: { authenticator: bearertokenauth } }
  service:
    extensions: [bearertokenauth]
  ```

  ```yaml
  # remote sender — add the token, and real TLS (a bearer token over insecure:true is sniffable)
  exporters:
    otlp:
      endpoint: <docker-host-ip>:4317
      headers: { authorization: "Bearer <same-token>" }
      tls: { ca_file: /etc/otelcol/ca.pem }   # server cert the container presents
  ```

## Notes

- **No port clash by design:** the *host* collector config has no listening
  receivers/extensions and disables self-metrics, so it coexists with the container
  collector. The container keeps owning `4317`/`4318`; each host collector only makes
  an outbound OTLP connection.
- **`host.name`** is set by the `resource` processor and lands in
  `ResourceAttributes['host.name']` — the key for telling hosts apart (contrib builds
  can fill it automatically with `resourcedetection`, detectors `[system]`).
