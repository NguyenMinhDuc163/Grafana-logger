# Add a service

Add these settings to the application Compose service, then recreate that
service. No change to Vector, Grafana or VictoriaLogs is needed.

```yaml
services:
  nro-admin:
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

Write logs to `stdout` or `stderr`. A JSON object on one line is preserved as
structured fields (for example `level`, `request_id` or `player_id`); ordinary
plain-text logs are kept as `message` too. Never log real credentials.

Check the labels and local fallback logs with:

```bash
docker inspect nro-admin --format '{{json .Config.Labels}}'
docker logs --tail 100 nro-admin
```
