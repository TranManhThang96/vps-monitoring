# Monitoring (Prometheus + Grafana)

Chạy trên network external `shared-net` của `../infra`, nên Prometheus scrape được PostgreSQL, Redis, Garage
bằng tên container. MySQL, nginx và host vẫn là native trên VPS.

| Job | Target | Nguồn |
|---|---|---|
| `postgres-exporter` | `postgres-exporter:9187` → `shared-postgres:5432` | role `exporter` (`pg_monitor`) |
| `redis-exporter` | `redis-exporter:9121` → `shared-redis:6379` | `REDIS_PASSWORD` |
| `garage` | `shared-garage:3903/metrics` | bearer token `GARAGE_METRICS_TOKEN` |
| `mysqld-exporter` | `mysqld-exporter:9104` → `host.docker.internal:3306` | user MySQL `exporter` |
| `nginx-exporter` | `nginx-exporter:9113` → `host.docker.internal:9114/nginx_status` | `stub_status` của nginx |
| `node-exporter` | `host.docker.internal:9100` | `network_mode: host`, không đổi |

## Cài đặt

Chạy `infra` trước (`make up` ở đó tạo `shared-net`, Postgres, Redis, Garage), rồi:

```bash
cd ../infra
make pg-monitor-user      # in POSTGRES_EXPORTER_PASSWORD một lần

cd ../monitoring
make init                 # tạo .env (chép REDIS_PASSWORD từ ../infra/.env) và secrets/garage_metrics_token
# điền MYSQL_PASSWORD và POSTGRES_EXPORTER_PASSWORD vào .env
make up                   # từ chối chạy nếu .env còn CHANGE_ME hoặc thiếu token
make targets              # health của từng target Prometheus
```

`.env` và `secrets/` đã nằm trong `.gitignore`. Prometheus: `http://127.0.0.1:9090`, Grafana: `http://127.0.0.1:4000`
(đổi `GF_SECURITY_ADMIN_PASSWORD` trong compose nếu mở ra ngoài).

| Lệnh | Tác dụng |
|---|---|
| `make up` / `down` / `restart` / `logs` / `pull` / `ps` | tất cả service |
| `make up SVC=grafana` hoặc `make up-grafana` | một service |
| `make secrets` | ghi lại `secrets/garage_metrics_token` từ `../infra/.env` |
| `make reload` | nạp lại `prometheus.yml`, không restart |
| `make targets` | job, health, lỗi scrape gần nhất |

Đổi `GARAGE_METRICS_TOKEN` trong `../infra/.env` thì chạy `make secrets reload`.

## Kiểm tra

```bash
make targets
```

Cả 6 job phải ở trạng thái `up`. Hai job `mysqld-exporter` và `nginx-exporter` vẫn `up` khi exporter chạy nhưng không nối được tới MySQL hay nginx.
Khi đó metric `mysql_up` hoặc `nginx_up` bằng 0, nên kiểm tra thêm:

```bash
curl -s 'localhost:9090/api/v1/query?query=mysql_up'   # value phải là "1"
curl -s 'localhost:9090/api/v1/query?query=nginx_up'   # value phải là "1"
```

Giá trị 0 nghĩa là chưa làm xong hai bước trên VPS bên dưới. `make logs SVC=mysqld-exporter` hiện lỗi cụ thể.

## Hai bước bắt buộc trên VPS (nằm ngoài repo)

Container nay nằm trong subnet của `shared-net`, nên MySQL và nginx thấy IP nguồn khác trước.

```bash
docker network inspect shared-net -f '{{(index .IPAM.Config 0).Subnet}}'   # ví dụ 172.20.0.0/16
```

**MySQL**: user của exporter phải khớp host nguồn. Gateway của `shared-net` là địa chỉ mà MySQL thấy
khi container gọi `host.docker.internal`:

```bash
docker network inspect shared-net -f '{{(index .IPAM.Config 0).Gateway}}'   # ví dụ 172.20.0.1
```

```sql
CREATE USER 'exporter'@'172.20.0.1' IDENTIFIED BY '...' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'172.20.0.1';
```

MySQL native cũng phải lắng nghe trên địa chỉ đó (`bind-address = 0.0.0.0` hoặc gateway), và firewall
phải cho phép subnet `shared-net` vào cổng 3306.

**nginx**: block `stub_status` (cổng 9114) phải cho phép subnet mới và lắng nghe được từ gateway:

```nginx
server {
    listen 9114;                 # không chỉ 127.0.0.1, vì exporter vào qua gateway
    location /nginx_status {
        stub_status;
        allow 172.20.0.0/16;     # subnet của shared-net
        allow 127.0.0.1;
        deny all;
    }
}
```

Chặn cổng 9114 từ internet bằng firewall.

## Chuyển từ cách cũ

Compose cũ dùng network `monitoring_monitoring` (bridge riêng). Tên project vẫn là `monitoring`, nên volume
`monitoring_prometheus_data` và `monitoring_grafana-data` được giữ nguyên.

```bash
make init     # compose nội suy biến bắt buộc lúc đọc file, nên cần .env đầy đủ trước mọi lệnh, kể cả down
              # điền MYSQL_PASSWORD và POSTGRES_EXPORTER_PASSWORD vào .env
make up       # tạo lại container trên shared-net, volume giữ nguyên
docker network rm monitoring_monitoring   # sau khi up, dọn network cũ
```

Thay đổi so với bản cũ:

- Bỏ `ports` 9104 và 9113: exporter chỉ truy cập được qua `shared-net`.
- `mysqld-exporter`: password chuyển sang env `MYSQLD_EXPORTER_PASSWORD`. README của exporter bản mới tách `--mysqld.username` và password; cú pháp `user:pass` trong một flag là cú pháp cũ.
- `.env` có `MYSQL_USER` và `MYSQL_PASSWORD` là bắt buộc (compose báo lỗi nếu thiếu).
- Có thêm `postgres-exporter`, `redis-exporter`, job `garage` trong `prometheus.yml`, và thư mục `secrets/` chứa token Garage.

## Dashboard

Provision tự động từ `grafana/dashboards/`: Garage (chính thức), PostgreSQL (grafana.com #9628, đã bỏ nhãn Kubernetes),
Redis (grafana.com #14091). Datasource Prometheus có `uid: prometheus`.

Dashboard PostgreSQL gốc viết cho bản exporter cũ và Postgres trước 17. Đã chỉnh cho Postgres 17 (phiên bản mặc định của `infra`):

- `postgres-exporter` bật thêm collector `postmaster` và `stat_checkpointer` (mặc định tắt), cho panel Start Time và Checkpoint Stats.
- Panel "Buffers (bgwriter)" và "Checkpoint Stats" đọc `pg_stat_checkpointer_*` thay cho `pg_stat_bgwriter_*`.
- Hai series `buffers_backend` và `backend_fsync` của panel "Buffers (bgwriter)" sẽ trống: Postgres 17 đã bỏ hai cột này khỏi `pg_stat_bgwriter` (chuyển sang `pg_stat_io`, exporter chưa xuất).
- Nếu hạ `POSTGRES_VERSION` xuống dưới 17, các panel checkpoint sẽ trống.

Dashboard được mount read-only (`allowUiUpdates: false`): sửa file JSON trong `grafana/dashboards/` rồi `make restart SVC=grafana`.
