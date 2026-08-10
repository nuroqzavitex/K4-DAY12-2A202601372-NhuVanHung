# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Như Văn Hùng  Mã học viên: 2A202601372

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Giả sử khi deploy ứng dụng lên Cloud (như Railway hay Render), bạn quên cài đặt biến môi trường `API_TOKEN`. Nếu để mặc định là `"changeme"`, ứng dụng vẫn khởi động thành công và chạy bình thường. Kẻ tấn công hoặc bất kỳ ai biết token mặc định này có thể gọi API của bạn liên tục, lạm dụng LLM và đốt sạch ngân sách tài khoản mà bạn không hề hay biết cho đến khi nhận hóa đơn. Ngược lại, với việc "Fail fast" (không có giá trị mặc định), ứng dụng sẽ dừng ngay lúc deploy với lỗi thiếu `API_TOKEN`, giúp bạn phát hiện và bổ sung secret trước khi ứng dụng nhận bất kỳ request nào.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log mẫu:
`{"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T07:36:25.123456+00:00", "client_id": "sv01", "prompt_tokens": 15, "completion_tokens": 30, "usd_cost": 0.00015}`

Hai việc làm được với log JSON:
1. **Lọc và tìm kiếm tự động (Structured Filtering)**: Các hệ thống gom log (như Datadog, GCP Logging, CloudWatch) có thể tự động parse JSON để tạo bộ lọc chính xác, ví dụ: lọc tất cả request của `client_id="sv01"` có `usd_cost > 0.001` hoặc tự tạo cảnh báo khi có `severity="ERROR"`. Lệnh `print()` văn bản thuần túy không hỗ trợ cấu trúc dữ liệu để lọc máy đọc như vậy.
2. **Thống kê và tính toán chi phí (Aggregation & Metrics)**: Dễ dàng trích xuất các trường số (`prompt_tokens`, `usd_cost`) để tính tổng chi phí sử dụng theo ngày/tháng hoặc vẽ biểu đồ tải theo thời gian thực mà không cần viết các hàm regex phức tạp để tách chuỗi.

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
| 1 stage (bản đầu) | 1850 MB |
| Multi-stage | 185 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~1.66 GB) bao gồm: các công cụ biên dịch mã nguồn (GCC, Make, g++), các header files (.h) phục vụ build C/C++, bộ nhớ cache của pip (`~/.cache/pip`), các gói bổ sung của Python bản đầy đủ và các thư viện hệ điều hành không cần thiết ở môi trường production. Multi-stage build loại bỏ toàn bộ bộ công cụ biên dịch này và chỉ copy kết quả cài đặt cuối cùng sang base image `python:3.11-slim` siêu nhẹ.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Với Dockerfile hiện tại: Các layer từ base image, `COPY requirements.txt .` và `RUN pip install` đều được **dùng lại từ cache** (CACHED). Chỉ có layer `COPY . .` và các layer phía sau mới phải chạy lại.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Khi sửa dù chỉ 1 ký tự trong `app/main.py`, Docker sẽ vô hiệu hóa cache (cache invalidation) ngay tại lệnh `COPY . .`. Do đó, lệnh `RUN pip install` đứng sau bắt buộc phải chạy lại từ đầu, khiến Docker phải tải và cài đặt lại toàn bộ dependencies mỗi lần sửa code, làm thời gian build kéo dài từ vài giây lên vài phút.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- Chuỗi sự kiện:
  1. Code Python gặp lỗ hổng (như Remote Code Execution - RCE qua `eval()`, `pickle`, hoặc Command Injection).
  2. Kẻ tấn công khai thác lỗ hổng để chạy lệnh shell bên trong container.
  3. Vì container chạy mặc định với user `root`, tiến trình của kẻ tấn công bên trong container có UID 0.
  4. Nếu kẻ tấn công khai thác tiếp lỗ hổng thoát khỏi container (container breakout), họ sẽ đứng trên máy Host với đúng quyền UID 0 (root của máy Host) và chiếm toàn bộ quyền điều khiển server.
