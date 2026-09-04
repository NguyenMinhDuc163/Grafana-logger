# Central Logging Operations

This stack is an independent Compose project. It discovers any Docker container
labelled `logging.enabled=true`; it does not own or gate the game server.

## Start

Add the logging values from `.env.example` to your private `.env`, especially a
strong `GRAFANA_ADMIN_PASSWORD` and the three prebuilt image references. On a
production server, run only:

```bash
docker login # required only for private Docker Hub repositories
docker compose config --quiet
docker compose pull
docker compose up -d
```

`compose.yml` contains no build instructions. Production therefore
pulls ready-to-run images and does not compile binaries or install the Grafana
plugin on the server.

## Build And Push Images With GitHub Actions

Create this Docker Hub repository once:

```text
nguyenduc1603/central-logging
```

Configure `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` in this repository's
GitHub Actions secrets. `.github/workflows/docker.yml` builds all three images
on a manual run or a relevant push to `main`. Their rolling tags are
`otel-latest`, `victorialogs-latest`, and `grafana-latest`; numbered tags are
`otel-v<run_number>`, `victorialogs-v<run_number>`, and
`grafana-v<run_number>`. No commit hash is appended to image tags.

Production consumes the rolling tags by default:

```bash
docker compose pull
docker compose up -d
```

## Optional Local Build

GitHub Actions is the normal build path. If it is ever necessary to build from
a local machine, authenticate and use the build overlay:

```bash
docker login
docker compose -f compose.yml -f compose.build.yml build --pull otel-collector victorialogs grafana
docker compose -f compose.yml -f compose.build.yml push otel-collector victorialogs grafana
```

The build overlay tags each result with the `image` value from `.env`. Build on
the same CPU architecture as production. For mixed AMD64/ARM64 environments,
use a multi-platform Buildx pipeline instead of a single-platform local build.

To run locally from images just built, include the overlay:

```bash
docker compose -f compose.yml -f compose.build.yml up -d
```

Starting the game is optional for the logging stack. Start order is not
important: Vector buffers while downstream services recover, and the game has
no dependency on logging.

## Status And Logs

```bash
docker compose ps
docker compose logs --tail=100 vector
docker compose logs --tail=100 otel-collector
docker compose logs --tail=100 victorialogs
docker compose logs --tail=100 grafana
```

Every service should become `healthy`. Grafana is available at:

```text
http://SERVER_IP:GRAFANA_PORT
```

The default bind address is `127.0.0.1`. Set `GRAFANA_BIND_ADDRESS=0.0.0.0` only
when remote access is required, and protect the port with a host firewall or an
authenticated TLS reverse proxy. Anonymous access and self-signup are disabled.

## Stop

This preserves named-volume data:

```bash
docker compose down
```

The following command is destructive and permanently deletes retained logs,
Grafana state, and the Vector disk buffer:

```bash
docker compose down -v
```

## Backup

Stop the stack for a consistent filesystem snapshot. Compose volume names use
the project prefix (`central-logging_` by default); confirm them first:

```bash
docker compose down
docker volume ls --filter name=central-logging
docker run --rm -v central-logging_victorialogs-data:/data:ro -v "$PWD":/backup alpine:3.22.1 tar czf /backup/victorialogs-data.tgz -C /data .
docker run --rm -v central-logging_grafana-data:/data:ro -v "$PWD":/backup alpine:3.22.1 tar czf /backup/grafana-data.tgz -C /data .
```

Store the archives outside the repository and protect them as operational data.

## Restore

Keep the stack stopped, verify the exact volume names, and restore into empty
volumes. Restoring overwrites the destination volume contents:

```bash
docker compose down
docker volume create central-logging_victorialogs-data
docker volume create central-logging_grafana-data
docker run --rm -v central-logging_victorialogs-data:/data -v "$PWD":/backup alpine:3.22.1 sh -c 'rm -rf /data/* && tar xzf /backup/victorialogs-data.tgz -C /data'
docker run --rm -v central-logging_grafana-data:/data -v "$PWD":/backup alpine:3.22.1 sh -c 'rm -rf /data/* && tar xzf /backup/grafana-data.tgz -C /data'
docker compose pull
docker compose up -d
```

## Retention And Capacity

Change retention in `.env`, then recreate VictoriaLogs:

```env
LOG_RETENTION=30d
```

```bash
docker compose up -d --force-recreate victorialogs
```

The default container limits are Vector 0.25 CPU/192 MB, Collector 0.50
CPU/384 MB, VictoriaLogs 1 CPU/1 GB, and Grafana 0.50 CPU/384 MB. Vector's disk
buffer is capped at 1 GiB and drops newest events when full, so logging cannot
backpressure the game indefinitely.

## Add Another Application

Emit one bounded JSON object per stdout line and opt in with generic labels:

```yaml
services:
  new-application:
    labels:
      logging.enabled: "true"
      logging.service: "new-application"
      logging.environment: "production"
    logging:
      driver: local
      options:
        max-size: "50m"
        max-file: "3"
```

No VictoriaLogs, Collector, Vector, or Grafana change is required. Containers
without `logging.enabled=true` are not collected, preventing the logging stack
from ingesting its own logs. For applications on other hosts, do not publish
the current OTLP ports; first add TLS, authentication, and network allow-listing.

## Security Notes

- Vector alone receives `/var/run/docker.sock`, mounted read-only. Treat Vector
  configuration and image upgrades as host-sensitive changes.
- VictoriaLogs ports 9428 and Collector ports 4317/4318 are internal only.
- Do not put database passwords, tokens, API keys, or real credentials in log
  messages, Compose files, dashboards, or tracked environment examples.
- The Vector redaction rule is defense in depth, not a substitute for avoiding
  secret logging at the application boundary.
- Database-backed currency, item, admin, gift-code, and account audit records
  remain authoritative; VictoriaLogs is for technical investigation. 
