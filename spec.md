# SPEC — Hardening current Grafana-logger implementation

## Goal

Improve the current implementation without redesigning the architecture.

Keep the current pipeline unchanged:

Docker services
    ↓
Vector
    ↓
VictoriaLogs
    ↓
Grafana

Do not add any new infrastructure component.

Do not reintroduce:
- OpenTelemetry Collector
- Beszel
- Loki
- Prometheus
- Tempo

The current central/agent split must remain unchanged.

---

## 1. Normalize `level` for every log event

### Problem

Current Vector config preserves `level` when the application emits structured JSON,
but plain-text logs such as PostgreSQL, Cloudflared, or other third-party services may not contain a `level` field.

Grafana dashboard filters and panels currently query using `level`, so logs without this field may be difficult to filter consistently.

### Required behavior

After parsing the log:

If `level` exists:
- convert it to string
- normalize it to uppercase

Supported common values should normalize to:

TRACE
DEBUG
INFO
WARN
ERROR
FATAL

Examples:

warn        -> WARN
warning     -> WARN
err         -> ERROR
error       -> ERROR
fatal       -> FATAL
information -> INFO
debug       -> DEBUG

If no recognized level exists:

level = "UNKNOWN"

Do not drop a log because level detection fails.

### Optional simple inference

For plain-text logs only, basic inference is allowed if it stays simple.

Examples:

"ERROR failed connection" -> ERROR
"FATAL database..."        -> FATAL
"WARNING ..."              -> WARN

Do not create a complex regex/parser framework.

If no obvious match:

UNKNOWN

### Acceptance

Every event stored in VictoriaLogs must contain:

level

and Grafana's Level variable must work for both:
- JSON application logs
- plain-text third-party logs

---

## 2. Protect the ingestion timestamp

### Problem

Vector currently parses JSON and merges it into the Docker event.

If an application emits:

{
  "timestamp": "..."
}

it may overwrite the timestamp supplied by Docker.

This can produce:
- invalid timestamps
- logs appearing at the wrong time
- ingestion/query problems

### Required behavior

Before parsing the application message:

save the original Docker event timestamp.

For example conceptually:

docker_timestamp = .timestamp

Then parse and merge application JSON.

After merge:

`.timestamp` must always be restored to the original Docker timestamp.

The Docker timestamp is the canonical ingestion timestamp.

### Preserve application timestamp

If the parsed application JSON contains its own timestamp:

preserve it separately as:

app_timestamp

Example:

Input application log:

{
  "timestamp": "2026-09-08T10:30:00Z",
  "level": "ERROR",
  "message": "database timeout"
}

Result:

timestamp     = Docker event timestamp
app_timestamp = 2026-09-08T10:30:00Z
level         = ERROR
message       = database timeout

If the application does not provide a timestamp:

do not create a fake `app_timestamp`.

### Acceptance

- Application JSON can never overwrite the canonical Vector/Docker timestamp.
- `_time` in VictoriaLogs is always based on Docker event time.
- Application timestamp is still queryable when present.

---

## 3. Add a clean local Windows / Docker Desktop test path

### Goal

Make it possible to test the exact production pipeline locally on Windows using Docker Desktop before deploying to Proxmox.

Do not create a separate implementation.

Use the same:

agent/
central/

configuration used in production.

### Add documentation

Create:

docs/LOCAL_TEST.md

Document a minimal local test.

### Local central configuration

For local Docker Desktop testing:

VICTORIALOGS_BIND_ADDRESS=0.0.0.0
VICTORIALOGS_PORT=9428

Grafana can be exposed locally:

GRAFANA_BIND_ADDRESS=0.0.0.0
GRAFANA_PORT=3000

Use:

LOG_RETENTION=1d
LOG_MAX_DISK=512MiB

to keep local resource usage small.

### Local agent configuration

Use:

HOST_NAME=windows-local
VICTORIALOGS_URL=http://host.docker.internal:9428

Explain why `host.docker.internal` is used:
Vector runs inside Docker and must reach the VictoriaLogs port published by the host.

### Add a minimal test container

Document or add an optional compose example such as:

log-test:
  image: alpine
  command:
    - sh
    - -c
    - |
      while true; do
        echo '{"level":"info","message":"hello central logging","user_id":123}'
        sleep 5
      done
  labels:
    logging.enabled: "true"
    logging.service: "log-test"
    logging.environment: "local"

The test must verify:

container
→ Docker logs
→ Vector
→ VictoriaLogs
→ Grafana

### Local test checklist

Document commands to verify:

1. central stack starts
2. Vector starts
3. Vector sees the labelled container
4. VictoriaLogs receives data
5. Grafana datasource works
6. Logs Overview displays `log-test`
7. filter by:
   - host_name
   - service_name
   - environment
   - level

Also test one plain-text log:

echo "ERROR plain text test"

and confirm:

level = ERROR

Also test:

echo "plain text without severity"

and confirm:

level = UNKNOWN

---

## 4. Do not change these working behaviors

Preserve:

- `logging.enabled=true` opt-in model
- Docker socket mounted read-only
- JSON structured fields
- plain-text message support
- secret redaction
- message truncation
- direct Vector -> VictoriaLogs flow
- Vector disk buffer
- `drop_newest`
- host_name/service_name/environment stream fields
- VictoriaLogs 2-day production retention
- Grafana provisioning
- current `central/` and `agent/` directory structure

Do not hard-code application/container names.

---

## 5. Dashboard compatibility

After level normalization, make sure the existing Logs Overview dashboard still works.

The dashboard should support:

Host
Environment
Service
Level
Time range

Expected Level values:

TRACE
DEBUG
INFO
WARN
ERROR
FATAL
UNKNOWN

Update Grafana dashboard queries only if required for the normalized values.

Prefer uppercase values consistently.

---

## 6. Tests / validation

At minimum validate these cases:

### Structured JSON

Input:

{"level":"error","message":"save failed","player_id":123}

Expected:

level = ERROR
message = save failed
player_id = 123

### JSON with custom timestamp

Input:

{"timestamp":"2020-01-01T00:00:00Z","level":"warn","message":"test"}

Expected:

timestamp = Docker timestamp
app_timestamp = 2020-01-01T00:00:00Z
level = WARN

### Plain text with level

Input:

ERROR database unavailable

Expected:

level = ERROR
message = ERROR database unavailable

### Plain text without level

Input:

player connected

Expected:

level = UNKNOWN
message = player connected

### Unusual JSON fields

Input:

{
  "foo":"bar",
  "request_id":"abc",
  "message":"hello"
}

Expected:

foo and request_id remain available.
level = UNKNOWN.

---

## 7. Acceptance criteria

Implementation is complete when:

✓ Current architecture remains Vector -> VictoriaLogs -> Grafana

✓ Every log contains normalized `level`

✓ Missing severity becomes UNKNOWN

✓ Common plain-text severity can be detected simply

✓ Docker timestamp cannot be overwritten by application JSON

✓ Application timestamp is preserved as app_timestamp

✓ Structured custom fields remain intact

✓ Existing secret redaction still works

✓ Existing bounded disk buffer still works

✓ Grafana dashboard works with normalized levels

✓ A Windows Docker Desktop local test is documented

✓ Local test uses the same central/agent implementation as production

✓ No new infrastructure components are added

---

## Implementation principle

Keep changes small.

Do not redesign working components.

Prefer a few explicit Vector transformations and clear documentation over new abstractions.

The project should remain understandable and operable by one person running a small home server.