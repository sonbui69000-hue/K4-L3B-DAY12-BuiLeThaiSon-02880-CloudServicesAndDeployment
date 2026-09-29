# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng trả lời mẫu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Bui Le Thai Son  Mã học viên: 02880.

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Trong lúc deploy, nếu quên đặt `AGENT_API_KEY` app sẽ dừng ngay với lỗi thiếu biến môi trường, giúp biết lỗi trước khi public service chạy. Nếu có khóa mặc định như `changeme`, service vẫn chạy và người khác có thể đoán được khóa để gọi API.
---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log thu được:

```text
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T04:46:31.222875+00:00", "user_id": "sv01", "tokens_in": 12, "tokens_out": 8, "cost_usd": 0.0001}
```

Từ dòng này có thể lọc user nào gọi nhiều và cộng chi phí theo user, hoăc đếm số token hoặc số lỗi theo thời gian. Một câu `print` bình thường không có format cố định nên khó lọc và thống kê tự động.
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
| 1 stage (bản đầu) | 1.25 GB |
| Multi-stage | 184 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Bản one-stage giữ base image đầy đủ và toàn bộ phần cài đặt trong cùng image. Bản multi-stage chỉ copy dependency và source cần chạy sang runtime, nên bỏ được phần thừa của stage build và dùng base slim.
---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Khi sửa một ký tự trong `app/main.py`, layer cài dependency ở builder vẫn được dùng lại vì `requirements.txt` không đổi. Các layer copy source ở runtime phải chạy lại và image được export lại. Nếu đặt `COPY . .` trước `pip install`, chỉ cần sửa source cũng làm layer đó đổi, nên Docker phải cài lại toàn bộ dependency và build chậm hơn.
---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Nếu app có lỗ hổng cho phép chạy lệnh, attacker có thể chạy lệnh với quyền của user trong container. Nếu container là root thì quyền đó rất cao và khi có thêm lỗi ở runtime hoặc Docker, nguy cơ ảnh hưởng tới host lớn hơn. `USER appuser` làm tiến trình chạy bằng user thường, nên lỗ hổng không tự động có quyền root trong container.
---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Với cách đếm theo phút đồng hồ, hạn mức 10 request/phút có thể nhận 20 request trong 2 giây. Người dùng gửi 10 request lúc 10:00:59, rồi gửi tiếp 10 request lúc 10:01:00 hoặc 10:01:01 khi bộ đếm đã reset. Sliding window nhìn lại đúng 60 giây nên không có khoảng hở này.
---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit giới hạn tốc độ hoặc số request, còn cost guard giới hạn tổng tiền của một user trong tháng. Rate limit có thể cho qua một request rất lớn về token, nhưng cost guard phải chặn nếu request đó làm vượt ngân sách. Ngược lại, user còn ngân sách nhưng gửi liên tục quá nhanh thì rate limit chặn trước, còn cost guard vẫn có thể cho qua.
---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Nếu Redis mất kết nối, endpoint gộp sẽ báo unhealthy cho cả ba container. Orchestrator có thể restart cả ba cùng lúc. Load balancer không còn instance nào để nhận request, dù lỗi Redis chỉ kéo dài 30 giây. Tách `/health` và `/ready` giúp `/health` vẫn thể hiện process còn sống, còn `/ready` trả 503 để load balancer tạm ngừng gửi traffic.
---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Với Redis, các request đi qua những instance khác nhau vẫn đọc được cùng history. Sau request đầu, `history_length` là 0; request kế tiếp thấy 2 message trước đó, rồi tiếp tục tăng theo từng cặp user và assistant. Nếu dùng dict Python, mỗi instance có dữ liệu riêng nên số này có thể quay lại 0 hoặc tăng không đều tùy request rơi vào container nào.
---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Railway Redis log báo `docker-entrypoint.sh: not found`, sau đó `/ready` của agent trả 503 với `redis: false`. Tìm ra nguyên nhân bằng cách xem service status và logs để xác định lỗi Redis. Sau khi recreate service Redis và gắn lại `REDIS_URL`, deploy lại thì `/ready` trả 200 với `{"status":"ready","redis":true}`.