# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền câu trả lời bên dưới mỗi câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Quý Dũng  Mã học viên: 2A202601200

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu đặt `api_token` mặc định là `"changeme"`, khi deploy ứng dụng lên môi trường Production mà quên khai báo biến `API_TOKEN` trên Cloud, ứng dụng vẫn sẽ khởi động thành công bình thường. Lúc này, hệ thống sẽ mở cổng nhận traffic public nhưng lại dùng chìa khóa bảo vệ mặc định là `"changeme"`. Kẻ xấu có thể dò tìm hoặc dùng chính giá trị mặc định `"changeme"` này để gọi API miễn phí, tiêu tốn tài nguyên và ngân sách LLM của bạn mà bạn không hề hay biết cho tới khi nhận hóa đơn. Ngược lại, việc không đặt giá trị mặc định sẽ ép ứng dụng "chết ngay" (Fail Fast) ở bước startup, phát ra lỗi thiếu biến `API_TOKEN` ngay lập tức, buộc người quản trị phải cấu hình đúng bí mật trước khi ứng dụng có thể phục vụ người dùng.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON mẫu thu được:
```json
{"event": "chat_completed", "timestamp": "2026-08-10T10:15:30Z", "service": "day12-chat-service", "version": "1.0.0", "client_id": "sv-test", "prompt_tokens": 15, "completion_tokens": 25, "usd_cost": 0.001}
```

Hai việc làm được với Structured Logging (log JSON) mà `print` không làm được:
1. **Truy vấn, lọc và phân tích tự động (Automated Parsing & Aggregation):** Các công cụ quản lý log tập trung (như Datadog, ELK Stack, Grafana Loki) có thể tự động parse các trường JSON để lọc theo `client_id`, tính tổng chi phí `usd_cost` theo ngày, hoặc thống kê lượng `prompt_tokens` mà không cần viết regex phức tạp.
2. **Cảnh báo theo ngưỡng (Alerting & Monitoring):** Có thể dễ dàng thiết lập quy tắc cảnh báo (Alert Rules) dựa trên dữ liệu kiểu số trong log (ví dụ: cảnh báo khi `usd_cost` của một `client_id` vượt ngưỡng bất thường trong khoảng thời gian ngắn), điều mà một dòng text thuần túy `print("đã trả lời xong")` hoàn toàn không hỗ trợ.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t chat:single .
docker build -t chat:multi .
docker images | grep chat
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | ~350 MB |
| Multi-stage | ~180 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~170 MB) là do trong kỹ thuật Multi-stage build, ở Stage 2 (Production image), chúng ta chỉ `COPY` các thư viện Python đã biên dịch xong từ Stage 1 (`builder`) mà bỏ lại toàn bộ các công cụ build/biên dịch không cần thiết như `gcc`, `pip cache`, các file `.whl` trung gian, header C (`python-dev`), và bộ đệm cài đặt của package manager. Điều này giúp giảm đáng kể kích thước Image cuối cùng và hạn chế diện tấn công (attack surface).

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Khi sửa `app/main.py` và build lại: Docker sẽ sử dụng lại Cache cho toàn bộ các layer phía trên bao gồm base image, `COPY requirements.txt .`, và layer tốn nhiều thời gian nhất là `RUN pip install ...`. Chỉ layer `COPY app/ app/` (và các layer sau nó) mới bị dỡ bỏ cache và chạy lại.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Mỗi khi có bất kỳ sự thay đổi nhỏ nào trong bất kỳ file nguồn nào (như `main.py`), layer `COPY . .` sẽ làm mất hiệu lực cache (cache invalidation). Hệ quả là Docker buộc phải chạy lại toàn bộ lệnh `RUN pip install` từ đầu, làm thời gian build bị kéo dài từ vài giây lên vài phút.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

1. **Chuỗi sự kiện:**
   - Code Python chứa lỗ hổng (ví dụ: Arbitrary Code Execution / Remote Code Execution thông qua `eval()`, `pickle.loads()`, hoặc Command Injection).
   - Kẻ tấn công khai thác lỗ hổng này để thực thi lệnh shell bên trong container.
   - Do container chạy dưới quyền `root` (UID 0), kẻ tấn công sở hữu quyền `root` bên trong container.
   - Nếu container có thêm lỗi cấu hình (như mount socket docker `/var/run/docker.sock`, chưa bỏ Linux capabilities, hoặc khai thác lỗ hổng Container Escape trong Linux Kernel), kẻ tấn công có thể thoát khỏi container và chiếm quyền kiểm soát `root` trên chính máy Host.
2. **Lệnh `USER appuser` cắt đứt chuỗi này:**
   Lệnh `USER appuser` ép process ứng dụng Python chạy dưới quyền một user không có đặc quyền (unprivileged user, UID 1000). Khi kẻ tấn công khai thác được lỗ hổng trong code Python, tiến trình shell chiếm được cũng chỉ có quyền của `appuser`, không thể sửa các file hệ thống của container, không thể cài phần mềm độc hại, và triệt hạ nguy cơ tiến công leo thang quyền `root` lên máy Host.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

