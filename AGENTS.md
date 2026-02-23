# AGENTS.md — Repository Guide for Coding Agents

## Overview

This is a **pure infrastructure/configuration repository**. It contains no application source
code. The entire codebase consists of Docker Compose YAML files and inline service configurations
for a lightweight monitoring and logging stack (VictoriaMetrics, Fluent Bit, VictoriaLogs,
Grafana, Blackbox Exporter, Speedtest Exporter). It is optimized for ARM/low-power hardware
such as Raspberry Pi.

**Primary language: YAML only.**

---

## Directory Structure

```
compose.yml                   # Root entrypoint — selectively includes sub-stacks
compose.override.yml          # Local dev overrides (entirely commented out; not linted)
compose.production.yml        # Git-ignored; generated via `docker compose config`
.dclintrc.yml                 # dclint (Docker Compose linter) configuration
.github/
  workflows/lint.yml          # CI: runs dclint on pull_request
  dependabot.yml              # Monthly GitHub Actions updates
<service>/
  compose.yml                 # Service runtime definition
  compose.config.yml          # Docker Config objects with inline service configuration
```

Each service lives in its own subdirectory (`fluent-bit/`, `victoriametrics/`,
`victorialogs/`, `grafana/`, `blackbox_exporter/`, `speedtest_exporter/`).

---

## Commands

### Lint

```bash
# Install the linter (one-time)
npm install --global dclint

# Run lint check (mirrors what CI runs)
dclint --config=.dclintrc.yml --recursive .

# Auto-fix lint issues
dclint --config=.dclintrc.yml --recursive . --fix
```

There are no build, test, or compile steps. Linting is the only automated quality check.

### Deploy

```bash
# Start the full stack (compose.override.yml is applied automatically)
docker compose up -d

# Generate the resolved production compose file (git-ignored, for inspection only)
docker compose config --no-normalize --no-interpolate --output compose.production.yml
```

### CI

CI (`.github/workflows/lint.yml`) runs on **pull_request** only and executes:
```
npm install --g dclint && dclint --config=.dclintrc.yml --recursive .
```

---

## Lint Rules (`.dclintrc.yml`)

| Rule | Setting | Effect |
|------|---------|--------|
| `require-quotes-in-ports` | `1` (warning) | Port mappings must be quoted strings |
| `service-image-require-explicit-tag` | `0` (disabled) | `:latest` image tags are allowed |
| `services-alphabetical-order` | `0` (disabled) | Service ordering is unrestricted |
| `no-unbound-port-interfaces` | `0` (disabled) | Binding to `0.0.0.0` is allowed |

**Excluded from linting:** `compose.config.yml` files and `compose.override.yml`.

Lint rule reference: https://github.com/zavoloklom/docker-compose-linter/blob/main/docs/rules.md

---

## YAML Style Guidelines

### File structure

- Every compose file starts with `---` (YAML document separator).
- Use 2-space indentation throughout. No tabs.
- Keys use snake_case (e.g., `metric_namespace`, `path.rootfs`, `interval_sec`).
- Environment variable names use SCREAMING_SNAKE_CASE (e.g., `GF_SECURITY_ADMIN_USER`).

### Compose file conventions

- Always use `compose.yml` as the filename, never `docker-compose.yml`.
- Config-only files are always named `compose.config.yml`.
- Each sub-stack compose file must declare a `name:` field at the top level.
- Sub-stacks include their own config file via `include: [compose.config.yml]` (relative path).
- The root `compose.yml` includes sub-stacks via relative paths; unused stacks are commented out.

### Services

- Every service must specify:
  - `image:` — use `latest` tag (explicit versioning is not enforced here)
  - `container_name:` — must match the service name exactly
  - `restart: unless-stopped`
  - `hostname:` — must match the service name exactly
- Ports must be **quoted strings**: `'8428:8428'`, not `8428:8428`.
- Configs injected at runtime go in `configs:` blocks with `source:` and `target:` fields.
- Read-only configs use `mode: 444`.

### Naming conventions

| Resource | Convention | Example |
|----------|-----------|---------|
| Directory names | lowercase, hyphen or underscore | `fluent-bit/`, `blackbox_exporter/` |
| Service names | lowercase, hyphen-separated | `victoria-metrics`, `fluent-bit` |
| Container names | match service name | `container_name: victoria-metrics` |
| Network names | lowercase, underscore-separated | `victoria_metrics_network` |
| Volume names | lowercase, underscore-separated | `victoria_metrics_data` |
| Config object names | match target filename | `fluent-bit.yml`, `scrape.yaml` |
| Environment variables | SCREAMING_SNAKE_CASE | `GF_INSTALL_PLUGINS` |

### Networks and volumes

- All named networks and volumes use an explicit `name:` field to ensure consistency
  regardless of the Docker Compose project name.
- Networks intended to be shared across stacks must set `attachable: true`.
- External networks (e.g., Traefik) are declared with `external: true`.

### Inline configurations (`compose.config.yml`)

- Inline service config is stored in `compose.config.yml` as a Docker Config object using
  the `content: |` block scalar.
