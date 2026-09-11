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

Your DHIS2 instance must expose a Prometheus-scrapeable metrics endpoint
(Spring Boot Actuator's `/actuator/prometheus`) with Micrometer bindings
enabled, and Prometheus must be scraping it into a `job` label (the
dashboard's variables default to discovering whatever `job`/`instance`
values are present).

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

If a metric family isn't present on your instance, the corresponding panels
will simply show no data — nothing else depends on them.

## Importing

In Grafana: **Dashboards → New → Import**, then upload
[`dashboards/dhis2-api-endpoint-performance.json`](dashboards/dhis2-api-endpoint-performance.json)
and select your Prometheus datasource when prompted.

The dashboard uses a `$datasource` template variable (so it isn't hardcoded
to a specific datasource UID) plus `$job`, `$instance`, and `$method`
filters, all defaulting to "All".

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
