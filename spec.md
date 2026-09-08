Được. **Giữ chung trong một repository là hướng mình khuyên dùng**, nhưng tách rõ hai *deployment target* bên trong repo:

```text
một repository
├── central/     # chạy trên LXC observability
└── agent/       # chạy trên dev-server / các host có Docker
```

Không cần tách thành nhiều repo.

Có một điểm mình sẽ **đơn giản hóa so với thiết kế hiện tại**: bỏ OpenTelemetry Collector khỏi runtime ở giai đoạn này. Hiện pipeline của repo đang là `Vector → OTLP → OTel Collector → VictoriaLogs`; Collector ở đây chủ yếu làm trung gian chuyển log.   VictoriaLogs hỗ trợ chính thức việc nhận log **trực tiếp từ Vector** qua Elasticsearch-compatible API hoặc HTTP JSON, nên với bài toán hiện tại chỉ có logs thì OTel Collector là một tầng không cần thiết. ([VictoriaMetrics Docs][1])

Dưới đây là spec mình nghĩ bạn có thể **đưa nguyên văn cho coding agent**.

---

# SPEC — Refactor `Grafana-logger` thành Central Observability cho Home Server

## 1. Mục tiêu

Refactor repository hiện tại từ một logging stack gắn với game backend thành một **central logging + monitoring stack dùng chung cho toàn bộ các service của home server**.

Repository vẫn là **một project duy nhất**.

Kiến trúc mục tiêu:

```text
                    Proxmox

        ┌───────────────────────────────┐
        │                               │
   dev-server                     observability LXC
   workload host                  monitoring backend
        │                               │
        │                               ├── VictoriaLogs
        │                               ├── Grafana
        │                               └── Beszel Hub
        │
        ├── Docker containers
        │      ├── dragon-ball
        │      ├── nro-admin
        │      ├── postgres
        │      ├── drizzle-gateway
        │      └── cloudflared
        │
        └── Vector
              │
              │ HTTP / private LAN
              ▼
         VictoriaLogs
```

Mục tiêu chính:

```text
Beszel       = infrastructure / host / container metrics
Vector       = log collector tại từng workload host
VictoriaLogs = centralized log storage + query
Grafana      = centralized log UI
```

Không dùng Grafana để thay Beszel trong phase này.

---

# 2. Nguyên tắc thiết kế

Ưu tiên:

1. Đơn giản.
2. Dễ deploy.
3. Dễ thêm host/service mới.
4. Ít tài nguyên.
5. Không coupling với game backend.
6. Không tạo abstraction chưa cần thiết.
7. Một component chỉ nên có một trách nhiệm rõ ràng.

Không thiết kế trước cho Kubernetes, multi-region, HA cluster hoặc enterprise scale.

Hệ thống hiện chỉ cần phục vụ:

```text
1 Proxmox
1 workload LXC
~5-10 services
log volume thấp
retention khoảng 2-3 ngày
```

---

# 3. Cấu trúc repository mục tiêu

Refactor thành:

```text
Grafana-logger/
│
├── README.md
├── .env.example
│
├── central/
│   ├── compose.yml
│   │
│   ├── grafana/
│   │   └── provisioning/
│   │       ├── datasources/
│   │       └── dashboards/
│   │
│   └── beszel/
│       └── README.md
│
├── agent/
│   ├── compose.yml
│   └── vector/
│       └── vector.yaml
│
└── docs/
    ├── ARCHITECTURE.md
    ├── DEPLOY.md
    └── ADD_SERVICE.md
```

Không tạo thêm repository.

Không rename repository trong refactor này.

---

# 4. Central stack

`central/compose.yml` chạy trên LXC `observability`.

Chỉ gồm:

```text
VictoriaLogs
Grafana
Beszel Hub
```

### Không chạy ở central

Không chạy:

```text
Vector
OTel Collector
application services
database
game server
```

ở LXC này.

---

# 5. Loại bỏ OpenTelemetry Collector

Pipeline hiện tại:

```text
Docker
 ↓
Vector
 ↓
OTLP
 ↓
OpenTelemetry Collector
 ↓
VictoriaLogs
```

Refactor thành:

```text
Docker
 ↓
Vector
 ↓
VictoriaLogs
```

Vector gửi trực tiếp sang VictoriaLogs.

Ưu tiên dùng Vector `elasticsearch` sink theo integration chính thức của VictoriaLogs:

```text
Vector
   ↓
http://VICTORIALOGS_HOST:9428/insert/elasticsearch/
```

