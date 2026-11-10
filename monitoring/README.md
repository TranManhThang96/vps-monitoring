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
| `node-exporter` | `host.docker.internal:9100` | `network_mode: host`, chỉ listen trên `HOST_GATEWAY_IP` (địa chỉ `docker0`) |

## Cài đặt

Chạy `infra` trước (`make up` ở đó tạo `shared-net`, Postgres, Redis, Garage), rồi:

```bash
cd ../infra
make pg-monitor-user      # in POSTGRES_EXPORTER_PASSWORD một lần

cd ../monitoring
make init                 # tạo .env (chép REDIS_PASSWORD từ ../infra/.env, sinh GRAFANA_ADMIN_PASSWORD) và secrets/garage_metrics_token
# điền MYSQL_PASSWORD và POSTGRES_EXPORTER_PASSWORD vào .env
make up                   # từ chối chạy nếu .env còn CHANGE_ME hoặc thiếu token
make targets              # health của từng target Prometheus
```

`.env` và `secrets/` đã nằm trong `.gitignore`. Prometheus (`http://127.0.0.1:9090`) và Grafana (`http://127.0.0.1:4000`, user `admin`,
mật khẩu là `GRAFANA_ADMIN_PASSWORD` trong `.env`) chỉ bind vào `BIND_ADDR` (mặc định `127.0.0.1`). Prometheus không có xác thực, đừng đặt `BIND_ADDR=0.0.0.0`.
Từ máy khác: `ssh -L 4000:127.0.0.1:4000 user@vps`, hoặc đặt nginx + TLS phía trước Grafana.

Grafana chỉ nhận `GRAFANA_ADMIN_PASSWORD` khi tạo volume `grafana-data` lần đầu. Volume đã có thì đổi mật khẩu bằng:

```bash
docker compose exec grafana grafana cli admin reset-admin-password "$(sed -n 's/^GRAFANA_ADMIN_PASSWORD=//p' .env)"
```

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

`host.docker.internal` trong container trỏ tới địa chỉ `docker0` của VPS (`HOST_GATEWAY_IP`, mặc định `172.17.0.1`; xem bằng
`ip -4 addr show docker0`). Nếu khác mặc định, đặt `HOST_GATEWAY_IP` trong `.env`. Địa chỉ nguồn mà MySQL và nginx nhìn thấy là
IP của chính container trong subnet `shared-net`, không phải gateway của network đó:

```bash
docker network inspect shared-net -f '{{(index .IPAM.Config 0).Subnet}}'   # ví dụ 172.20.0.0/16
```

**MySQL**: user của exporter phải khớp subnet đó. MySQL nhận mặt nạ mạng thay cho wildcard, đúng với mọi độ dài prefix
(`172.20.0.0/16` → `255.255.0.0`):

```sql
CREATE USER 'exporter'@'172.20.0.0/255.255.0.0' IDENTIFIED BY '...' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'172.20.0.0/255.255.0.0';
```

MySQL native phải lắng nghe được trên `HOST_GATEWAY_IP` (`bind-address = 0.0.0.0` hoặc địa chỉ đó), và firewall phải cho phép
subnet `shared-net` vào cổng 3306 của host.

**nginx**: block `stub_status` (cổng 9114) phải lắng nghe trên `HOST_GATEWAY_IP` và cho phép subnet `shared-net`:

```nginx
server {
    listen 127.0.0.1:9114;
    listen 172.17.0.1:9114;      # HOST_GATEWAY_IP, không dùng 0.0.0.0 để cổng này không ra internet
    location /nginx_status {
        stub_status;
        allow 172.20.0.0/16;     # subnet của shared-net
        allow 127.0.0.1;
        deny all;
    }
}
```

