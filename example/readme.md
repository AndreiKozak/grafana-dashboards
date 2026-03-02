This directory contains a minimal Docker and Promtail example for the root dashboard.

- `compose.yml` shows a container configured with Docker `json-file` logging.
- `promtail-config.yml` extracts the `app` label from the Docker log tag and adds a `machine` external label.

Use these files together with [../readme.md](../readme.md).
