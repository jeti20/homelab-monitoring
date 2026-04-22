# Homelab Monitoring Stack

A minimal monitoring setup using the **Prometheus textfile collector pattern**. A Bash script writes `.prom` files that Node Exporter exposes to Prometheus, with Grafana for visualization.

## Architecture

```
simulate.sh → textfiles/my_script.prom → Node Exporter (:9100) → Prometheus (:9090) → Grafana (:3000)
```

Instead of running scripts as standalone exporters, they write metrics in Prometheus exposition format to `.prom` files. Node Exporter picks them up via `--collector.textfile.directory` and exposes them alongside its own metrics.

## Stack

| Service | Image | Port |
|---|---|---|
| Node Exporter | `prom/node-exporter` | 9100 |
| Prometheus | `prom/prometheus` | 9090 |
| Grafana | `grafana/grafana` | 3000 |

## Getting Started

**Prerequisites:** Docker, Docker Compose

```bash
# Start all services
docker-compose up -d

# Run the simulation script
./simulate.sh
```

| URL | Credentials |
|---|---|
| Grafana — http://localhost:3000 | admin / admin |
| Prometheus — http://localhost:9090 | — |
| Node Exporter — http://localhost:9100/metrics | — |

## Simulation Script

`simulate.sh` mimics a real script with three possible outcomes and writes the result to `textfiles/my_script.prom`.

```bash
./simulate.sh             # random outcome (60% success, 20% no_result, 20% error)
./simulate.sh success     # force success  (exit code 0)
./simulate.sh no_result   # force no result (exit code 1)
./simulate.sh error       # force error    (exit code 2)
```

## Metrics

| Metric | Type | Description |
|---|---|---|
| `my_script_last_exit_status` | Gauge | Last run outcome: `0` success, `1` no result, `2` error |
| `my_script_last_run_timestamp_seconds` | Gauge | Unix timestamp of the last run |

Prometheus stores the history of both metrics. Query `my_script_last_exit_status` in the expression browser to see a timeline of every run and its outcome.

## Key Design Decisions

**Atomic writes** — the script writes to a `.tmp` file and then moves it into place, preventing Prometheus from reading a partially-written file.

**No standalone exporter** — scripts don't need to run as long-lived HTTP servers. They just write a file and exit, which is safer and simpler in cron-based environments.

**Scrape interval** — Prometheus scrapes Node Exporter every 15 seconds (`prometheus/prometheus.yml`). New metric values appear in Prometheus within 15 seconds of the script running.

## Applying This Pattern to Your Own Script

Add this block at the end of any Bash script to expose its result as metrics:

```bash
EXIT_CODE=$?
TIMESTAMP=$(date +%s)
PROM_FILE="/path/to/textfiles/your_script.prom"

cat > "${PROM_FILE}.tmp" <<EOF
your_script_last_exit_status $EXIT_CODE
your_script_last_run_timestamp_seconds $TIMESTAMP
EOF

mv "${PROM_FILE}.tmp" "$PROM_FILE"
```

Make sure the `textfiles/` directory is bind-mounted into the Node Exporter container (see `docker-compose.yml`).
