# llamacpp-monitoring

Prometheus + Grafana monitoring for [llama.cpp](https://github.com/ggml-org/llama.cpp) inference servers. Activate the environment, point it at a llama.cpp instance, and get a pre-built dashboard.

## Usage

```bash
# Monitor llama.cpp on a remote host
LLAMACPP_HOST=192.168.0.126 flox activate -s

# Monitor llama.cpp on localhost (default)
flox activate -s
```

Then open:

- **Grafana**: http://localhost:3000 (anonymous access enabled)
- **Prometheus**: http://localhost:9090

## Platform support

The `llamacpp-flox-monitoring` package is currently pinned via `store-path` to version 0.9.1. Because `store-path` uses a single Nix store hash for all platforms, it will only work on the platform it was built for. This will be migrated to `pkg-path` once the package is published to FloxHub.

## Environment variables

Override any variable at activation time: `VAR=value flox activate -s`

| Variable | Default | Description |
|----------|---------|-------------|
| `LLAMACPP_HOST` | `localhost` | llama.cpp hostname or IP (also used for health check) |
| `LLAMACPP_SCRAPE_HOST` | `127.0.0.1` | Address Prometheus uses to scrape llama.cpp |
| `LLAMACPP_PORT` | `8080` | llama.cpp server port |
| `PROMETHEUS_HOST` | `0.0.0.0` | Prometheus listen address |
| `PROMETHEUS_PORT` | `9090` | Prometheus listen port |
| `GF_SERVER_HTTP_ADDR` | `0.0.0.0` | Grafana listen address |
| `GF_SERVER_HTTP_PORT` | `3000` | Grafana listen port |
| `GF_SECURITY_ADMIN_PASSWORD` | `admin` | Grafana admin password |

### Host sync

When `LLAMACPP_HOST` is overridden but `LLAMACPP_SCRAPE_HOST` is not, the scrape host is automatically synced so Prometheus targets the correct address.

## Dashboard

8-panel dashboard monitoring llama.cpp inference performance.

| Panel | Type | Metric(s) |
|-------|------|-----------|
| Generation Speed (tokens/sec) | timeseries | `llamacpp:predicted_tokens_seconds` |
| Prompt Speed (tokens/sec) | timeseries | `llamacpp:prompt_tokens_seconds` |
| Token Throughput (rate/min) | timeseries | `rate(llamacpp:tokens_predicted_total[1m])`, `rate(llamacpp:prompt_tokens_total[1m])` |
| Decode Rate | timeseries | `rate(llamacpp:n_decode_total[1m])` |
| Cumulative Tokens | timeseries | `llamacpp:tokens_predicted_total`, `llamacpp:prompt_tokens_total` |
| Active Requests | stat | `llamacpp:requests_processing` |
| Deferred Requests | stat | `llamacpp:requests_deferred` |
| Busy Slots per Decode | timeseries | `llamacpp:n_busy_slots_per_decode` |

## How it works

This environment installs `prometheus`, `grafana`, `curl`, `jq`, and the `llamacpp-flox-monitoring` package (v0.9.1, pinned via `store-path`). On activation:

1. `llamacpp-monitoring-init` is sourced -- sets defaults, syncs scrape host, creates mutable dirs in `$FLOX_ENV_CACHE`, generates Prometheus and Grafana provisioning configs with expanded variables
2. `llamacpp-monitoring-prometheus` starts Prometheus with the generated config
3. `llamacpp-monitoring-grafana` starts Grafana with the pre-built dashboard

Static assets (dashboard JSON, `grafana.ini`) live in the immutable Nix store. Mutable state (Prometheus TSDB, Grafana data, generated configs) lives in `$FLOX_ENV_CACHE` -- the project directory stays clean.

## Manifest

> **Note**: The `llamacpp-flox-monitoring` package is pinned via `store-path` to version 0.9.1. This will be migrated to `pkg-path` once the package is published to FloxHub.

```toml
version = 1

[install]
prometheus.pkg-path = "prometheus"
grafana.pkg-path = "grafana"
curl.pkg-path = "curl"
jq.pkg-path = "jq"
llamacpp-flox-monitoring.store-path = "/nix/store/s297bkawd0fg0niqjzqamly68pq57gyk-llamacpp-flox-monitoring-0.9.1"

[hook]
on-activate = '''
  . llamacpp-monitoring-init
'''

[services]
prometheus.command = "llamacpp-monitoring-prometheus"
grafana.command = "llamacpp-monitoring-grafana"

[options]
```

## Troubleshooting

```bash
# Check Prometheus targets
curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {scrapeUrl, health}'

# Check llama.cpp health directly
curl -s http://$LLAMACPP_HOST:$LLAMACPP_PORT/health

# Check llama.cpp metrics directly
curl -s http://$LLAMACPP_HOST:$LLAMACPP_PORT/metrics | head -20

# View service logs
flox services logs prometheus
flox services logs grafana

# Restart services after config change
flox services restart
```
