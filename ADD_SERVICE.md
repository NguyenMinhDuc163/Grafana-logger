# Thêm service mới vào Central Logging

Tài liệu này áp dụng khi service mới chạy bằng Docker trên cùng máy với
`Grafana-logger`. Service chỉ cần ghi log ra `stdout` hoặc `stderr`; không cần
kết nối trực tiếp tới Grafana, VictoriaLogs hay mạng `observability`.

## 1. Thêm Docker labels

Trong file Compose của ứng dụng cần thu thập log, thêm ba labels vào đúng
service:

```yaml
services:
  payment-service:
    image: example/payment-service:latest
    labels:
      logging.enabled: "true"
      logging.service: "payment-service"
      logging.environment: "production"
```

Ý nghĩa:

- `logging.enabled`: cho phép Vector thu thập log của container.
- `logging.service`: tên dùng để lọc log trên Grafana. Mỗi service nên có một
  tên riêng, ổn định và không chứa khoảng trắng.
- `logging.environment`: môi trường chạy, ví dụ `development`, `staging` hoặc
  `production`.

Nếu muốn giới hạn dung lượng log Docker, có thể thêm:

```yaml
    logging:
      driver: local
      options:
        max-size: "50m"
        max-file: "3"
```

Sau khi sửa Compose, tạo lại container của ứng dụng để Docker nhận labels:

```bash
docker compose up -d --force-recreate payment-service
```

Không cần restart `Grafana-logger`. Vector sẽ tự phát hiện container mới trên
cùng Docker host.

## 2. Kiểm tra labels và log của ứng dụng

Kiểm tra labels đã xuất hiện trên container:

```bash
docker inspect payment-service --format '{{json .Config.Labels}}'
```

Kiểm tra ứng dụng thực sự ghi log ra Docker:

```bash
docker logs --tail 100 payment-service
```

Nếu `docker logs` không có dữ liệu thì Central Logging cũng không có dữ liệu để
thu thập. Cấu hình hiện tại không đọc log nằm riêng trong file bên trong
container.

## 3. Kiểm tra trên Grafana

1. Mở Grafana và vào **Explore**.
2. Chọn datasource **VictoriaLogs**.
3. Trong **Options**, chọn **Type: Raw Logs**.
4. Nhập query:

   ```logsql
   service_name:="payment-service"
   ```

5. Chọn khoảng thời gian phù hợp rồi nhấn **Run query**.

Lọc thêm theo môi trường:

```logsql
service_name:="payment-service" environment:="production"
```

Tìm các dòng có nội dung giống lỗi khi ứng dụng đang ghi log dạng text:

```logsql
service_name:="payment-service" _msg:~"(?i)(error|exception|failed)"
```

Thay `payment-service` bằng giá trị đã đặt trong `logging.service`.

## 4. Tạo dashboard bằng Grafana UI

1. Vào **Dashboards > New > New dashboard**.
2. Thêm một **Panel** và chọn datasource **VictoriaLogs**.
3. Mở **Options** và chọn **Type: Raw Logs**.
4. Nhập query `service_name:="payment-service"`.
5. Nhấn **Run query** và chọn visualization **Logs**.
6. Đặt tên panel, nhấn **Save**, sau đó đặt tên dashboard.

Dashboard tạo trên UI được lưu trong volume `grafana-data`. Không chạy
`docker compose down -v` nếu muốn giữ dashboard và dữ liệu Grafana.

## 5. Định dạng log

JSON không bắt buộc. Log dạng text vẫn được thu thập và tìm kiếm trong `_msg`.
Nếu ứng dụng ghi mỗi dòng dưới dạng một JSON object, các trường như `level`,
`trace_id`, `user_id` hoặc `event_name` sẽ có thể được lọc riêng trên Grafana.

Ví dụ JSON trên một dòng:

```json
{"level":"ERROR","message":"Payment failed","trace_id":"abc-123","user_id":"42"}
```

Không ghi mật khẩu, token, API key hoặc dữ liệu nhạy cảm vào log.

## 6. Khi không thấy log

Kiểm tra lần lượt:

```bash
docker compose ps
docker logs --tail 100 payment-service
docker inspect payment-service --format '{{json .Config.Labels}}'
```

Trong thư mục `Grafana-logger`, kiểm tra Vector:

```bash
docker compose logs --tail 100 vector
```

Các nguyên nhân thường gặp:

- Container chưa được tạo lại sau khi thêm labels.
- `logging.enabled` không có giá trị chính xác là `"true"`.
- Tên trong query khác với `logging.service`.
- Ứng dụng ghi log vào file thay vì `stdout`/`stderr`.
- Khoảng thời gian trên Grafana không chứa thời điểm phát sinh log.
- Service chạy trên Docker host khác. Cấu hình hiện tại chỉ tự phát hiện
  container trên cùng máy với Vector.
