# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder câu hỏi bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Phùng Quốc Việt  Mã học viên: 2A202602456

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi triển khai ứng dụng lên môi trường Production (như Render hoặc Railway), nếu người triển khai sơ suất quên cấu hình biến môi trường `AGENT_API_KEY`, việc không có giá trị mặc định sẽ khiến container dừng ngay lập tức (fail fast) trong quá trình khởi động. Nhờ cơ chế này, hệ thống điều phối cloud lập tức phát hiện container không pass healthcheck, từ chối đưa container vào nhận traffic và báo lỗi đỏ rõ ràng trên dashboard để kỹ sư bổ sung ngay lập tức.
Ngược lại, nếu để giá trị mặc định là `"changeme"`, server vẫn sẽ khởi động thành công và báo trạng thái xanh mượt, nhưng lúc này dịch vụ đang chạy với một khóa công khai ai cũng đoán được. Kẻ tấn công hoặc bot quét tự động có thể lợi dụng khóa `"changeme"` này để liên tục gọi API `/ask`, làm tiêu hao hạn mức tài khoản LLM hoặc phát sinh rủi ro bảo mật nghiêm trọng mà đội ngũ vận hành hoàn toàn không hay biết.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

```json
{"timestamp": "2026-09-29T05:37:09.123456Z", "level": "info", "event": "ask_completed", "user_id": "sv-test", "duration_ms": 142.5, "cost_usd": 0.00003465, "tokens": {"in": 43, "out": 47}}
```

Hai việc làm được với dòng log JSON mà lệnh print thông thường không làm được:
1. **Tự động bóc tách và lập Dashboard giám sát thời gian thực:** Các hệ thống thu thập log tập trung (như ElasticSearch, Datadog, Grafana Loki) có thể tự động parse các trường JSON có cấu trúc (`duration_ms`, `cost_usd`, `user_id`, `tokens`) để vẽ biểu đồ theo dõi độ trễ, lưu lượng token và chi phí phát sinh mà không cần viết các hàm regex bóc tách chuỗi phức tạp và dễ gãy vỡ.
2. **Thiết lập hệ thống cảnh báo tự động (Alerting) theo ngưỡng:** Dựa trên cấu trúc JSON, ta có thể cài đặt rule cảnh báo tự động gửi thông báo qua Slack/Telegram khi phát hiện bất thường, ví dụ: kích hoạt cảnh báo khi `cost_usd > 0.05` trên một request hoặc khi tỷ lệ lỗi `level == "error"` vượt ngưỡng 5% trong 5 phút.

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
| 1 stage (bản đầu) | ~850 MB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (gần 600 MB) bao gồm:
1. Toàn bộ các công cụ biên dịch (compilers), header C/C++ và build-tools như `gcc`, `g++`, `make`, `python3-dev` cần thiết trong lúc cài đặt một số thư viện Python.
2. Thư mục cache của pip (`/root/.cache/pip`) lưu trữ các file wheel và source code tải về từ Internet.
3. Các file tạm thời sinh ra trong quá trình cài đặt package.
Multi-stage build đã tách biệt hoàn toàn giữa stage biên dịch (`builder`) và stage thực thi (`runtime`). Chỉ có các package hoàn chỉnh đã cài đặt mới được copy sang image runtime (dùng base image `python:3.11-slim`), giúp loại bỏ sạch sẽ toàn bộ công cụ biên dịch thừa, giữ cho image cuối cùng cực kỳ gọn nhẹ và an toàn.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Khi sửa một ký tự trong `app/main.py` rồi build lại:
  - Các layer từ đầu cho tới trước lệnh `COPY app/ ./app/` (bao gồm: khai báo base image, tạo non-root `appuser`, thiết lập `WORKDIR`, `COPY requirements.txt` và `RUN pip install`) đều không có bất kỳ thay đổi nào nên Docker sẽ **dùng lại hoàn toàn từ cache (CACHED)**.
  - Chỉ có layer `COPY app/ ./app/` và các chỉ thị phía sau (`USER`, `EXPOSE`, `CMD`) là phải thực thi lại. Do đó, quá trình build chỉ mất 1-2 giây.
- Nếu đặt `COPY . .` lên trước `RUN pip install`:
  - Mỗi khi sửa bất kỳ file mã nguồn nào (dù chỉ đổi 1 ký tự), checksum của layer `COPY . .` sẽ bị thay đổi.
  - Theo cơ chế caching của Docker, khi một layer bị invalid cache thì toàn bộ các layer phía sau nó đều bị hủy cache. Điều này buộc Docker phải tải lại toàn bộ các thư viện và chạy lại lệnh `RUN pip install` từ đầu, khiến thời gian build kéo dài từ vài giây lên vài phút mỗi lần lập trình viên sửa code.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện dẫn đến chiếm quyền máy host:
1. Mã nguồn Python tồn tại lỗ hổng (ví dụ: Command Injection hoặc Remote Code Execution - RCE qua `pickle`/`eval`).
2. Kẻ tấn công gửi payload khai thác lỗ hổng và mở được một interactive shell bên trong container.
3. Nếu container chạy với user mặc định là `root`, tiến trình shell của kẻ tấn công sẽ có UID 0. Vì container dùng chung kernel với máy host, nếu máy chủ tồn tại lỗ hổng kernel (như Dirty COW), lỗi trong Docker runtime (`runc`) hoặc cấu hình mount nhầm `/var/run/docker.sock`, kẻ tấn công với quyền UID 0 trong container có thể khai thác để thoát khỏi container (container escape).
4. Sau khi thoát ra máy host, vì ban đầu tiến trình mang UID 0, kẻ tấn công lập tức có được quyền `root` tối cao trên toàn bộ máy chủ vật lý của bạn.

Lệnh `USER appuser` cắt đứt chuỗi tấn công ngay tại **bước 2**: Tiến trình của ứng dụng bị giam trong một tài khoản người dùng bình thường không có đặc quyền (UID 10001). Khi kẻ tấn công chiếm được shell, chúng chỉ có quyền của user thường: không thể ghi vào hệ thống file root, không dùng được `sudo`, và không đủ quyền thực hiện các cuộc tấn công leo thang đặc quyền để escape ra máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

- Con số tối đa: **20 request** trong 2 giây liên tiếp.
- Cách đạt được:
  - Giả sử phút thứ nhất kết thúc ở mốc `12:00:59` và phút tiếp theo bắt đầu ở mốc `12:01:00`.
  - Lúc `12:00:59`, người dùng gửi dồn dập 10 request (vừa vặn chạm hạn mức 10 req/phút của phút hiện tại).
  - Ngay 1 giây sau đó, tức mốc `12:01:00`, đồng hồ bước sang phút mới và bộ đếm fixed window tự động reset về 0. Người dùng lập tức gửi tiếp 10 request nữa.
  - Tổng cộng trong khoảng thời gian chỉ vỏn vẹn 2 giây (từ giây 59 sang giây 00), hệ thống đã phải tiếp nhận và xử lý tới 20 request (gấp đôi hạn mức quy định).
- Thuật toán Sliding Window (dùng Sorted Set trong Redis) giải quyết triệt để vấn đề này vì nó luôn tính tổng số request trong đúng cửa sổ 60 giây trượt tính lùi từ thời điểm gửi request hiện tại (`now - 60s`), loại bỏ hoàn toàn hiện tượng dồn tải ở ranh giới phút đồng hồ.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- **Điểm khác biệt cốt lõi:**
  - `Rate Limiting`: Bảo vệ hạ tầng kỹ thuật và tính sẵn sàng của server trước nguy cơ quá tải hoặc tấn công từ chối dịch vụ (DoS) trong thời gian ngắn (tính theo giây/phút), không quan tâm request đó tốn bao nhiêu chi phí tài chính.
  - `Cost Guard`: Bảo vệ ngân sách tài chính của dự án trong dài hạn (tính theo tháng), ngăn chặn việc chi tiêu vượt ngân sách do gọi LLM API đắt đỏ, không quan tâm request gửi nhanh hay chậm.

- **Tình huống Rate Limit cho qua nhưng Cost Guard chặn:**
  - Một người dùng gọi API cực kỳ chậm rãi (chỉ 1 request mỗi 5 phút, hoàn toàn nằm trong hạn mức 10 req/phút). Tuy nhiên, request đó lại gửi một tài liệu khổng lồ 200.000 tokens kèm yêu cầu phân tích sâu, ước tính chi phí vượt quá ngân sách tháng 10.0 USD của tài khoản đó. Rate limit cho phép gửi, nhưng Cost guard sẽ phát hiện và ném lỗi `402 Payment Required`.

- **Tình huống Cost Guard cho qua nhưng Rate Limit chặn:**
  - Một bot gửi spam 50 request trong vòng 1 giây với payload câu hỏi rỗng hoặc chỉ có 1 từ. Chi phí LLM cho mỗi request này gần như bằng 0 (tổng chi phí chỉ $0.0001, rất nhỏ so với budget $10/tháng). Cost guard thấy tiền vẫn còn rất nhiều, nhưng Rate limiter sẽ lập tức can thiệp và trả về lỗi `429 Too Many Requests` từ request thứ 11 để ngăn chặn làm nghẽn CPU và mạng của server.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự các sự kiện xảy ra:
