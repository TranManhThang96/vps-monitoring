# Hạ tầng dùng chung (Docker)

PostgreSQL, Redis và Garage (lưu trữ tương thích S3) chạy trên một Docker network external,
để dùng chung cho các dịch vụ native trên VPS và cho các ứng dụng Docker khác.

Không dùng MinIO: bản community đã bị archive ngày 2026-04-25 (không còn bản vá bảo mật).
Garage là một binary tĩnh, RAM khi rảnh dưới 100 MB.

## Cài đặt

```bash
cd infra     # mọi lệnh make và docker compose chạy từ thư mục này
make init    # tạo .env với mật khẩu ngẫu nhiên (không ghi đè nếu .env đã có)
make up      # tạo network `shared-net` nếu chưa có, rồi chạy toàn bộ
make ps
```

Sửa `.env` trước khi chạy `make up` nếu muốn đổi phiên bản, port hoặc giới hạn bộ nhớ.

## Điều khiển

| Lệnh | Tác dụng |
|---|---|
| `make up` / `down` / `restart` / `logs` / `pull` | tất cả service |
| `make up SVC=redis` hoặc `make up-redis` | một service (`postgres`, `redis`, `garage`) |
| `make down-garage` | xóa container của một service, giữ nguyên dữ liệu |
| `make psql`, `make redis-cli`, `make garage-status` | mở shell / xem trạng thái |
| `make garage-bucket NAME=photos` | tạo bucket và cấp quyền cho key mặc định |
| `make db-create NAME=myapp` | tạo user + database `myapp` với mật khẩu ngẫu nhiên |

`make down` không bao giờ xóa network hay thư mục `./data`.

## Kết nối

Các port chỉ bind vào `127.0.0.1` (biến `BIND_ADDR`), nên PHP/nginx native kết nối qua loopback.

| Service | Từ host | Từ container khác trong `shared-net` |
|---|---|---|
| PostgreSQL | `127.0.0.1:5432` | `shared-postgres:5432` |
| Redis | `127.0.0.1:6379` (có mật khẩu) | `shared-redis:6379` |
| Garage S3 | `http://127.0.0.1:3900`, region `garage`, path-style | `http://shared-garage:3900` |

Cho một compose project khác tham gia network:

```yaml
services:
  app:
    networks: [shared]
networks:
  shared:
    name: shared-net   # giá trị SHARED_NETWORK trong .env
    external: true
```

## Database riêng cho từng app

Mỗi app nên có một database và một user riêng, không dùng superuser `postgres`.
Nếu một app bị lộ mật khẩu hoặc dính SQL injection, kẻ tấn công chỉ chạm được database của app đó,
không đọc hay xóa được dữ liệu của app khác.

```bash
make db-create NAME=myapp
```

Lệnh này (Postgres phải đang chạy):

1. Kiểm tra `NAME` chỉ gồm chữ thường, số, dấu gạch dưới, bắt đầu bằng chữ cái, tối đa 63 ký tự.
2. Dừng nếu user hoặc database trùng tên đã tồn tại, không sửa gì.
3. Tạo user `myapp` (không phải superuser, không có quyền tạo database hay user khác) với mật khẩu ngẫu nhiên 32 ký tự hex.
4. Tạo database `myapp` với chủ sở hữu là `myapp`.
5. Thu hồi quyền `CONNECT` của mọi user khác trên database này. Mặc định Postgres cho phép mọi user kết nối vào mọi database, bước này là bước cô lập thật sự.
6. In user, mật khẩu và chuỗi kết nối **một lần duy nhất**. Mật khẩu không được lưu ở đâu, hãy chép vào `.env` của app ngay.

Ví dụ cấu hình Laravel:

```
DB_CONNECTION=pgsql
DB_HOST=127.0.0.1          # app native trên VPS; app Docker trong shared-net dùng shared-postgres
DB_PORT=5432
DB_DATABASE=myapp
DB_USERNAME=myapp
DB_PASSWORD=<mật khẩu đã in>
```

Sao lưu riêng một app:

```bash
docker compose exec -T postgres pg_dump -U postgres myapp > myapp.sql
```

Xóa hẳn một app (không hoàn tác được):

```bash
make psql
# DROP DATABASE myapp; DROP USER myapp;
```

Quên mật khẩu thì đặt lại bằng `make psql` rồi `ALTER USER myapp WITH PASSWORD '...';`.

## Monitoring

Prometheus và Grafana nằm ở `../monitoring`, cùng tham gia `shared-net`. Phía `infra` cần ba thứ:

- **Garage**: `/metrics` ở cổng admin (3903) luôn đòi bearer token (`metrics_require_token = true`). Token là `GARAGE_METRICS_TOKEN` trong `.env`; `make init` tự sinh. Với `.env` có sẵn, thêm dòng này trước khi `make up`, nếu không compose sẽ báo lỗi:

  ```bash
  echo "GARAGE_METRICS_TOKEN=$(openssl rand -hex 32)" >> .env
  ```

- **PostgreSQL**: `make pg-monitor-user` tạo role `exporter` chỉ có quyền `pg_monitor` (đọc thống kê, không phải superuser) và in mật khẩu một lần. Chép mật khẩu vào `../monitoring/.env`.
- **Redis**: exporter dùng chính `REDIS_PASSWORD`.

Cách cấu hình phía monitoring: xem `../monitoring/README.md`.

## Lưu ý

- Dữ liệu nằm trong `./data` (biến `DATA_DIR`). Sao lưu Postgres bằng `pg_dump`, không copy file khi đang chạy.
- Garage chạy single-node với `replication_factor = 1`: không có dự phòng. Hãy sao lưu `data/garage` hoặc nhân bản object sang nơi khác.
- Redis ghi AOF vào `data/redis` (`--appendonly yes`). Sao lưu bằng cách chạy `docker compose exec redis redis-cli BGREWRITEAOF`, đợi xong rồi copy `data/redis`, hoặc dừng Redis (`make down-redis`) trước khi copy.
- Log của mọi container xoay vòng (10 MB × 3 file) nên không làm đầy ổ đĩa.
- Postgres có `/dev/shm` 256 MB (`POSTGRES_SHM_SIZE`); Docker mặc định chỉ 64 MB, không đủ cho `shared_buffers=1GB` với truy vấn song song.
- Garage không có web console. Dùng `aws s3`, `rclone` hoặc thư viện S3.
- Đổi `POSTGRES_VERSION` sang major version mới cần dump/restore.
- Không đổi `GARAGE_RPC_SECRET` sau lần chạy đầu tiên.
- Makefile đọc `SHARED_NETWORK` từ `.env` và ưu tiên hơn biến export trong shell, còn compose thì ưu tiên biến shell. Chỉ để các biến này trong `.env`, đừng export thêm.
