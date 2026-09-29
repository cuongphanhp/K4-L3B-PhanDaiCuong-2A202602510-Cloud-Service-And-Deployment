# Rubric Chấm Điểm

## Phần bắt buộc — 100 điểm

| Tiêu chí | Điểm | Bằng chứng cần có | Điều kiện mất điểm |
|---|---:|---|---|
| CP1 — 12-Factor Config, Health & Logging | 15 | `tests/test_cp1.py` pass; cấu hình đọc từ environment; `/health` và log JSON hoạt động | Thiếu trường cấu hình, hardcode secret, health phụ thuộc Redis hoặc log không đúng định dạng |
| CP2 — Docker | 15 | `tests/test_cp2.py` pass; multi-stage image, non-root user, healthcheck, Compose có `agent` và `redis` | Image không build/chạy, chạy root, thiếu `.dockerignore`, hardcode secret hoặc cấu hình Redis sai |
| CP3 — API Security | 20 | `tests/test_cp3.py` pass; API key, sliding-window rate limit và cost guard hoạt động | `/ask` không yêu cầu key, limiter/cost guard sai hoặc kiểm tra sau khi đã gọi LLM |
| CP4 — Scaling & Reliability | 20 | `tests/test_cp4.py` pass; Redis store, `/ready`, giới hạn history và graceful shutdown hoạt động | State giữ trong process, readiness sai, không xử lý SIGTERM hoặc làm mất history |
| CP5 — Cloud Deployment | 15 | `tests/test_cp5.py` pass; URL HTTPS hoạt động, `/health`, `/ready` và auth đúng; `DEPLOYMENT.md` cùng ảnh minh chứng đầy đủ | URL không hoạt động, thiếu Redis/config, tài liệu còn placeholder hoặc làm lộ secret |
| Phản ánh trong `exercises.md` | 15 | Đủ 10 câu, dựa trên quan sát thực tế và học viên giải thích được | Bỏ trống, trả lời chung chung/sao chép hoặc không giải thích được nội dung đã viết |
| **Tổng phần bắt buộc** | **100** | | |

Điểm checkpoint được tính theo tỷ lệ test đạt. Test bị bỏ qua vì thiếu Docker
hoặc điều kiện môi trường không tự động được xem là đã đạt; Lab Coach có thể
yêu cầu bằng chứng chạy thực tế.

Nếu dùng `LOCAL_FALLBACK=true`, CP5 bị giới hạn tối đa 9/15 điểm.

## Bonus — tối đa 10 điểm

| Tiêu chí bonus | Điểm tối đa | Bằng chứng |
|---|---:|---|
| CI/CD bằng GitHub Actions | +10 | `tests/test_bonus_cicd.py` pass; workflow test, build và chỉ deploy sau khi test xanh; badge trong README báo `passing` |

Tổng bonus của toàn bài lab không vượt quá **10 điểm**. Đây là điểm cộng cho
sản phẩm CI/CD của bài lab, **không phải điểm giơ tay, phát biểu hoặc pitching**.
Phần Nginx/load balancing là hoạt động mở rộng để học, không tạo thêm một quỹ
điểm bonus riêng. Tổng điểm cuối do `grade.py` tính vẫn được giới hạn ở 100.

## Các khoản trừ và điều kiện hủy điểm

- Sai mẫu tên repository: trừ 5 điểm.
- Commit `.env`, API key hoặc secret: trừ 10 điểm; học viên phải thu hồi và đổi
  secret ngay. Xóa ở commit mới không loại secret khỏi lịch sử Git.
- Không giải thích được phần code hoặc nội dung đã nộp khi được hỏi: hủy điểm
  phần tương ứng.
- Sao chép bài hoặc có hai bài trùng nhau bất thường: cả hai bài có thể nhận
  0 điểm theo [RULES.md](RULES.md).
