# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: ..........................  Mã học viên: ..........................

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi tôi bỏ `API_TOKEN` và khởi tạo `Settings`, chương trình dừng ngay với
> `ValidationError` thay vì tiếp tục chạy. Tình huống cụ thể là lúc deploy tôi
> quên khai báo secret trên dashboard: service sẽ fail ở bước khởi động nên tôi
> phát hiện ngay qua log/health check. Nếu có mặc định `"changeme"`, service vẫn
> lên bình thường và người biết hoặc đoán được token mẫu có thể gọi `/chat`, dùng
> hết rate limit và ngân sách LLM trước khi tôi nhận ra cấu hình bị thiếu.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng tôi thu được khi chạy test gọi hàm log là:
> `{"event": "chat_completed", "severity": "INFO", "ts": "2026-08-11T00:41:21.227806+00:00", "client_id": "sv01", "usd_cost": 0.12}`.
> Từ dòng này tôi có thể (1) lọc hoặc đếm riêng sự kiện `chat_completed` theo
> `client_id` và khoảng thời gian, và (2) cộng `usd_cost` để làm dashboard/cảnh
> báo chi phí. Chuỗi `print("đã trả lời xong")` không có trường dữ liệu ổn định,
> không có timestamp và cũng không cho biết client hay chi phí nên máy khó tổng
> hợp chính xác.

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
| 1 stage (bản đầu) | Chưa đo được — máy hiện không có lệnh `docker` |
| Multi-stage | Chưa đo được — 2 test build/image bị skip vì thiếu Docker |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi không ghi số MB giả: khi chạy `docker version`, PowerShell báo
> `docker: The term 'docker' is not recognized`, còn test CP2 cho 14 test cấu
> hình pass và 2 test cần Docker bị skip. Nhìn vào Dockerfile hiện tại, phần
> chênh lệch dự kiến chủ yếu đến từ việc bản đầu dùng image Python đầy đủ và giữ
> cả môi trường build. Bản multi-stage dùng `python:3.11-slim`; stage runtime chỉ
> nhận thư viện trong `/install`, mã `app` và `utils`, không mang toàn bộ filesystem
> của builder, cache cài đặt hay các công cụ chỉ cần lúc build. Muốn có số đo thật
> tôi vẫn cần cài/bật Docker rồi chạy đúng ba lệnh trong đề.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Máy hiện không có Docker nên tôi chưa quan sát được các dòng `CACHED` của một
> lần build thật. Tuy nhiên test tĩnh xác nhận Dockerfile đang theo thứ tự
> `COPY requirements.txt` → `RUN pip install` → `COPY app`. Vì vậy khi chỉ sửa
> `app/main.py`, stage builder và layer cài dependency vẫn dùng lại được; cache
> bắt đầu mất ở `COPY app ./app`, rồi các layer đứng sau nó phải được tạo lại.
> Nếu đặt `COPY . .` trước `RUN pip install`, chỉ một ký tự trong source cũng làm
> hash của layer `COPY` đổi, kéo theo `pip install` chạy lại dù
> `requirements.txt` không đổi. Đây là phần tốn thời gian nhất nhưng hoàn toàn
> không cần thiết cho thay đổi mã nguồn đó.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi rủi ro tôi hình dung là: lỗi Python cho phép thực thi lệnh trong
> container → tiến trình đang là root nên mã độc có toàn quyền trên filesystem
> và capability được cấp cho container → nếu container còn bị cấu hình nguy hiểm
> như mount Docker socket, mount thư mục host có quyền ghi hoặc có lỗ hổng
> container-runtime, kẻ tấn công có thể thoát ra và chiếm quyền cao trên host.
> Root trong container không tự động đồng nghĩa root trên host, nhưng làm hậu quả
> của bước khai thác nặng hơn nhiều. Dockerfile hiện chuyển sang `USER appuser`
> (UID 10001), nên mã khai thác từ ứng dụng trước hết chỉ có quyền của user thường,
> không thể tùy ý sửa file hệ thống hay dùng các đặc quyền root trong container;
> chuỗi tấn công bị chặn ngay sau bước thực thi mã.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> Khi tôi chạy các test xác thực, response 401 có
> `WWW-Authenticate: Bearer`. Header này là lời “challenge” theo chuẩn HTTP: nó
> cho client biết tài nguyên yêu cầu Bearer token và thư viện HTTP có thể chọn
> đúng cơ chế xác thực. Cả thiếu header, sai scheme và sai token đều trả
> `invalid or missing bearer token` để endpoint không trở thành một oracle cho
> người đang dò. Nếu trả lỗi quá chi tiết, kẻ tấn công biết ngay mình đã viết
> đúng scheme/cấu trúc và chỉ còn phải đoán token; thông báo chung làm giảm thông
> tin rò rỉ trong khi client hợp lệ vẫn có header chuẩn để biết cách sửa request.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> Tôi đã chạy test với mốc thời gian cố định: 10 token được tiêu hết tại
> `t=1000`, sau 30 giây có đúng 5 token; test im lặng một ngày cũng chỉ trả về
> 10 token. Vì vậy sau 10 phút, client vẫn chỉ gửi liên tiếp được 10 request;
> request thứ 11 nhận 429. Nếu bỏ `min(capacity, ...)`, số token tăng thêm là
> `10 phút × 10 token/phút = 100`. Trong kịch bản xô đã cạn trước lúc nghỉ, nó
> sẽ bắn được 100 request; trong đúng setup test đã gửi một request rồi nghỉ
> (còn 9 token), nó sẽ tích thành 109. Lỗi nằm ở chỗ thời gian im lặng biến
> thành sức chứa không giới hạn, trái với ý nghĩa `capacity=10`.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> Với hạn mức $30/tháng, một sự cố từ 2 giờ sáng có thể đốt gần như toàn bộ phần
> ngân sách tháng còn lại, tối đa khoảng $30 nếu đầu tháng chưa dùng gì; service
> chỉ tự cho client gọi lại khi sang tháng mới. Với $1/ngày, thiệt hại của cùng
> sự cố bị giới hạn khoảng $1 cho mỗi ngày và tự hồi phục khi khóa chuyển sang
> ngày UTC mới. Tôi thấy `CostGuard.today()` dùng UTC, nên ở múi giờ Bangkok mốc
> reset là 07:00 sáng, không phải 00:00 giờ địa phương. Các test cũng cho thấy
> key của hai ngày khác nhau tách biệt, nên sang ngày mới không cần người vận
> hành xóa Redis bằng tay.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện sẽ là: Redis mất kết nối → endpoint gộp kiểm tra Redis của cả
> ba container cùng trả 503 → orchestrator hiểu nhầm cả ba process đã chết và
> đồng thời rút/restart chúng → request đang xử lý bị ngắt và cụm không còn
> instance phục vụ → trong lúc Redis chưa hồi, container mới tiếp tục báo lỗi,
> có thể tạo vòng restart → Redis trở lại sau 30 giây nhưng ứng dụng vẫn phải
> khởi động lại và nối lại dependency, kéo dài thời gian gián đoạn. Khi chạy test
> hiện tại, Redis sống thì `/readyz` trả 200, Redis chết thì `/readyz` trả 503;
> `/healthz` được giữ nhẹ và không nhận dependency Redis. Vì thế lỗi dependency
> chỉ làm load balancer ngừng gửi traffic, không kích hoạt restart hàng loạt.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Tôi chưa deploy được một service cloud thật trong workspace này. Lỗi thực tế
> khi chạy test CP5 là: `Chưa điền Public URL thật vào DEPLOYMENT.md`; test dừng
> ngay tại fixture `base_url`. Tôi kiểm tra `DEPLOYMENT.md` và thấy URL vẫn là
> `https://TODO-thay-bang-url-that.up.railway.app`, các thông tin cá nhân/platform
> còn để trống; thư mục `screenshots` cũng chỉ có README. Nguyên nhân hiện tại
> không phải lỗi FastAPI mà là chưa hoàn tất bước deploy/cấu hình đầu vào, và máy
> này cũng chưa có Docker để dùng phương án local fallback. Cách sửa là deploy
> image lên Railway/Render, khai báo `API_TOKEN`, `REDIS_URL` cùng các biến cấu
> hình, kiểm tra `/healthz` và `/readyz`, rồi thay URL mẫu trong `DEPLOYMENT.md`
> bằng URL HTTPS thật. Tôi chưa ghi rằng lỗi đã được sửa vì chưa có URL chạy thật
> để kiểm chứng.

