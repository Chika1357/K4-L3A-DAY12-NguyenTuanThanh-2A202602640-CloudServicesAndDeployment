# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Tuấn Thành  Mã học viên: 2A202602640

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy lên Railway, nếu quên set biến AGENT_API_KEY trong dashboard thì app sẽ báo ValidationError ngay lúc khởi động và container không start được — mình thấy lỗi ngay trên log và sửa được luôn. Nếu để mặc định "changeme" thì app vẫn chạy bình thường, ai cũng có thể gọi API bằng khóa "changeme" mà mình không biết, đến khi nhận hóa đơn LLM thì đã muộn.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log thu được: `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T10:15:30+00:00", "user_id": "sv01", "tokens_in": 12, "tokens_out": 58, "cost_usd": 0.0001}`
>
> Hai việc làm được: (1) Lọc tất cả log theo user_id cụ thể để xem ai đang tiêu nhiều tiền nhất hôm nay — chỉ cần query `jq 'select(.user_id == "sv01")'`, còn print thì không có cấu trúc nên không lọc được. (2) Tính tổng cost_usd trong 5 phút qua để đặt cảnh báo tự động khi chi phí vượt ngưỡng — Datadog/CloudWatch đọc JSON field trực tiếp, còn chuỗi print thì phải parse bằng regex rất dễ vỡ.

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
| 1 stage (bản đầu) | ~1.05 GB |
| Multi-stage | ~195 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần chênh lệch khoảng 850MB chủ yếu là compiler (gcc, build-essential), header files của C, cache pip, và các package phát triển mà stage builder cài để biên dịch các thư viện có phần native (như uvloop, pydantic-core). Với multi-stage, những thứ đó chỉ tồn tại trong stage builder rồi bị bỏ đi — stage runtime chỉ copy kết quả đã biên dịch xong qua, nên image nhẹ hơn rất nhiều. Ngoài ra base image python:3.11 đầy đủ cũng nặng hơn python:3.11-slim khoảng 600MB vì chứa thêm các công cụ hệ thống.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Với Dockerfile hiện tại, khi sửa 1 ký tự trong main.py thì các layer `COPY requirements.txt` và `RUN pip install` được dùng lại từ cache (vì requirements.txt không đổi), chỉ layer `COPY app ./app` trở đi phải chạy lại — build rất nhanh, khoảng 2-3 giây. Nếu đặt `COPY . .` trước `RUN pip install`, thì bất kỳ thay đổi nào trong source code cũng làm cache của `COPY . .` bị hủy, kéo theo `pip install` phải chạy lại từ đầu — tốn thêm 30-60 giây mỗi lần build dù chỉ sửa 1 dấu phẩy.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện: (1) Code Python có lỗ hổng injection cho phép kẻ tấn công thực thi lệnh tùy ý bên trong container. (2) Vì container chạy bằng root, lệnh đó có quyền root trong container — đọc được mọi file, cài thêm tool. (3) Kẻ tấn công lợi dụng quyền root để escape khỏi container (qua lỗi kernel hoặc mount volume nhạy cảm), lúc này họ trở thành root trên máy host. Lệnh `USER appuser` cắt đứt chuỗi ở bước 2: dù kẻ tấn công thực thi được lệnh, họ chỉ là user thường (UID 10001), không đọc được file nhạy cảm, không cài được tool, và không thể khai thác các kỹ thuật escape cần quyền root.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> 20 request trong 2 giây. Cách đạt được: gửi 10 request lúc 10:00:59 (cuối phút 10:00), rồi ngay lúc 10:01:01 (đầu phút 10:01) gửi thêm 10 request. Vì đếm theo phút đồng hồ, bộ đếm reset ở giây 00, nên cả 2 nhóm đều "hợp lệ" — mỗi nhóm nằm trong phút riêng. Nhưng thực tế 20 request chỉ cách nhau 2 giây. Sliding window không có lỗ hổng này vì nó luôn nhìn 60 giây gần nhất tính từ thời điểm hiện tại.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn số lượng request trên đơn vị thời gian, cost guard giới hạn tổng số tiền đã tiêu trong tháng. (1) Rate limit cho qua nhưng cost guard chặn: user gửi đều đặn 5 request/phút (dưới hạn mức 10), nhưng mỗi request dùng prompt rất dài tốn 50.000 token — sau vài giờ tổng chi phí vượt $10 ngân sách tháng. (2) Rate limit chặn nhưng cost guard cho qua: user mới tạo tài khoản, chưa tiêu đồng nào (spent = $0), nhưng viết script gọi API 20 lần trong 1 giây — rate limit chặn ở request thứ 11 dù ngân sách vẫn còn dư.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> (1) Redis mất kết nối. (2) Cả 3 container gọi Redis trong health check đều thất bại, trả về lỗi. (3) Orchestrator thấy cả 3 container unhealthy, bắt đầu restart từng container. (4) Trong lúc restart, không còn container nào phục vụ request — toàn bộ service sập. (5) Redis quay lại sau 30 giây, nhưng cả 3 container đang restart nên chưa sẵn sàng. (6) User thấy lỗi 502/503 kéo dài hàng phút dù Redis chỉ mất 30 giây. Nếu tách riêng: /health không kiểm tra Redis nên container không bị restart, chỉ /ready báo 503 khiến load balancer tạm ngừng gửi request — Redis quay lại là service phục vụ lại ngay.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis, history_length tăng đều: 0, 2, 4, 6, 8... vì cả 3 container cùng đọc/ghi vào một Redis chung. Nếu lưu trong dict Python, history_length sẽ nhảy không đều, ví dụ: 0, 0, 2, 0, 2, 4... Lý do: request bị load balancer phân phối ngẫu nhiên vào 3 container, mỗi container có dict riêng không chia sẻ với nhau. Container A có 4 message, nhưng request tiếp theo vào container B thì thấy history_length = 0 — agent "mất trí nhớ" ngẫu nhiên.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi gặp phải: sau khi deploy lên Railway, health check liên tục timeout và container bị restart. Thông báo lỗi trên dashboard: "Health check failed: connection refused". Tìm nguyên nhân bằng cách đọc log trên Railway — thấy uvicorn đang bind 127.0.0.1:8000 thay vì 0.0.0.0. Nguyên nhân là ban đầu CMD trong Dockerfile cố định `--host 127.0.0.1` — trong container thì 127.0.0.1 là loopback, bên ngoài không gọi vào được. Sửa bằng cách đổi thành `--host 0.0.0.0` và dùng `--port ${PORT:-8000}` để đọc cổng từ biến môi trường mà Railway tự gán.