- Lệnh `USER appuser` cắt đứt chuỗi ở bước 3: Chuyển tiến trình sang chạy với user thường (unprivileged user). Ngay cả khi kẻ tấn công thực thi được lệnh hoặc thoát được khỏi container, họ chỉ có quyền hạn cực kỳ hạn chế của user thường, không thể sửa đổi file hệ thống hay can thiệp tiến trình khác trên máy host.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

- Header `WWW-Authenticate: Bearer` là bắt buộc theo chuẩn **HTTP RFC 6750** đối với phản hồi 401 Unauthorized, giúp client/browser biết chính xác cơ chế xác thực mà server yêu cầu để tự động điều chỉnh request thích hợp.
- Trả cùng một thông báo lỗi ("invalid or missing bearer token") giúp bảo mật chống lại việc rò rỉ thông tin (Information Disclosure). Nếu báo chi tiết "sai scheme" hay "sai token", kẻ tấn công sẽ thu thập được thông tin về cấu trúc xác thực và dễ dàng thực hiện brute-force hoặc dò tìm token hơn.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

- Khi có `min(capacity, ...)`: Client chỉ gửi được tối đa **10 request** liên tiếp trước khi bị lỗi 429, vì xô token chỉ tích lũy tối đa bằng sức chứa `capacity=10`.
- Nếu bỏ `min(capacity, ...)`: Sau 10 phút im lặng, số token tích lũy sẽ là $10 \text{ phút} \times 10 \text{ token/phút} = 100 \text{ token}$. Client có thể bắn liên tiếp **100 request** trong 1 giây mà không bị chặn, làm mất tác dụng chống bùng nổ traffic (burst protection) của Rate Limiter.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

- Hạn mức **$30/tháng**: Thiệt hại tối đa là **$30** (toàn bộ ngân sách tháng bị đốt sạch chỉ trong vài giờ đầu tiên của sự cố). Service bị khóa và chỉ tự hồi phục vào đầu tháng sau (hoặc khi nạp thêm tiền).
- Hạn mức **$1/ngày**: Thiệt hại tối đa chỉ là **$1** (giới hạn thiệt hại xuống 1/30). Service sẽ **tự động hồi phục** vào 00:00 UTC sáng hôm sau khi hạn mức ngày mới bắt đầu.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện:
1. Redis gặp sự cố đứt kết nối 30 giây.
2. Endpoint gộp báo thất bại (503 / unhealthy) cho cả 3 container.
3. Orchestrator (Docker/K8s/Cloud) dựa vào liveness probe (`/healthz`) thấy unhealthy nên kết luận cả 3 container đã chết, lập tức **kill và restart cả 3 container**.
4. Trong lúc 3 container đang bị restart, toàn bộ cụm rơi vào trạng thái ngừng hoạt động hoàn toàn (total downtime).
5. Khi container mới khởi động lại, Redis vẫn chưa khôi phục kết nối (trong khoảng 30s sự cố), làm container mới rớt healthcheck tiếp và rơi vào vòng lặp khởi động lại liên tục (CrashLoopBackOff), biến sự cố tạm thời của Redis thành thảm họa sập toàn bộ hệ thống.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Thông báo lỗi**: `Error: Invalid value for '--port': '$PORT' is not a valid integer` (dẫn tới `Healthcheck failure` trên Railway).
- **Cách tìm nguyên nhân**: Xem log chi tiết trong mục **View logs** trên Railway dashboard, phát hiện ra file `railway.toml` định nghĩa `startCommand = "uvicorn app.main:app --host 0.0.0.0 --port $PORT"` thực thi trực tiếp mà không qua Shell, khiến `$PORT` không được giải mã thành số nguyên mà truyền nguyên chuỗi ký tự `"$PORT"`.
- **Cách sửa**: Sửa dòng `startCommand` trong [railway.toml](file:///d:/VinAI/LABS/K4-DAY12-2A202601372-NhuVanHung/railway.toml) thành `startCommand = "python -m app.main"`. Khi đó Python sẽ dùng `pydantic-settings` đọc biến môi trường `PORT` và chuyển thành số nguyên `int` chuẩn xác.