1. **Cần kèm header `WWW-Authenticate: Bearer`:**
   Theo chuẩn RFC 7235 và RFC 6750, khi HTTP Server trả về trạng thái `401 Unauthorized`, bắt buộc phải chứa header `WWW-Authenticate` để chỉ dẫn cho HTTP Client (như trình duyệt hoặc ứng dụng) biết phương thức xác thực mà Server yêu cầu là gì (ở đây là phương thức `Bearer` token), giúp client biết cách chuẩn bị request đúng chuẩn cho lần gọi tiếp theo.
2. **Trả cùng một thông báo lỗi:**
   Đây là nguyên tắc bảo mật thông tin (Information Disclosure Avoidance). Nếu trả về thông báo lỗi chi tiết (ví dụ: *"Token này tồn tại nhưng bị hết hạn"* hoặc *"Scheme đúng nhưng chữ ký token bị sai"*), kẻ tấn công có thể lợi dụng sự khác biệt này để dò tìm xem token nào là hợp lệ hoặc phân tích cơ chế xác thực của hệ thống. Trả về cùng một phản hồi `401` chung chung ngăn chặn việc kẻ tấn công suy đoán trạng thái hệ thống.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

1. **Số request gửi được trước khi bị 429:** **10 requests**. Dù client có im lặng bao lâu (10 phút hay 100 phút), số token nạp lại tối đa luôn bị chặn ở mức `capacity = 10`.
2. **Nếu bỏ đoạn `min(capacity, ...)`:**
   Sau 10 phút (600 giây), với tốc độ refill là 10 tokens/phút, thuật toán sẽ tính toán cộng thêm 10 * 10 = 100 tokens. Khi đó, client có thể gửi liên tiếp **110 requests** (10 tokens ban đầu + 100 tokens tích lũy) trước khi bị dính lỗi 429.
   *Lý do:* Đoạn `min(capacity, ...)` có vai trò đặt "trần" cho xô (bucket limit). Bỏ đoạn này sẽ làm xô bị "tràn vô hạn", dẫn đến tình trạng client có thể tích trữ một lượng token cực lớn trong thời gian không hoạt động và xả ra một đợt tấn công bùng nổ (traffic burst) làm sập hệ thống.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

1. **Hạn mức $30/tháng:**
   - **Thiệt hại tối đa:** **$30.0** (client có thể tiêu sạch toàn bộ ngân sách của cả tháng chỉ trong vài giờ đầu tiên của ngày đầu tháng).
   - **Tự hồi phục khi nào:** Service sẽ chặn client đó cho đến tận **đầu tháng sau** (mất gần 30 ngày client không thể sử dụng dịch vụ).
2. **Hạn mức $1/ngày:**
   - **Thiệt hại tối đa:** **$1.0** (ngay khi chi phí vượt $1 trong ngày, Cost Guard sẽ lập tức chặn client đó).
   - **Tự hồi phục khi nào:** Service sẽ tự động mở lại cho client đó vào **00:00:00 ngày hôm sau (UTC)** khi chu kỳ ngân sách ngày mới bắt đầu.
   *Kết luận:* Hạn mức theo ngày giúp khoanh vùng thiệt hại (blast radius) nhỏ hơn 30 lần và khôi phục hoạt động nhanh hơn rất nhiều.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

1. Redis bị sự cố mất kết nối trong 30 giây.
2. Endpoint gộp chung (kiểm tra cả app và Redis) bắt đầu trả về lỗi `500/503` do không thể kết nối tới Redis.
3. Orchestrator (Docker Swarm/Kubernetes) gọi Healthcheck probe và thấy endpoint báo lỗi. Nó hiểu lầm rằng **chính container Python/FastAPI đang bị treo/chết**.
4. Orchestrator lập tức tiến hành tiêu diệt (kill) cả 3 container ứng dụng và khởi động lại (restart) chúng.
5. Trong suốt 30 giây Redis ngắt kết nối, cả 3 container liên tục rơi vào vòng lặp bị kill -> khởi động lại -> fail healthcheck -> bị kill tiếp.
6. Khi Redis phục hồi sau 30 giây, các container ứng dụng vẫn đang trong quá trình khởi động lại rải rác, dẫn tới kéo dài thời gian gián đoạn dịch vụ (downtime) không cần thiết và làm tăng tải cho CPU máy Host.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Thông báo lỗi:**
  `Error: Invalid value for '--port': '$PORT' is not a valid integer.`
- **Nguyên nhân:**
  Trong file `railway.toml`, thuộc tính `startCommand` được đặt là `"uvicorn app.main:app --host 0.0.0.0 --port $PORT"`. Khi Railway thực thi lệnh này, nó chạy trực tiếp tiến trình mà không thông qua Shell wrapper. Do đó, chuỗi `$PORT` không được Shell expand thành giá trị cổng thực tế (như `8000`), làm `uvicorn` nhận giá trị dạng string `"$PORT"` và báo lỗi không phải số nguyên (`integer`).
- **Cách sửa:**
  Cập nhật thuộc tính `startCommand` trong file `railway.toml` bọc lệnh khởi chạy qua `sh -c`:
  `startCommand = "sh -c 'exec uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}'"`
