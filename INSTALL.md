# Install and Configure

This guide builds a single-host, systemd-based stack on RHEL-compatible Linux. It is for a fresh install; on an existing stack, inventory and back up current configuration first. The live stack used Grafana 13.1.0, Prometheus 3.1.0, and Loki 3.7.3. These are observed pins, not a claim that they are the latest releases. Review current releases before a new production install.

Install the tools used by the steps below:

```sh
sudo dnf install curl tar gzip
```

## 1. Install Grafana

Add Grafana's official RPM repository:

```sh
sudo tee /etc/yum.repos.d/grafana.repo >/dev/null <<'EOF'
[grafana]
name=grafana
baseurl=https://rpm.grafana.com
enabled=1
repo_gpgcheck=1
gpgcheck=1
gpgkey=https://rpm.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
EOF
sudo dnf install grafana
sudo systemctl enable --now grafana-server
```

For an exact version, check available package versions with `dnf --showduplicates list grafana` and install the reviewed version.

## 2. Install Prometheus

Set the version to the release you reviewed. The example uses the version observed on the source stack.

```sh
PROMETHEUS_VERSION=3.1.0
ARCH=linux-amd64
WORK=$(mktemp -d)
curl -fL "https://github.com/prometheus/prometheus/releases/download/v${PROMETHEUS_VERSION}/prometheus-${PROMETHEUS_VERSION}.${ARCH}.tar.gz" -o "$WORK/prometheus.tar.gz"
tar -xzf "$WORK/prometheus.tar.gz" -C "$WORK"

getent passwd prometheus >/dev/null || sudo useradd --system --user-group --no-create-home --shell /sbin/nologin prometheus
sudo install -d -o prometheus -g prometheus /etc/prometheus/file_sd /var/lib/prometheus
sudo install -m 0755 "$WORK/prometheus-${PROMETHEUS_VERSION}.${ARCH}/prometheus" /usr/local/bin/prometheus
sudo install -m 0755 "$WORK/prometheus-${PROMETHEUS_VERSION}.${ARCH}/promtool" /usr/local/bin/promtool
sudo install -m 0644 config/prometheus.yml /etc/prometheus/prometheus.yml
sudo install -m 0644 config/node-exporters.yml /etc/prometheus/file_sd/node-exporters.yml
sudo install -m 0644 systemd/prometheus.service /etc/systemd/system/prometheus.service
sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
```

The file service-discovery list starts empty. Add the actual node-exporter targets to `config/node-exporters.yml`, then copy it to `/etc/prometheus/file_sd/node-exporters.yml`. Prometheus reloads file-based targets automatically.

Install node_exporter on each Linux host that should appear in the host dashboards:

```sh
printf 'Node Exporter release version (without the leading v): '
IFS= read -r NODE_EXPORTER_VERSION
ARCH=linux-amd64
WORK=$(mktemp -d)
curl -fL "https://github.com/prometheus/node_exporter/releases/download/v${NODE_EXPORTER_VERSION}/node_exporter-${NODE_EXPORTER_VERSION}.${ARCH}.tar.gz" -o "$WORK/node-exporter.tar.gz"
tar -xzf "$WORK/node-exporter.tar.gz" -C "$WORK"
getent passwd node_exporter >/dev/null || sudo useradd --system --user-group --no-create-home --shell /sbin/nologin node_exporter
sudo install -m 0755 "$WORK/node_exporter-${NODE_EXPORTER_VERSION}.${ARCH}/node_exporter" /usr/local/bin/node_exporter
sudo install -m 0644 clients/node-exporter.service /etc/systemd/system/node_exporter.service
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
```

Add each host to the file-service-discovery list. Replace the placeholder with a reachable hostname:

```yaml
- targets:
    - "<NODE_EXPORTER_HOST>:9100"
```

Node Exporter does not authenticate requests. Allow access to port `9100` only from the monitoring host.

The Prometheus service enables the remote-write receiver at `/api/v1/write`. Configure each compatible sender with this `remote_write` section:

```yaml
remote_write:
  - url: http://<PROMETHEUS_HOST>:9090/api/v1/write
```

Restrict access to trusted senders with a firewall or an authenticated proxy. Do not expose the unauthenticated receiver to the public internet.

## 3. Install Loki

