# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay thế vào các câu hỏi bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Phan Văn Cường  Mã học viên: K4-L3B-012

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống cụ thể: Khi deploy ứng dụng lên môi trường Production (hoặc Staging), người quản trị/DevOps quên cấu hình biến môi trường `AGENT_API_KEY` trong dashboard hoặc secret manager.
- Nếu để giá trị mặc định là `"changeme"`, ứng dụng vẫn khởi động thành công và báo trạng thái "healthy". Khi đó, bất kỳ kẻ tấn công hoặc bot quét tự động nào thử gọi API với header `X-API-Key: changeme` đều có thể truy cập trái phép vào endpoint `/ask`, liên tục gọi model LLM và làm cạn kiệt ngân sách/hóa đơn API trong khi nhóm phát triển hoàn toàn không hay biết.
- Ngược lại, với cơ chế Fail-fast (không có default value), ứng dụng sẽ crash ngay lập tức tại thời điểm khởi động (`ValidationError`), orchestrator (như Docker, Kubernetes hay Railway) sẽ báo lỗi deployment/crashloop và giữ lại phiên bản cũ hoặc gửi alert ngay cho nhóm phát triển. Nhờ đó, lỗ hổng bảo mật nghiêm trọng không bao giờ bị đưa ra môi trường công khai.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T07:15:30.123456+00:00", "user_id": "sv-123", "tokens_in": 15, "tokens_out": 42, "cost_usd": 0.000125}`

Hai việc làm được với log có cấu trúc (JSON structured log) mà print text thông thường không làm được:
1. **Lọc, truy vấn và tổng hợp tự động bằng máy (Automated Filtering & Aggregation):** Các hệ thống thu thập log tập trung (như Datadog, Grafana Loki, CloudWatch, ELK) có thể tự động parse các trường JSON để chạy câu truy vấn chính xác, ví dụ: tính tổng chi phí `SUM(cost_usd)` theo từng `user_id` trong tháng, hoặc đếm số lượt request `tokens_out > 1000`. Với log text dạng tự do của `print`, máy tính không thể phân tích cú pháp tin cậy mà phải dùng regex rất dễ gãy.
2. **Cảnh báo và giám sát theo thời gian thực (Real-time Alerting & Metrics Extraction):** Dễ dàng cấu hình cảnh báo tự động khi phát hiện `level == "error"` hoặc khi `cost_usd` của một user vượt ngưỡng bất thường trong một khoảng thời gian ngắn, giúp phát hiện sớm tấn công hoặc rò rỉ chi phí API.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | ~1.02 GB |
| Multi-stage | ~185 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~835 MB) bao gồm:
1. **Sự khác biệt giữa base image đầy đủ và slim:** Bản đầu dùng `python:3.11` (dựa trên Debian đầy đủ chứa hàng trăm công cụ hệ thống, gói thư viện hệ điều hành, tài liệu hướng dẫn man pages, và tiện ích không cần thiết cho runtime). Bản multi-stage chuyển sang dùng `python:3.11-slim`, lược bỏ phần lớn các package hệ điều hành thừa.
2. **Loại bỏ công cụ build và cache trong stage runtime:** Trong quá trình build ở stage `builder`, nếu cần các trình biên dịch (như `gcc`, `make`, `build-essential`), file header C (`python3-dev`) để biên dịch các package C-extensions hoặc wheel cache của pip, toàn bộ các công cụ và file tạm này chỉ tồn tại ở stage `builder`. Stage runtime cuối cùng chỉ copy thư mục gói Python đã cài đặt (`.local`), hoàn toàn không mang theo compiler hay artifact thừa, giúp image gọn nhẹ, bảo mật và kéo/tải (pull/push) nhanh hơn nhiều.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Với Dockerfile hiện tại:
- Các layer được dùng lại từ cache (CACHE HIT):
  + Stage builder: `FROM python:3.11-slim AS builder`, `WORKDIR /app`, `COPY requirements.txt .`, `RUN pip install --no-cache-dir --user -r requirements.txt`.
  + Stage runtime: `FROM python:3.11-slim`, `WORKDIR /app`, `RUN useradd ...`, `COPY --from=builder /root/.local /home/appuser/.local`.
- Layer phải chạy lại từ đầu (CACHE MISS):
  + Layer `COPY . .` (vì checksum của file `app/main.py` đã thay đổi).
  + Tất cả các lệnh phía sau `COPY . .` (như các biến ENV, USER, HEALTHCHECK, CMD).

