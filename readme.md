# Grafana Logs Dashboards

This repository contains Grafana dashboards for browsing Loki logs, plus a minimal Docker Compose stack for Grafana, Loki, Promtail, and one sample app container.

The dashboards live in `dashboards/`.

## Repository Layout

- `compose.yml`: Docker Compose stack with Loki, Promtail, Grafana, and one sample app container
- `promtail-config.yml`: Promtail config for Docker container logs and nginx log files
- `ds.yml`: Grafana datasource provisioning for Loki
- `dashboards.yml`: Grafana dashboard provisioning
- `dashboards/grafana-loki-logs-dashboard.json`: general container logs dashboard
- `dashboards/grafana-nginx-logs-dashboard.json`: nginx logs dashboard

## Dashboards

### `grafana-loki-logs-dashboard.json`

This dashboard is a general container logs view:

- Title: `Logs`
- UID: `qwertyz`
- Default time range: `now-30m` to `now`
- Filters on `machine` and `app`

Variables:

- `datasource`: Grafana datasource selector for Loki
- `Machine`: label values for `machine` from `{job="containerlogs"}`
- `App`: label values for `app` from `{job="containerlogs", machine="$Machine"}`

Main query:

```logql
{job="containerlogs", app=~`${App:regex}`, machine="$Machine"}
```

Required labels:

- `job="containerlogs"`
- `machine`
- `app`

### `grafana-nginx-logs-dashboard.json`

This dashboard is an nginx logs view:

- Title: `Nginx-logs`
- UID: `qwertyx`
- Default time range: `now-30m` to `now`
- Filters on `machine`, `domain`, and `log_type`

Variables:

- `datasource`: Grafana datasource selector for Loki
- `Machine`: label values for `machine` from `{job="nginx"}`
- `domain`: label values for `domain` from `{job="nginx", machine="$Machine"}`
- `logtype`: label values for `log_type` from `{job="nginx", machine="$Machine"}`

Main query:

```logql
{machine="$Machine", job="nginx", domain=~`${domain:regex}`, log_type=~`${logtype:regex}`}
```

Required labels:

- `job="nginx"`
- `machine`
- `domain`
- `log_type`

## Promtail Label Contract

### Container logs

The Docker container log pipeline produces labels for the Loki dashboard:

- Docker uses the `json-file` logging driver
- Docker log tags are formatted as `{{.ImageName}}|{{.Name}}`
- Promtail parses the Docker JSON log envelope
- Promtail extracts `image` and `app` from the log tag
- Promtail attaches `machine` as an external label
- Promtail sets `job: containerlogs`

### Nginx logs

The nginx log pipeline reads files from:

```text
/var/log/nginx/*.log
```

It expects filenames in this format:

```text
<domain>-access.log
<domain>-error.log
```

Example:

```text
test.example.com-access.log
test.example.com-error.log
```

From the filename, Promtail extracts:

- `domain`: `test.example.com`
- `log_type`: `access` or `error`

Promtail also attaches `machine` as an external label and sets `job: nginx`.

## Importing Dashboards

To import manually:

1. Open Grafana.
2. Go to `Dashboards` -> `New` -> `Import`.
3. Upload the dashboard JSON you want from `dashboards/`.
4. Choose a Loki datasource when prompted.

## Local Stack

The top-level Compose stack does the following:

1. Runs Loki at `http://loki:3100`.
2. Runs Promtail and mounts Docker container logs plus `/var/log/nginx`.
3. Pushes logs to Loki at `http://loki:3100/loki/api/v1/push`.
4. Runs Grafana with file-based dashboard provisioning from `./dashboards`.
5. Includes one sample `app` container that writes Docker `json-file` logs for the general Loki dashboard.

## Notes

- The stack is Linux-specific as written because it reads `/var/lib/docker/containers` and `/var/log/nginx` from the host.
- The container logs dashboard only depends on the labels produced by the checked-in Docker pipeline.
- The nginx dashboard depends on nginx filenames matching the documented `<domain>-<level>.log` convention.
- The stack uses `grafana/grafana:latest` and `grafana/loki:latest`, which is convenient for testing but not ideal for reproducible deployments.

## Troubleshooting

- If `Machine` is empty in the Loki dashboard, Loki likely has no streams with `job="containerlogs"`.
- If `App` is empty, the Docker log tag extraction for `app` is probably failing.
- If `Machine` is empty in the nginx dashboard, Loki likely has no streams with `job="nginx"`.
- If `domain` or `logtype` is empty, verify the nginx filenames match `<domain>-access.log` or `<domain>-error.log`.
- If a dashboard imports but shows no logs, verify Grafana is using the same Loki instance Promtail writes to.
