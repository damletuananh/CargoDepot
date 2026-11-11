# Runbook — Chạy StorageHub full stack (BE · FE · PayOS · tunnel)

Hướng dẫn từ số 0 đến khi thanh toán PayOS thật chạy được trên máy local.
Cập nhật: 2026-10-20.

---

## 0. Yêu cầu môi trường

| Thành phần | Phiên bản / ghi chú |
|---|---|
| JDK | 21+ (repo test với Java 25) |
| Maven | 3.9+ |
| Node.js + npm | Node 20+ |
| MySQL | 8.4 LTS, đang chạy ở `localhost:3306` |
| ngrok | 3.x (chỉ cần khi test PayOS thật) |
| Git | bất kỳ |

Kiểm tra nhanh:

```bash
java -version
mvn -v
node -v && npm -v
mysql --version
ngrok version
```

---

## 1. Lấy code

```bash
git clone https://github.com/<your-org-or-username>/StorageHub.git
cd StorageHub

git checkout master
```

---

## 2. Chuẩn bị MySQL

App tự tạo database nhờ `createDatabaseIfNotExist=true` trong JDBC URL, nhưng nên tạo sẵn:

```sql
CREATE DATABASE storagehub_demo
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_0900_ai_ci;
```

> **Quan trọng:** dùng database **mới** (`storagehub_demo`) nếu DB `storagehub` cũ đã từng chạy migration khác.
> Flyway sẽ báo `Migration checksum mismatch`. Xem `docs/backend/BACKEND-DATABASE-GUIDE.md`.
>
> Profile `dev` chạy thêm seed demo (`V2`–`V5`) gồm user, unit, policy, reservation mẫu.

---
## 3. Cấu hình `backend/.env`

App tự đọc `backend/.env` qua `spring.config.import` trong `application.yml`.
Biến môi trường của OS (nếu có) sẽ **ưu tiên hơn** file `.env`.

```bash
cd backend
cp .env.example .env
```

Điền `backend/.env`:

```env
# --- Database ---
DB_HOST=localhost
DB_PORT=3306
DB_NAME=storagehub_demo
DB_USER=root
DB_PASS=mat_khau_mysql

# --- JWT (Base64, >= 32 bytes) ---
# Tạo mới: openssl rand -base64 48
JWT_SECRET=thay_bang_chuoi_base64_ngau_nhien

# --- PayOS ---
# false = chỉ demo CARD/MOMO/VNPAY, tab PayOS báo chưa cấu hình
PAYOS_ENABLED=true
PAYOS_CLIENT_ID=...
PAYOS_API_KEY=...
PAYOS_CHECKSUM_KEY=...

# FE quay về sau khi thanh toán / hủy
PAYOS_RETURN_URL=http://localhost:5173/payment
PAYOS_CANCEL_URL=http://localhost:5173/payment
```