- The Config object `name:` matches the target file name (`fluent-bit.yml`, `scrape.yaml`, etc.).
- These files are excluded from dclint (they are not valid top-level compose files).
- Fluent Bit pipelines follow the `pipeline.inputs / filters / outputs` structure.
- Prometheus scrape configs follow standard scrape job format with `job_name`, `static_configs`.

### Comments

- Comments that disable optional stack includes use `# - path/to/compose.yml` format.
- Comments on configuration options use `# GF_SERVER_ROOT_URL: ...` (hash + space + content).
- Lint rule comments reference the rules documentation URL at the top of `.dclintrc.yml`.

---

## Infrastructure Patterns

### Restart policy

All services use `restart: unless-stopped` — they restart automatically on failure but not
when manually stopped.

### Dependency management

Use `depends_on:` with explicit conditions:
- `condition: service_completed_successfully` — for init/setup containers
- `condition: service_started` — for soft dependencies

### DNS

Services that need reliable external DNS (e.g., blackbox prober) set:
```yaml
dns:
  - 1.1.1.1
```

### User mapping

Services that need host UID/GID alignment use environment variable substitution:
```yaml
user: ${PUID}:${PGID}
```

### Network topology

- `victoria_metrics_network` — scrape network; attachable so external stacks can join
- `victoria_logs_network` — log ingestion network; attachable for Fluent Bit and other log shippers
- `grafana_network` — Grafana internal network connecting Grafana to VictoriaMetrics and VictoriaLogs
- `blackbox_network` — Blackbox Exporter network
- `speedtest_network` — Speedtest Exporter network
- `traefik_public_network` — external, managed by a separate Traefik stack (optional; declared external)

---

## Known Gotchas

### Docker Compose variable interpolation in `content: |` blocks

The `content: |` block in `compose.config.yml` files is processed by Docker Compose for
variable substitution before being written to the container. This means:

- `${MY_VAR}` in `content: |` will be substituted with the host environment variable `MY_VAR`
  (or empty string if unset), **not** left for the service to resolve at runtime.
- `$${MY_VAR}` in `content: |` produces the literal string `${MY_VAR}` in the file written
  to the container — use this only when the service itself should perform variable substitution.
- **Do not** use `${TAG}` or similar Fluent Bit template variables directly in `content: |`
  blocks. Use static values or Fluent Bit's `modify` filter with hardcoded field values instead.

### Fluent Bit `modify` filter for log stream labeling

Fluent Bit's `${TAG}` placeholder is only reliable in specific contexts (e.g., `rewrite_tag`
output). In `modify` filter `add:` operations, it is resolved by Docker Compose (not Fluent Bit)
and will be empty if `TAG` is not set in the host environment. Use per-tag `modify` filters
with hardcoded `log_source` values instead:

```yaml
- name: modify
  match: host.syslog
  add: log_source host.syslog
```

### Shared network `external: true` conflict

Declaring a shared network as `external: true` in one sub-stack while another sub-stack
declares it as `attachable: true` causes a merge conflict when `docker compose config` resolves
them. **Fix:** never use `external: true` for shared cross-stack networks. Instead, declare all
shared networks in `compose.override.yml` with only a `name:` field, and reference them in
sub-stacks without `external: true`.

### Host log file format (syslog)

On modern Ubuntu/Debian systems, `/var/log/syslog` uses ISO 8601 timestamps
(`2026-02-23T19:25:30.485528+00:00 hostname app[pid]: message`), **not** BSD syslog format.
Fluent Bit's `syslog-rfc3164` parser will fail to parse these lines. Use no parser (raw line
stored in the `log` field) and send to VictoriaLogs with `_msg_field=log`.

### VictoriaLogs `/insert/jsonline` field mapping

When using Fluent Bit's `http` output to send to VictoriaLogs `/insert/jsonline`:

- `_msg_field` must match the actual record field containing the log message:
  - Docker container logs (docker json format): `log`
  - Raw tail input (no parser): `log`
  - Syslog-rfc5424 parsed: `message`
- `_time_field` must match the timestamp field:
  - Docker container logs: `time`
  - Fluent Bit json_lines with `json_date_format: iso8601`: `date`
- `_stream_fields` must reference fields that actually exist in the record body.
  `tag` is **not** a record field — inject it explicitly via `modify` filter.

### cgroup v2 Docker metrics

On hosts running cgroup v2 (default on Ubuntu 22.04+), Fluent Bit's `docker` input plugin
reports `cpu_used` in **microseconds** (`usage_usec`), not nanoseconds. On cgroup v1 it is
nanoseconds. `mem_used` and `mem_limit` are bytes on both versions.

---

## Adding a New Service

1. Create a new directory: `<service-name>/`
2. Create `<service-name>/compose.yml` with `name:`, `services:`, and optional `include:`.
3. If the service needs a config file, create `<service-name>/compose.config.yml` with the
   config as a `configs:` block using `content: |`.
4. Add the include entry (commented out by default) to the root `compose.yml`:
   ```yaml
   # - <service-name>/compose.yml
   ```
5. Run `dclint --config=.dclintrc.yml --recursive .` to verify no lint errors.
