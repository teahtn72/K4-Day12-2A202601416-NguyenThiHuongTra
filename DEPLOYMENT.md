# Thông Tin Deploy — Checkpoint 5

Tài liệu này ghi nhận cấu hình và kết quả kiểm tra bản triển khai công khai.
Chỉ tên biến môi trường và nguồn cấp giá trị được ghi lại; không lưu giá trị
secret trong repository.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Nguyễn Thị Hương Trà |
| Mã học viên | 2A202601416 |
| Repo | https://github.com/teahtn72/K4-Day12-2A202601416-NguyenThiHuongTra |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-chat-3qxa.onrender.com |
| Platform | Render Blueprint, Docker runtime |
| Ngày deploy và xác minh | 2026-08-10 |
| Web service | `day12-chat` |
| Data service | `day12-chat-redis` — Render Key Value tương thích Redis |
| Health check | `/healthz` |

Render đọc cấu hình hạ tầng từ `render.yaml`, build image bằng `Dockerfile`,
chạy Uvicorn trên cổng do biến `PORT` cung cấp và kết nối web service với
Render Key Value qua private connection string.

## Biến Môi Trường Đã Set Trên Cloud

| Biến | Trạng thái | Nguồn giá trị |
|------|------------|---------------|
| `PORT` | ✅ | Render tự cấp cho web service |
| `API_TOKEN` | ✅ | Secret nhập trong Render Dashboard qua `sync: false` |
| `REDIS_URL` | ✅ | `connectionString` của service `day12-chat-redis` |
| `BUCKET_CAPACITY` | ✅ | Cấu hình Blueprint: `10` |
| `REFILL_PER_MINUTE` | ✅ | Cấu hình Blueprint: `10` |
| `DAILY_BUDGET_USD` | ✅ | Cấu hình Blueprint: `1.0` USD/client/ngày |
| `LOG_LEVEL` | ✅ | Cấu hình Blueprint: `INFO` |

## Lệnh Kiểm Tra

Đặt token lấy từ Render Dashboard vào biến môi trường cục bộ trước khi chạy
các kiểm tra có xác thực. Không commit giá trị này:

```bash
export API_TOKEN='token-lấy-từ-Render-Dashboard'
```

### 1. Liveness

```bash
curl -i https://day12-chat-m7xz.onrender.com/healthz
```

Mong đợi HTTP 200 và `{"status":"ok"}`.

### 2. Readiness

```bash
curl -i https://day12-chat-m7xz.onrender.com/readyz
```

Mong đợi HTTP 200 và `{"status":"ready","redis":true}`. Nếu endpoint trả
500 hoặc 503, kiểm tra log Render, implementation `ChatStore.ping()` và biến
`REDIS_URL`, sau đó redeploy commit mới nhất.

### 3. Endpoint được bảo vệ

```bash
curl -i -X POST https://day12-chat-m7xz.onrender.com/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"Hello"}'
```

Mong đợi HTTP 401 và header `WWW-Authenticate: Bearer`.

### 4. Chat với token hợp lệ

```bash
curl -i -X POST https://day12-chat-m7xz.onrender.com/chat \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "X-Client-Id: sv-test" \
  -d '{"message":"Deploy là gì?"}'
```

Mong đợi HTTP 200 với các trường `reply`, `client_id`, `turns_before`,
`usd_cost` và `usage`.

### 5. Rate limit

```bash
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " \
    -X POST https://day12-chat-m7xz.onrender.com/chat \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $API_TOKEN" \
    -H "X-Client-Id: rate-limit-smoke-test" \
    -d '{"message":"test"}'
done
echo
```

Với bucket capacity bằng 10, các request đầu trả 200 và các request sau khi
hết token trả 429. Token có thể được nạp thêm nếu vòng lặp chạy chậm.

## Kết Quả Chạy Thật

Các endpoint không cần secret được kiểm tra trực tiếp ngày 2026-08-10:

```text
GET /healthz
HTTP/2 200
{"status":"ok","service":"day12-chat-service","version":"1.0.0"}

GET /readyz
HTTP/2 500
Internal Server Error

POST /chat (không có Authorization)
HTTP/2 401
www-authenticate: Bearer
{"detail":"invalid or missing bearer token"}
```

Liveness và Bearer authentication đang hoạt động đúng. Readiness chưa đạt vì
bản deploy hiện tại trả HTTP 500; cần kiểm tra log của service, hoàn thiện
`ChatStore.ping()` nếu bản cloud chưa chứa code CP4 mới nhất, rồi redeploy.

Để test CP5 kiểm tra chat bằng token thật, thêm cùng giá trị `API_TOKEN` trên
Render vào `.env` cục bộ dưới tên `DEPLOY_API_TOKEN`:

```dotenv
DEPLOY_API_TOKEN=token-lấy-từ-Render-Dashboard
```

`.env` phải tiếp tục nằm trong `.gitignore` và `.dockerignore`.

## CI/CD và Rollback

Workflow `.github/workflows/ci.yml` chạy pytest và build Docker image cho mỗi
push hoặc pull request. Job deploy chỉ chạy sau khi test và build thành công
trên nhánh `main`, rồi gọi Render Deploy Hook lưu trong GitHub Secret
`RENDER_DEPLOY_HOOK_URL`.

Nếu deploy mới lỗi:

1. Mở service `day12-chat` trong Render Dashboard.
2. Mở tab **Events** hoặc **Deploys**.
3. Chọn bản deploy ổn định gần nhất.
4. Chọn **Rollback** hoặc **Redeploy** cho commit đó.
5. Kiểm tra lại `/healthz`, `/readyz` và `/chat`.

## Ảnh Chụp Màn Hình

Lưu ảnh nộp kèm trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — Render Dashboard của web service và Key Value.
- `screenshots/healthz.png` — `/healthz` trả HTTP 200.
- `screenshots/readyz.png` — `/readyz` trả HTTP 200 sau khi khắc phục Redis.

Không để ảnh hiển thị `API_TOKEN`, `REDIS_URL`, deploy hook hoặc secret khác.

## Phương Án Dự Phòng Cục Bộ

Chỉ sử dụng nếu Render không khả dụng:

1. Đặt `LOCAL_FALLBACK=true` trong `.env`.
2. Chạy `docker compose up -d` và `docker compose ps`.
3. Kiểm tra `http://localhost:8000/healthz` và `/readyz`.
4. Chụp kết quả vào thư mục `screenshots/`.
5. Chạy `pytest tests/test_cp5.py -v`.

Bản nộp hiện tại sử dụng public deployment trên Render, không dùng fallback.
