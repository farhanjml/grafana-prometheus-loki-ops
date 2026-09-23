# Grafana, Prometheus & Loki Ops Skill

A Hermes runbook and deployment starter for a single-host Grafana, Prometheus, and Loki stack.

## Includes

- Step-by-step install and configuration guide.
- Prometheus and Loki configuration plus systemd units.
- Grafana data-source and dashboard provisioning files.
- Portable dashboard JSON for host metrics, disk usage, and Loki logs.
- Operational checks and troubleshooting notes in `SKILL.md`.

The guide records the versions observed on the source stack: Grafana 13.1.0, Prometheus 3.1.0, and Loki 3.7.3. Review version pins before a new production install.

Start with [INSTALL.md](INSTALL.md). The `config/`, `systemd/`, `provisioning/`, `clients/`, and `dashboards/` directories contain the matching templates.
