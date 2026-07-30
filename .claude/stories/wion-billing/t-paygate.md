# T-PayGate – Tài liệu tích hợp cho đối tác thứ 3

---

## 0. Mục lục


 1. [Tổng quan & môi trường](#1-t%E1%BB%95ng-quan--m%C3%B4i-tr%C6%B0%E1%BB%9Dng)
 2. [Quy trình chính](#2-quy-tr%C3%ACnh-ch%C3%ADnh)
 3. [Bước 1 – OAuth](#3-b%C6%B0%E1%BB%9Bc-1--oauth-l%E1%BA%A5y-access_token)
 4. [Bước 2 – Connect](#4-b%C6%B0%E1%BB%9Bc-2--connect-k%E1%BA%BFt-n%E1%BB%91i-ng%C3%A2n-h%C3%A0ng)
 5. [Bước 3 – Config](#5-b%C6%B0%E1%BB%9Bc-3--config-qu%E1%BA%A3n-l%C3%BD-k%E1%BA%BFt-n%E1%BB%91i)
 6. [Bước 4 – Bill](#6-b%C6%B0%E1%BB%9Bc-4--bill-t%E1%BA%A1o--v%E1%BA%A5n-tin-h%C3%B3a-%C4%91%C6%A1n)
 7. [Webhook (T-PayGate → đối tác)](#7-webhook-t-paygate--%C4%91%E1%BB%91i-t%C3%A1c)
 8. [Bảo mật](#8-b%E1%BA%A3o-m%E1%BA%ADt)
 9. [Mã lỗi](#9-m%C3%A3-l%E1%BB%97i)
10. [Testing & Go-live](#10-testing--go-live)
11. [Phụ lục](#11-ph%E1%BB%A5-l%E1%BB%A5c)


---

## 1. Tổng quan & môi trường

### 1.1 T-PayGate là gì

Cổng kết nối thanh toán trung gian, đứng giữa đối tác thứ 3 (3rd party — POS, ERP, eCommerce, ứng dụng quản lý bán hàng) và các ngân hàng đối tác. Đối tác chỉ cần tích hợp **một bộ API duy nhất** của T-PayGate, không phải làm việc kỹ thuật trực tiếp với từng ngân hàng.

### 1.2 Hai môi trường

| Mục | Staging (UAT) | Production |
|----|----|----|
| **Base URL** | `https://t-paygate.tpos.dev` | `https://t-paygate.tpos.app` |
| OAuth token | `/api/v1/oauth/token` | `/api/v1/oauth/token` |
| Public API prefix | `/api/v1/public-api/...` | `/api/v1/public-api/...` |
| Connect-bank UI | `/api/v1/public-api/view/connect` | `/api/v1/public-api/view/connect` |
| Webhook timeout | 30 giây | 10 giây |
| Webhook retry | 3 lần × 5 phút | 3 lần × 5 phút |

### 1.3 Đối tác cần chuẩn bị

| Mục | Khi nào cần |
|----|----|
| `clientId`, `tenantId`, `source` (T-PayGate cấp) | Bước 1 OAuth |
| Thông tin ngân hàng đối tác (số tài khoản, các mã định danh do bank cấp…) | Bước 2 Connect |
| URL webhook public HTTPS | Trước go-live |
| IP outbound để T-PayGate whitelist | Trước go-live PROD |


---

## 2. Quy trình chính

```
   ┌───────────────────────────────────────────────────────────────────┐
   │                                                                   │
   │   [1] OAUTH      [2] CONNECT       [3] CONFIG       [4] BILL      │
   │                                                                   │
   │   POST /token →  POST /connect →   GET /list   →   POST /bill     │
   │   (mỗi 60p)      POST /confirm     POST /dis-     GET /get-*      │
   │                  (1 lần / merchant) connect       (mỗi đơn hàng)  │
   │                                                                   │
   │                                                  ↓ webhook        │
   │                                                  PAYMENT_RECEIVED │
   └───────────────────────────────────────────────────────────────────┘
```

| Bước | Tần suất | Output | Lưu lại |
|----|----|----|----|
| **1. OAuth** | Mỗi 60 phút | `access_token` JWT | Cache phía đối tác |
| **2. Connect** | 1 lần / merchant / bank | `configBankId`, `vaNumber` | DB phía đối tác |
| **3. Config** | Khi cần xem/hủy | List / xóa | – |
| **4. Bill** | Mỗi đơn hàng | `billCode`, QR data | DB phía đối tác |


---

## 3. Bước 1 – OAuth (lấy `access_token`)

### 3.1 Endpoint

```
POST /api/v1/oauth/token

Content-Type: application/x-www-form-urlencoded
```

### 3.2 Request

| Field | Type | Bắt buộc | Mô tả |
|----|----|----|----|
| `clientId` | string | ✓ | T-PayGate cấp khi onboarding |
| `tenantId` | string | ✓ | T-PayGate cấp |
| `source` | string | ✓ | Phân kênh  |

```http

POST /api/v1/oauth/token HTTP/1.1

Host: t-paygate.tpos.dev

Content-Type: application/x-www-form-urlencoded

clientId=demo_client&tenantId=abc-123&source=WEB
```

### 3.3 Response (200)

```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR...",
  "expiresIn": 3600
}
```

### 3.4 Sử dụng token

Mọi API ở bước 2/3/4 đều phải gửi header:

```
Authorization: Bearer <accessToken>
```

### 3.5 Lưu ý

* Token TTL = **3600 giây (60 phút)** → đối tác cache + refresh trước hết hạn \~5 phút.
* Không gọi `/token` mỗi request — sẽ bị rate-limit.
* Nếu token hết hạn → API trả **HTTP 401** → gọi lại `/token`.


---

## 4. Bước 2 – Connect (kết nối ngân hàng)

Mục tiêu: liên kết tài khoản ngân hàng của merchant với T-PayGate **một lần duy nhất**, sau đó dùng `configBankId` cho mọi giao dịch ở bước 4.

### 4.1 Hai cách kết nối

| Cách | Ưu điểm | Phù hợp |
|----|----|----|
| **A. Embedded UI** (`/view/connect`) | T-PayGate lo UI, OTP, validate; đối tác chỉ mở iframe / tab | Web app, ít custom |
| **B. API trực tiếp** (`/config-bank/connect`) | Đối tác tự design UI | Mobile app, native |

### 4.2 Cách A – Embedded UI

#### Bước 2.A.1 – Mở UI

```
GET /api/v1/public-api/view/connect?bankCode={bankCode}&token={access_token}
```

T-PayGate sẽ render form đăng ký phù hợp theo từng ngân hàng. Form fields cụ thể (số tài khoản, mã merchant, OTP, …) thay đổi theo ngân hàng đối tác chọn — đối tác không cần biết trước, chỉ truyền `bankCode` đã được cấp.

#### Bước 2.A.2 – User submit form

T-PayGate tự gọi API `/connect` ở backend, sau đó:

* Nếu cần OTP → render màn hình OTP, user nhập → POST `/view/otp`.
* Nếu không cần OTP → render màn hình thành công luôn.

#### Bước 2.A.3 – Tab đóng / postMessage

Sau thành công T-PayGate post message `tabClosed` về `window.opener` để parent app refresh trạng thái.

### 4.3 Cách B – API trực tiếp

#### Bước 2.B.1 – Tạo kết nối

```
POST /api/v1/public-api/config-bank/connect

Authorization: Bearer <token>
Content-Type: application/json
```

Body — `ConnectConfigBankDto`:

| Field | Type | Bắt buộc | Mô tả |
|----|----|----|----|
| `bankCode` | string | ✓ | Mã ngân hàng (T-PayGate cung cấp danh sách qua `GET /bank`) |
| `merchantName` | string | ✓ | Tên cửa hàng / merchant |
| `accountName` | string | ✓ | Tên chủ tài khoản |
| `accountNo` | string | ✓ | Số tài khoản nhận tiền |
| `identity` | string | ◯ | CMND/CCCD (cần với một số bank) |
| `phone` | string | ◯ | SĐT đăng ký dịch vụ |
| `email` | string | ◯ | Email |
| `prefix` | string | ◯ | Mã merchant do ngân hàng cấp |
| `clientId` | string | ◯ | Mã định danh do ngân hàng cấp |
| `encryptKey` | string | ◯ | Khóa do ngân hàng cấp (nếu yêu cầu) |
| `secretKey` | string | ◯ | Khóa do ngân hàng cấp (nếu yêu cầu) |

> Mỗi ngân hàng yêu cầu một tập field khác nhau. Tham khảo tài liệu kỹ thuật riêng của từng ngân hàng trong gói onboarding để biết field nào bắt buộc.

```json
{
  "bankCode": "XXX",
  "merchantName": "Cửa hàng A",
  "accountName": "NGUYEN VAN A",
  "accountNo": "108800888060",
  "identity": "012345678901",
  "phone": "0987654321",
  "email": "shop@example.com",
  "prefix": "merchant_xxx",
  "clientId": "provider_xxx"
}
```

Response:

```json
{
  "success": true,
  "results": {
    "configBankId": "abc-uuid",
    "isConnected": false,
    "isOTPConfirmation": true
  }
}
```

#### Bước 2.B.2 – Xác thực OTP (chỉ khi `isOTPConfirmation = true`)

```
POST /api/v1/public-api/config-bank/confirm

Authorization: Bearer <token>
Content-Type: application/json
```

Body — `ConfirmBankDto`:

```json
{
  "configBankId": "abc-uuid",
  "otpNumber": "123456"
}
```

Response thành công:

```json
{
  "success": true,
  "results": {
    "configBankId": "abc-uuid",
    "vaNumber": "9XYZ200309052356",
    "isConnected": true
  }
}
```

### 4.4 Sequence (cách B với OTP)

```
3rd Party             T-PayGate           Bank
   │                     │                  │
   │ POST /connect       │                  │
   │────────────────────>│                  │
   │                     │ Register         │
   │                     │─────────────────>│
   │                     │<─────────────────│ OTP gửi SMS đến KH
   │ {isOTPConfirmation} │                  │
   │<────────────────────│                  │
   │                     │                  │
   │ KH nhập OTP         │                  │
   │ POST /confirm       │                  │
   │────────────────────>│                  │
   │                     │ Verify           │
   │                     │─────────────────>│
   │                     │<─────────────────│ OK + VA number
   │ {isConnected:true}  │                  │
   │<────────────────────│                  │
```


---

## 5. Bước 3 – Config (quản lý kết nối)

### 5.1 Liệt kê kết nối hiện có

```
GET /api/v1/public-api/config-bank/list?bankCode={optional}
Authorization: Bearer <token>
```

Response:

```json
[
  {
    "configBankId": "abc-uuid",
    "bankCode": "XXX",
    "accountNo": "108800888060",
    "accountName": "NGUYEN VAN A",
    "vaNumber": "9XYZ200309052356",
    "merchantId": "INTERNAL_ID",
    "clientId": "provider_xxx",
    "phone": "0987654321",
    "isConnected": true,
    "urlLogo": "https://.../logo.png",
    "dateCreated": "2026-05-01T08:00:00Z"
  }
]
```

### 5.2 Hủy kết nối

```
POST /api/v1/public-api/config-bank/disconnect

Authorization: Bearer <token>
Content-Type: application/json
```

Body — `DeleteConfigBankDto`:

```json
{ "configBankId": "abc-uuid" }
```

T-PayGate sẽ:


1. Gọi API hủy phía bank tương ứng.
2. Set `IsConnected = false` (giữ record cho đối soát lịch sử, không xóa cứng).

### 5.3 Danh sách ngân hàng được hỗ trợ

```
GET /api/v1/public-api/bank
```

Trả full list ngân hàng kèm logo, BIN, mã chuẩn NAPAS — đối tác dùng để build dropdown cho user chọn.


---

## 6. Bước 4 – Bill (tạo & vấn tin hóa đơn)

Sau khi connect xong (có `vaNumber`), đối tác tạo hóa đơn cho mỗi đơn hàng để khách hàng quét QR thanh toán.

| Field | Type | Bắt buộc | Mô tả |
|----|----|----|----|
| `x-client-id` | string | ✓ | T-PayGate cấp khi onboarding |
| `x-tenant-id` | string | ✓ | T-PayGate cấp |
| `x-source` | string | ✓ | Phân kênh (VD: WiCloud, TPOS,..) |
| `x-config-id` | string | ✓ | Id cấu hình ngân hàng |

### 6.1 Tạo hóa đơn

```
POST /api/v1/public-api/order/bill

Authorization: Bearer <token>
Content-Type: application/json
```

Body — `PayGateOrderBillRequestDto`:

| Field | Type | Bắt buộc | Mô tả |
|----|----|----|----|
| `refTransactionId` | string | ✓ | Mã giao dịch tham chiếu phía đối tác |
| `amount` | decimal | ✓ | Số tiền (VND) |
| `description` | string |    | Nội dung giao dịch hiển thị trên QR |

```json
{
  "refTransactionId": "ORDER-2026-0001",
  "amount": 250000,
  "description": "Thanh toan don hang 0001"
}
```

Response:

```json
{
  "success": true,
  "results": {
    "billCode": "B2026050500001",
    "vaNumber": "9XYZ200309052356",
    "amount": 250000,
    "qrContent": "00020101021238...",
    "qrImageBase64": "iVBORw0KGgo...",
    "expiredAt": "2026-05-05T15:30:00Z"
  }
}
```

> Đối tác tự render QR từ `qrContent` (dùng thư viện QR ở client) hoặc dùng luôn `qrImageBase64`.

### 6.2 Vấn tin theo `refTransactionId`

```
GET /api/v1/public-api/order/get-refTransactionId?refTransactionId=ORDER-2026-0001

Authorization: Bearer <token>
```

Trả danh sách bill có cùng `refTransactionId`.

### 6.3 Vấn tin theo `billCode`

```
GET /api/v1/public-api/order/get-billCode?billCode=B2026050500001

Authorization: Bearer <token>
```

### 6.4 Vòng đời hóa đơn

```
CREATED → WAITING_PAYMENT → PAID (success)
                ↘
                  EXPIRED  (hết hạn)
                  CANCELED (đối tác hủy)
```

| Trạng thái | Mô tả |
|----|----|
| `CREATED` | Vừa tạo, chưa quét QR |
| `WAITING_PAYMENT` | Có khách quét, đang chờ chuyển khoản |
| `PAID` | Bank đã ghi có, T-PayGate đã callback đối tác |
| `EXPIRED` | Quá hạn (mặc định 24h, cấu hình theo tenant) |
| `CANCELED` | Đối tác chủ động hủy |


---

## 7. Webhook (T-PayGate → đối tác)

### 7.1 Đăng ký endpoint

* 1 URL public HTTPS, đăng ký lúc onboarding.
* T-PayGate POST JSON về URL này khi có sự kiện

### 7.2 Body Webhook- v1

```json
{
    "RefTransactionId": "TXN202606030001",
    "BillCode": "HD20260603001",
    "Amount": 1500000.00,
    "VirtualAccount": "970422000123456789",
    "ActualAccount": "1234567890123",
    "PaymentTime": "2026-06-03T14:30:00"
}
```


### 7.2 Body Webhook- v2 - Releases Pending

```json
{
  "EventType": "PAYMENT_RECEIVED",
  "Timestamp": "2026-05-05T08:30:00Z",
  "Data": {
    "RefTransactionId": "TXN202606030001",
    "BillCode": "HD20260603001",
    "Amount": 1500000.00,
    "VirtualAccount": "970422000123456789",
    "ActualAccount": "1234567890123",
    "PaymentTime": "2026-06-03T14:30:00"
  }
}
```

### 7.2.1 Sự kiện chính

|    |    |    |
|----|----|----|
|    |    |    |
|    |    |    |
|    |    |    |
|    |    |    |

### 7.3 Phản hồi từ đối tác

* HTTP 200 + `{"MessageError": true}` trong **< 10 giây** (PROD) / **< 30 giây** (UAT).
* Fail/timeout → T-PayGate retry **3 lần × 5 phút**.
* **Bắt buộc idempotent** theo `billCode` / `bankTransId`.


---

## 8. Bảo mật

### 8.1 Lớp 1 – Kênh

* HTTPS bắt buộc (TLS 1.2+), cert hợp lệ.
* Whitelist IP (PROD) — đối tác đăng ký dải IP outbound.

### 8.2 Lớp 2 – Identity

* OAuth 2.0 Client Credentials.
* JWT TTL = 60 phút.

### 8.3 Lớp 3 – Toàn vẹn dữ liệu

* Mọi chữ ký số (giữa T-PayGate ↔ ngân hàng đối tác, giữa T-PayGate ↔ webhook) do T-PayGate tự xử lý nội bộ. Đối tác không phải tự ký hoặc verify.

### 8.4 Lớp 4 – Idempotency

* `refTransactionId` (đối tác sinh) — T-PayGate cache 24h.
* `billCode` / `bankTransId` (T-PayGate sinh) — đối tác cache 24h.

### 8.5 Lớp 5 – Bí mật

* `clientId`/`clientSecret` không commit Git, không log.
* Khi rotate: T-PayGate thông báo trước **30 ngày**.
* Lộ credentials → đối tác báo T-PayGate trong 1 giờ; T-PayGate revoke trong 4 giờ.


---

## 9. Mã lỗi

### 9.1 HTTP code

| Code | Ý nghĩa |
|----|----|
| 200 | Thành công |
| 400 | Sai format / validate |
| 401 | Token sai/hết hạn |
| 403 | Không có quyền / chữ ký sai |
| 404 | Resource không tồn tại |
| 409 | Trùng `refTransactionId` |
| 429 | Quá rate limit |
| 5xx | Lỗi nội bộ T-PayGate |

### 9.2 Business code (response body)

| Code | Ý nghĩa |
|----|----|
| `0` / `00` | Thành công |
| `01` | Sai chữ ký |
| `02` | Mã KH/Hóa đơn không tồn tại |
| `03` | Lỗi nghiệp vụ phía bank |
| `05` | Trùng `transId` |
| `99` | Lỗi không xác định |

> Lỗi cụ thể từ phía ngân hàng đối tác sẽ được T-PayGate đóng gói lại trong `message`, đối tác **không cần xử lý theo từng bank** — chỉ cần check `success` và đọc `message` để hiển thị.


---

## 10. Testing & Go-live

### 10.1 Bộ test data UAT

T-PayGate cung cấp khi onboarding:

* `clientId` + `tenantId` + `source` UAT
* 1 tài khoản test ngân hàng đối tác đã đăng ký sẵn
* Số CMND + SĐT đã đăng ký dịch vụ test
* OTP cố định (nếu áp dụng)
* Postman collection

### 10.2 Testcase tối thiểu

| # | Kịch bản | Pass criteria |
|----|----|----|
| 1 | OAuth thành công | Có `accessToken` |
| 2 | OAuth sai `clientId` | HTTP 401 |
| 3 | Connect ngân hàng có OTP, OTP đúng | `isConnected=true`, có `vaNumber` |
| 4 | Connect sai OTP 3 lần | Mã lỗi rõ, không cấp VA |
| 5 | List config-bank | Trả đúng kết nối vừa tạo |
| 6 | Tạo bill, scan, thanh toán | Webhook `PAYMENT_RECEIVED` về đúng `refTransactionId` |
| 7 | Webhook timeout → retry | Retry đủ 3 lần, dừng đúng quy tắc |
| 8 | Trùng `refTransactionId` | Trả response cũ, không double-process |
| 9 | Disconnect | `IsConnected=false`, bank revoke OK |
| 10 | Đối soát T+1 | File khớp với webhook đã nhận |

### 10.3 Go-live checklist

#### Phía đối tác

- [ ] Hoàn tất 100% testcase UAT
- [ ] Endpoint webhook PROD chạy + cert SSL hợp lệ
- [ ] IP outbound PROD đã gửi cho T-PayGate
- [ ] Email/group oncall xác nhận
- [ ] Idempotency verified
- [ ] Plan rollback

#### Phía T-PayGate

- [ ] Cấp `clientId`/`tenantId` PROD (riêng UAT)
- [ ] Cert ký số PROD (nếu bank yêu cầu — xử lý nội bộ)
- [ ] Whitelist IP đối tác
- [ ] Bật monitoring/alert
- [ ] Job đối soát hàng ngày chạy
- [ ] Backup config bank trước enable


---

## 11. Phụ lục

### 11.1 Tóm tắt endpoints

| Bước | Method | Path | Auth |
|----|----|----|----|
| OAuth | POST | `/api/v1/oauth/token` | – |
| Connect (UI) | GET/POST | `/api/v1/public-api/view/connect` | Token query |
| Connect (UI OTP) | POST | `/api/v1/public-api/view/otp` | Token query |
| Connect (API) | POST | `/api/v1/public-api/config-bank/connect` | Bearer |
| Confirm OTP | POST | `/api/v1/public-api/config-bank/confirm` | Bearer |
| List config | GET | `/api/v1/public-api/config-bank/list` | Bearer |
| Disconnect | POST | `/api/v1/public-api/config-bank/disconnect` | Bearer |
| Bank list | GET | `/api/v1/public-api/bank` | – |
| Tạo bill | POST | `/api/v1/public-api/order/bill` | Bearer |
| Vấn tin theo ref | GET | `/api/v1/public-api/order/get-refTransactionId` | Bearer |
| Vấn tin theo billCode | GET | `/api/v1/public-api/order/get-billCode` | Bearer |

### 11.2 Mã ngân hàng

Đối tác lấy danh sách động qua `GET /api/v1/public-api/bank` và dùng đúng `code` trả về (chuẩn NAPAS) làm `bankCode` ở các bước tiếp theo. Không hardcode danh sách.

### 11.3 Postman collection

T-PayGate cấp khi onboarding:

* `t-paygate-staging.postman_collection.json`
* `t-paygate-prod.postman_collection.json`

Đã set sẵn các environment variable: `baseUrl`, `clientId`, `tenantId`, `source`, `accessToken`.

### 11.4 Hỗ trợ

Đối tác liên hệ qua kênh đã đăng ký lúc onboarding (email / group chat / hotline) để được hỗ trợ kỹ thuật và vận hành.


---

## 12. Lịch sử thay đổi

| Ngày | Phiên bản | Thay đổi |
|----|----|----|
| 2026-05-05 | 1.0 | Khởi tạo outline |
| 2026-05-05 | 1.1 | Cập nhật URL prod `t-paygate.tpos.app`, restructure theo 4 bước OAuth → Connect → Config → Bill |
| 2026-05-05 | 1.2 | Tài liệu standalone cho T-PayGate, ẩn thông tin bank cụ thể (đối tác lấy động qua `/bank`) |
| 2026-05-05 | 1.3 | Bỏ chi tiết verify chữ ký webhook (T-PayGate tự xử lý nội bộ) |