Không tự build OTLP envelope nữa.

Không giữ OTel Collector chỉ vì "có thể tương lai sẽ cần traces".

Nếu sau này dự án thật sự có distributed tracing thì OTel Collector có thể được thêm lại bằng một feature riêng.

Git history đã giữ implementation cũ nên không cần archive duplicate configuration trong repository.

---

# 6. Agent stack

`agent/compose.yml` chạy trên mỗi Docker workload host.

Hiện tại host đầu tiên là:

```text
dev-server
```

Agent stack chỉ cần:

```text
Vector
```

Vector được mount:

```text
/var/run/docker.sock:ro
```

Không expose Docker socket qua network.

Không để VictoriaLogs hoặc Grafana truy cập Docker socket.

Current implementation đã đi đúng hướng khi chỉ Vector được mount Docker socket.

---

# 7. Service discovery

Giữ cơ chế hiện tại:

```yaml
labels:
  logging.enabled: "true"
  logging.service: "dragon-ball"
  logging.environment: "production"
```

Vector chỉ ingest container:

```text
logging.enabled=true
```

Cơ chế này hiện đã tồn tại và phải được giữ nguyên.

Không hard-code container names trong Vector config.

Ví dụ các service hiện tại:

```text
dragon-ball
nro-admin
postgres
drizzle-gateway
cloudflared
```

chỉ cần thêm labels.

Portainer không cần collect log mặc định.

---

# 8. Host identity

Thêm khái niệm host chung.

Agent phải nhận:

```env
HOST_NAME=dev-server
```

và mọi log gửi lên phải có:

```text
host_name=dev-server
```

Khi sau này có:

```text
dev-server
media-server
game-server-2
```

không cần sửa central configuration.

Chỉ deploy Vector agent với:

```env
HOST_NAME=<host>
```

---

# 9. Common log schema

Loại bỏ tư tưởng schema lấy game làm trung tâm.

Mọi log nên có các field chung nếu có thể:

```text
timestamp
host_name
service_name
environment
container_name
container_id
level
message
source_type
```

Các field ứng dụng tự phát ra phải **được giữ nguyên**, không ép chuyển sang schema chung.

Ví dụ game có thể có:

```text
player_id
command
event_name
server_id
```

API có thể có:

```text
request_id
user_id
method
path
status_code
duration_ms
```

Postgres có thể có field riêng.

Vector không cần khai báo thủ công từng field như:

```text
player_id
user_id
command
event_name
...
```

Nếu log là JSON:

```json
{
  "level": "ERROR",
  "message": "Failed saving player",
  "player_id": 123
}
```

thì sau normalize vẫn phải giữ:

```text
level
message
player_id
```

---

# 10. Plain text logs

Không yêu cầu mọi container phải xuất JSON.

Nếu message không parse được JSON:

```text
FATAL: database connection failed
```

vẫn ingest bình thường:

```text
message="FATAL: database connection failed"
```

Không drop log vì parsing failure.

JSON structured logging chỉ là khuyến nghị cho các application do mình kiểm soát.

---

# 11. Secret redaction

Giữ cơ chế redaction đơn giản hiện có cho:

```text
password
passwd
token
secret
api_key
api-key
```

Không xây một DLP engine phức tạp.

Không recursively scan mọi nested object trong phase này.

Giới hạn kích thước `message` để một stack trace hoặc payload lỗi không làm Vector tiêu tốn bất thường.

Current giới hạn khoảng 64 KiB cho message có thể giữ nguyên.

---

# 12. Vector buffering

Vector phải tiếp tục hoạt động nếu VictoriaLogs restart.

Sử dụng disk buffer.

Tuy nhiên giảm từ mức hiện tại 1 GiB xuống:

```text
256 MiB
```

hoặc tối đa:

```text
512 MiB
```

do log volume hiện rất nhỏ.

Behavior khi buffer đầy:

```text
drop_newest
```

Logging không bao giờ được phép gây backpressure khiến application/game gặp lỗi.

---

# 13. Docker local logging

Central logging **không thay thế hoàn toàn** local Docker logging.

Application containers nên dùng:

```yaml
logging:
  driver: local
  options:
    max-size: "20m"
    max-file: "3"
```

Mục đích:

```text
Docker local logs
= emergency fallback

VictoriaLogs
= centralized history/search
```

Không dùng Docker local log làm storage dài hạn.

---

# 14. VictoriaLogs retention