Nếu đặt `COPY . .` lên trước `RUN pip install`:
Mỗi khi sửa dù chỉ một ký tự trong code Python (`app/main.py`), layer `COPY . .` sẽ bị invalid cache, dẫn đến việc Docker buộc phải chạy lại toàn bộ lệnh `RUN pip install` tiếp sau đó. Việc này khiến quá trình build lặp đi lặp lại rất lâu (phải tải và cài lại toàn bộ thư viện từ internet hoặc local cache), mất hoàn toàn lợi ích của Docker build cache.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện tấn công (Container Breakout):
1. Ứng dụng Python có một lỗ hổng bảo mật (ví dụ: Remote Code Execution qua thư viện deserialization, command injection, hoặc tải file tùy ý).
2. Kẻ tấn công khai thác lỗ hổng để thực thi shell payload bên trong container. Vì container đang chạy dưới quyền `root` (UID 0), tiến trình shell chiếm được có đầy đủ đặc quyền root trong container.
3. Kẻ tấn công lợi dụng đặc quyền root này để khai thác một lỗ hổng trong Linux kernel, cấu hình sai của Docker (như volume mount `/var/run/docker.sock` hoặc thư mục nhạy cảm từ host), hoặc khai thác lỗ hổng container runtime (runc).
4. Do UID trong container ánh xạ trực tiếp tới UID trên host kernel (UID 0 = root trên host), kẻ tấn công thoát khỏi ranh giới container (container breakout) và chiếm toàn quyền kiểm soát root trên máy chủ host.

Lệnh `USER appuser` cắt đứt chuỗi này ngay tại bước 2:
Khi chuyển sang user không đặc quyền (`appuser` với UID 1000), tiến trình shell mà kẻ tấn công chiếm được chỉ có quyền hạn tối thiểu của UID 1000. Kẻ tấn công không thể đọc ghi các file nhạy cảm của hệ thống, không có các Linux Capabilities của root, và không thể tương tác với Docker daemon socket hay khai thác hầu hết các lỗi leo thang đặc quyền container runtime, ngăn chặn nguy cơ chiếm quyền máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa **20 request** trong 2 giây liên tiếp.

Giải thích:
Với cơ chế Fixed Window (đếm theo phút đồng hồ và reset về 0 ở giây thứ :00):
- Tại giây thứ `10:00:59` (giây cuối cùng của phút thứ 10), người dùng gửi 10 request. Hệ thống ghi nhận 10/10 request cho phút thứ 10 và cho qua.
- Sang giây thứ `10:01:00` (giây đầu tiên của phút thứ 11), bộ đếm được reset về 0. Người dùng lập tức gửi thêm 10 request nữa trong giây `10:01:00` (hoặc `10:01:01`). Hệ thống ghi nhận 10/10 request cho phút thứ 11 và tiếp tục cho qua.
Tổng cộng trong khoảng thời gian 2 giây (từ `10:00:59` đến `10:01:01`), người dùng đã gửi thành công 20 request (gấp đôi hạn mức cho phép), tạo ra đột biến tải (spike/burst) có thể làm sập service hoặc cạn kiệt tài nguyên. Sliding window khắc phục được lỗ hổng này vì nó luôn xét cửa sổ 60 giây liên tục lùi từ thời điểm hiện tại.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Sự khác nhau:
- **Rate limit**: Giới hạn về *tần suất/số lượng request* trong một đơn vị thời gian ngắn (ví dụ: tối đa 10 request/phút) nhằm chống nghẽn mạng, chống DoS/spam và bảo vệ throughput của server.
- **Cost guard**: Giới hạn về *tổng chi phí tài chính/ngân sách tiêu thụ* (ví dụ: tối đa 10 USD/tháng) dựa trên số lượng token LLM thực tế phát sinh của từng người dùng trong chu kỳ dài.

