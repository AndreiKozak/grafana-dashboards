# Grafana Logs Dashboard

This repository contains a Grafana dashboard for browsing Docker container logs stored in Loki.

The dashboard file is `grafana-logs-dashboard.json`.

## Features

- Filters logs by `machine`
- Filters logs by `app`
- Uses a Grafana Loki datasource variable so the dashboard can be imported into different environments

## Expected Labels

The dashboard expects Loki streams to include:

- `machine`
- `app`
- `job="containerlogs"`

The example Promtail configuration in `example/promtail-config.yml` produces those labels from Docker container logs.

## Import

1. Open Grafana.
2. Go to `Dashboards` -> `New` -> `Import`.
3. Upload `grafana-logs-dashboard.json`.
4. Select your Loki datasource when prompted.

## Example Setup

The `example/` directory contains a minimal Docker Compose and Promtail configuration.

Use:

- [example/compose.yml](example/compose.yml) for the Docker Compose example
- [example/promtail-config.yml](example/promtail-config.yml) for the Promtail configuration

## Notes

- The dashboard queries logs with `{job="containerlogs", app=~"${App:regex}", machine="$Machine"}`.
- The `Machine` variable should be selected first, then `App` will narrow the stream list for that machine.
