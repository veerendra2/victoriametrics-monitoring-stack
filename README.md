# Victoriametrics Monitoring Stack

> _Good enough observability, minimal complexity, and low resource usage._

A lightweight, Docker Compose–based monitoring and logging stack designed for home servers and edge devices such as Raspberry Pi.

This project replaces traditional Prometheus-based setups with [VictoriaMetrics](https://victoriametrics.com/) for metrics, [Fluent Bit](https://fluentbit.io/) as a unified agent for basic system metrics and log collection, and [VictoriaLogs](https://docs.victoriametrics.com/victorialogs/) for centralized log storage.

Fluent Bit is used to keep the stack simple by providing an all-in-one metrics and logs push model, but it is not intended to fully replace dedicated exporters such as [node-exporter](https://github.com/prometheus/node_exporter) or [cAdvisor](https://github.com/google/cadvisor). The focus is on minimal resource usage, ease of deployment, and practicality for small-scale environments.

The stack is optimized for ARM architectures and low-power hardware, making it ideal for home labs, Raspberry Pi clusters, and small self-hosted servers.

## Deploy

```bash
git clone https://github.com/veerendra2/victoriametrics-monitoring-stack.git

cd victoriametrics-monitoring-stack

docker compose -f compose.yml up -d
```

- VictoriaMetrics UI - http://localhost:8428/vmui/

To get single docker compose file, run below

```bash
docker compose config --no-normalize --no-interpolate --output compose.production.yml
docker compose -f compose.production.yml up -d
```

## Docker Compose Lint

```bash
npm install --global dclint

# Run lint
dclint --config=.dclintrc.yml --recursive .

# Fix lint issues
dclint --config=.dclintrc.yml --recursive . --fix
```

## Resources

### VictoriaMetrics

- [Scrape config examples](https://docs.victoriametrics.com/victoriametrics/scrape_config_examples/)

### Fluent Bit

- [Node exporter metrics](https://docs.fluentbit.io/manual/data-pipeline/inputs/node-exporter-metrics)

```bash
# To see available config for input `node_exporter_metrics`
docker run -it --rm fluent/fluent-bit:latest /fluent-bit/bin/fluent-bit -i node_exporter_metrics -h
```