Lấy 3 key PayOS tại [my.payos.vn](https://my.payos.vn) → kênh thanh toán → **Cấu hình API**.

`backend/.env` đã nằm trong `backend/.gitignore` — **không commit**.

---

## 4. Chạy Backend

```bash
cd backend
mvn spring-boot:run -Dspring-boot.run.profiles=dev
```

Hoặc trong IntelliJ: chọn `StorageHubApplication`, thêm VM/program argument:

```
-Dspring-boot.run.profiles=dev
```

Kiểm tra:

- Health: <http://localhost:8080/actuator/health> → `{"status":"UP"}`
- Swagger: <http://localhost:8080/swagger-ui.html>

Log khởi động phải có `Started StorageHubApplication`.

### Lỗi thường gặp khi boot

| Log | Nguyên nhân / cách xử lý |
|---|---|
| `FlywayValidateException: Migration checksum mismatch` | DB cũ khác migration. Tạo DB mới (`DB_NAME=storagehub_demo`) hoặc xử lý theo `docs/backend/BACKEND-DATABASE-GUIDE.md` mục 5. |
| `PAYOS_ENABLED=true nhưng thiếu PAYOS_...` | Điền đủ 3 key trong `.env` rồi restart. |
| `Port 8080 was already in use` | Tắt process cũ: `netstat -ano \| findstr :8080` → `taskkill /PID <pid> /F`. |

---

## 5. Chạy Frontend

```bash
cd frontend
npm install
npm run dev
```

Mở <http://localhost:5173>.

Vite proxy `/api` → `http://localhost:8080` (xem `frontend/vite.config.js`), nên FE và BE không cần CORS.

> Xem chi tiết toàn diện về kiến trúc kỹ thuật Frontend, Data Flow, đặc tả API và ma trận phân quyền tại [docs/frontend/README.md](frontend/README.md).

### Tài khoản seed

| Vai trò | Email | Mật khẩu |
|---|---|---|
| Customer | `lan@demo.storagehub.vn` | `password` |
| Customer | `trang@demo.storagehub.vn` | `password` |
| Staff | `minh@demo.storagehub.vn` | `password` |
| Facility Manager | `tuan@demo.storagehub.vn` | `password` |
| Business Ops | `hang@demo.storagehub.vn` | `password` |
| System Admin | `nam@demo.storagehub.vn` | `password` |

---

## 6. Ba phương thức thanh toán

| Method | Chế độ | Hành vi |
|---|---|---|
| `CARD` | Demo | Bấm thanh toán → `PROCESSING` → poll ~1s → `SUCCEEDED`. Không cần thẻ hợp lệ. |
| `MOMO` | Demo | Bấm thanh toán → tự `SUCCEEDED`; không có bước OTP. |
| `VNPAY` | Demo | Bấm tab → tự tạo phiên → `SUCCEEDED` sau ~1s. |
| `PAYOS` | Thật | Tạo link PayOS + VietQR; trạng thái chỉ chuyển `SUCCEEDED` khi PayOS gọi webhook. |

Demo methods vẫn chạy state machine thật trong `PaymentService`: sinh receipt `RT-…`, reservation → `RESERVED`, contract draft, notification, activity log.

Đổi tốc độ demo trong `application.yml`:

```yaml
payment:
  mock:
    auto-complete-delay: PT0S   # 0s = gần như tức thì; PT2S = chậm 2 giây
```

---

## 7. Tunnel + Webhook PayOS

PayOS là server ngoài internet, không gọi được `localhost`. Cần tunnel.

### 7.1 Chạy ngrok

```bash
ngrok http 127.0.0.1:8080 --domain=<your-static-domain>.ngrok-free.dev
```

> Dùng **`127.0.0.1:8080`**, không dùng `localhost:8080`.
> Trên Windows, `localhost` có thể resolve sang IPv6 `::1`, trong khi Tomcat chỉ listen IPv4 → ngrok trả
> `ERR_NGROK_8012` (502 Bad Gateway).
>
> Nếu không có static domain: `ngrok http 127.0.0.1:8080` rồi copy URL `https://xxxx.ngrok-free.app`.
> Mỗi lần ngrok restart URL có thể đổi → phải cập nhật lại dashboard PayOS.

### 7.2 Khai báo webhook trên PayOS

Vào [my.payos.vn](https://my.payos.vn) → kênh thanh toán → **Cấu hình Webhook**:

```
https://<domain-ngrok>/api/v1/payments/webhooks/payos
```

Backend cũng nhận alias (tương thích cấu hình cũ):

```
https://<domain-ngrok>/api/v1/payments/webhook
```

Endpoint công khai, **không dùng JWT**; xác thực bằng chữ ký HMAC-SHA256 với `PAYOS_CHECKSUM_KEY`.

### 7.3 Kiểm tra tunnel đã tới backend

```bash
curl -i -X POST https://<domain-ngrok>/api/v1/payments/webhooks/payos \
  -H "Content-Type: application/json" -d "{}"
```

Kết quả đúng:

```http
HTTP/1.1 400 Bad Request
{"code":"PAYMENT_WEBHOOK_INVALID","message":"Chữ ký webhook PayOS không hợp lệ.","fieldErrors":[]}
```

- `400 PAYMENT_WEBHOOK_INVALID` = tunnel OK, backend đã nhận request và từ chối chữ ký rỗng.
- `502` + `ERR_NGROK_8012` = ngrok không kết nối được backend → kiểm tra upstream `127.0.0.1:8080`.
- `404` + `ERR_NGROK_3200` = tunnel đang offline.
- `404` JSON của backend = sai path webhook.

### 7.4 Lưu ý trạng thái PENDING

- PayOS tạo link/QR thành công nhưng khách chưa chuyển khoản → payment vẫn `PENDING`.
- Khách chuyển khoản → PayOS POST webhook → `PaymentService` chuyển `SUCCEEDED`, chạy handler đúng một lần.
- Webhook retry hoặc bấm "Làm mới" nhiều lần không tạo receipt trùng: xử lý idempotent theo trạng thái terminal.
- Nếu webhook không tới: FE poll `GET /api/v1/payments/{id}` sẽ hỏi trực tiếp PayOS; link hết hạn → `EXPIRED`.
- Link PayOS mặc định hết hạn sau ~5 phút (`payment.payos.qr-ttl`).

---

## 8. Test end-to-end PayOS thật

1. Đăng nhập customer.
2. Vào một unit còn trống → **Tiến hành thanh toán cọc**.
   - Nếu vào từ `/units/:unitId/book`, FE tự tạo reservation thật trước khi tạo payment.
3. Chọn tab **PayOS QR**.
4. Quét VietQR bằng app ngân hàng, hoặc bấm **Mở cổng PayOS**.
5. Chuyển đúng số tiền hiển thị (ví dụ `114.000 ₫`).
6. Chờ webhook (vài giây). Modal tự chuyển **Thanh toán cọc thành công**, hiện biên lai `RT-…`.
7. Kiểm tra DB:

```sql
SELECT payment_id, receipt_code, purpose, method, amount, status
FROM payments ORDER BY payment_id DESC LIMIT 5;

SELECT reservation_id, code, status FROM reservations ORDER BY reservation_id DESC LIMIT 5;
```

---

## 9. Kiểm thử tự động

```bash
# Backend
mvn test

# Frontend
cd frontend
npm run lint
npm run build
```

Test PayOS tập trung ở:

- `backend/src/test/java/com/storagehub/gateway/PayOsPaymentGatewayTest.java`
  — chữ ký hợp lệ / giả mạo / thiếu signature / orderCode lạ.
- `backend/src/test/java/com/storagehub/service/PaymentWebhookServiceTest.java`
  — success chạy handler 1 lần, webhook retry idempotent, lệch tiền bị chặn.

---

## 10. Checklist lỗi nhanh

| Hiện tượng | Kiểm tra |
|---|---|
| `You do not have permission to access this resource` | Reservation không thuộc user đang đăng nhập. Đăng xuất/đăng nhập lại, hoặc vào flow đặt kho để tạo reservation mới. |
| `PayOS chưa được cấu hình` | `PAYOS_ENABLED=true` + đủ 3 key trong `.env`; restart backend. |
| Webhook trả `PAYMENT_WEBHOOK_INVALID` khi PayOS gọi | `PAYOS_CHECKSUM_KEY` sai hoặc dashboard gửi payload khác kênh. Sửa key → restart. |
| Payment mãi `PENDING` dù đã chuyển khoản | Tunnel offline, webhook URL sai, hoặc PayOS chưa gửi. Kiểm tra **Webhook deliveries** trên dashboard. |
| `ERR_NGROK_8012` | Chạy lại ngrok với `127.0.0.1:8080`. |
| `ERR_NGROK_3200` | Tunnel offline — bật lại ngrok và cập nhật URL nếu domain đổi. |
| `401` sau khi reset DB | JWT cũ trỏ user không còn tồn tại — đăng nhập lại. |
| Flyway checksum mismatch | Dùng DB mới hoặc xử lý theo `docs/backend/BACKEND-DATABASE-GUIDE.md` mục 5. |
| Port đã bị chiếm | `netstat -ano \| findstr :8080` (hoặc `:5173`) → `taskkill /PID <pid> /F`. |

---

## 11. Reset môi trường demo

```bash
# Dừng backend trước
mysql -uroot -p -e "DROP DATABASE IF EXISTS storagehub_demo;"
cd backend
mvn spring-boot:run -Dspring-boot.run.profiles=dev
```

Flyway chạy lại `V1` + seed `V2`–`V5`; DB sạch, reservation mẫu trở lại trạng thái ban đầu.

---

## 12. Bảo mật

- Không commit `backend/.env`, key PayOS, JWT secret, mật khẩu DB.
- Nếu key PayOS lộ: vào my.payos.vn tạo lại key và cập nhật `.env` + dashboard webhook.
- `PAYOS_ENABLED=false` khi chỉ demo offline; tab PayOS sẽ báo chưa cấu hình thay vì gọi mạng.
