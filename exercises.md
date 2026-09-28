# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng mẫu bên dưới mỗi câu hỏi bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Hà Huy Nhất  Mã học viên: 2A202602401

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu em quên set `AGENT_API_KEY` trên Railway, app chết ngay ở lúc endpoint cần đọc settings thay vì âm thầm chạy với khóa mặc định. Việc này đã giúp em không gặp tình huống public URL đã mở nhưng ai cũng có thể dùng khóa `"changeme"` để gọi `/ask` và tiêu quota.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log em thu được có dạng: `{"event":"ask_completed","level":"info","timestamp":"2026-09-28T10:17:19+00:00","user_id":"cp5-check","tokens_in":3,"tokens_out":35,"cost_usd":0.00002145}`. Với log JSON như này thì em có thể lọc theo `event` hoặc `user_id` để điều tra một người dùng cụ thể, và có thể cộng dồn `cost_usd`/tokens để cảnh báo chi phí. `print("đã trả lời xong")` làm cho máy không  biết request của ai, tốn bao nhiêu tiền, hay xảy ra lúc nào theo cấu trúc chuẩn.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản                 | Dung lượng |
| -------------------- | ------------ |
| 1 stage (bản đầu) | 1.73 GB      |
| Multi-stage          | 331 MB       |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần chênh lệch chủ yếu đến từ base image `python:3.11` đầy đủ và các thứ nằm trong layer build/runtime của bản một stage. Bản multi-stage dùng `python:3.11-slim`, cài dependency trong builder rồi chỉ copy virtualenv và source cần thiết sang runtime, nên không mang theo nhiều file hệ thống và công cụ thừa. `.dockerignore` cũng giúp không copy `.venv`, `.git`, `.env` vào image.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi em sửa một ký tự trong `app/main.py`, các layer `FROM`, `WORKDIR`, `COPY requirements.txt`, `pip install`, tạo user và copy virtualenv từ builder vẫn dùng cache; layer phải chạy lại là `COPY app ./app`, các layer sau nó và export image. Nếu đặt `COPY . .` trước `RUN pip install`, chỉ cần sửa code là Docker mất cache từ layer copy toàn bộ source, và làm em phải cài lại toàn bộ dependency, build chậm hơn.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu app Python có lỗ hổng cho phép chạy lệnh trong container, kẻ tấn công sẽ chạy lệnh với quyền user của process. Nếu container chạy root, quyền đó là root bên trong container thì khi có mount nhạy cảm, Docker socket, hoặc lỗi escape/container runtime, rủi ro leo lên quyền cao trên host lớn hơn. Lệnh `USER appuser` cắt chuỗi này ở bước đầu: process app chỉ có quyền user thường, nên kể cả khai thác được app thì quyền trong container cũng bị giới hạn.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Có thể gửi tối đa 20 request trong 2 giây. Cách làm là gửi 10 request ở cuối một phút, rồi gửi tiếp 10 request ngay sau khi đồng hồ reset sang 10:01:00 hoặc 10:01:01. Bộ đếm theo phút sẽ xem đó là hai phút khác nhau, còn sliding window 60 giây sẽ nhìn lại 60 giây gần nhất và chặn nhóm thứ hai.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn tốc độ gọi, ví dụ 10 request/phút/user thì cost guard giới hạn tổng tiền đã tiêu trong tháng. Rate limit cho qua nhưng cost guard chặn: user gọi không nhanh, chỉ 1 request, nhưng trước đó đã tiêu gần hết ngân sách tháng nên request mới vượt budget. Ngược lại, cost guard cho qua nhưng rate limit chặn: user còn ngân sách rất nhiều nhưng gửi request thứ 11 trong cùng cửa sổ 60 giây.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu gộp `/health` và `/ready` rồi cho nó kiểm tra Redis, khi Redis mất kết nối 30 giây thì cả 3 container đều báo unhealthy. Orchestrator hiểu là process hỏng và restart cả 3 container thay vì chỉ rút traffic khỏi chúng. Trong lúc restart, không còn instance nào phục vụ request; khi Redis quay lại, hệ thống vẫn phải chờ các container khởi động lại.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis, các lần gọi cùng `X-User-Id` thấy `history_length` tăng ổn định: lượt đầu là 0, lượt sau thấy 2 message trước đó, rồi tăng tiếp theo từng cặp user/assistant. Nếu dùng dict Python trong RAM, khi scale 3 instance thì request có thể rơi vào container khác nhau thì `history_length` sẽ nhảy lung tung, có lúc quay về 0 vì container đó chưa từng thấy lịch sử của user.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi em gặp khi đã deploy và call url ready thì log trong phần deploy báo sai redis url và em lấy url đúng trong redis đã deploy và set lại biến là xong.
