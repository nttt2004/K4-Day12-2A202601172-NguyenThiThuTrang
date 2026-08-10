# Thông tin deploy — Checkpoint 5

## Thông tin học viên

| Mục | Nội dung |
|---|---|
| Họ và tên | Nguyễn Thị Thu Trang |
| Mã học viên | 2A202601172 |
| Repo | https://github.com/nttt2004/K4-Day12-2A202601172-NguyenThiThuTrang |

## Service

| Mục | Nội dung |
|---|---|
| Public URL | https://day12-chat-rvcs.onrender.com |
| Platform | Render Web Service (Docker), Blueprint managed |
| Ngày deploy | 2026-08-10 |
| Trạng thái dashboard | Deploy live — commit f69ed22 (`checkpoint 4`) |

## Biến môi trường đã set trên Render

| Biến | Đã set | Nguồn/ghi chú |
|---|---|---|
| `PORT` | ✅ | Render tự cấp |
| `API_TOKEN` | ✅ | Render Environment Secret |
| `REDIS_URL` | ✅ | Render Redis add-on `day12-chat-redis` |
| `BUCKET_CAPACITY` | ✅ | Render Environment Variable: 10 |
| `REFILL_PER_MINUTE` | ✅ | Render Environment Variable: 10 |
| `DAILY_BUDGET_USD` | ✅ | Render Environment Variable: 1.0 |
| `LOG_LEVEL` | ✅ | Render Environment Variable: INFO |

## Kiểm tra live

Contract của service dùng `/healthz`, `/readyz`, `/chat`.

```text
GET /healthz
HTTP 200
{"status":"ok","service":"day12-chat-service","version":"1.0.0"}

GET /readyz
HTTP 200
{"status":"ready","redis":true}

POST /chat (không có Authorization)
HTTP 401
{"detail":"invalid or missing bearer token"}
```

Kiểm tra `/chat` có xác thực bằng token live, chạy tại máy cá nhân sau khi đặt
`DEPLOY_API_TOKEN` trong file `.env` (không commit file này):

```powershell
$url = "https://day12-chat-rvcs.onrender.com"
Invoke-WebRequest -Method Post "$url/chat" `
  -ContentType "application/json" `
  -Headers @{ Authorization = "Bearer $env:DEPLOY_API_TOKEN"; "X-Client-Id" = "cp5-test" } `
  -Body '{"message":"Deploy là gì?"}'
```

Kết quả mong đợi: HTTP 200 và JSON có trường `reply`.

## Ảnh bằng chứng

- `screenshots/dashboard.png` — ảnh dashboard Render hiển thị deploy live.
- `screenshots/healthz.png` — ảnh kết quả gọi endpoint health.
