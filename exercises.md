# Phiếu phản ánh — K4 Ngày 12

Họ và tên: Nguyễn Thị Thu Trang  
Mã học viên: 2A202601172

---

### Câu 1 — Fail fast (CP1)

Khi deploy lên Render mà quên đặt `API_TOKEN`, app sẽ không khởi động được ngay
do Pydantic báo thiếu trường bắt buộc. Điều này tốt hơn việc app vẫn chạy với
`changeme`: service có thể trông như đang khỏe nhưng endpoint đã được bảo vệ bằng
một token công khai mà bất kỳ ai cũng đoán được. Fail fast giúp phát hiện lỗi cấu
hình trong log deploy trước khi có request thật hoặc chi phí phát sinh.

### Câu 2 — Log cho máy đọc (CP1)

Một dòng log mình quan sát được là:

```json
{"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T10:32:33.688350+00:00", "client_id": "sv-test", "prompt_tokens": 3, "completion_tokens": 37, "usd_cost": 2.26e-05}
```

Từ một dòng JSON như vậy, mình có thể lọc tất cả request theo `event`, hoặc tính
tổng chi phí theo `client_id`/`usd_cost`. Mình cũng có thể cảnh báo khi số token,
thời gian hoặc mức `severity` bất thường. Một câu `print` thông thường không có
cấu trúc ổn định để máy lọc và thường thiếu timestamp, loại sự kiện và các trường
định lượng.

### Câu 3 — Kích thước image (CP2)

Kết quả build image production hiện tại mình đo được:

| Bản | Dung lượng |
|---|---:|
| 1 stage (bản đầu theo template) | khoảng 1.8 GB theo hướng dẫn lab |
| Multi-stage | 70.9 MB |

Bản multi-stage nhỏ hơn vì runtime dùng `python:3.11-slim`, chỉ giữ virtualenv
đã cài dependency cùng thư mục `app` và `utils`; source test, git, cache và file
`.env` bị loại khỏi build context. Bản một stage dùng image Python đầy đủ và copy
cả repository nên mang theo nhiều thành phần không cần khi chạy service.

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Với Dockerfile hiện tại, `COPY requirements.txt` và bước tạo/cài virtualenv ở
stage builder được cache lại khi chỉ sửa một ký tự trong `app/main.py`. Các layer
copy source ở stage runtime và layer `chown` phải chạy lại, vì nội dung source đã
thay đổi. Nếu đặt `COPY . .` trước `pip install`, mọi thay đổi code cũng làm
invalid layer chứa toàn bộ source, khiến Docker phải cài lại toàn bộ dependency,
build chậm hơn đáng kể.

### Câu 5 — Vì sao không chạy bằng root (CP2)

Nếu có lỗi trong thư viện hoặc một lỗ hổng cho phép kẻ tấn công thực thi lệnh,
process trong container có thể đọc/ghi các file mà user của process được phép đọc
ghi. Nếu process là root, quyền đó có thể được dùng để khai thác thêm Docker hoặc
đường kết nối tới host, biến lỗi trong app thành quyền cao trên máy chạy. Trong
image của mình, `USER appuser` chuyển process sang user thường sau khi các file đã
được chuẩn bị; vì vậy chuỗi tấn công bị cắt ở quyền user của container, không phải
quyền root.

### Câu 6 — Bearer token (CP3)

`WWW-Authenticate: Bearer` cho client biết server yêu cầu kiểu xác thực nào và
token phải được gửi theo scheme Bearer. Mình đã quan sát request `/chat` không có
token trên Render trả `401` với thông báo `invalid or missing bearer token`.

Dùng cùng một thông báo cho thiếu header, sai scheme và sai token làm giảm thông
tin mà endpoint tiết lộ. Nếu trả lời “scheme đúng nhưng token sai” hoặc “thiếu
header”, người dò có thể dùng khác biệt đó để tối ưu việc thử token. Việc so sánh
giá trị hợp lệ còn dùng `secrets.compare_digest` để tránh rò rỉ qua thời gian phản
hồi.

### Câu 7 — Token bucket (CP3)

Với capacity 10, tốc độ nạp 10 token/phút và im lặng 10 phút, client tích được
`10 + 10 × 10 = 110` token theo công thức lý thuyết, nhưng `min(capacity, ...)`
chặn lượng thực tế ở 10. Vì vậy client gửi được 10 request liên tiếp, request thứ
11 nhận 429.

Nếu bỏ `min`, client sẽ gửi được 110 request trước khi bị 429. Đây là lý do phải
giới hạn trần: thời gian im lặng không được biến thành một kho token vô hạn và tạo
ra một đợt request lớn đột biến.

### Câu 8 — Ngân sách theo ngày (CP3)

Với hạn mức tháng 30 USD, sự cố bắt đầu lúc 2 giờ sáng có thể làm mất toàn bộ
30 USD trước khi sang tháng mới; service chỉ tự hồi phục khi tháng kế tiếp bắt
đầu. Với hạn mức 1 USD/ngày, thiệt hại tối đa trong ngày UTC hiện tại chỉ là 1
USD, sau đó cost guard trả 402; ngân sách tự tách sang key của ngày UTC mới và
được dùng lại vào ngày hôm sau. Vì vậy hạn mức ngày giới hạn phạm vi của sự cố
nhỏ hơn rất nhiều so với hạn mức tháng.

### Câu 9 — `/healthz` khác `/readyz` (CP4)

Nếu cả hai endpoint cùng ping Redis, khi Redis mất kết nối trong 30 giây, cả ba
container sẽ cùng trả trạng thái unhealthy. Orchestrator có thể hiểu nhầm đó là
lỗi process và restart cả ba instance gần như cùng lúc. Khi Redis quay lại, không
còn instance ổn định nào phục vụ hoặc các instance vừa khởi động lại tiếp tục
fail probe, làm lỗi ngắn của Redis thành outage toàn cụm.

Thiết kế hiện tại giữ `/healthz` nhẹ, chỉ kiểm tra process và trả 200; `/readyz`
mới kiểm tra Redis và trả 503 khi Redis không sẵn sàng. Load balancer vì vậy rút
instance khỏi traffic mà không restart cả cụm.

### Câu 10 — Deploy thật (CP5)

Khi kiểm tra Render, mình thấy free instance có cảnh báo sẽ spin down lúc không
hoạt động, nên request đầu tiên có thể chậm khoảng 50 giây. Lúc kiểm tra ban đầu
request hết thời gian chờ; mình xem trạng thái deploy trên dashboard, đợi service
wake up rồi gọi lại bằng `curl.exe`. Kết quả sau đó là `/healthz` 200,
`/readyz` 200 với `redis: true`, còn `/chat` không có token trả 401. Như vậy đây
là độ trễ cold start của gói free, không phải lỗi `REDIS_URL` hay lỗi bind port.
