# Dashboard JSON

These portable dashboards use the data-source UIDs in `../provisioning/datasources.yml`:

- `host-disk-usage.json` — filesystem overview adapted from the live disk-usage dashboard.
- `host-node-overview.json` — CPU, memory, load, and filesystem panels for node_exporter metrics.
- `loki-log-overview.json` — log volume and recent log lines from Loki.

They are not a full export of every dashboard on the Grafana instance. Dashboards that depend on application-specific data sources or plugins are not included.
