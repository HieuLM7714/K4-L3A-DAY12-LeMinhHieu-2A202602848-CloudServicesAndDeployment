# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder bên dưới bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lê Minh Hiếu  Mã học viên: 2A202602848

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống cụ thể: Khi deploy service lên cloud (Render/Railway), lập trình viên quên khai báo biến `AGENT_API_KEY` trong dashboard cấu hình biến môi trường.
- Nếu có giá trị mặc định như `"changeme"`: Service vẫn khởi động thành công, health check báo xanh và lập trình viên ngỡ rằng hệ thống đã an toàn. Tuy nhiên, khóa `"changeme"` là khóa phổ biến đã nằm trong mã nguồn. Các bot tự động quét Internet liên tục rà soát các API public bằng từ điển khóa mặc định sẽ nhanh chóng bypass lớp xác thực và gọi ồ ạt vào endpoint `/ask`. Hậu quả là hạn mức tài khoản LLM bị đốt sạch trong vài giờ và phát sinh chi phí khổng lồ trước khi bạn kịp nhận ra.
- Khi không có mặc định (fail fast): Ngay khi khởi động, Pydantic ném lỗi `ValidationError: Field required: agent_api_key`. Container crash ngay lập tức tại bước deployment preview, log hiển thị rõ ràng thông báo lỗi trước mắt bạn. Việc app "chết sớm" buộc bạn phải cấu hình đúng secret ngay lập tức trước khi traffic bên ngoài có thể tiếp cận service, ngăn chặn triệt để rủi ro rò rỉ bảo mật và thiệt hại tài chính.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được từ service:
```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T15:46:50.123456+00:00", "user_id": "sv-test", "tokens_in": 43, "tokens_out": 47, "cost_usd": 0.00003465}
```
Hai việc làm được với dòng log này:
1. **Truy vấn, lọc và tổng hợp số liệu theo trường (Structured Querying & Aggregation)**: Khi log được đẩy về các công cụ tập trung như Datadog, CloudWatch hay ElasticSearch, ta có thể viết truy vấn chính xác: `user_id == "sv-test" AND cost_usd > 0.001` hoặc vẽ biểu đồ tổng chi phí tích lũy theo từng user trong 24 giờ qua. Với `print("đã trả lời xong")`, log không có thông tin user, chi phí hay token, máy móc không thể bóc tách để tính toán.
2. **Cấu hình cảnh báo tự động (Automated Alerting & Metrics)**: Ta có thể thiết lập alert rule: "Nếu tỷ lệ log có `level == "error"` vượt quá 5% trong 5 phút" hoặc "Nếu có log event `ask_completed` với `cost_usd > 1.0` thì gửi thông báo PagerDuty/Slack ngay lập tức". `print()` dạng chuỗi tự do không có `level`, không có `timestamp` chuẩn hóa ISO-8601 nên không thể kích hoạt các cảnh báo tự động một cách tin cậy.

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

Phần dung lượng chênh lệch hơn 800 MB bao gồm:
1. **Hệ điều hành nền**: `python:3.11` đầy đủ dựa trên Debian bản chuẩn chứa toàn bộ các tiện ích hệ thống, trình biên dịch C/C++ (`gcc`, `g++`), `make`, thư viện header C (`libc-dev`), công cụ đồ họa/âm thanh, gói man pages và tài liệu không bao giờ dùng khi chạy app. Trong khi đó, `python:3.11-slim` đã loại bỏ hoàn toàn các gói thừa này.
2. **Dữ liệu tạm trong quá trình build**: Ở bản 1-stage, cache của pip (`~/.cache/pip`), các file object `.o`, `.whl` trung gian và apt cache (`/var/lib/apt/lists/*`) bị lưu vĩnh viễn vào các layer của Docker image. Với multi-stage, stage `builder` chịu trách nhiệm cài đặt và biên dịch, sau đó stage `runtime` chỉ copy thư mục kết quả `/install` sang `/usr/local`, hoàn toàn bỏ lại toàn bộ compiler và cache build ở phía sau.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- **Với Dockerfile hiện tại**:
  - Các layer trước `COPY app ./app` (bao gồm `FROM python:3.11-slim`, `WORKDIR /app`, `COPY requirements.txt .`, `RUN pip install ...`, `COPY --from=builder ...`, `RUN useradd ...`) đều không thay đổi nên **100% được lấy từ Docker cache (`CACHED`)**.
  - Chỉ có layer `COPY app ./app` và các layer sau nó (`USER appuser`, `CMD ...`) là phải chạy lại. Quá trình build hoàn thành trong chưa đầy 1 giây.
