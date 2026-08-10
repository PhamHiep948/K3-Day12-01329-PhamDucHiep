# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder câu trả lời bằng nội dung của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Phạm Đức Hiệp  Mã học viên: 01329

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy (hoặc `docker compose up`), nếu quên set `AGENT_API_KEY` thì
> `Settings()` ném `ValidationError` ngay lúc start — container không lên,
> nhìn log là biết thiếu secret. Nếu mặc định `"changeme"`, service vẫn xanh
> trên dashboard nhưng ai đoán được khóa mặc định cũng gọi `/ask` được; mình
> chỉ phát hiện khi quota/chi phí tăng bất thường.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log thật khi gọi `/ask` thành công:
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T03:00:00+00:00", "user_id": "sv-test", "tokens_in": 12, "tokens_out": 80, "cost_usd": 0.00005}`
>
> 1) Lọc theo `user_id` hoặc `event=ask_completed` trên CloudWatch/Datadog.
> 2) Tính tổng `cost_usd` hoặc cảnh báo khi `cost_usd` cao — `print` thuần
> chữ không parse/aggregate được.

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
| 1 stage (bản đầu) | 1.69 GB (`agent:single`, base `python:3.11`) |
| Multi-stage | 270 MB (`day12-agent:cp2-test`) |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Đo thật trên máy: single ~1.69GB, multi-stage 270MB. Phần chênh (~1.4GB)
> chủ yếu là base image đầy đủ (không slim) và các lớp cài đặt thừa không
> cần lúc runtime. Multi-stage chỉ mang `/install` sang `python:3.11-slim`.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Dockerfile hiện tại: `COPY requirements.txt` → `pip install` → rồi mới
> `COPY app/` / `COPY utils/`. Sửa `main.py` thì layer requirements + pip
> vẫn cache; chỉ các bước COPY code và sau đó chạy lại. Nếu `COPY . .` trước
> `pip install`, mỗi lần sửa một dòng code Docker hủy cache từ COPY → cài
> lại toàn bộ thư viện dù `requirements.txt` không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Lỗ hổng RCE trong app → process đang là root trong container → kẻ tấn công
> cài tool, đọc secret, lợi dụng cấu hình mount/privileged để leo thang ra
> host. `USER appuser` cắt ngay từ đầu: process app không còn root trong
> container, giảm quyền sau khi khai thác thành công.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa 20 request trong ~2 giây: gửi 10 request lúc 10:00:59 (hết quota
> phút đó), sang 10:01:00 cửa sổ cố định reset, gửi tiếp 10 request nữa.
> Sliding window 60 giây gần nhất không có kẽ hở kiểu “đổi phút”.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn **số request / thời gian** (bảo vệ hạ tầng). Cost guard
> giới hạn **tiền / tháng** (bảo vệ ngân sách).
> - Rate cho qua, cost chặn: 1 request/phút nhưng mỗi câu hỏi rất dài/tốn
>   token → chưa vượt 10/phút nhưng `spent` đã gần `MONTHLY_BUDGET_USD` → 402.
> - Cost cho qua, rate chặn: còn nhiều ngân sách nhưng spam 15 request trong
>   vài giây → 429 dù tiền vẫn dư.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Redis chết → cả 3 instance báo probe “đỏ” → orchestrator coi container
> unhealthy và **restart cả 3** → khi Redis sống lại có thể không còn instance
> nào sẵn sàng phục vụ (hoặc restart đồng loạt gây sóng). Tách `/health`
> (không đụng Redis) và `/ready` (có Redis): LB chỉ ngừng đẩy traffic, không
> restart oan cả cụm.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis + nginx round-robin, `history_length` tăng đều giữa các lần gọi
> (0 → 2 → 4…) dù request rơi container khác nhau. Nếu dùng dict trong RAM,
> mỗi container một bộ nhớ riêng → số nhảy lung tung hoặc về 0 tùy instance
> nhận request, agent như “mất trí nhớ”.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lúc chạy local qua compose (chuẩn bị fallback/deploy), gọi `/ready` qua
> nginx trả `500 Internal Server Error`; log agent ghi
> `NotImplementedError: TODO (CP4): cài đặt /ready` — image còn bản code cũ
> trước khi làm xong `/ready`. Nguyên nhân tìm bằng `docker compose logs agent`.
> Sửa: `docker compose up -d --build agent`, sau đó `/ready` trả 200
> `{"status":"ready","redis":true}`. Vì chưa có tài khoản Railway/Render nên
> dùng `LOCAL_FALLBACK=true` + screenshots thay cho URL HTTPS.
