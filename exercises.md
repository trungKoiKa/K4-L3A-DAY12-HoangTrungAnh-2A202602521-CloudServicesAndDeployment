# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng đánh dấu bên dưới bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Hoàng Trung Anh  Mã học viên: 2A202602521

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi thiếu AGENT_API_KEY, service dừng ngay lúc khởi động và Render đánh dấu deploy lỗi hoặc health check không đạt. Điều này giúp phát hiện cấu hình sai trước khi nhận traffic. Nếu đặt mặc định "changeme", deploy vẫn có thể xanh nhưng credential dùng chung đã bị biết trước và có thể bị dùng để gọi API trái phép.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log tôi quan sát được là: {"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T15:04:23.593346+00:00", "user_id": "sv01", "tokens_in": 12, "tokens_out": 34, "cost_usd": 2e-05}. Từ một dòng này, hệ thống log có thể lọc hoặc đếm theo event, user_id, level và cảnh báo theo cost_usd mà không cần đọc chuỗi tự do. Nó cũng có thể ghép các request theo timestamp và truy vấn bằng công cụ log tập trung; print() chỉ tạo một câu chữ khó phân tích ổn định.

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
| 1 stage (bản đầu) | ... MB |
| Multi-stage | ... MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Bản một stage ban đầu dùng python:3.11 đầy đủ có kích thước khoảng 1,1 GB, còn image multi-stage hiện tại tôi đo bằng docker images là 271 MB (day12-agent:prod). Chênh lệch đến từ việc image một stage giữ cả base image đầy đủ, cache hoặc build artefacts và toàn bộ dependency trong cùng layer; bản multi-stage chỉ chép runtime dependencies sang python:3.11-slim nên không mang theo phần thừa.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Sau khi sửa một ký tự trong app/main.py, các layer base image, WORKDIR, COPY requirements.txt và pip install được dùng lại từ cache; layer COPY app hoặc COPY utils và các layer sau nó phải chạy lại. Nếu đặt COPY . . trước pip install, mọi thay đổi source sẽ làm mất cache của layer đó và buộc Docker cài lại toàn bộ dependency, khiến build chậm hơn.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu app chạy bằng root, một lỗ hổng cho phép kẻ tấn công thực thi lệnh trong container; tiến trình đó có quyền đọc hoặc ghi các file và thiết bị mà root trong container được phép dùng. Nếu khai thác tiếp được runtime hoặc container boundary, quyền root có thể trở thành quyền rất cao trên host. USER appuser chạy tiến trình bằng UID thường, nên cắt chuỗi leo thang quyền ngay từ lớp container.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Với giới hạn 10 request/phút nhưng đếm theo phút đồng hồ, người dùng có thể gửi 10 request vào giây 59 của phút hiện tại và thêm 10 request vào giây 00 của phút kế tiếp. Như vậy có 20 request trong khoảng 2 giây mà cả hai nhóm đều được tính ở hai bucket khác nhau.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn số lần gọi trong một cửa sổ thời gian; cost guard giới hạn tổng chi phí theo tháng. Ví dụ user mới gửi 9 request nhỏ nên rate limit còn chỗ, nhưng spent cộng estimated_cost vượt ngân sách tháng thì cost guard phải trả 402. Ngược lại, user còn ngân sách nhưng gửi request thứ 11 trong 60 giây thì rate limit trả 429; hai cơ chế bảo vệ hai tài nguyên khác nhau.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu /health cũng kiểm tra Redis, khi Redis mất 30 giây cả ba container sẽ lần lượt bị đánh dấu unhealthy; load balancer có thể loại cả cụm và health check liên tục khiến chúng restart, tạo outage lan rộng dù tiến trình app vẫn sống. Tách endpoint giúp /health chỉ chứng minh process còn chạy, còn /ready trả 503 khi Redis lỗi để ngừng nhận traffic mới nhưng giữ container sống chờ Redis hồi phục.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis dùng chung, ba instance đều đọc hoặc ghi cùng history nên history_length tăng tuần tự theo cùng một X-User-Id, bất kể request được route vào instance nào. Nếu dùng dict Python trong từng process, mỗi instance có một bản history riêng; khi load balancer chuyển request, history_length sẽ bị reset hoặc nhảy theo instance và dữ liệu không nhất quán.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi thực tế của tôi là gọi /ask trên Render bị 401 invalid or missing API key dù /health và /ready vẫn trả 200. Tôi kiểm tra response và đối chiếu độ dài hoặc giá trị của key qua Render CLI mà không in secret; phát hiện DEPLOY_API_KEY cục bộ khác AGENT_API_KEY đang chạy trên service. Sau khi đồng bộ đúng key, redeploy Render và chạy lại tests/test_cp5.py, test auth /ask trả 200 và CP5 đạt 9/9 test.