---

### Câu 11 — Bài học sau khi hoàn thành lab

Sau khi chạy và kiểm tra service, quan niệm của bạn về một ứng dụng “chạy được
trên production” đã thay đổi như thế nào? Nêu ít nhất ba điều ngoài logic trả
lời chat mà bạn cho là bắt buộc.

> Trước bài lab, tôi dễ xem việc endpoint trả được câu trả lời là dấu hiệu ứng
> dụng đã sẵn sàng. Sau khi chạy test, tôi thấy logic `/chat` chỉ là một phần nhỏ.
> Thứ nhất, cấu hình và secret phải lấy từ môi trường, đồng thời phải fail fast;
> test bỏ `API_TOKEN` cho thấy app dừng ngay sẽ an toàn hơn âm thầm chạy với token
> mẫu. Thứ hai, service phải có liveness và readiness riêng: Redis lỗi thì
> `/readyz` trả 503 nhưng không nên kéo theo việc restart tất cả process qua
> `/healthz`. Thứ ba, state, rate limit và chi phí phải nằm trong Redis để nhiều
> instance cùng nhìn thấy một kết quả và không bị mất khi container khởi động
> lại. Tôi còn thấy structured log và graceful shutdown cũng không phải phần
> trang trí: log JSON giúp truy vết theo client/chi phí, còn draining giúp ngừng
> nhận request mới trước khi process thoát. Với tôi, “production-ready” bây giờ
> có nghĩa là ứng dụng vẫn kiểm soát được cấu hình, traffic, state, chi phí và
> vòng đời khi dependency hoặc container gặp sự cố, chứ không chỉ chạy đúng ở
> máy cá nhân.
