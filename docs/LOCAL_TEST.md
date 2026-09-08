# Local Windows / Docker Desktop test

This uses the same `central/` and `agent/` Compose files as production. Run it
on the same Windows machine running Docker Desktop.

## 1. Start the central stack

Copy `central/.env.example` to `central/.env` and set these local values:

```env
LOG_RETENTION=1d
LOG_MAX_DISK=512MiB
VICTORIALOGS_BIND_ADDRESS=0.0.0.0
VICTORIALOGS_PORT=9428
GRAFANA_BIND_ADDRESS=0.0.0.0
GRAFANA_PORT=3000
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=CHANGE_ME
```

Start it:

```powershell
cd central
docker compose up -d
docker compose ps
```

Open Grafana at `http://localhost:3000`.

## 2. Start the Vector agent

Copy `agent/.env.example` to `agent/.env` and set:

```env
HOST_NAME=windows-local
VICTORIALOGS_URL=http://host.docker.internal:9428
```

`host.docker.internal` resolves from a Docker Desktop container to the Windows
host, where VictoriaLogs port 9428 is published.

```powershell
cd ..\agent
docker compose up -d
docker compose ps
docker compose logs --tail 50 vector
```

## 3. Run the test log producer

From the repository root:

```powershell
docker compose -f docs/local-test.compose.yml up -d
docker compose -f docs/local-test.compose.yml logs --tail 20 log-test
```

The container emits structured JSON logs (including one with an application
timestamp), one `ERROR` plain-text log, and one plain-text log with no severity
every five seconds. Stop it when finished:

```powershell
docker compose -f docs/local-test.compose.yml down
```

## 4. Verify in Grafana

In **Explore** select the VictoriaLogs datasource, set the time range to the
last hour, then run:

```logsql
host_name:="windows-local" service_name:="log-test"
```

Confirm these fields in the log details:

| Input | Expected fields |
| --- | --- |
| JSON `{"level":"info",...}` | `level=INFO`, `user_id=123` |
| JSON with `timestamp` and `level=warn` | Docker `timestamp`, `app_timestamp`, `level=WARN` |
| `ERROR plain text test` | `level=ERROR` |
| `plain text without severity` | `level=UNKNOWN` |

Run the same filter in **Dashboards → Logs → Logs Overview**, then choose
`windows-local`, `local`, `log-test`, and a level from the dropdowns.

For an application JSON timestamp, verify that `timestamp` remains the Docker
event time and the application's value is available as `app_timestamp`.
