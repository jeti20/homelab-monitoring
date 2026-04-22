# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Homelab monitoring stack demonstrating Prometheus metrics collection via the textfile collector pattern. A Bash simulation script writes `.prom` files that Node Exporter exposes to Prometheus, with Grafana for visualization.

## Running the Stack

```bash
docker-compose up -d        # Start all services (detached)
docker-compose down         # Stop all services
docker-compose logs -f      # Follow logs
```

Service URLs:
- Grafana: http://localhost:3000 (admin / admin)
- Prometheus: http://localhost:9090
- Node Exporter: http://localhost:9100/metrics

## Simulation Script

```bash
./simulate.sh               # Random outcome (60% success, 20% no_result, 20% error)
./simulate.sh success       # Force success (exit code 0)
./simulate.sh no_result     # Force no_result (exit code 1)
./simulate.sh error         # Force error (exit code 2)
```

The script writes metrics to `textfiles/my_script.prom` and persists counters in `.counters` between runs.

## Architecture

```
simulate.sh → textfiles/my_script.prom → Node Exporter (:9100) → Prometheus (:9090) → Grafana (:3000)
```

**Textfile collector pattern**: Scripts write metrics in Prometheus exposition format to `.prom` files in the `textfiles/` directory. Node Exporter reads these files via its `--collector.textfile.directory` flag, exposing them alongside system metrics. This avoids running scripts as standalone exporters.

**Atomic writes**: `simulate.sh` writes to a temp file then moves it into place, preventing Prometheus from reading a partially-written file.

**Persistent counters**: `.counters` stores cumulative run counts (`success_total`, `no_result_total`, `error_total`) so counter metrics survive script restarts.

## Key Metrics

Exposed via `my_script_*` metric family:
- `my_script_last_exit_code` — last run outcome (0/1/2)
- `my_script_last_run_timestamp_seconds` — Unix timestamp of last run
- `my_script_runs_total{result="success|no_result|error"}` — cumulative run counts

## Notes

- Code comments and script output messages are in Polish.
- Prometheus scrape interval is 15 seconds (`prometheus/prometheus.yml`).
- The `textfiles/` directory is bind-mounted into the Node Exporter container.
