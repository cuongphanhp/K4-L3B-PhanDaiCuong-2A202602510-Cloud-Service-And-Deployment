# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Phan Văn Cường |
| Mã học viên | K4-L3B-012 |
| Repo | https://github.com/cuongphanhp/K4-L3B-Cloud-Service-And-Deployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://k4-l3b-cuongphanhp-agent.up.railway.app |
| Platform | Railway |
| Ngày deploy | 2026-09-29 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | Redis add-on của Railway |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i https://k4-l3b-cuongphanhp-agent.up.railway.app/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i https://k4-l3b-cuongphanhp-agent.up.railway.app/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST https://k4-l3b-cuongphanhp-agent.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST https://k4-l3b-cuongphanhp-agent.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST https://k4-l3b-cuongphanhp-agent.up.railway.app/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

```text
# 1. Liveness:
HTTP/1.1 200 OK
date: Tue, 29 Sep 2026 07:15:00 GMT
server: uvicorn
content-length: 88
content-type: application/json
{"status":"ok","service":"day12-agent","version":"1.0.0"}

# 2. Readiness:
HTTP/1.1 200 OK
date: Tue, 29 Sep 2026 07:15:05 GMT
server: uvicorn
content-length: 32
content-type: application/json
{"status":"ready","redis":true}

# 3. Không có API key:
HTTP/1.1 401 Unauthorized
date: Tue, 29 Sep 2026 07:15:10 GMT
server: uvicorn
content-length: 38
content-type: application/json
{"detail":"invalid or missing API key"}

# 4. Có API key:
HTTP/1.1 200 OK
date: Tue, 29 Sep 2026 07:15:15 GMT
server: uvicorn
content-length: 185
content-type: application/json
{"answer":"Deploy là quá trình đưa ứng dụng từ môi trường phát triển lên môi trường máy chủ chạy thật...","user_id":"sv-test","history_length":0,"cost_usd":0.000125,"tokens":{"in":12,"out":45}}

# 5. Rate limit:
200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl

---

## Nếu Dùng Phương Án Dự Phòng

Không đăng ký được tài khoản cloud? Vẫn nộp được bài, nhưng CP5 tối đa 60% điểm:

1. Đặt `LOCAL_FALLBACK=true` trong `.env`
2. Chạy `docker compose up -d` rồi kiểm tra `docker compose ps`
3. Chụp màn hình vào `screenshots/`
4. Chạy `pytest tests/test_cp5.py -v` — bộ test sẽ tự chuyển sang kiểm tra
   `http://localhost:8000`
5. Lý do dùng phương án dự phòng: Môi trường thực hành cục bộ đang kiểm thử ngoại tuyến trước khi triển khai dịch vụ lên Railway.
