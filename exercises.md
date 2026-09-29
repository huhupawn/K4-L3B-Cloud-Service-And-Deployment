# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền câu trả lời trực tiếp bên dưới mỗi câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Anh Hoàng  Mã học viên: 2A202602816

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống cụ thể: Khi deploy ứng dụng lên môi trường cloud (như Railway hoặc Render), developer có thể sơ suất quên cấu hình biến môi trường `AGENT_API_KEY` trong mục Variables của dashboard.
- Nếu để giá trị mặc định (ví dụ `"changeme"` hoặc `"sk-default"`): Ứng dụng vẫn khởi động bình thường, health check báo 200 OK. Hệ thống chạy trong nhiều ngày mà developer không hề hay biết rằng API đang dùng khóa mặc định. Kẻ tấn công có thể quét các endpoint công khai và dùng các khóa mặc định phổ biến để gọi trái phép vào service `/ask`, làm tiêu hao ngân sách hoặc spam mô hình LLM mà developer chỉ phát hiện ra khi nhận hóa đơn chi phí phát sinh.
- Với việc không đặt giá trị mặc định (Fail Fast): Ứng dụng sẽ ném ra ngoại lệ `ValidationError` của Pydantic và dừng tiến trình ngay lúc khởi động. Quá trình deploy lập tức báo Failed trên màn hình dashboard. Developer phát hiện lỗi ngay tại thời điểm deploy và bắt buộc phải cấu hình khóa API an toàn trước khi service tiếp nhận bất kỳ request nào từ bên ngoài.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON mẫu thu được từ service:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T08:15:30.123456+00:00", "user_id": "sv01", "tokens_in": 15, "tokens_out": 42, "cost_usd": 0.000171}`

Hai việc làm được với dòng log có cấu trúc này mà `print("đã trả lời xong")` không làm được:
1. **Lọc, tổng hợp và phân tích định lượng tự động (Metrics & Aggregation)**: Các hệ thống thu thập log tập trung (như Datadog, Grafana Loki, CloudWatch, Elasticsearch) có thể tự động parse JSON theo dòng và lập chỉ mục từng trường. Ta có thể chạy truy vấn tính toán như `SELECT sum(cost_usd) GROUP BY user_id` để biết user nào tiêu nhiều chi phí nhất trong ngày, tính tổng lượng token tiêu thụ, hoặc thống kê thời gian phản hồi mà không cần viết regex bóc tách chuỗi phức tạp.
2. **Cảnh báo tự động theo thời gian thực (Real-time Alerting)**: Có thể cấu hình cảnh báo tự động khi một user có `cost_usd` vượt quá ngưỡng cho phép trong ngày, hoặc khi tần suất sự kiện có `level == "error"` tăng đột biến vượt quá 5% trong 5 phút. Với `print("đã trả lời xong")`, log không có trường `level`, không có timestamp ISO và không có metadata nên không thể thiết lập cảnh báo chính xác.

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
| 1 stage (bản đầu) | 1.02 GB |
| Multi-stage | 185 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~835 MB) bao gồm:
- Bản 1 stage dùng base image `python:3.11` đầy đủ dựa trên Debian tiêu chuẩn, chứa toàn bộ build toolchain (gcc, g++, make), các gói header (python-dev, libc-dev), các tiện ích phát triển (curl, git, apt package cache, man pages...).
- Bản multi-stage tách thành stage `builder` để cài đặt thư viện và stage runtime sử dụng `python:3.11-slim`, chỉ sao chép các gói Python đã cài đặt trong `/root/.local` từ builder sang. Runtime image không chứa compiler, không có build dependencies, không có cache của pip (`--no-cache-dir`) và không có các công cụ dev thừa thãi, giúp image nhỏ gọn, kéo image nhanh hơn và hạn chế tối đa lỗ hổng bảo mật.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Với Dockerfile hiện tại: Khi sửa 1 ký tự trong `app/main.py` rồi build lại:
  + Các layer từ `FROM ...`, `WORKDIR`, `COPY requirements.txt .`, `RUN pip install ...` đều được dùng lại hoàn toàn từ cache (`CACHED`) vì file `requirements.txt` không có thay đổi.
  + Chỉ các layer từ `COPY . .` trở xuống (bao gồm bước copy mã nguồn và phân quyền user) mới phải chạy lại. Thời gian build lại chỉ mất 1-2 giây.
