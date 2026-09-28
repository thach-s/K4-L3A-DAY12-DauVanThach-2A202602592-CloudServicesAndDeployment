# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay từng dòng trả lời mẫu bên dưới bằng câu trả lời thực tế.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đậu Văn Thạch  Mã học viên: 2A202602592

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Tình huống cụ thể là lúc deploy service công khai lên Railway. Nếu tôi quên đặt `AGENT_API_KEY`, cấu hình bắt buộc làm service báo lỗi ngay trong quá trình kiểm tra/deploy, khi tôi còn đang theo dõi log và có thể sửa trước khi mở traffic. Nếu code dùng mặc định `"changeme"`, service vẫn chạy và người ngoài có thể đoán khóa công khai đó để gọi `/ask`, tiêu rate quota và ngân sách của tôi. Fail fast biến lỗi cấu hình thành lỗi triển khai rõ ràng thay vì một sự cố bảo mật và chi phí âm thầm.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log thật trên Railway là: `{"user_id":"cp5-rate-test","tokens_in":302,"tokens_out":43,"message":"","level":"info","cost_usd":0.0000711,"timestamp":"2026-09-28T08:12:44.950596+00:00","event":"ask_completed"}`. Từ các field có cấu trúc này tôi có thể (1) lọc theo `user_id`, cộng `cost_usd` và token để biết user nào dùng nhiều tài nguyên nhất; (2) nhóm theo `event`, `level` và `timestamp` để dựng biểu đồ, tính tỷ lệ lỗi hoặc đặt cảnh báo theo thời gian. Chuỗi `print("đã trả lời xong")` không chứa các chiều dữ liệu đó nên máy không thể tổng hợp đáng tin cậy.

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
| 1 stage (bản đầu) | Build bị gián đoạn; riêng base nén khoảng 409 MB |
| Multi-stage | 310 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Image multi-stage đo thực tế bằng `docker images` là 310 MB và image inspect cho biết runtime chạy bằng user `agent`. Lần build một-stage trên máy bị gián đoạn do Docker Hub tải quá chậm; log build cho thấy riêng các layer nén của base `python:3.11` đã khoảng 409 MB trước khi cài dependency, nên tôi không ghi một số final giả. Bản một-stage lớn hơn vì mang toàn bộ base Python đầy đủ, công cụ/OS package và mọi thứ dùng lúc build sang runtime. Bản multi-stage dùng `python:3.11-slim` và chỉ copy virtualenv cùng `app/`, `utils/`, nên không mang phần builder và file không cần thiết vào image cuối.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Với Dockerfile hiện tại, thay đổi một ký tự trong `app/main.py` không làm đổi `requirements.txt`, nên toàn bộ stage builder, gồm tạo virtualenv và `pip install`, được lấy lại từ cache. Các layer runtime trước `COPY app ./app`, như base slim, tạo user và copy `/opt/venv`, cũng tái sử dụng được; từ `COPY app ./app` trở đi Docker phải tạo lại layer vì source đã đổi. Nếu đặt `COPY . .` trước `RUN pip install`, mọi thay đổi source sẽ làm layer COPY đổi và kéo theo việc cài lại toàn bộ dependency, khiến build chậm và tải mạng không cần thiết.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu code Python có lỗ hổng thực thi lệnh, kẻ tấn công trước hết chạy được lệnh với quyền của process trong container. Khi process là root, họ có thể sửa file hệ thống của container, cài công cụ, đọc secret dễ hơn và lợi dụng thêm cấu hình runtime sai, volume/socket nhạy cảm hoặc lỗ hổng kernel để tăng tác động tới host. Root trong container không tự động đồng nghĩa root trên host, nhưng làm hậu quả của bước tiếp theo lớn hơn nhiều. Lệnh `USER agent` cắt chuỗi ở lớp quyền trong container: mã bị khai thác chỉ có UID thường, không được phép sửa các vùng chỉ root truy cập; đây là giảm quyền và giảm thiệt hại, không thay thế cho vá lỗ hổng hay cô lập runtime.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Cách đếm theo phút đồng hồ cho phép tối đa 20 request trong khoảng hai giây: gửi 10 request ở cuối phút, ví dụ 10:00:59, rồi ngay sau khi bộ đếm reset ở giây 00 gửi tiếp 10 request lúc 10:01:00. Mỗi phút riêng vẫn chỉ có 10 request nhưng tải dồn thực tế là 20 request gần như liên tiếp. Sliding window nhìn lại đúng 60 giây gần nhất nên đợt thứ hai sẽ bị chặn.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn tần suất trong cửa sổ ngắn, ở bài này là 10 request trong 60 giây; cost guard giới hạn tổng tiền theo từng user trong cả tháng, ở bài này là 10 USD. Trường hợp rate limit cho qua nhưng cost guard chặn: user chỉ gửi một request trong phút nhưng đã tiêu gần hết 10 USD hoặc request mới làm tổng dự kiến vượt ngân sách. Trường hợp ngược lại: user còn gần nguyên ngân sách và gửi các request rất rẻ, nhưng request thứ 11 trong cùng 60 giây vẫn bị rate limiter trả 429. Kết quả chạy thật của tôi là 10 mã 200 rồi 5 mã 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Trình tự sẽ là: Redis mất kết nối; cả ba container gọi endpoint gộp và cùng trả 503; orchestrator hiểu nhầm process đã chết, loại cả ba khỏi nhận traffic rồi restart chúng; request đang xử lý bị gián đoạn và cụm tạm thời không còn instance phục vụ; các container cùng khởi động lại nhưng Redis vẫn chưa sẵn sàng nên tiếp tục fail/restart. Khi Redis trở lại, hệ thống còn phải chờ các container boot lại. Tách endpoint tránh vòng lặp này: `/health` vẫn 200 để không restart process khỏe, còn `/ready` trả 503 để load balancer chỉ tạm ngừng gửi traffic.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Khi kiểm tra với Redis dùng chung, request đầu tiên trả `history_length=0`, request tiếp theo của cùng user thấy hai message trước và trả `history_length=2`. Dashboard Railway cũng cho thấy key `history:cp5-auth-test` có 2 phần tử và `history:cp5-rate-test` được cắt ở 20 phần tử. Nếu dùng dict Python, mỗi container giữ một bản riêng nên khi request bị phân phối qua ba instance, số có thể lặp hoặc nhảy kiểu 0, 0, 2, 0, 2, 4 thay vì tăng nhất quán; restart container còn làm lịch sử của instance đó về 0.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi thực tế tôi gặp là Railway CLI báo `Unauthorized. Please login with railway login` dù tôi đã đăng nhập trên website. Tôi chạy `railway whoami` để xác nhận CLI vẫn chưa có phiên hợp lệ và nhận ra mã device-code đầu tiên chưa được authorize hoàn tất. Tôi hủy phiên đang chờ, tạo mã browserless mới, bấm Authorize, rồi kiểm tra lại thấy `Logged in as Đậu Văn Thạch`. Sau đó deployment chuyển sang `SUCCESS`; URL công khai trả `/health` 200, `/ready` 200 và request không có key trả 401.
