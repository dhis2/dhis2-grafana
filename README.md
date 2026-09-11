# DHIS2 API Endpoint Performance (Grafana Dashboard)

A Grafana dashboard for monitoring a running DHIS2 instance, built around the
metrics exposed by Spring Boot Actuator / Micrometer. It focuses on:

- **Hot endpoints** — which API endpoints receive the most traffic (GET vs.
  write methods broken out separately)
- **Slow endpoints** — average and max latency per endpoint
- **Errors** — 5xx rate overall and per-endpoint
- **JVM & System** — heap/non-heap memory, GC pause time, threads, CPU,
  open file descriptors, process memory
- **Ehcache** — hit ratio, hot caches by GET/PUT rate, worst hit-ratio
  caches, evictions
- **JDBC connection pool** — pool utilization, pending/waiting connections,
  acquire/usage/creation latency percentiles, acquire timeouts

## Requirements

- **DHIS2 2.37 or later.** DHIS2 core ships its own Prometheus metrics
  endpoint at `GET /api/metrics`
  ([`PrometheusScrapeEndpointController`](https://github.com/dhis2/dhis2-core/blob/master/dhis-2/dhis-web-api/src/main/java/org/hisp/dhis/webapi/controller/PrometheusScrapeEndpointController.java)),
  backed by a Micrometer `PrometheusMeterRegistry`. This is **not** the
  generic Spring Boot Actuator `/actuator/prometheus` path — it's DHIS2's own
  controller, and it requires authentication (DHIS2 scrapes it like any other
  API endpoint, with credentials on every request).
- Prometheus (or a compatible remote-write/VictoriaMetrics-style backend)
  scraping that endpoint into a datasource Grafana can query, with `job` and
  `instance` labels present (the dashboard's variables discover whatever
  values exist).

### Enabling the metrics DHIS2 collects

Each metric family is off by default and enabled independently in
`dhis.conf` — see
[`ConfigurationKey.java`](https://github.com/dhis2/dhis2-core/blob/master/dhis-2/dhis-support/dhis-support-external/src/main/java/org/hisp/dhis/external/conf/ConfigurationKey.java)
for the authoritative list:

```properties
# HTTP request metrics (http_server_requests_seconds_*) — Hot/Slow/Errors sections
monitoring.api.enabled = on

# JVM metrics (jvm_*, process_*) — JVM & System section
monitoring.jvm.enabled = on

# CPU and uptime metrics — JVM & System section
monitoring.cpu.enabled = on
monitoring.uptime.enabled = on

# Hibernate 2nd-level (Ehcache) cache region metrics — Ehcache section
monitoring.ehcache.enabled = on

# JDBC connection pool metrics (jdbc_connections_*) — JDBC Connection Pool section
monitoring.dbpool.enabled = on
db.pool.type = hikari   # default; the jdbc_connections_* naming only applies to HikariCP
```

(DHIS2 renames HikariCP's native `hikaricp.connections*` meters to
`jdbc.connections*` internally for naming stability — see
[`PrometheusMonitoringConfig`](https://github.com/dhis2/dhis2-core/blob/master/dhis-2/dhis-support/dhis-support-system/src/main/java/org/hisp/dhis/monitoring/metrics/PrometheusMonitoringConfig.java) —
which is why the dashboard queries `jdbc_connections_*`, not
`hikaricp_connections_*`.)

### Example Prometheus scrape config

This mirrors the production config used in
[`dhis2-server-tools`](https://github.com/dhis2/dhis2-server-tools/blob/main/deploy/roles/monitoring/templates/scrape-configs.j2):

```yaml
scrape_configs:
  - job_name: dhis2
    metrics_path: /api/metrics
    basic_auth:
      username: <a DHIS2 user with API access>
      password: <password>
    static_configs:
      - targets:
          - <dhis2-host>:8080
```

Metric families this dashboard uses:

- `http_server_requests_seconds_{count,sum,max}` — HTTP request timing
- `jvm_memory_used_bytes`, `jvm_memory_max_bytes`, `jvm_gc_pause_seconds_*`,
  `jvm_threads_*`, `process_cpu_usage`, `process_resident_memory_bytes`,
  `process_open_fds`, `process_max_fds`, `system_load_average_1m`,
  `system_cpu_count`
- `ehcache_{hits,gets,puts,evictions}_total`
- `jdbc_connections{,_active,_idle,_min,_max,_pending}`,
  `jdbc_connections_timeout_total`,
  `jdbc_connections_{acquire,creation,usage}_seconds_*` (needs histogram
  buckets for the latency percentile panels)

If a metric family isn't present on your instance (because the corresponding
`monitoring.*.enabled` flag is off), the corresponding panels will simply
show no data — nothing else depends on them.

## Importing

In Grafana: **Dashboards → New → Import**, then upload
[`dashboards/dhis2-api-endpoint-performance.json`](dashboards/dhis2-api-endpoint-performance.json)
and select your Prometheus datasource when prompted.

The dashboard uses a `$datasource` template variable (so it isn't hardcoded
to a specific datasource UID) plus `$job`, `$instance`, and `$method`
filters, all defaulting to "All". It also carries the `__inputs`/`__requires`
metadata block Grafana's own "Export for sharing externally" adds, so it
imports cleanly through grafana.com's community dashboard uploader too (built
and tested against Grafana 12.1.1).

## Notes

- `http_server_requests_seconds` on most DHIS2 instances is a Micrometer
  **Summary**, not a Histogram — there are no `_bucket` samples, so latency
  panels use average (`sum/count`) and max rather than true percentiles.
- `jdbc_connections_*` histograms (acquire/usage/creation) *do* typically
  have buckets, so those panels use `histogram_quantile` for real p50/p95/p99.
- Several "top 10" panels use `topk`/`bottomk` with a minimum-traffic guard
  (e.g. `> 0.01` req/s) to avoid noisy ratios on near-idle endpoints/caches.
- The eviction and timeout "top N" tables use `increase(...[$__range])`
  rather than a fixed 5-minute rate window, so they reflect whatever time
  range you have the dashboard set to, not just the last 5 minutes.

## License

BSD 3-Clause — see [LICENSE](LICENSE).
