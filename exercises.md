# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay từng dòng giữ chỗ bên dưới bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Bùi Hoàng Việt  Mã học viên: 2A202601392

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy lên Railway mà quên khai báo API_TOKEN, fail fast làm tiến trình dừng ngay và log chỉ rõ biến còn thiếu. Nếu dùng mặc định changeme, app vẫn khởi động nhưng ai đoán được token đều có thể gọi /chat và tiêu tốn ngân sách. Nhờ chết sớm, tôi phát hiện sai cấu hình trước khi service mở ra Internet.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log: {"event":"chat_completed","severity":"INFO","ts":"2026-08-10T10:30:00+00:00","client_id":"sv-test","usd_cost":0.00002265,"prompt_tokens":3,"completion_tokens":37}. Tôi có thể lọc, đếm sự kiện theo client_id và cộng usd_cost để theo dõi chi phí hoặc cảnh báo. Một câu print không cho biết client, thời điểm hay chi phí theo cấu trúc máy đọc được.

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
| 1 stage (bản đầu) | khoảng 1.800 MB |
| Multi-stage | khoảng 180 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Bản một stage giữ base image đầy đủ, công cụ build, cache và thành phần chỉ cần lúc cài thư viện. Bản multi-stage chỉ chép dependency đã cài sang python:3.11-slim; phần chênh lệch chủ yếu là compiler, công cụ build, pip cache và gói hệ điều hành không cần lúc chạy. Docker Desktop không hoạt động ở lần kiểm tra cuối nên đây là số đo ghi nhận từ lần build trước.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa app/main.py, Docker dùng lại layer base, WORKDIR, layer copy requirements.txt và layer cài dependency; chỉ các layer từ lúc copy source trở đi chạy lại. Nếu đặt COPY . . trước RUN pip install, mọi thay đổi source làm mất cache và kéo theo cài lại toàn bộ dependency dù requirements.txt không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu lỗ hổng cho phép thực thi lệnh, kẻ tấn công chạy lệnh trong container. Nếu tiến trình là root, kết hợp Docker socket, thư mục host được mount hoặc lỗ hổng runtime, quyền này có thể dẫn tới quyền cao trên host. USER appuser cắt chuỗi ở bước thực thi: mã bị khai thác chỉ có UID 10001 với quyền tối thiểu.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> WWW-Authenticate: Bearer cho client biết tài nguyên dùng cơ chế Bearer theo chuẩn HTTP. Thiếu header, sai scheme và sai token đều trả cùng 401 để không tiết lộ cho người dò quét rằng phần nào đã đúng. Client hợp lệ vẫn đủ thông tin để gửi lại header đúng.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> Client im lặng 10 phút vẫn chỉ gửi được 10 request liên tiếp trước khi bị 429 vì bucket chứa tối đa 10. Nếu bỏ min(capacity, ...), nó tích lũy khoảng 100 token refill, hoặc 110 nếu cộng cả 10 token ban đầu, tạo burst lớn và làm capacity mất ý nghĩa.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> Hạn mức 30 USD/tháng có thể làm mất tối đa 30 USD còn lại của tháng và client chỉ tự dùng lại khi sang kỳ tháng mới. Hạn mức 1 USD/ngày giới hạn thiệt hại ngày đó ở khoảng 1 USD và tự phục hồi khi sang ngày mới, nên thu nhỏ cả thiệt hại lẫn thời gian chờ.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Redis mất kết nối làm probe chung của cả ba container lỗi. Orchestrator restart cả ba; container mới vẫn không nối được Redis nên tiếp tục fail và rơi vào vòng lặp restart. Khi tách probe, /healthz vẫn 200 để không restart web process vô ích, còn /readyz trả 503 để ngừng nhận traffic. Redis trở lại thì readiness tự xanh.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lần deploy đầu, /healthz trả 200 nhưng /readyz trả 503. Tôi gọi riêng hai endpoint và xem log Railway nên biết lỗi nằm ở Redis, không phải Docker hay PORT. REDIS_URL chưa trỏ đúng Redis add-on; tôi tạo Redis service, đặt reference variable Redis.REDIS_URL rồi redeploy. Sau đó /readyz trả 200 và CP5 pass.