Default:

```env
LOG_RETENTION=3d
```

VictoriaLogs command:

```text
-retentionPeriod=3d
```

và thêm giới hạn storage:

```text
-retention.maxDiskSpaceUsageBytes=2GiB
```

Cả hai phải configurable bằng environment variables.

VictoriaLogs hỗ trợ chính thức retention theo thời gian và giới hạn disk space. ([VictoriaMetrics Docs][2])

Ví dụ:

```env
LOG_RETENTION=3d
LOG_MAX_DISK=2GiB
```

Không đặt retention mặc định 14 hoặc 30 ngày cho workload hiện tại.

---

# 15. VictoriaLogs stream fields

Không dùng field game-specific như:

```text
server_id
```

làm stream dimension mặc định.

Dùng:

```text
host_name
service_name
environment
```

hoặc nếu integration cần:

```text
host_name
container_name
```

Mục tiêu là cardinality thấp và dễ query.

Không dùng:

```text
user_id
player_id
request_id
trace_id
```

làm stream field.

---

# 16. Grafana

Grafana chỉ phục vụ **logs** trong scope hiện tại.

Không dựng lại dashboard CPU/RAM/disk vì Beszel đã đảm nhận.

Chỉ cần một dashboard chính:

```text
Logs Overview
```

Có các variables:

```text
Host
Environment
Service
Level
```

Và tối đa các panel:

```text
Log volume theo thời gian
Log count theo level
Recent errors
Logs table
```

Không tạo hàng chục dashboard cho từng application.

Query cụ thể của game có thể sử dụng Explore.

---

# 17. Beszel

Thêm Beszel Hub vào `central/compose.yml`.

Không fork Beszel.

Không thêm VictoriaLogs tab vào Beszel trong phase này.

Không cố merge Grafana và Beszel thành một UI.

Hai UI là chấp nhận được:

```text
Beszel
→ monitoring

Grafana
→ logs
```

Beszel Agent được cài ở:

```text
Proxmox host
dev-server
```

theo deployment chính thức của Beszel.

Repo chỉ cần document cách connect agents tới central Beszel Hub.

---

# 18. Networking

Central services giao tiếp trên private Proxmox network.

Ví dụ:

```text
dev-server
10.10.1.90

observability
10.10.1.x
```

Vector gửi tới private IP của VictoriaLogs.

VictoriaLogs ingestion port:

```text
9428
```

không được expose công khai ra Internet.

Nếu firewall được cấu hình:

```text
allow 10.10.1.90 → observability:9428
deny external → 9428
```

Grafana và Beszel có thể được publish qua reverse proxy/Cloudflare Tunnel riêng.

Không thêm TLS/service-to-service authentication vào phase này vì traffic chỉ đi trong private LAN.

---

# 19. Resource budget

Target cho `observability` LXC:

```text
2 vCPU
2 GB RAM
8–10 GB root
5 GB log data volume
```

Central containers:

```text
VictoriaLogs
CPU <= 1
RAM <= ~768 MB - 1 GB

Grafana
CPU <= 0.5
RAM <= ~384 MB

Beszel
CPU thấp
RAM <= ~256 MB nếu phù hợp
```

Agent Vector:

```text
CPU <= 0.25
RAM <= 192 MB
```

Không tối ưu micro-resource usage nếu chưa có vấn đề thực tế.

---

# 20. Deployment independence

Workload không được phụ thuộc vào logging stack.

Nếu VictoriaLogs chết:

```text
game       vẫn chạy
postgres   vẫn chạy
admin      vẫn chạy
```

Vector buffer tạm thời.

Nếu Vector chết:

```text
application vẫn chạy
```

Logging không được xuất hiện trong:

```yaml
depends_on:
```

của application stack.

---

# 21. Documentation

`README.md` mới phải giải thích ngắn gọn:

```text
What this project is
Architecture
Central deployment
Agent deployment
Add a service
Basic troubleshooting
```

`docs/ADD_SERVICE.md` phải cho thấy chỉ cần:

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

Không yêu cầu sửa Vector/Grafana/VictoriaLogs khi thêm service.

---

# 22. `.env.example`

Phân biệt central và agent config rõ ràng.

Có thể tạo:

```text
central/.env.example
agent/.env.example
```

Central:

```env
LOG_RETENTION=3d
LOG_MAX_DISK=2GiB

GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=CHANGE_ME

BESZEL_PORT=8090
```

Agent:

