# Deployment

## Central LXC

Create an observability LXC with 2 vCPU, 2 GB RAM and a persistent volume (at
least 5 GB) for VictoriaLogs. Copy `central/.env.example` to `central/.env`.
Set a strong Grafana password and replace `VICTORIALOGS_BIND_ADDRESS` with the
LXC private-LAN IP (for example `10.10.1.20`). Allow only workload-host IPs to
reach port 9428.

```bash
cd central
docker compose config --quiet
docker compose pull
docker compose up -d
```

Grafana and Beszel bind to loopback by default. Put a reverse proxy or
Cloudflare Tunnel in front of them if browser access from another machine is
required. Do not publish VictoriaLogs to the internet.

## Agent on a workload host

Copy `agent/.env.example` to `agent/.env`. Set `HOST_NAME` to a stable unique
value and `VICTORIALOGS_URL` to the private LXC address, for example
`http://10.10.1.20:9428`.

```bash
cd agent
docker compose config --quiet
docker compose pull
docker compose up -d
```

The Vector disk buffer is 256 MiB by default. It lets applications continue
while VictoriaLogs restarts, but uses `drop_newest` rather than applying
backpressure when full.

## Migration from the old stack

1. Deploy and check the central LXC.
2. Deploy the new Vector agent on `dev-server`.
3. Confirm logs in Grafana Explore and the Logs Overview dashboard.
4. Stop the old combined stack only after confirmation.
5. Delete old volumes only when their retained data is no longer needed.
