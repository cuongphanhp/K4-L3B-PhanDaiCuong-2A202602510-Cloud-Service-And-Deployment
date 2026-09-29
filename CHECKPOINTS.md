# Checkpoints

Mỗi checkpoint gồm ba phần: sản phẩm phải hoàn thành, kiến thức học viên phải
giải thích được và cách tự kiểm tra. Hãy commit sau mỗi checkpoint.

Ghi nhận thời điểm buổi lab bắt đầu là `Start`; lịch checkpoint không phụ thuộc
vào giờ bắt đầu cụ thể:

| Giai đoạn | Khung thời gian | Mốc kiểm tra |
|---|---|---|
| CP0 — Setup | Start +0–20 phút | Start +20 phút |
| CP1 — Config, Health & Logging | Start +20–60 phút | Start +60 phút |
| CP2 — Docker | Start +60–105 phút | Start +105 phút |
| Giải lao | Start +105–115 phút | — |
| CP3 — API Security | Start +115–160 phút | Start +160 phút |
| CP4 — Scaling & Reliability | Start +160–200 phút | Start +200 phút |
| CP5 — Cloud Deployment | Start +200–230 phút | Start +230 phút |
| Wrap-up và nộp bài | Start +230–240 phút | Start +240 phút |

## CP0 — Setup

**Sản phẩm:** repo cá nhân đúng tên, môi trường Python cài được dependency,
`.env` cục bộ được tạo từ `.env.example`, Redis khởi động được hoặc dùng
`fake://` tạm thời.

**Cần hiểu:** vì sao `.env` không được commit và vì sao lỗi test ở thời điểm
chưa viết code là bình thường.

**Tự kiểm tra:** `pytest tests/ -v -m "not docker"` chạy được, không gặp
`ModuleNotFoundError` hay lỗi setup môi trường.

## CP1 — 12-Factor Config, Health & Logging

**Sản phẩm:** `Settings` đủ sáu trường, secret bắt buộc, log JSON một dòng và
endpoint `/health` độc lập với dependency ngoài.

**Cần hiểu:** phân biệt code với config, ý nghĩa fail fast và lý do liveness
không nên gọi Redis.

**Tự kiểm tra:** `pytest tests/test_cp1.py -v`.

## CP2 — Docker

**Sản phẩm:** Dockerfile multi-stage dùng image gọn, chạy non-root, có
healthcheck, đọc `$PORT`; `.dockerignore` an toàn; Compose có `agent` và
`redis`.

**Cần hiểu:** Docker layer cache, ranh giới mạng giữa container, rủi ro chạy
root và nguy cơ secret lọt vào build context.

**Tự kiểm tra:** chạy `pytest tests/test_cp2.py -v`, sau đó build và chạy thật:

```bash
docker build -t day12-agent:prod .
docker compose up -d
curl http://localhost:8000/health
```

## CP3 — API Security

**Sản phẩm:** xác thực API key bằng so sánh constant-time, sliding-window rate
limit và cost guard theo user/tháng; `/ask` kiểm tra trước khi gọi mock LLM.

**Cần hiểu:** sự khác nhau giữa rate limit và budget limit, mã lỗi 401/402/429
và lý do member trong Redis sorted set phải duy nhất.

**Tự kiểm tra:** `pytest tests/test_cp3.py -v`.

## CP4 — Scaling & Reliability

**Sản phẩm:** history lưu trong Redis có giới hạn và TTL; `/ready` kiểm tra
Redis; liveness/readiness phản ánh shutdown; SIGTERM/SIGINT được chuyển tiếp
đúng cho handler cũ.

**Cần hiểu:** stateless service, khác nhau giữa liveness và readiness, cách
graceful shutdown tránh làm rớt request.

**Tự kiểm tra:** `pytest tests/test_cp4.py -v`; nếu có Docker, thử scale nhiều
instance theo hướng dẫn trong `LAB_GUIDE.md`.

## CP5 — Cloud Deployment

**Sản phẩm:** service có URL HTTPS công khai, kết nối Redis, environment được
set trên platform, `DEPLOYMENT.md` và ảnh minh chứng đã hoàn thiện.

**Cần hiểu:** cách platform cấp `$PORT`, cách đọc build/runtime log, nơi lưu
secret và cách chẩn đoán health/readiness khi deploy.

**Tự kiểm tra:** `pytest tests/test_cp5.py -v` và chạy các lệnh curl trong
`DEPLOYMENT.md`.

Nếu không thể dùng cloud, làm phương án `LOCAL_FALLBACK=true`; CP5 khi đó tối
đa 9/15 điểm.

## Bonus — CI/CD

Bonus không bắt buộc và chỉ nên làm sau CP1–CP5. Sản phẩm là GitHub Actions
workflow chạy khi push/pull request, cài dependency, chạy test, build image và
chỉ deploy nhánh chính sau khi test xanh.

**Tự kiểm tra:** `pytest tests/test_bonus_cicd.py -v` và xác nhận badge trong
README báo `passing`.

Bonus CI/CD tối đa 10 điểm cho bài lab; không phải điểm giơ tay, phát biểu hay
pitching. Nginx/load balancing là phần mở rộng, không phải bonus riêng.
