# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đỗ Tú Anh  Mã học viên: 2A202601272

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu để mặc định "changeme", app sẽ vẫn khởi động trơn tru. Hệ lụy là bất kỳ ai biết token mặc định này đều có thể truy cập hợp lệ vào endpoint `/chat`, thoải mái gọi LLM và "đốt" sạch ngân sách OpenAI của bạn. Việc "chết sớm" (Fail fast) giúp bạn ngay lập tức phát hiện ra cấu hình bị thiếu lúc vừa bật server, ngăn chặn sự cố lộ API ra ngoài public trước khi nó kịp xảy ra.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log: `{"timestamp": "2026-08-10T08:00:00Z", "level": "INFO", "event": "chat_completed", "client_id": "sv-test", "usd_cost": 0.0015}`
1. Các hệ thống như Datadog/ELK có thể dễ dàng parse tự động (không cần regex) để vẽ biểu đồ tổng chi phí theo từng `client_id` theo thời gian thực.
2. Dễ dàng filter (lọc) tất cả các request có `usd_cost > 0.005` hoặc đếm số lượng tin nhắn của một user cụ thể bằng các công cụ truy vấn log chuẩn, điều mà dòng chữ "đã trả lời xong" hoàn toàn vô dụng.

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
| 1 stage (bản đầu) | ~1.0 GB |
| Multi-stage | ~150 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần chênh lệch khổng lồ (~850MB) chính là các công cụ build C/C++ (gcc, make), bộ nhớ đệm (cache) của thư viện apt/pip, cũng như mã nguồn thừa của các thư viện trung gian trong quá trình `pip install`. Stage 2 chỉ sao chép kết quả đã biên dịch xong từ Stage 1 sang một hệ điều hành trống (slim), vứt bỏ toàn bộ "rác" của quá trình build.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Khi sửa mã nguồn, các layer cài đặt thư viện (`COPY requirements.txt` và `RUN pip install`) được Docker dùng lại từ cache vì file `requirements.txt` không đổi. Chỉ layer `COPY app/ app/` và phần sau đó mới bị build lại, tốn 1-2 giây. 
Nếu đưa `COPY . .` lên trước `RUN pip install`, mọi thay đổi dù nhỏ nhất ở bất kỳ file code nào (như `main.py`) đều sẽ vô hiệu hóa cache ở dòng COPY, khiến Docker phải tải lại và cài đặt lại toàn bộ thư viện `pip` từ đầu (tốn thời gian rất lâu).

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Kẻ tấn công khai thác lỗ hổng (ví dụ RCE) trong code Python để thực thi các lệnh Linux. Do app đang chạy dưới quyền root, kẻ tấn công cũng trở thành root bên trong container. Mặc dù bị cô lập trong container, nhưng với quyền root, hắn có thể khai thác tiếp các lỗ hổng của nhân hệ điều hành Linux để phá vỡ lớp cách ly (container breakout) và chiếm quyền máy host. Lệnh `USER appuser` biến process thành user thường, nên dù kẻ tấn công có khai thác được code Python, hắn cũng không có quyền cài đặt mã độc hay thay đổi cấu hình mạng, cắt đứt hoàn toàn chuỗi leo thang.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

`WWW-Authenticate` là quy định bắt buộc của HTTP để trình duyệt hoặc client tự động nhận biết cơ chế xác thực (Bearer) và bật prompt yêu cầu nhập token/password. 
Ta trả cùng một thông báo lỗi để phòng chống kiểu tấn công thăm dò (Enumeration Attack). Nếu bạn báo "Sai token", kẻ gian sẽ biết cú pháp Header của chúng đã chuẩn (chỉ cần brute-force token). Nếu bạn báo "Sai Header", kẻ gian sẽ thử các chuẩn khác. Gom chung lại làm mù thông tin của kẻ tấn công.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

Nó gửi được đúng 10 request (toàn bộ sức chứa của bucket) trước khi bị 429. 
Nếu bỏ `min(capacity, ...)`, số token sẽ được cộng dồn theo thời gian thành 10 phút * 10 = 100 token. Khi đó client có thể "bắn" liên hoàn 100 request trong 1 giây, phá hỏng hoàn toàn mục tiêu chống quá tải (burst) của Token Bucket.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

Với hạn mức $30/tháng, thiệt hại tối đa ngay trong đêm đó là 30 đô la, và user này sẽ hoàn toàn bị chặn suốt phần còn lại của tháng trừ khi quản trị viên can thiệp thủ công.
Với hạn mức $1/ngày, thiệt hại tối đa trong đêm đó bị giới hạn chỉ là 1 đô la. Tới rạng sáng ngày hôm sau (qua mốc 0h), ngân sách mới sẽ được tái cấp phát và dịch vụ tự động hồi phục lại trạng thái bình thường.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

1. Redis mất kết nối.
2. Load Balancer gọi endpoint gộp (kiểm tra Redis) và nhận về 503 lỗi.
3. Load Balancer coi container này là đã chết (Liveness probe failed) nên lập tức tắt và khởi động lại container (Restart).
4. Container mới vừa khởi động xong, Redis vẫn chết, lại báo 503, lại bị Restart.
5. Cả 3 container rơi vào CrashLoop, làm gián đoạn toàn bộ request, kể cả các tính năng không liên quan tới Redis. Việc tách rời Liveness và Readiness đảm bảo App vẫn sống khi Redis tạm đứt.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Lỗi: "Deploy thất bại do Health check timeout trên Render". 
Nguyên nhân: Khi đọc log deploy trên Render, tôi thấy app đang lắng nghe ở cổng mặc định `8000`, trong khi Render tự động cấp phát một cổng ngẫu nhiên thông qua biến môi trường `$PORT` và yêu cầu ping thử vào đó. Do không khớp cổng, Render không nhận được phản hồi và giết process.
Cách sửa: Thay đổi dòng CMD cuối Dockerfile để ưu tiên đọc biến `$PORT`: `CMD ["sh", "-c", "exec uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]`.
