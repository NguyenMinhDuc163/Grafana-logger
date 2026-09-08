# Grafana-logger

A small, central observability stack for a home server. It separates the
logging backend from Docker workload hosts, so adding another host means
deploying one lightweight Vector agent rather than another Grafana stack.

```text
workload host: Docker -> Vector -> VictoriaLogs <- Grafana : observability LXC
Proxmox / hosts: Beszel agents ----------------> Beszel Hub : observability LXC
```

Grafana is deliberately logs-only here. Beszel handles host and container
metrics. The central stack keeps logs for three days and caps data at 2 GiB by
default, which suits a small machine with low log volume.

## Repository layout

- `central/` — VictoriaLogs, Grafana and Beszel Hub; deploy on the observability LXC.
- `agent/` — Vector only; deploy one copy on every Docker workload host.
- `docs/` — architecture, deployment and service-onboarding notes.

## Central deployment

Copy `central/.env.example` to `central/.env`, set the private-LAN address and
a strong Grafana password, then run:

```bash
cd central
docker compose config --quiet
docker compose pull
docker compose up -d
```

Only permit workload hosts to reach the VictoriaLogs ingestion port (9428). The
Grafana and Beszel ports bind to loopback by default; publish them through a
reverse proxy or Cloudflare Tunnel if needed.

## Agent deployment

Copy `agent/.env.example` to `agent/.env`. Set `HOST_NAME` (for example
`dev-server`) and `VICTORIALOGS_URL` to the central private-LAN endpoint.

```bash
cd agent
docker compose config --quiet
docker compose pull
docker compose up -d
```

Vector is the only service with read-only Docker socket access. It has a
256 MiB disk buffer and drops newest logs only if that buffer becomes full;
applications never depend on it.

## Add a service

Opt in on the application's own Docker Compose service:

```yaml
labels:
  logging.enabled: "true"
  logging.service: "nro-admin"
  logging.environment: "production"
logging:
  driver: local
  options:
    max-size: "20m"
    max-file: "3"
```

No central configuration change is needed. JSON log fields are preserved;
plain-text logs are still searchable as the message. See
[docs/ADD_SERVICE.md](docs/ADD_SERVICE.md) for the complete short guide.

## Basic troubleshooting

Run `docker compose ps` in the relevant directory. On a workload host, inspect
`docker compose logs --tail=100 vector` and confirm the target container's
labels with `docker inspect`. In Grafana, use **Explore** with the VictoriaLogs
datasource and filter with `host_name:="dev-server"` or
`service_name:="nro-admin"`.

See [docs/DEPLOY.md](docs/DEPLOY.md) for migration and network details.
