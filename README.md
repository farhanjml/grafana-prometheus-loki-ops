# Grafana, Prometheus & Loki Ops Skill

A Hermes runbook for checking and troubleshooting an existing Grafana, Prometheus, and Loki stack.

It covers service health, live Prometheus data, Loki ingestion, Grafana data sources, and common failure modes.

**This is not a stack installer.** It does not include step-by-step deployment/configuration instructions or dashboard JSON files.

## Use with Hermes

Copy `SKILL.md` to `$HERMES_HOME/skills/grafana-prom-loki-ops/SKILL.md`. If `HERMES_HOME` is unset, use `~/.hermes/skills/grafana-prom-loki-ops/SKILL.md`. Start a new Hermes session to load it.