```sh
LOKI_VERSION=3.7.3
WORK=$(mktemp -d)
sudo dnf install unzip
curl -fL "https://github.com/grafana/loki/releases/download/v${LOKI_VERSION}/loki-linux-amd64.zip" -o "$WORK/loki.zip"
unzip -p "$WORK/loki.zip" loki-linux-amd64 > "$WORK/loki"

getent passwd loki >/dev/null || sudo useradd --system --user-group --no-create-home --shell /sbin/nologin loki
sudo install -m 0755 "$WORK/loki" /usr/local/bin/loki
sudo install -d -o loki -g loki /etc/loki /var/lib/loki
sudo install -m 0644 config/loki.yaml /etc/loki/local-config.yaml
sudo install -m 0644 systemd/loki.service /etc/systemd/system/loki.service
sudo chown -R loki:loki /var/lib/loki
sudo systemctl daemon-reload
sudo systemctl enable --now loki
```

The sample config disables Loki's built-in authentication for a trusted private network. Restrict access at the firewall, or place Loki behind a TLS/authentication proxy before allowing untrusted clients to connect.

## 4. Configure Grafana Data Sources and Dashboards

The provisioning templates use stable data-source UIDs so the JSON dashboards can resolve their data sources.

Before applying them to an existing Grafana installation, back up its provisioning directory and check for duplicate data-source names or UIDs. Conflicting provisioning files can prevent Grafana from starting. Do not copy the files over an existing installation without checking first.

```sh
sudo install -d -o grafana -g grafana /etc/grafana/provisioning/datasources
sudo install -d -o grafana -g grafana /etc/grafana/provisioning/dashboards
sudo install -d -o grafana -g grafana /var/lib/grafana/dashboards
sudo install -m 0644 provisioning/datasources.yml /etc/grafana/provisioning/datasources/monitoring.yml
sudo install -m 0644 provisioning/dashboards.yml /etc/grafana/provisioning/dashboards/monitoring.yml
sudo install -m 0644 dashboards/*.json /var/lib/grafana/dashboards/
sudo chown -R grafana:grafana /var/lib/grafana/dashboards
sudo systemctl restart grafana-server
```

Set a unique Grafana administrator password during first login. Keep passwords and tokens in a secret store, not in this repository.

## 5. Send Logs from Client Hosts

Promtail reached end of life on March 2, 2026. Use Grafana Alloy for new client installs. On each client, add Grafana's official RPM repository using the steps in section 1, then install Alloy:

```sh
sudo dnf install alloy
```

Copy `clients/alloy-logs.alloy` to `/etc/alloy/config.alloy`.

Replace `<LOKI_PUSH_URL>` with the Loki push endpoint reachable from that client, for example `http://<LOKI_HOST>:3100/loki/api/v1/push`. Review the log glob so the client reads only the intended files. Then enable Alloy:

```sh
sudo systemctl enable --now alloy
sudo systemctl status alloy --no-pager
```

Do not store client credentials in the Alloy file. If authentication is required, use an environment-specific secret file with restricted permissions and the supported Alloy authentication settings.

## 6. Verify the Stack

```sh
sudo /usr/local/bin/promtool check config /etc/prometheus/prometheus.yml
systemctl is-active grafana-server prometheus loki
curl -fsS http://localhost:9090/-/ready
curl -fsS http://localhost:3100/ready
curl -fsS http://localhost:3000/api/health
curl -fsSG http://localhost:9090/api/v1/query --data-urlencode 'query=count(up)'
curl -fsS http://localhost:3100/metrics | grep loki_distributor_lines_received_total
```

In Grafana, open the provisioned dashboards and confirm that the Prometheus and Loki data sources are healthy. Empty panels can mean that no matching exporter or log stream is configured yet.

## Official References

- [Grafana RPM installation](https://grafana.com/docs/grafana/latest/setup-grafana/installation/redhat-rhel-fedora/)
- [Prometheus installation](https://prometheus.io/docs/prometheus/latest/installation/)
- [Node Exporter guide](https://prometheus.io/docs/guides/node-exporter/)
- [Node Exporter releases](https://github.com/prometheus/node_exporter/releases)
- [Prometheus configuration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)
- [Loki single-binary installation](https://grafana.com/docs/loki/latest/setup/install/local/)
- [Alloy Linux installation](https://grafana.com/docs/alloy/latest/set-up/install/linux/)
- [Promtail-to-Alloy migration](https://grafana.com/docs/alloy/latest/set-up/migrate/from-promtail/)
