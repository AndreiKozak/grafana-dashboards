# Grafana Logs Dashboard

This repository contains a single Grafana dashboard for browsing Docker container logs stored in Loki, plus a minimal Docker Compose stack for Grafana, Loki, Promtail, and one sample app container.

The primary artifact is `dashboards/grafana-logs-dashboard.json`.

## Repository Layout

- `compose.yml`: Docker Compose stack with Loki, Promtail, Grafana, and one sample app container
- `promtail-config.yml`: Promtail pipeline that turns Docker `json-file` logs into Loki streams with the labels expected by the dashboard
- `ds.yml`: Grafana datasource provisioning for Loki
- `dashboards.yml`: Grafana dashboard provisioning
- `dashboards/grafana-logs-dashboard.json`: dashboard definition mounted into Grafana

## What The Dashboard Does

The dashboard is intentionally narrow:

- It has one `logs` panel titled `Logs`
- The default time range is `now-30m` to `now`
- It filters on two Loki labels: `machine` and `app`
- It uses a datasource variable so the dashboard can be imported into different Grafana environments without hard-coding a datasource UID

### Variables

The dashboard defines three variables:

- `datasource`: Grafana datasource selector for Loki
- `Machine`: populated from Loki label values for `machine` using `{job="containerlogs"}`
- `App`: populated from Loki label values for `app` using `{job="containerlogs", machine="$Machine"}`

`App` is multi-select and supports `All`. `Machine` is effectively the primary filter and should be selected first.

### Main Query

The log panel uses this selector:

```logql
{job="containerlogs", app=~"${App:regex}", machine="$Machine"}
```

That means the dashboard assumes all relevant streams carry:

- `job="containerlogs"`
- `machine`
- `app`

## Log Label Contract

The example Promtail configuration builds those labels from Docker container logs:

- Docker writes container logs with the `json-file` logging driver
- Docker log tags are formatted as `{{.ImageName}}|{{.Name}}`
- Promtail parses the Docker JSON log envelope
- Promtail extracts `image` and `app` from the log tag
- Promtail attaches `machine` as an external label
- Promtail sets `job: containerlogs` in the scrape config

If your environment uses different labels, `dashboards/grafana-logs-dashboard.json` must be updated to match.

## Importing The Dashboard

To import manually:

1. Open Grafana.
2. Go to `Dashboards` -> `New` -> `Import`.
3. Upload `dashboards/grafana-logs-dashboard.json`.
4. Choose a Loki datasource when prompted.

## Local Stack

The top-level Compose stack implements this flow:

1. A container writes logs with Docker `json-file`.
2. Promtail reads `/var/lib/docker/containers/*/*log`.
3. Promtail pushes logs to Loki at `http://loki:3100/loki/api/v1/push`.
4. Grafana connects to Loki through the provisioned datasource in `ds.yml`.
5. Grafana loads dashboards from `/var/lib/grafana/dashboards` based on `dashboards.yml`.

### Stack Files

- `compose.yml`: starts `loki`, `promtail`, `grafana`, and one sample `app` container
- `promtail-config.yml`: parses Docker `json-file` logs, extracts `app` and `image` from the Docker log tag, and adds `machine` as an external label
- `ds.yml`: provisions the Loki datasource in Grafana
- `dashboards.yml`: tells Grafana to load dashboards from the mounted `./dashboards` directory

## Current Gaps And Assumptions

The repository is small and functional in intent, but there are a few assumptions worth making explicit:

- `compose.yml` mounts `./dashboards` into Grafana, and this repository now includes that directory with the dashboard JSON inside it.
- `promtail-config.yml` reads Docker container logs directly from `/var/lib/docker/containers`, so the stack is Linux-specific as written.
- The example uses `grafana/grafana:latest` and `grafana/loki:latest`, which is convenient for testing but not ideal for reproducible deployments.
- The dashboard only exposes `machine` and `app`; labels such as `env`, `delivery`, and `image` are ingested by the example config but are not surfaced in the dashboard UI.
- The Docker logging tag format must remain `{{.ImageName}}|{{.Name}}` unless you also update the regex stage in `promtail-config.yml`.

## Operational Notes

- If `Machine` is empty in Grafana, Loki likely does not contain streams with `job="containerlogs"`.
- If `Machine` works but `App` is empty, the `app` label extraction from Docker log tags is probably failing.
- If the dashboard imports correctly but shows no logs, verify that the selected datasource points to the same Loki instance Promtail is writing to.