1. Khi Redis mất kết nối, endpoint `/health` (vì đã gộp chung logic kiểm tra Redis) lập tức trả về lỗi `503 Service Unavailable`.
2. Hệ thống điều phối (Orchestrator như Docker Swarm / Kubernetes / Cloud Health Checker) thăm dò liveness định kỳ. Khi thấy `/health` trả về 503 liên tiếp nhiều lần, nó kết luận là tiến trình agent đã bị hỏng hoặc deadlock.
3. Orchestrator lập tức cưỡng chế **kill (SIGKILL) và restart lại toàn bộ 3 container agent**.
4. Khi 3 container vừa khởi động lại xong, sự cố Redis vẫn chưa kết thúc (vì sự cố kéo dài 30 giây). Các container kiểm tra Redis tiếp tục thất bại và lại trả về 503 $\rightarrow$ Orchestrator lại tiếp tục restart cụm container lần nữa.
5. Cụm dịch vụ rơi vào trạng thái thảm họa **CrashLoopBackOff**: Tiêu hao tối đa CPU máy chủ để khởi động lại tiến trình, log hệ thống ngập tràn rác, và ngay cả khi Redis đã kết nối trở lại sau 30 giây, hệ thống vẫn mất thêm vài phút chập chờn để các container thoát khỏi chu kỳ restart liên tục.
*(Tách riêng `/health` không kiểm tra Redis và `/ready` kiểm tra Redis giúp container tiếp tục sống bình thường, load balancer chỉ tạm thời điều hướng traffic ra khỏi container qua `/ready` mà không restart tiến trình).*

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- Khi lưu lịch sử trong Redis (Stateless): Vì cả 3 container agent đều trỏ chung vào một cơ sở dữ liệu Redis, bất kể request tiếp theo của người dùng được Nginx điều phối vào container nào, container đó đều đọc được toàn bộ tin nhắn trước đó từ Redis List. Do đó, trường `history_length` tăng đều đặn và chuẩn xác (`0 -> 2 -> 4 -> 6...`).
- Nếu lưu lịch sử trong một dict Python in-memory: Mỗi container sở hữu một bộ nhớ RAM hoàn toàn cô lập. Khi Nginx phân phối request theo cơ chế Round-Robin:
  - Request 1 đến Container 1: `history_length = 0`, Container 1 lưu tin nhắn vào dict của riêng nó.
  - Request 2 đến Container 2: Container 2 chưa từng thấy user này trong dict của nó nên `history_length` lại là `0`!
  - Request 3 đến Container 3: Container 3 cũng chưa có dữ liệu nên `history_length` tiếp tục là `0`!
  - Request 4 quay lại Container 1: `history_length` đột ngột nhảy lên `2`.
  Kết quả là người dùng sẽ thấy `history_length` nhảy loạn xạ không theo quy luật, và AI Agent bị hiện tượng "mất trí nhớ từng chặng", không thể duy trì ngữ cảnh hội thoại ổn định.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Thông báo lỗi gặp phải:**
  Khi truy cập vào đường link gốc trên trình duyệt `https://day12-agent-wjd3.onrender.com/`, trang web trả về mã lỗi HTTP `404 Not Found` kèm nội dung JSON: `{"detail":"Not Found"}`. Ngoài ra, khi test bằng `curl.exe` từ PowerShell với cú pháp bọc nháy kép thông thường, lệnh bị trả về lỗi `422 Unprocessable Entity` (`JSON decode error: Expecting property name enclosed in double quotes`).
- **Cách tìm ra nguyên nhân:**
  1. Với lỗi 404: Kiểm tra lại mã nguồn `app/main.py` và log trên Render Dashboard, nhận thấy ứng dụng chỉ khai báo các endpoint `/health`, `/ready` và `/ask`, hoàn toàn không khai báo route gốc `@app.get("/")`. Do đó FastAPI trả về 404 cho route `/` là hoàn toàn chính xác theo thiết kế.
  2. Với lỗi 422: Nhận thấy Windows PowerShell tự ý cắt bỏ các dấu nháy kép `"` bên trong payload JSON khi truyền đối số cho file `.exe` bên ngoài, dẫn đến body gửi lên server bị mất định dạng JSON.
- **Cách khắc phục:**
  1. Với việc kiểm thử: Truy cập đúng các đường dẫn nghiệp vụ đã định nghĩa như `https://day12-agent-wjd3.onrender.com/health` (trả về 200 OK) và `/ready` (trả về 200 OK kèm `redis: true`).
  2. Với các lệnh kiểm thử trên Windows: Sử dụng `Invoke-RestMethod` của PowerShell hoặc bọc chặt payload JSON bằng dấu nháy đơn `'{"question":"..."}'` để đảm bảo chuỗi JSON không bị biến dạng khi gửi lên server.
