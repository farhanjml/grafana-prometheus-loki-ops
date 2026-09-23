---
name: grafana-prom-loki-ops
description: "Use when operating an existing Grafana, Prometheus, and Loki stack."
version: 1.0.0
license: MIT
metadata:
  hermes:
    tags: [grafana, prometheus, loki, monitoring, remote-write, promtail, ops]
    category: devops
---

# Grafana / Prometheus / Loki Operations

This skill is an operations and troubleshooting runbook for an existing stack. It does not install or configure the stack from scratch.

## Scope

- Check Grafana, Prometheus, and Loki service health.
- Verify live metrics and log ingestion through service APIs.
- Inspect configured Grafana data sources.
- Diagnose common no-data and service-startup problems.

The common data paths are Prometheus remote-write from monitored clients and Promtail pushes into Loki. Confirm the actual paths on each installation; do not infer them from filenames alone.

## Default Ports

| Service | Port |
|---|---:|
| Grafana | 3000 |
| Prometheus | 9090 |
| Loki HTTP | 3100 |
| Loki gRPC | 9096 |

Ports and service names can differ by deployment. Check the live host before using these defaults.

## Survey the Stack

Run commands on the monitoring host. Use the correct service names and access method for the target system.

1. Check service state:

   ```sh
   systemctl status grafana-server prometheus loki --no-pager
   ```

2. Check listening ports:

   ```sh
   ss -tlnp | grep -E ':(3000|9090|3100|9096)'
   ```

3. Read the active Prometheus configuration:

   ```sh
   curl -fsS http://localhost:9090/api/v1/status/config
   ```

4. Read live labels and query data:

   ```sh
   curl -fsS http://localhost:9090/api/v1/label/job/values
   curl -fsS http://localhost:9090/api/v1/label/instance/values
   curl -fsSG http://localhost:9090/api/v1/query --data-urlencode 'query=count(up)'
   curl -fsSG http://localhost:9090/api/v1/query --data-urlencode 'query=node_load1'
   ```

5. Check whether remote-write receiving is enabled:

   ```sh
   systemctl cat prometheus.service
   ```

   Look for `--web.enable-remote-write-receiver` when clients push metrics to Prometheus.

6. Check Loki readiness and ingestion:

   ```sh
   curl -fsS http://localhost:3100/ready
   curl -fsS http://localhost:3100/metrics | grep loki_distributor_lines_received_total
   ```

7. Query recent Loki streams. Replace the time range with the interval you need:

   ```sh
   curl -fsSG http://localhost:3100/loki/api/v1/query_range \
     --data-urlencode 'query={job=~".+"}' \
     --data-urlencode 'start=<RFC3339_START>' \
     --data-urlencode 'end=<RFC3339_END>' \
     --data-urlencode 'limit=5'
   ```

8. Check Grafana data sources through the UI or authenticated API. If the installation uses SQLite, the database is often `/var/lib/grafana/grafana.db`; confirm the configured database path before querying it.

## Troubleshooting Rules

- **Trust live APIs over one config file.** Metrics can arrive through remote write even when the scrape configuration appears minimal. Check active configuration, label values, and query results before calling the stack empty.
- **Use a valid LogQL selector.** `{}` is not a valid selector. Use a matcher such as `{job=~".+"}`.
- **Do not use the labels endpoint alone to prove logs exist.** It can return success without a useful data list. Check ingestion counters and run a query-range request.
- **Check Loki's configured query limit.** A time range wider than `max_query_length` can fail. Do not assume the same limit on every installation.
- **Recheck transient readiness errors.** Loki can briefly report that an ingester is not ready after startup. Query `/ready` again before treating it as a persistent failure.
- **Avoid duplicate Grafana datasource provisioning.** Conflicting provisioning files can stop Grafana from starting. Back up the files first, identify the conflicting definition, and change one file at a time. After each change, restart Grafana and verify that its HTTP port is listening and the service remains active. Do not delete files as a first response.
- **Check exporter recovery logic after database errors.** A long-running exporter can stay active while it repeatedly fails to write. Confirm that its error path reconnects for the exception types it can encounter, and inspect recent service logs.
- **An empty dashboard panel is not enough to prove a pipeline failure.** Query the underlying data with the panel's filters, check the record's freshness, and confirm that the upstream system should have produced a non-empty result.
- **Never put passwords or tokens in commands, files, or shell history.** Use the site's approved secret store or a protected interactive prompt for authenticated operations.

## Related Repository Files

- `INSTALL.md` contains the single-host installation and configuration steps.
- `config/` and `systemd/` contain portable service templates.
- `provisioning/` and `dashboards/` contain Grafana provisioning and dashboard JSON.