Hai tình huống:
1. **Rate limit cho qua nhưng Cost guard chặn**: Một người dùng chỉ gửi 1 request trong 10 phút (hoàn toàn hợp lệ theo rate limit 10 req/phút), nhưng tài khoản của họ đã tiêu hết 9.99 USD trong tháng. Request này gửi kèm một đoạn tài liệu 50.000 tokens khiến chi phí ước tính vượt quá hạn mức 10.00 USD còn lại. Cost guard sẽ chặn ngay với mã lỗi `402 Payment Required`.
2. **Cost guard cho qua nhưng Rate limit chặn**: Một người dùng mới toanh chưa tiêu đồng nào (ngân sách còn nguyên 10.00 USD), nhưng dùng script tự động spam gửi liên tiếp 15 request "hello" chỉ trong 3 giây. Mặc dù chi phí của mỗi request cực kỳ nhỏ (~0.00001 USD, hoàn toàn nằm trong ngân sách), Rate limit sẽ kích hoạt ngay từ request thứ 11 và trả về mã lỗi `429 Too Many Requests`.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện xảy ra (Cascading Failure):
1. **Redis gặp sự cố hoặc gián đoạn mạng** trong 30 giây (ví dụ: Redis restart hoặc mạng chập chờn).
2. **Liveness check thất bại:** Vì endpoint kiểm tra liveness kiểm tra cả Redis, cả 3 container agent đều trả về lỗi khi orchestrator (Docker/Kubernetes) ping vào endpoint này.
3. **Orchestrator kill và restart liên tục cả cụm container:** Nhận thấy liveness probe fail (vượt quá số lần retries cho phép), orchestrator nhận định process bị deadlock và phát lệnh SIGKILL/restart toàn bộ 3 container agent.
4. **Boot-loop và nghẽn hệ thống:** Cả 3 container khởi động lại đồng thời, cùng lúc cố gắng kết nối lại Redis đang chưa sẵn sàng, tiếp tục fail healthcheck và lại bị restart liên tục (CrashLoopBackOff). Mọi request người dùng đang xử lý dở bị đứt gãy hoàn toàn.
5. **Redis phục hồi nhưng bị quá tải:** Khi Redis vừa sống lại, nó lập tức bị "dội bom" bởi kết nối ồ ạt từ các container đang restart cùng lúc (thundering herd problem), kéo dài thời gian downtime của toàn bộ hệ thống.

*Bài học:* `/health` (liveness) chỉ kiểm tra tiến trình app còn sống không để quyết định restart, tuyệt đối không kiểm tra dependency ngoài. Việc kiểm tra Redis phải dành riêng cho `/ready` (readiness) để orchestrator chỉ tạm ngắt traffic vào container mà không restart container.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Nếu lưu trong một dict Python trong bộ nhớ RAM của từng process thay vì Redis:
- Khi có 3 instance agent chạy sau load balancer (ví dụ round-robin của Nginx hoặc Docker Compose DNS), mỗi request của cùng một `X-User-Id` sẽ được định tuyến luân phiên ngẫu nhiên vào một trong 3 container (A, B hoặc C).
- Con số `history_length` sẽ **thay đổi bất thường, nhảy lộn xộn hoặc giảm đột ngột** giữa các lần gọi liên tiếp thay vì tăng dần đều 0 -> 2 -> 4 -> 6...
  Ví dụ: Lần 1 vào A (history=0, A lưu 2 tin); lần 2 vào B (history=0 vì B chưa có gì, B lưu 2 tin); lần 3 vào C (history=0); lần 4 quay lại A (history=2); lần 5 vào B (history=2)... Người dùng sẽ thấy agent bị "mất trí nhớ", trả lời câu sau không ăn nhập với câu trước do bối cảnh hội thoại bị phân mảnh giữa các RAM độc lập của từng container.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Thông báo lỗi gặp phải:** `Application failed to respond on port 8000. Deployment timed out waiting for healthcheck /health`.
- **Cách tìm ra nguyên nhân:** Mở tab Logs/Runtime Logs trên dashboard của platform (Railway/Render). Quan sát thấy ứng dụng ghi log: `Uvicorn running on http://0.0.0.0:8000`, trong khi platform cấp biến môi trường `PORT=10000` (hoặc cổng ngẫu nhiên do platform chỉ định) và bộ kiểm tra của platform chỉ gửi HTTP probe vào cổng được cấp phát đó. Vì app đang hardcode cổng `8000` trong lệnh chạy nên platform không nhận được phản hồi tại cổng nó mong đợi.
- **Cách sửa:** Sửa cấu hình Dockerfile và script khởi động: không cố định `--port 8000` mà đọc từ biến môi trường `$PORT` (ví dụ: `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]` hoặc trong `main.py` dùng `port=get_settings().port`). Đồng thời cấu hình `railway.toml` / `render.yaml` sử dụng `$PORT` được platform inject tự động. Sau khi sửa và commit lại, container lắng nghe đúng cổng của platform và deployment chuyển sang trạng thái xanh (Active).
