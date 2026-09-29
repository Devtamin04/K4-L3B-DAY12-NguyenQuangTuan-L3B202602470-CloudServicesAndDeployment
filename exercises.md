# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder dưới mỗi câu hỏi bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: ..........................  Mã học viên: ..........................

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Ví dụ khi deploy lên Railway mà quên set `AGENT_API_KEY`: nếu mặc định là `"changeme"` thì app vẫn lên bình thường, health check xanh, nhưng ai biết khóa mặc định (nó nằm ngay trong code public) đều gọi được `/ask` và đốt tiền LLM của mình. Vì không có mặc định, app chết ngay lúc khởi động với lỗi rõ ràng "thiếu AGENT_API_KEY", deploy bị đánh dấu fail và mình sửa ngay trước khi có người dùng thật, thay vì phát hiện sau khi bị lạm dụng.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log thật lấy từ `railway logs`:
>
> ```
> [INFO]  event="ask_completed" timestamp="2026-09-29T08:56:48.380013+00:00" user_id="sv-rl" tokens_in=392 tokens_out=43 cost_usd=0.0000846
> ```
>
> Hai việc làm được mà `print("đã trả lời xong")` không làm được: (1) lọc/tìm theo trường, ví dụ chỉ lấy các dòng `user_id="sv-rl"` hoặc `event="ask_completed"`; (2) tổng hợp số liệu bằng máy, ví dụ cộng `cost_usd` theo user hoặc vẽ biểu đồ token theo thời gian, và đặt cảnh báo khi giá trị vượt ngưỡng. Dòng print chỉ là chuỗi tự do, không có trường để máy đọc.

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

> | Bản | Dung lượng |
> |-----|-----------|
> | 1 stage (bản đầu, `python:3.11` đầy đủ) | 1.73 GB |
> | Multi-stage (`python:3.11-slim`) | 310 MB |
>
> Phần chênh (khoảng 1.4 GB) chủ yếu là base image đầy đủ của `python:3.11` chứa sẵn trình biên dịch, header, công cụ build (gcc, make, git...) mà lúc chạy không cần. Bản multi-stage dùng base slim và chỉ copy virtualenv đã cài cùng source sang stage runtime, nên không mang theo các công cụ build và cache của pip.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Dockerfile của mình copy `requirements.txt` rồi `RUN pip install` trước, sau đó mới `COPY app/` và `COPY utils/`. Khi sửa một ký tự trong `app/main.py`, các layer base, tạo user, `WORKDIR`, `COPY requirements.txt`, `pip install` và copy venv được dùng lại từ cache. Chỉ layer `COPY app/` và các layer sau nó chạy lại, nên build chỉ mất vài giây. Nếu đặt `COPY . .` lên trước `pip install` thì sửa một dòng code làm đổi layer đó, mọi layer phía sau (kể cả `pip install`) bị vô hiệu hóa và phải tải lại toàn bộ thư viện mỗi lần build.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện: code Python có lỗ hổng (ví dụ command injection hoặc deserialize dữ liệu không tin cậy) khiến kẻ tấn công chạy được lệnh trong container. Nếu process chạy bằng root thì họ có quyền root trong container: cài công cụ, đọc mọi file, và nếu container có mount volume, cấu hình dư quyền hoặc dính lỗ hổng runtime thì có thể thoát ra ngoài và có quyền cao trên máy host. Lệnh `USER app` cắt chuỗi ở bước đầu: sau khi bị chiếm, kẻ tấn công chỉ có quyền của user `app` (không shell, không home, không ghi được vào hệ thống), nên rất khó cài thêm gì hoặc leo thang lên host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Với đếm theo phút đồng hồ và hạn mức 10/phút, một người dùng có thể gửi tối đa 20 request trong 2 giây: 10 request vào giây 59 của phút này (chạm hạn mức) rồi 10 request vào giây 00 của phút kế tiếp (bộ đếm vừa reset). Sliding window 60 giây tính các request trong 60 giây gần nhất tại mọi thời điểm, nên trong bất kỳ khoảng 60 giây nào cũng không vượt 10, kể cả tại ranh giới phút.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn tốc độ (số request trên mỗi khoảng thời gian, trả 429) để chống spam, còn cost guard giới hạn tổng chi phí tích lũy theo user mỗi tháng (trả 402). Rate limit cho qua nhưng cost guard chặn: một user gửi đều đặn 5 request/phút (dưới hạn mức) nhưng gửi câu hỏi rất dài, cả tháng tích lũy vượt ngân sách 10 USD thì bị chặn. Ngược lại: một user mới, chi phí gần 0, gửi 15 request trong vài giây thì rate limit chặn (429) dù ngân sách còn nguyên. Khi chạy thật trên Railway mình thấy 10 lần đầu trả 200, 5 lần sau trả 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu gộp làm một và cho kiểm tra Redis, khi Redis mất kết nối 30 giây thì: (1) endpoint gộp bắt đầu trả lỗi ở cả 3 container; (2) platform coi liveness fail nên khởi động lại (restart) từng container; (3) container vừa restart vẫn không kết nối được Redis nên tiếp tục fail và bị restart lặp lại; (4) cả cụm cùng bị restart hoặc bị loại khỏi load balancer, service down hoàn toàn dù code vẫn ổn; (5) khi Redis quay lại, các container còn phải khởi động lại nên phục hồi chậm hơn. Tách riêng thì `/health` vẫn 200 nên không bị restart oan, còn `/ready` trả 503 để chỉ tạm ngừng nhận traffic đến khi Redis về. Mình đã thấy đúng hành vi này: khi `REDIS_URL` trỏ sai, `/health` vẫn 200 còn `/ready` trả 503 `{"redis": false}`.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Nếu lịch sử lưu trong dict Python của từng process thì mỗi instance có bộ nhớ riêng. Với `--scale agent=3` và load balancer chia request luân phiên, cùng một `X-User-Id` sẽ có `history_length` nhảy lung tung (mỗi instance đếm riêng, ví dụ 2, 2, 2 rồi 4, 4, 4 thay vì tăng dần đều), và mất sạch khi container restart. Khi lưu ở Redis, mọi instance đọc chung một danh sách nên `history_length` tăng đơn điệu theo số lượt bất kể instance nào xử lý. Trên Railway mình cũng thấy `history_length` của user `sv-test` tiếp tục tăng qua các lần gọi và qua lần redeploy.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi: sau `railway add --database redis` rồi `railway up`, deploy báo "Deploy crashed" và log lặp lại `/bin/sh: 1: exec: docker-entrypoint.sh: not found`. Mình tìm ra nguyên nhân bằng `railway logs` và `railway status`: CLI đã tự link vào service Redis sau khi thêm database, nên `railway up` đã đẩy image Python của mình lên chính service Redis. Service này vẫn giữ lệnh khởi động `docker-entrypoint.sh` của Redis, mà image Python không có file đó. Cách sửa: tạo service riêng `railway add --service app`, deploy bằng `railway up --service app`, dùng một service Redis nguyên bản, và set `REDIS_URL` bằng biến tham chiếu `${{<tên service Redis>.REDIS_URL}}`. Sau đó còn gặp thêm `/ready` trả 503 vì biến trỏ nhầm vào service Redis hỏng, đổi sang service Redis mới thì `/ready` trả 200.