- Nếu đặt `COPY . .` lên trước `RUN pip install`:
  + Mỗi khi sửa code dù chỉ 1 ký tự, checksum của thư mục thay đổi làm mất hiệu lực cache ngay từ layer `COPY . .`.
  + Docker buộc phải thực thi lại toàn bộ layer `RUN pip install` phía sau, tải và cài đặt lại tất cả các gói thư viện từ Internet mỗi lần build, làm thời gian build kéo dài từ vài giây lên vài phút và lãng phí băng thông mạng.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện dẫn tới việc chiếm quyền máy host:
1. Ứng dụng Python tồn tại lỗ hổng (ví dụ Remote Code Execution qua injection hoặc thư viện bên thứ ba bị tấn công).
2. Kẻ tấn công kích hoạt mã độc từ xa và chiếm quyền điều khiển tiến trình. Vì container mặc định chạy bằng root (UID 0), kẻ tấn công sở hữu toàn quyền root bên trong container namespace.
3. Kẻ tấn công khai thác tiếp một lỗ hổng container breakout (như lỗ hổng kernel Linux, bug trong runc/Docker daemon, hoặc container bị mount volume nhạy cảm từ host).
4. Do tiến trình trong container mang UID 0, khi thoát được ra ngoài host namespace, tiến trình đó tiếp tục có quyền root (UID 0) trên máy host thực tế, cho phép kẻ tấn công kiểm soát toàn bộ hệ thống máy chủ.

Lệnh `USER appuser` cắt đứt chuỗi tấn công ngay tại bước 2: Tiến trình Python chạy với quyền người dùng thông thường không có đặc quyền (unprivileged user, UID 1000). Dù kẻ tấn công RCE thành công, họ không thể chỉnh sửa file hệ thống của container, không có các Linux capabilities nguy hiểm (như CAP_SYS_ADMIN), và bị chặn đứng hoàn toàn khả năng thực hiện container breakout sang máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

- Con số tối đa: 20 request trong 2 giây liên tiếp.
- Cách đạt được:
  Cơ chế đếm theo phút đồng hồ (fixed window) reset bộ đếm về 0 vào mỗi đầu phút (giây :00).
  Người dùng có thể gửi toàn bộ 10 request vào giây cuối cùng của phút thứ nhất (10:00:59) - hệ thống ghi nhận 10 request và cho phép vì chưa vượt hạn mức của phút đó.
  Ngay 1 giây sau, khi đồng hồ chuyển sang phút tiếp theo (10:01:00), bộ đếm bị reset về 0. Người dùng lập tức gửi tiếp 10 request trong giây này và tiếp tục được cho qua.
  Tổng cộng trong vòng 2 giây (10:00:59 - 10:01:00), người dùng đã gửi thành công 20 request, gây ra một đợt burst lưu lượng gấp đôi hạn mức quy định và có thể làm quá tải hoặc nghẽn hệ thống.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- Sự khác nhau:
  + **Rate Limit**: Giới hạn **tần suất / số lượng** request trong một khoảng thời gian ngắn (ví dụ: 10 request/phút) để bảo vệ server khỏi quá tải tài nguyên CPU/RAM, nghẽn mạng và tấn công DoS/Spam.
  + **Cost Guard**: Giới hạn **tổng chi phí tài chính / mức tiêu thụ token** trong chu kỳ dài (ví dụ: tối đa 10 USD/tháng) nhằm bảo vệ ngân sách trước chi phí gọi các mô hình AI/LLM.

- Tình huống Rate Limit cho qua nhưng Cost Guard phải chặn:
  Một người dùng chỉ gửi 2 request trong 1 phút (tần suất rất thấp, rate limit 10 req/phút cho qua dễ dàng), nhưng mỗi request kèm theo một tài liệu văn bản khổng lồ 100.000 tokens kèm yêu cầu phân tích sâu, tiêu tốn 5.5 USD mỗi request. Sau 2 request, người dùng này tiêu hết 11 USD (vượt ngân sách 10 USD/tháng). Ở request tiếp theo, Cost Guard sẽ lập tức chặn lại và trả về lỗi 402 Payment Required.

