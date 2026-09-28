# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền câu trả lời bên dưới mỗi câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Chử Trần Phương Nam  Mã học viên: 2A202602675

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi triển khai lên môi trường Cloud/Production, nếu lập trình viên quên khai báo biến `AGENT_API_KEY` trên Dashboard quản lý:
> - Nếu có giá trị mặc định `"changeme"`, app vẫn khởi động thành công, endpoint `/ask` mở công khai và bất kỳ ai cũng có thể gọi API bằng khóa mặc định, dẫn đến nguy cơ bị kẻ xấu khai thác làm cạn kiệt ngân sách LLM.
> - Khi không có giá trị mặc định, Pydantic ném lỗi `ValidationError` ngay lúc khởi động (Fail-fast), container lập tức dừng lại và báo lỗi deployment rõ ràng cho kỹ sư biết để bổ sung cấu hình trước khi có bất kỳ traffic nào đi vào hệ thống.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log JSON thực tế:
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T08:15:30.123456+00:00", "user_id": "sv-test", "tokens_in": 12, "tokens_out": 45, "cost_usd": 0.00015}`
> 
> Hai việc làm được với Structured JSON log:
> 1. **Lọc và truy vấn có cấu trúc (Structured Querying):** Các công cụ gom log tập trung (Datadog, Grafana Loki, CloudWatch) có thể tự động parse các trường để lọc nhanh mọi log của `user_id = 'sv-test'` hoặc tìm kiếm các request có `cost_usd > 0.01` mà không cần viết regex phức tạp.
> 2. **Trích xuất chỉ số và thiết lập cảnh báo tự động (Metrics & Alerting):** Trích xuất trực tiếp các trường định lượng (`cost_usd`, `tokens_in`, `tokens_out`) để vẽ biểu đồ chi phí thời gian thực và cấu hình cảnh báo tự động khi chi phí trung bình tăng đột biến.

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
| 1 stage (bản đầu) | 1045 MB |
| Multi-stage | 182 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần dung lượng chênh lệch (~863 MB) bao gồm:
> 1. Trình biên dịch C/C++ (`gcc`, `g++`, `make`), các header files hệ thống và công cụ build package (`build-essential`) có trong image `python:3.11` đầy đủ nhưng không cần thiết ở môi trường runtime.
> 2. Các file rác và bộ nhớ đệm tạm thời phát sinh trong quá trình `pip install` (nhờ dùng `--no-cache-dir` và chỉ copy các binary đã hoàn thiện sang stage runtime).
> 3. Hệ điều hành cơ sở tối giản của `python:3.11-slim` đã loại bỏ các package tiện ích và documentation dư thừa của bản Debian chuẩn.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> - Với Dockerfile hiện tại: Các layer `FROM`, `WORKDIR`, `COPY requirements.txt`, `COPY wheels/`, `RUN pip install ...`, và `RUN useradd ...` đều được lấy từ cache (`CACHED`). Chỉ có layer `COPY app/ app/` và các bước sau đó phải chạy lại, giúp thời gian build chỉ mất dưới 2 giây.
> - Nếu đặt `COPY . .` trước `RUN pip install`: Mỗi khi sửa bất kỳ file mã nguồn nào, layer `COPY . .` bị mất cache (cache invalidation), khiến toàn bộ lệnh `RUN pip install` phải cài đặt lại từ đầu, làm thời gian build bị chậm đi rất nhiều lần.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> - Chuỗi sự kiện tấn công:
>   1. Code Python có lỗ hổng RCE (ví dụ qua deserialize, command injection).
>   2. Kẻ tấn công thực thi mã trong container. Do container chạy dưới quyền `root` (UID 0), kẻ tấn công chiếm toàn quyền root trong không gian namespace của container.
>   3. Kẻ tấn công khai thác lỗ hổng kernel Linux hoặc các file mount nhạy cảm (`/var/run/docker.sock`) để thoát khỏi container (container escape).
>   4. Khi thoát ra host, vì UID trong container là 0 nên kẻ tấn công nghiễm nhiên sở hữu quyền `root` trên toàn bộ máy host.
> - Lệnh `USER appuser` cắt đứt chuỗi ở bước 2: Tiến trình Python chỉ chạy với quyền user thường (UID 1000). Kẻ tấn công không có quyền can thiệp hệ thống container, ngăn chặn việc cài đặt rootkit và vô hiệu hóa các kỹ thuật container breakout thông thường.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> - Người dùng có thể gửi tối đa **20 request** trong 2 giây liên tiếp.
> - Giải thích:
>   - Ở giây `10:00:59` (cuối phút 10), người dùng gửi 10 request (hợp lệ với quota phút 10).
>   - Đúng giây `10:01:00`, đồng hồ nhảy phút mới và bộ đếm reset về 0.
>   - Ở giây `10:01:01` (đầu phút 11), người dùng gửi tiếp 10 request nữa (hợp lệ với quota phút 11).
>   - Như vậy từ `10:00:59` đến `10:01:01` (chỉ trong 2 giây), hệ thống đã phải gánh 20 request, gây quá tải đột ngột. Thuật toán Sliding Window ngăn chặn triệt để điều này bằng cách luôn tính chính xác trong 60 giây gần nhất tính từ thời điểm hiện tại.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> - **Khác biệt:** Rate limit kiểm soát **số lượng request/tần suất** theo chu kỳ ngắn (ví dụ: 10 req/phút) để bảo vệ hạ tầng máy chủ khỏi spam/DDoS. Cost guard kiểm soát **chi phí tài chính / token** theo chu kỳ dài (ví dụ: $10/tháng) để bảo vệ ngân sách khỏi bị cạn kiệt.
> - **Rate limit cho qua nhưng Cost guard chặn:** Người dùng chỉ gửi 1 request trong 10 phút (tần suất rất thấp, rate limit cho qua), nhưng request này chứa prompt cực lớn yêu cầu xử lý 100.000 tokens tốn $1.50, trong khi ngân sách tháng của user chỉ còn dư $0.20 -> Cost guard phát hiện vượt budget và chặn với mã lỗi `402 Payment Required`.
> - **Cost guard cho qua nhưng Rate limit chặn:** Đầu tháng, người dùng còn nguyên ngân sách $10.00. Người dùng chạy script bắn liên tiếp 30 request ngắn trong 2 giây (mỗi request chỉ tốn $0.0001, tổng chỉ $0.003, rất rẻ so với $10) -> Cost guard cho phép nhưng Rate limiter lập tức chặn từ request thứ 11 với mã lỗi `429 Too Many Requests`.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện xảy ra (hiệu ứng sụp đổ dây chuyền - Cascading Failure):
> 1. Redis gặp sự cố ngắt kết nối mạng trong 30 giây.
> 2. Orchestrator định kỳ gọi liveness probe vào endpoint chung. Do kiểm tra Redis thất bại, cả 3 container đều trả về lỗi 503.
> 3. Orchestrator đánh giá liveness probe hỏng nên lập tức gửi tín hiệu kill và restart đồng loạt cả 3 container.
> 4. Cả cụm container rơi vào vòng lặp restart liên tục (CrashLoop), gây quá tải CPU/RAM trên node và tạo ra 100% thời gian chết (total downtime) cho người dùng cuối thay vì chỉ tạm dừng điều phối traffic.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> - **Hiện tại (Dùng Redis):** `history_length` tăng tuần tự và nhất quán: 0 -> 2 -> 4 -> 6... bất kể request đi vào container nào trong cụm 3 instance.
> - **Nếu lưu trong dict Python (State trong bộ nhớ tiến trình):** Do mỗi container có vùng nhớ riêng biệt và load balancer điều phối theo cơ chế round-robin, `history_length` sẽ bị phân mảnh và nhảy lộn xộn (ví dụ: request 1 vào agent 1 có history 0, request 2 vào agent 2 có history 0, request 3 vào agent 3 có history 0, request 4 vào agent 1 mới thấy history 2). AI sẽ liên tục bị "mất trí nhớ" và phản hồi sai lệch ngữ cảnh.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> - **Lỗi gặp phải:** `Health check timeout / Service failed to bind to port`.
> - **Cách tìm ra nguyên nhân:** Kiểm tra log container trên Dashboard của Cloud Platform. Nhận thấy nền tảng tự động gán cổng qua biến môi trường `$PORT` (ví dụ: `PORT=10000`), trong khi service FastAPI trong mã nguồn lại mặc định lắng nghe cố định ở cổng `8000`.
> - **Cách sửa:** Cập nhật lệnh khởi động trong `Dockerfile` và `Settings` để ưu tiên đọc cổng từ biến môi trường `$PORT`: `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]`.

