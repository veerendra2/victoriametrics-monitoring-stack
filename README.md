# Victoriametrics Monitoring Stack

> _Good enough observability, minimal complexity, and low resource usage._

A lightweight, Docker Compose–based monitoring and logging stack designed for home servers and edge devices such as Raspberry Pi.

This project replaces traditional Prometheus-based setups with [VictoriaMetrics](https://victoriametrics.com/) for metrics, [Fluent Bit](https://fluentbit.io/) as a unified agent for basic system metrics and log collection, and [VictoriaLogs](https://docs.victoriametrics.com/victorialogs/) for centralized log storage.

Fluent Bit is used to keep the stack simple by providing an all-in-one metrics and logs push model, but it is not intended to fully replace dedicated exporters such as [node-exporter](https://github.com/prometheus/node_exporter) or [cAdvisor](https://github.com/google/cadvisor). The focus is on minimal resource usage, ease of deployment, and practicality for small-scale environments.

The stack is optimized for ARM architectures and low-power hardware, making it ideal for home labs, Raspberry Pi clusters, and small self-hosted servers.