- **Nếu đặt `COPY . .` lên trước `RUN pip install`**:
  - Docker cache hoạt động theo nguyên tắc: khi một layer bị thay đổi (cache miss), toàn bộ các layer tiếp theo sau nó đều bị vô hiệu hóa (cache invalidated).
  - Do `COPY . .` chứa `app/main.py`, khi sửa 1 ký tự, layer `COPY . .` bị miss. Kéo theo lệnh `RUN pip install` phía sau buộc phải chạy lại từ đầu: tải lại và cài đặt lại toàn bộ các thư viện Python mỗi lần commit code. Thời gian build bị kéo dài từ vài giây lên vài phút mỗi lần phát triển hoặc CI build.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện tấn công (Container Breakout Attack Chain):
1. **Khai thác lỗ hổng cấp ứng dụng**: Kẻ tấn công khai thác một lỗi Remote Code Execution (RCE) trong code Python (ví dụ: `eval()`, `pickle.loads()`, hoặc lỗ hổng parser) để thực thi lệnh shell tùy ý bên trong container.
2. **Quyền mặc định trong container**: Do container chạy bằng user `root` (UID 0), tiến trình của kẻ tấn công có toàn quyền quản trị bên trong container namespace (đọc ghi mọi file hệ thống, cài thêm công cụ tấn công).
3. **Thoát container (Container Breakout)**: Kẻ tấn công lợi dụng một lỗ hổng trong nhân Linux kernel (như Dirty COW, cgroup release_agent, hoặc container mount socket docker `/var/run/docker.sock`) để thoát ra ngoài host. Vì UID trong container ánh xạ trực tiếp sang UID trên Linux host (đều là UID 0), kẻ tấn công lập tức có quyền `root` tối cao trên toàn bộ máy chủ vật lý/máy ảo của hạ tầng.
- **Lệnh `USER appuser` cắt đứt chuỗi ở bước 2**: Lệnh `USER` hạ quyền tiến trình xuống một tài khoản unprivileged (`UID 10001`). Khi bị RCE, kẻ tấn công chỉ có quyền của user thường trong container: không thể ghi đè file hệ thống, không thể load kernel module, không thể khai thác hầu hết các kỹ thuật breakout đòi hỏi quyền root/CAP_SYS_ADMIN. Ngay cả khi có lỗ hổng thoát container, kẻ tấn công ra ngoài host cũng chỉ là một user vô danh không có quyền hạn.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

- **Số request tối đa trong 2 giây liên tiếp**: **20 request** (gấp đôi hạn mức cho phép).
- **Cách đạt được**:
  - Với cơ chế fixed window đếm theo phút đồng hồ (reset lúc giây 00):
    - Người dùng gửi 10 request dồn dập vào giây cuối cùng của phút thứ nhất: lúc `10:00:59`. Vì trong phút `10:00`, họ mới dùng 10 request $\le$ hạn mức 10, nên cả 10 request đều được cho qua.
    - Đúng 1 giây sau, đồng hồ chuyển sang `10:01:00`. Bộ đếm của phút mới được reset về 0. Người dùng lập tức gửi tiếp 10 request nữa lúc `10:01:01`. Vì trong phút `10:01`, số request là 10 $\le$ 10, nên 10 request này tiếp tục được cho qua.
    - Kết quả: Từ `10:00:59` đến `10:01:01` (khoảng thời gian chỉ 2 giây), hệ thống đã phải gánh chịu **20 request** liên tiếp. Hiện tượng này gọi là Boundary Bursting (tràn lưu lượng ở ranh giới cửa sổ). Thuật toán Sliding Window sử dụng Redis ZSET giải quyết triệt để lỗ hổng này bằng cách đo lường chính xác khoảng thời gian liên tục `[t - 60s, t]`.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- **Sự khác nhau cốt lõi**:
  - **Rate Limit** kiểm soát **tần suất/vận tốc gọi** (Velocity/Frequency - đơn vị: request/phút) để bảo vệ hạ tầng máy chủ và API gateway không bị quá tải (DDoS/traffic spike).
  - **Cost Guard** kiểm soát **tổng chi phí tài chính tích lũy** (Financial Budget - đơn vị: USD/tháng) dựa trên số lượng token LLM tiêu thụ thực tế để bảo vệ ngân sách của doanh nghiệp.
- **Tình huống Rate Limit cho qua nhưng Cost Guard chặn**:
  - User chỉ gửi đúng 1 request trong 10 phút (vận tốc cực thấp: 0.1 req/phút $\ll$ 10 req/phút của rate limit nên rate limit cho qua). Tuy nhiên, tài liệu đính kèm trong prompt dài tới 100.000 token khiến chi phí ước tính là $0.05. Lúc này ngân sách tháng của user đã tiêu hết ($10.0 / $10.0), Cost Guard lập tức chặn với mã lỗi `402 Payment Required`.
- **Tình huống Cost Guard cho qua nhưng Rate Limit chặn**:
  - Đầu tháng, user có nguyên vẹn ngân sách $10.0 (chưa tiêu đồng nào). User dùng script gửi 25 câu hỏi ngắn liên tiếp ("hi", "test") trong vòng 10 giây. Tổng chi phí của 25 câu hỏi này chỉ khoảng $0.00025 (rất nhỏ so với ngân sách $10.0 nên Cost Guard hoàn toàn cho phép). Nhưng vì tốc độ gọi vượt quá ngưỡng 10 request/phút, Rate Limiter lập tức chặn từ request thứ 11 với mã lỗi `429 Too Many Requests`.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện (Cascading Failure / CrashLoopBackOff):