**node-exporter** chỉ listen trên `HOST_GATEWAY_IP:9100`, nên không ra internet. Cổng 9114 của nginx đã tránh `0.0.0.0` như trên;
nếu bạn vẫn mở nó rộng hơn, hãy chặn bằng firewall.

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
- Prometheus và Grafana bind vào `BIND_ADDR` (mặc định `127.0.0.1`), không còn `0.0.0.0`.
- `.env` cần thêm `GRAFANA_ADMIN_PASSWORD` (bắt buộc; thay cho mật khẩu cố định `admin`). Với `.env` có sẵn, thêm dòng này trước khi `make up`:

  ```bash
  echo "GRAFANA_ADMIN_PASSWORD=$(openssl rand -hex 16)" >> .env
  ```

  và đổi mật khẩu của volume Grafana đã tồn tại bằng lệnh `grafana cli` ở trên. Các biến mới còn lại (`BIND_ADDR`, `HOST_GATEWAY_IP`, version image, giới hạn) đều có mặc định, xem `.env.example`.
- Image được ghim version (biến `*_VERSION` trong `.env`) thay cho `:latest`. Nâng version: sửa `.env`, rồi `make pull && make up`.
- Prometheus có `--storage.tsdb.retention.size` (`PROMETHEUS_RETENTION_SIZE`, mặc định 5GB) ngoài `retention.time=200h`; mọi service có `mem_limit`, log json-file xoay vòng (10 MB × 3 file); Prometheus và Grafana có healthcheck, Grafana chỉ khởi động sau khi Prometheus healthy.
- `mysqld-exporter`: password chuyển sang env `MYSQLD_EXPORTER_PASSWORD`. README của exporter bản mới tách `--mysqld.username` và password; cú pháp `user:pass` trong một flag là cú pháp cũ.
- `.env` có `MYSQL_USER` và `MYSQL_PASSWORD` là bắt buộc (compose báo lỗi nếu thiếu).
- Có thêm `postgres-exporter`, `redis-exporter`, job `garage` trong `prometheus.yml`, và thư mục `secrets/` chứa token Garage.

## Dashboard

Provision tự động từ `grafana/dashboards/`: Garage (chính thức), PostgreSQL (grafana.com #9628, đã bỏ nhãn Kubernetes),
Redis (grafana.com #14091), MySQL (grafana.com #14057), Node Exporter Full (grafana.com #1860), NGINX (repo `nginx-prometheus-exporter`). Datasource Prometheus có `uid: prometheus`.

Dashboard PostgreSQL gốc viết cho bản exporter cũ và Postgres trước 17. Đã chỉnh cho Postgres 17 (phiên bản mặc định của `infra`):

- `postgres-exporter` bật thêm collector `postmaster` và `stat_checkpointer` (mặc định tắt), cho panel Start Time và Checkpoint Stats.
- Panel "Buffers (bgwriter)" và "Checkpoint Stats" đọc `pg_stat_checkpointer_*` thay cho `pg_stat_bgwriter_*`.
- Hai series `buffers_backend` và `backend_fsync` của panel "Buffers (bgwriter)" đã bị xóa khỏi dashboard: Postgres 17 bỏ hai cột này khỏi `pg_stat_bgwriter` (chuyển sang `pg_stat_io`, exporter chưa xuất).
- Nếu hạ `POSTGRES_VERSION` xuống dưới 17, các panel checkpoint sẽ trống.

Dashboard được mount read-only (`allowUiUpdates: false`): sửa file JSON trong `grafana/dashboards/` rồi `make restart SVC=grafana`.

## Sao lưu

- **Grafana**: dữ liệu (user, thay đổi qua UI) nằm trong volume `monitoring_grafana-data`. Dashboard đã nằm trong git nên không cần sao lưu. Sao lưu volume:

  ```bash
  docker run --rm -v monitoring_grafana-data:/data:ro -v "$PWD":/backup alpine tar czf /backup/grafana-data.tgz -C /data .
  ```

- **Prometheus**: `monitoring_prometheus_data` chỉ giữ metrics 200 giờ hoặc `PROMETHEUS_RETENTION_SIZE`, thường không cần sao lưu.
- Redis AOF của `../infra` nằm ở `infra/data/redis`: xem `../infra/README.md`.
