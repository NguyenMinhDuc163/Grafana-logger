# Architecture

`central/` runs on the observability LXC and contains VictoriaLogs and Grafana.
`agent/` runs on each Docker workload host and contains only Vector.

```text
Docker containers --(stdout/stderr)--> Vector --(private LAN)--> VictoriaLogs
                                                               --> Grafana
```

Only containers labelled `logging.enabled=true` are collected. Vector adds the
common fields `host_name`, `service_name`, `environment`, `container_name`,
`container_id` and `source_type`; application JSON fields remain intact. The
VictoriaLogs stream consists only of `host_name`, `service_name` and
`environment` to keep cardinality low.

Applications never depend on either compose project. If Vector or the central
LXC is unavailable, Docker keeps the application's rotated local logs and
Vector drops only new events after its bounded disk buffer is full.
