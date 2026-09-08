Được. Bạn có thể đưa agent spec ngắn gọn này:

# Spec refactor `Grafana-logger`

## Mục tiêu

Biến project hiện tại từ logging riêng cho game backend thành **central logging dùng chung cho toàn bộ Docker services trên `dev-server`**.

Giữ kiến trúc đơn giản:

```text
Docker services
   ↓
Vector
   ↓
VictoriaLogs
   ↓
Grafana
```

Không dùng Beszel.
Không dùng OpenTelemetry Collector ở phase hiện tại.

## Kiến trúc deployment

### `dev-server`

Chạy:

```text
Applications
Postgres
Cloudflared
Portainer
Vector
```

Vector đọc Docker logs qua:

```text
/var/run/docker.sock:ro
```

và gửi log sang VictoriaLogs ở VM/LXC observability.

### `observability`

Chạy:

```text
VictoriaLogs
Grafana
```

Grafana dùng VictoriaLogs làm datasource.

## Cấu trúc repo

Giữ **một repository**, nhưng tách:

```text
Grafana-logger/
├── agent/
│   ├── compose.yml
│   └── vector/
│       └── vector.yaml
│
├── central/
│   ├── compose.yml
│   └── grafana/
│       └── provisioning/
│
├── docs/
│   ├── DEPLOY.md
│   └── ADD_SERVICE.md
│
└── README.md
```

## Vector

Giữ cơ chế opt-in bằng Docker labels:

```yaml
labels:
  logging.enabled: "true"
  logging.service: "dragon-ball"
  logging.environment: "production"
```

Chỉ collect container có:

```text
logging.enabled=true
```

Không hard-code container name.

Mỗi log phải có metadata chung:

```text
host_name
service_name
environment
container_name
container_id
level
message
timestamp
```

Nếu log là JSON thì giữ thêm các field riêng như:

```text
player_id
user_id
request_id
duration_ms
```

Nếu log là plain text thì vẫn ingest bình thường.

## Host config

Vector agent nhận:

```env
HOST_NAME=dev-server
VICTORIALOGS_URL=http://<observability-ip>:9428
```

để sau này thêm host mới không cần sửa central stack.

## VictoriaLogs

Dùng làm nơi lưu log duy nhất.

Default:

```env
LOG_RETENTION=2d
LOG_MAX_DISK=1GiB
```

Mục tiêu:

* giữ log 1–2 ngày
* không cho log làm đầy disk
* query theo host/service/environment

Không dùng field như `player_id`, `request_id` làm stream field.

## Grafana

Grafana là UI chính để xem log.

Chỉ cần một dashboard đơn giản:

```text
Logs Overview
```

Có filter:

```text
Host
Service
Environment
Level
Time range
```

Có các panel:

```text
log volume
error count
recent errors
logs table
```

Không tạo dashboard riêng cho từng service.

Không thêm Prometheus/metrics vào phase này.

## Docker service hiện tại

Các service chỉ cần thêm labels và log rotation:

```yaml
labels:
  logging.enabled: "true"
  logging.service: "postgres"
  logging.environment: "production"

logging:
  driver: local
  options:
    max-size: "20m"
    max-file: "3"
```

Không cần sửa code nếu hiện tại:

```bash
docker logs <container>
```

đã xem được log.

## Loại bỏ

Xóa khỏi runtime:

```text
OpenTelemetry Collector
Beszel
```

Không thêm:

```text
Loki
Prometheus
Tempo
Kubernetes
HA
systemd log collection
```

trong refactor này.

## Yêu cầu quan trọng

Application **không được phụ thuộc vào logging stack**.

Nếu:

```text
Vector chết
VictoriaLogs chết
Grafana chết
```

thì:

```text
game
admin
postgres
```

vẫn phải chạy bình thường.

## Acceptance criteria

Hoàn thành khi:

```text
✓ Vector chạy trên dev-server
✓ VictoriaLogs + Grafana chạy trên observability host
✓ Vector gửi trực tiếp log sang VictoriaLogs
✓ Grafana xem/search được log
✓ Filter được theo host/service/level/time
✓ Thêm service mới chỉ cần Docker labels
✓ JSON log giữ structured fields
✓ Plain-text log vẫn hoạt động
✓ retention mặc định 2 ngày
✓ log storage giới hạn 1 GiB
✓ Docker local logs có rotation
✓ không còn OTel Collector
✓ không còn Beszel
```

Một câu nên thêm cho agent:

> Keep the project small and operationally simple. Do not introduce new infrastructure or abstractions unless required by this spec. Prefer removing unnecessary layers over adding new ones.

Đây là hướng mình thấy sạch nhất cho setup hiện tại của bạn: **Proxmox lo host/LXC, Portainer lo Docker, còn project này chỉ lo centralized logging: `Vector → VictoriaLogs → Grafana`.**