- Tình huống Cost Guard cho qua nhưng Rate Limit phải chặn:
  Đầu tháng người dùng chưa tiêu đồng nào (số dư chi tiêu = 0 USD). Người dùng chạy script gửi liên tục 15 request trong 2 giây với nội dung rất ngắn: 'hi', 'ping' (mỗi request chỉ vài chục token, tương đương 0.00001 USD, tổng chi phí không đáng kể ~0.00015 USD so với ngân sách 10 USD). Cost Guard thấy ngân sách còn rất nhiều, nhưng Rate Limit sẽ phát hiện tần suất vượt quá 10 req/phút và chặn ngay từ request thứ 11 với mã lỗi 429 Too Many Requests.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện xảy ra khi gộp `/health` và `/ready` làm một và phụ thuộc vào Redis khi Redis mất kết nối 30 giây:
1. Redis gặp sự cố tạm thời (mạng gián đoạn hoặc Redis đang restart trong 30 giây).
2. Liveness probe của cả 3 container agent kiểm tra Redis thất bại và trả về mã lỗi 503 hoặc timeout.
3. Bộ điều phối (Docker/Kubernetes/Cloud platform) hiểu rằng cả 3 container agent đã bị deadlock/treo và tự động gửi tín hiệu kill, sau đó restart lại đồng loạt cả 3 container.
4. Quá trình khởi động lại đồng thời (Boot Storm) gây quá tải CPU/RAM của host, các container liên tục khởi động, cố gắng kết nối lại Redis nhưng Redis vẫn chưa sẵn sàng, dẫn đến việc container lại bị fail health check và tiếp tục bị restart (vòng lặp CrashLoopBackOff).
5. Toàn bộ cụm dịch vụ sập hoàn toàn (Cascading failure/Outage), không còn container nào phục vụ người dùng kể cả các request tĩnh không cần Redis.
(Nếu tách riêng: `/health` chỉ kiểm tra process Python còn sống thì container không bị restart; `/ready` trả về 503 để load balancer tạm ngưng điều phối traffic tới agent cho đến khi Redis kết nối lại bình thường).

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- Khi lưu lịch sử trong một dict Python trong RAM của process (Stateful):
  Do 3 container agent chạy trên 3 tiến trình riêng biệt với các vùng nhớ RAM hoàn toàn cô lập:
  + Request 1 gửi đến Container A: `history_length` trả về là 0 (Container A lưu câu 1 vào dict của mình).
  + Request 2 được load balancer chuyển sang Container B: Vì RAM của Container B chưa có thông tin user này, `history_length` vẫn trả về là 0!
  + Request 3 chuyển sang Container C: `history_length` tiếp tục trả về là 0.
  + Request 4 quay lại Container A: `history_length` nhảy lên 2 (vì nhớ câu 1 từ request đầu).
  Người dùng sẽ thấy `history_length` tăng giảm thất thường, agent liên tục mất trí nhớ và trả lời không liên quan đến mạch hội thoại trước đó.
- Ngược lại khi dùng Redis (Stateless): Cả 3 container đều đọc/ghi vào một nguồn dữ liệu tập trung duy nhất trên Redis (`history:<user_id>`), do đó dù request rơi vào bất kỳ container nào, `history_length` luôn tăng đều đặn: 0 -> 2 -> 4 -> 6..., đảm bảo hội thoại nhất quán và liền mạch.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- Thông báo lỗi thực tế khi deploy lên cloud (như Render / Railway):
  `Health check failed: connection refused to port 8000. Application failed to respond within 60 seconds. Container stopped.`
- Cách tìm ra nguyên nhân:
  Mở tab Deployment Logs trên dashboard của nền tảng cloud. Nhận thấy nền tảng tự động gán một cổng ngẫu nhiên qua biến môi trường (ví dụ `PORT=10000`) và thực hiện probe vào cổng này. Tuy nhiên, lệnh khởi chạy uvicorn trong Dockerfile ban đầu bị hardcode `--port 8000`, khiến uvicorn chỉ lắng nghe ở 8000 thay vì cổng được gán, dẫn tới việc nền tảng probe vào port 10000 bị từ chối kết nối (`Connection Refused`).
- Cách sửa:
  Sửa lệnh chạy uvicorn trong Dockerfile để đọc biến môi trường:
  `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]`
  Đồng thời cấu hình `HEALTHCHECK` trong Dockerfile đọc đúng cổng `${PORT:-8000}`. Sau khi cập nhật và push lại, service bind đúng cổng của cloud và health check chuyển sang màu xanh (200 OK).
