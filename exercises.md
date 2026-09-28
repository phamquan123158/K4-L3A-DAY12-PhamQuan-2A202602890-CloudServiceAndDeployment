# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay phần trả lời mẫu bên dưới mỗi câu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Phạm Quân  Mã học viên: 2A202602890

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu deploy thiếu `AGENT_API_KEY`, cấu hình bắt buộc khiến app dừng ngay thay vì âm thầm chạy với khóa mặc định dễ đoán. Ví dụ một service mới vừa được public nhưng người vận hành quên nhập secret: fail fast làm deployment báo lỗi để sửa trước khi nhận traffic; nếu dùng `changeme`, người ngoài có thể đoán khóa và gọi `/ask`, làm lộ quyền truy cập hoặc phát sinh chi phí.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> ```json
> {"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T10:41:13.509246+00:00", "user_id": "sv-test", "tokens_in": 3, "tokens_out": 37, "cost_usd": 2.265e-05}
> ```
> Có thể lọc các lượt theo `user_id` và cộng `cost_usd` để theo dõi chi phí; cũng có thể thống kê token và thời điểm để tìm xu hướng sử dụng. Một câu `print("đã trả lời xong")` không có trường dữ liệu ổn định để máy lọc hay tổng hợp.

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
| 1 stage (image cũ trước khi đổi Dockerfile) | 1.26 GB |
| Multi-stage (`day12-agent:cp2-test`) | 184 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Chênh lệch chủ yếu do image 1 stage dùng base Python đầy đủ và giữ chung môi trường cài đặt với mọi thứ phục vụ build. Bản multi-stage dùng `python:3.11-slim`; dependency được cài ở stage `builder`, còn stage runtime chỉ nhận các package cần chạy và source `app`/`utils`, không mang nguyên stage builder vào image cuối. Số 1.26 GB là phép đo của image cũ trong lần baseline; 184 MB là image CP2 hiện tại.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> `requirements.txt` được copy và cài dependency trong stage `builder` trước khi copy source. Khi chỉ sửa `app/main.py`, Docker có thể dùng lại layer cài dependency; layer copy source/runtime phía sau phải chạy lại. Nếu `COPY . .` nằm trước `RUN pip install`, thay đổi một file bất kỳ cũng làm layer copy đổi và khiến bước cài dependency bị chạy lại, làm build chậm hơn.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu có lỗ hổng cho phép thực thi lệnh trong Python, kẻ tấn công trước hết có quyền của process trong container. Chạy root khiến họ có toàn quyền root bên trong container; nếu container còn quyền/mount nguy hiểm như Docker socket, privileged mode hoặc có lỗi thoát container, rủi ro có thể lan tới host. `USER app` chạy process bằng tài khoản thường, giảm quyền ngay tại bước thực thi và giới hạn tác động; nó không thay thế các biện pháp cô lập khác.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Có thể gửi 20 request trong khoảng hai giây qua ranh giới phút: 10 request ở giây cuối của phút hiện tại, rồi 10 request ngay đầu phút tiếp theo. Bộ đếm theo phút lịch reset giữa hai nhóm nên cho qua cả hai; sliding window vẫn thấy đủ 20 request nằm trong 60 giây và chặn sau khi quota 10 đã dùng hết.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn số request trong 60 giây; cost guard giới hạn tổng USD của từng user trong tháng. Một user có thể gửi ít request nhưng mỗi request rất tốn token, khiến rate limit cho qua còn cost guard chặn vì hết ngân sách. Ngược lại, nhiều request rất rẻ có thể vượt số lần/phút dù tổng chi phí tháng vẫn thấp; khi đó rate limit chặn trước.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Redis mất kết nối: probe gộp kiểm tra Redis bắt đầu trả lỗi cho cả ba container. Orchestrator hiểu liveness thất bại là process cần restart, nên lần lượt hoặc đồng thời restart cả ba dù ứng dụng vẫn chạy được; chúng lại thất bại khi Redis chưa hồi phục, gây vòng lặp restart và làm cụm mất phục vụ. Tách probe giúp `/health` tiếp tục báo process còn sống, trong khi `/ready` trả lỗi để load balancer ngừng gửi traffic cho instance chưa sẵn sàng mà không restart nhầm.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis dùng chung, mỗi lượt `/ask` thêm hai message nên `history_length` lần lượt là `0`, `2`, `4`, ... bất kể request tới container nào. Nếu mỗi process giữ một dict riêng, container khác không thấy phần lịch sử vừa ghi; khi load balancer phân phối request, các giá trị sẽ không tăng đều (ví dụ ba container mới lần lượt cùng có thể trả `0`, rồi mỗi container chỉ thấy những request đã vào chính nó). Đây là lý do state hội thoại phải ở Redis thay vì RAM của từng container.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi quan sát được là `GET /ready` trên domain Railway trả HTTP 500, trong khi `/health` trả 200 và `/ask` không có key trả 401. Mình khoanh vùng bằng cách gọi riêng ba endpoint và chạy `tests/test_cp5.py`; test chỉ còn fail ở readiness. Railway reference mình cung cấp là `${{redis.REDIS_URL}}`, nhưng mình chưa truy cập được runtime logs/dashboard để xác minh biến đã được áp dụng và tìm nguyên nhân chi tiết, nên chưa thể nói lỗi đã sửa. Bước tiếp theo là kiểm tra deployment logs và service Redis/biến reference trong Railway rồi redeploy, sau đó xác nhận `/ready` trả 200.
