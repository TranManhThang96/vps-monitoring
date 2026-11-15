# Deploy chat-app lên VPS (Postgres → pgvector)

Runbook cho lần deploy đầu: đổi `shared-postgres` sang image có pgvector, tạo database cho RAG, chạy
Open WebUI sau nginx.

Điều kiện áp dụng: **Postgres chưa có dữ liệu cần giữ**. Bước 1.3 xoá hẳn cluster cũ. Nếu sau này đã
có dữ liệu, xem mục [Khi đã có dữ liệu](#khi-đã-có-dữ-liệu) thay vì xoá.

## 0. Đưa code mới lên VPS

```bash
cd ~/vps-script                 # đường dẫn repo trên VPS
git pull                        # hoặc rsync từ máy dev
grep -n 'pgvector' infra/docker-compose.yml
```

Checkpoint: phải thấy `image: ${POSTGRES_IMAGE:-pgvector/pgvector:0.8.7-pg17}`. Không thấy thì code
trên VPS còn cũ, dừng lại.

## 1. Đổi image Postgres

### 1.1. Vào `infra`, tạo `.env` nếu chưa có

```bash
cd ~/vps-script/infra
test -f .env || make init
```

`make init` chỉ chạy lần đầu. Nếu `.env` đã tồn tại nó thoát với
`.env already exists, not overwriting` — không ghi đè secret, không cần chạy lại.

### 1.2. Dừng + xoá container Postgres cũ

```bash
make down SVC=postgres
```

### 1.3. Xoá cluster cũ

`data/postgres` thuộc uid 999 nên `rm -rf` bằng user thường sẽ `Permission denied`; xoá qua container:

```bash
mkdir -p data
docker run --rm -v "$PWD/data:/d" alpine rm -rf /d/postgres
```

### 1.4. Pull image pgvector + start lại

```bash
make pull SVC=postgres      # kéo image pgvector
make up SVC=postgres        # tạo shared-net nếu chưa có, rồi start
```

`make up` (không phải `docker compose up`) là cần thiết vì compose khai báo network `shared-net` là
`external`: gọi `docker compose up` trực tiếp khi network chưa tồn tại sẽ lỗi
`network shared-net declared as external, but could not be found`.

Lần đầu pull image Debian (~150 MB) rồi initdb, mất khoảng vài chục giây.

### 1.5. Xác nhận

```bash
docker inspect -f '{{.Config.Image}}' shared-postgres
# pgvector/pgvector:0.8.7-pg17

docker compose ps postgres
# shared-postgres   Up (healthy)

docker exec shared-postgres psql -U postgres -tAc \
  "select name, default_version from pg_available_extensions where name='vector'"
# vector|0.8.7
```

Nếu image vẫn là `postgres:17-alpine` hoặc không có dòng `vector|0.8.7`: code trên VPS chưa đúng,
quay lại bước 0.

Ghi chú: không cần thêm `POSTGRES_IMAGE` vào `.env`. Compose đã có default; biến cũ `POSTGRES_VERSION`
nếu còn trong `.env` thì bị bỏ qua, xoá đi cho gọn:

```bash
sed -i '/^POSTGRES_VERSION=/d' .env
```

Muốn ghim cứng image trong `.env`:

```bash
grep -q '^POSTGRES_IMAGE=' .env \
  && sed -i 's|^POSTGRES_IMAGE=.*|POSTGRES_IMAGE=pgvector/pgvector:0.8.7-pg17|' .env \
  || echo 'POSTGRES_IMAGE=pgvector/pgvector:0.8.7-pg17' >> .env
```

## 2. Tạo database + extension cho chat app

```bash
cd ~/vps-script/chat-app
test -f .env || make init        # in ra WEBUI_ADMIN_PASSWORD một lần, lưu lại
vi .env                          # WEBUI_URL=https://chat.<domain>, OPENAI_API_KEY=user_...
make config                      # kiểm tra compose + interpolation
make up                          # chạy `make db` trước: role openwebui + CREATE EXTENSION vector
```

`make up` idempotent: lần đầu tạo role/database và sinh `DB_PASSWORD` vào `.env`, các lần sau chỉ đồng
bộ lại mật khẩu và extension.

### 2.1. Xác nhận

```bash
make health                     # Up (healthy) + {"status":true}
make chat                       # gọi thẳng CommandCode, kiểm tra key
make psql                       # trong psql:  \dx   →  vector | 0.8.7 | public
```

Thử RAG: upload một file trong UI, rồi

```bash
make psql                       # select count(*) from document_chunk;
```

`document_chunk` chưa có index vector là bình thường: dưới 5000 chunk Open WebUI dùng exact scan
(recall tốt nhất). Khi vượt ngưỡng đó mới đặt `PGVECTOR_INDEX_METHOD=hnsw` trong `.env` rồi
`make up`.

## 3. nginx

Thêm vhost (hoặc sửa file trong `/etc/nginx/conf.d/chat.conf`):

```nginx
server {
    listen 443 ssl;
    http2 on;
    server_name chat.<domain>;

    # TLS: certbot --nginx -d chat.<domain>  hoặc Cloudflare Origin Certificate

    client_max_body_size 50m;          # file upload của RAG

    location / {
        proxy_pass http://127.0.0.1:3080;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket /ws/socket.io
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # SSE của chat streaming: không được buffer
        proxy_buffering off;
        proxy_cache off;
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }
}
```

```bash
sudo nginx -t && sudo systemctl reload nginx
sudo certbot --nginx -d chat.<domain>      # nếu chưa có TLS
```

Mở `https://chat.<domain>`, đăng nhập bằng `WEBUI_ADMIN_EMAIL` / `WEBUI_ADMIN_PASSWORD` trong
`chat-app/.env`.

## Tóm tắt một dòng cho bước 1

```bash
cd ~/vps-script/infra && make down SVC=postgres && \
docker run --rm -v "$PWD/data:/d" alpine rm -rf /d/postgres && \
make pull SVC=postgres && make up SVC=postgres
```

## Thứ tự phụ thuộc

| Bước | Cần trước đó |
|---|---|
| `infra: make init && make up` (chỉ postgres cũng được) | — |
| `chat-app: make init` | — (chỉ cần điền key/domain) |
| `chat-app: make up` | `shared-postgres` healthy + network `shared-net` tồn tại |
| nginx vhost | app healthy trên `127.0.0.1:3080` |

## Khi đã có dữ liệu

Không xoá `data/postgres`. Image Debian đọc được cluster do Alpine tạo (cùng major 17), nên chỉ cần:

```bash
cd ~/vps-script/infra
make pull SVC=postgres
make up SVC=postgres                 # recreate, giữ data/
docker compose ps postgres
```

Sao lưu trước khi đổi image, và luôn dùng `pg_dump` (không copy file khi cluster đang chạy):

```bash
docker exec -t shared-postgres pg_dumpall -U postgres > postgres-all.sql
```

Nếu đổi sang major version Postgres khác (16 → 17), bắt buộc dump/restore hoặc `pg_upgrade`.

## Xử lý sự cố

| Hiện tượng | Nguyên nhân / cách sửa |
|---|---|
| `docker compose` báo `network shared-net not found` | `cd ../infra && make network` |
| `rm: cannot remove ... Permission denied` khi xoá `data/postgres` | dùng lệnh `docker run ... alpine rm -rf` ở bước 1.3 |
| App lỗi `could not translate host name "shared-postgres"` | container không ở `shared-net`: kiểm tra `docker network inspect shared-net`, chạy `cd ../infra && make up` |
| Upload tài liệu lỗi `type "vector" does not exist` | database chưa có extension: `cd ../chat-app && make db` |
| `make db` in `WARNING: document_chunk.vector is vector(N) but PGVECTOR_DIM=M` | đổi embedding model khác số chiều: `DROP TABLE document_chunk;` rồi index lại |
| Chat trả 401 dù container healthy | key CommandCode hết hạn: `make chat` xem lỗi gốc, cập nhật key (Admin Panel → Connections nếu config đã persist) |
| Container `starting` vài phút ở lần đầu | đang tải model embedding RAG (~90 MB) về `data/cache` |