1. **Redis gặp sự cố**: Lúc `t=0s`, Redis tạm thời mất kết nối (restart, network blip hoặc failover) trong 30 giây.
2. **Health probe thất bại**: Vì `/health` (liveness probe) kiểm tra Redis, khi Redis không phản hồi, `/health` của cả 3 container agent đều trả về lỗi hoặc timeout.
3. **Orchestrator kill container**: Container orchestrator (Docker daemon / Kubernetes / Cloud platform) cho rằng tiến trình của container đã bị treo/hỏng (unhealthy liveness) $\rightarrow$ Orchestrator gửi lệnh `SIGKILL` và restart đồng loạt cả 3 container agent.
4. **Hệ thống sập hoàn toàn (Total Outage)**: Trong khi container đang khởi động lại (restart cycle), không còn bất kỳ instance nào hoạt động để phục vụ traffic. Toàn bộ người dùng truy cập web đều nhận lỗi `502 Bad Gateway`.
5. **Vòng lặp khởi động chết chóc (Crash Loop)**: Khi các container vừa bật lên, Redis vẫn chưa xong 30 giây phục hồi $\rightarrow$ container mới lại kiểm tra Redis và lại fail $\rightarrow$ orchestrator lại kill tiếp. Một sự cố nhỏ ở tầng cache đã bị khuếch đại thành sập toàn bộ hệ thống.
*(Nếu tách riêng: `/ready` trả 503 giúp load balancer tạm ngưng điều phối traffic tới agent, trong khi `/health` vẫn 200 giúp container sống bình yên chờ Redis online trở lại)*.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- **Khi lưu trên Redis (Stateless - Hiện tại)**:
  - Dù request được Load Balancer điều phối ngẫu nhiên (Round-Robin) tới container 1, container 2 hay container 3, vì cả 3 container đều đọc/ghi vào cùng một Redis tập trung, giá trị `history_length` sẽ **tăng đơn điệu và liên tục theo từng lượt hỏi**: `0 -> 2 -> 4 -> 6 -> 8...`. Agent ghi nhớ hoàn hảo ngữ cảnh cuộc trò chuyện.
- **Nếu lưu trong dict Python (Stateful trong RAM của container)**:
  - Mỗi container sở hữu một vùng nhớ RAM hoàn toàn cô lập.
  - Lượt 1: Request vào Container A $\rightarrow$ A lưu vào dict của A $\rightarrow$ `history_length` = 0.
  - Lượt 2: Load balancer đẩy sang Container B $\rightarrow$ Dict của B rỗng $\rightarrow$ `history_length` vẫn là 0 (agent B bị "mất trí nhớ").
  - Lượt 3: Load balancer đẩy sang Container C $\rightarrow$ Dict của C rỗng $\rightarrow$ `history_length` lại là 0.
  - Lượt 4: Request quay lại Container A $\rightarrow$ A thấy 2 message từ lượt 1 $\rightarrow$ `history_length` = 2.
  - Kết quả: `history_length` sẽ nhảy lung tung gián đoạn (0, 0, 0, 2, 2, 4...) tùy thuộc vào việc request rơi trúng container nào, gây ra hiện tượng agent mất trí nhớ ngẫu nhiên đối với người dùng.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Lỗi gặp phải**: Khi deploy web service lên nền tảng đám mây Render, dịch vụ build thành công nhưng health check probe của Render liên tục báo timeout và chuyển trạng thái service sang `Failed`.
- **Thông báo lỗi trên log**:
  `Timed out waiting for health check at /health on port 10000` hoặc `Container failed to respond on port 10000`.
- **Cách tìm ra nguyên nhân**:
  - Mở tab **Logs** trên Render dashboard, quan sát thấy dòng lệnh Uvicorn khởi động:
    `INFO: Started server process ... Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)`
  - Trong khi đó, Render cấp phát cổng ngẫu nhiên cho container thông qua biến môi trường `$PORT` (ví dụ `PORT=10000`).
  - Do Dockerfile ban đầu hardcode `--port 8000`, Uvicorn chỉ lắng nghe trên cổng 8000 trong khi Load Balancer của Render gửi health check probe vào cổng 10000, dẫn đến kết nối bị từ chối (Connection Refused / Timeout).
- **Cách sửa**:
  - Cập nhật lệnh `CMD` trong `Dockerfile` sử dụng shell parameter expansion để đọc biến môi trường `$PORT` do platform cấp:
    `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]`
  - Đồng thời cập nhật `app/config.py` để trường `port: int = 8000` tự động ánh xạ với biến môi trường `PORT`. Sau khi commit và push lại, Render kích hoạt probe thành công và service chuyển sang trạng thái `Live`.