```env
HOST_NAME=dev-server
VICTORIALOGS_URL=http://10.10.1.x:9428
VECTOR_BUFFER_SIZE=268435456
```

Không commit password thật.

---

# 23. CI/CD

Không mở rộng CI/CD quá mức.

Nếu hiện đang build custom Grafana image để preinstall VictoriaLogs datasource plugin thì có thể tiếp tục dùng.

Nếu VictoriaLogs custom image chỉ tồn tại để phục vụ OTel Collector/healthcheck thì xem xét bỏ và dùng image chính thức.

Sau refactor:

```text
không build OTel image
```

nếu Collector đã bị loại bỏ.

Không thêm:

```text
Terraform
Ansible
Kubernetes
Helm
GitOps
```

trong scope này.

---

# 24. Migration

Migration phải không yêu cầu downtime application.

Thứ tự:

```text
1. Tạo observability LXC
2. Deploy VictoriaLogs
3. Deploy Grafana
4. Deploy Beszel Hub
5. Xác nhận central healthy
6. Deploy Vector agent mới trên dev-server
7. Xác nhận logs xuất hiện trong VictoriaLogs
8. Xác nhận Grafana query được logs
9. Stop old centralized stack trên dev-server
10. Remove old VictoriaLogs/Grafana/OTel volumes khi đã xác nhận không cần
```

Không xóa old volume trước khi hệ thống mới hoạt động.

---

# 25. Acceptance criteria

Refactor được coi là hoàn tất khi:

```text
✓ Central stack chạy độc lập trên observability LXC.

✓ dev-server chỉ cần Vector để gửi logs.

✓ Vector không cần OTel Collector.

✓ VictoriaLogs nhận trực tiếp logs từ Vector.

✓ Chỉ container có logging.enabled=true được ingest.

✓ Thêm container mới chỉ cần labels, không sửa central config.

✓ Logs có thể filter theo host/service/environment.

✓ Structured JSON fields như player_id vẫn query được.

✓ Plain-text PostgreSQL/cloudflared logs vẫn xem được.

✓ VictoriaLogs giữ mặc định 3 ngày.

✓ VictoriaLogs có storage ceiling configurable.

✓ Docker local logs được rotate.

✓ Grafana có Logs Overview dashboard đơn giản.

✓ Beszel Hub chạy ở central LXC.

✓ Application vẫn chạy bình thường nếu toàn bộ observability stack offline.

✓ Không còn game-specific assumptions trong common logging pipeline.

✓ README mô tả rõ central vs agent deployment.
```

---

# 26. Non-goals — rất quan trọng

**Không thực hiện trong refactor này:**

```text
❌ Kubernetes
❌ Prometheus
❌ Loki
❌ distributed VictoriaLogs
❌ HA
❌ multi-tenancy
❌ traces
❌ OpenTelemetry application instrumentation
❌ custom Beszel fork
❌ metrics trong Grafana
❌ log alerting phức tạp
❌ automatic service onboarding
❌ systemd/journald collection
❌ Proxmox log collection
❌ S3 log archive
❌ 30/90-day retention
```

Những thứ này chỉ được thêm khi có use case thực tế.

---

## Một chỉ dẫn cuối cùng mình sẽ thêm cho agent

> **Do not redesign working pieces unless required by this spec. Prefer deleting unnecessary layers over introducing new abstractions. Preserve existing Grafana provisioning and Vector redaction behavior where they still fit. Keep the implementation understandable by one person operating a small home server.**

Điểm mình thích nhất ở hướng này là repo hiện tại đã có nền rất tốt: discovery bằng generic Docker labels, resource limits, disk buffer, Grafana provisioning và VictoriaLogs đều đã tồn tại.  Ta không xây lại hệ thống; chủ yếu là **đổi boundary từ “logging của game” thành “central logging của host”, tách collector khỏi backend, và bỏ tầng OTel chưa mang lại giá trị hiện tại**.

Nếu agent làm đúng spec này thì kết quả cuối sẽ khá “production-like”, nhưng vẫn chỉ còn **4 component có lý do rõ ràng để tồn tại: Beszel, Vector, VictoriaLogs và Grafana**.

[1]: https://docs.victoriametrics.com/victorialogs/data-ingestion/vector/?utm_source=chatgpt.com "VictoriaLogs: Data Ingestion: Vector Setup"
[2]: https://docs.victoriametrics.com/victorialogs/?utm_source=chatgpt.com "VictoriaLogs"
