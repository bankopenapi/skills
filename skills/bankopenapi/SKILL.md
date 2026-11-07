---
name: bankopenapi
description: Tích hợp BankOpenAPI để nhận tiền chuyển khoản vào tài khoản ngân hàng Việt Nam - tạo yêu cầu thanh toán (mã TT… trong nội dung chuyển khoản), hiện mã VietQR, nhận webhook payment_request.paid / transaction.received, verify chữ ký Standard Webhooks, chống xử lý trùng. Dùng khi người dùng muốn thu tiền chuyển khoản cho đơn hàng, tự xác nhận thanh toán, hiện QR chuyển khoản, nhận báo có qua webhook, hoặc nhắc tới BankOpenAPI, app.bankopenapi.com, API key boa_live_. Use for BankOpenAPI bank-transfer payments, VietQR checkout and payment webhooks in Vietnam.
---

# BankOpenAPI

BankOpenAPI báo cho hệ thống của người dùng biết khi có tiền chuyển khoản vào tài khoản ngân hàng Việt Nam của họ. Tiền đi thẳng vào tài khoản ngân hàng của người dùng; BankOpenAPI không giữ tiền.

Tài liệu đầy đủ: https://bankopenapi.com/tai-lieu

## Luồng tích hợp chuẩn

1. Người dùng đã liên kết tài khoản ngân hàng và tạo API key trong app (https://app.bankopenapi.com). Việc này làm trên giao diện, không có API.
2. Khi khách đặt đơn: backend gọi `POST /v1/payment-requests` → nhận `code` (vd. `TT7K3M9Q`), `transfer` (thông tin chuyển khoản), `qr` (chuỗi QR dựng sẵn) và `hostedUrl`.
3. Vẽ `qr.payload` thành ảnh QR bằng thư viện QR bất kỳ (hoặc chuyển hướng người trả tới `hostedUrl`). `qr.type` là `dynamic` (QR động một lần của ngân hàng, khớp theo chính mã QR, hết hạn lúc `qr.expiresAt`) hoặc `static` (VietQR tĩnh, khớp theo mã trong nội dung); code không cần phân biệt. `qr` là `null` khi yêu cầu không còn nhận tiền.
4. Tiền về đúng mã → BankOpenAPI gửi webhook `payment_request.paid` → endpoint verify chữ ký, chống trùng theo `id`, đánh dấu đơn đã thanh toán, trả 200.

## Quy tắc bắt buộc

- **API key chỉ dùng ở backend.** Đọc từ biến môi trường `BANKOPENAPI_API_KEY`. Không đặt trong code frontend, app di động, không commit. API không bật CORS nên gọi từ trình duyệt sẽ lỗi.
- **Số tiền là số nguyên VND** (`150000`), từ 1.000 đến 500.000.000. Không dùng số thập phân.
- **Verify chữ ký webhook trên body thô** trước khi tin dữ liệu. Không parse JSON rồi stringify lại.
- **Chống xử lý trùng** bằng `id` của sự kiện (= header `webhook-id`): lưu vào bảng có khóa duy nhất, trong cùng transaction với việc cập nhật đơn. Đã thấy thì bỏ qua và vẫn trả 200.
- **Trả 2xx trong 10 giây.** Việc nặng đưa vào hàng đợi. BankOpenAPI retry tối đa 8 lần trong khoảng 27 giờ nếu không nhận 2xx.
- **Không dựa vào thứ tự sự kiện.** Quyết định theo `data.status`. Cần chắc thì gọi `GET /v1/payment-requests/:id`.
- **Sandbox:** API key `boa_test_…` chỉ làm việc với dữ liệu thử. Tạo tài khoản ngân hàng thử bằng `POST /v1/test/bank-accounts`, giả lập thanh toán bằng `POST /v1/test/payment-requests/:id/pay` (hoặc tiền vào tùy ý: `POST /v1/test/transactions`). Webhook của dữ liệu thử có `livemode: false`. Chi tiết: https://bankopenapi.com/tai-lieu/sandbox
- Không tự đặt nội dung chuyển khoản khác `transfer.content`; không dùng số tài khoản gốc thay cho `transfer.accountNumber` (có thể là virtual account).

## API

Base URL: `https://app.bankopenapi.com/api/v1` (sẽ chuyển sang `api.bankopenapi.com`; đặt base URL trong biến môi trường `BANKOPENAPI_API_BASE`).

Header: `Authorization: Bearer <API key>` (dạng `boa_live_` + 36 ký tự, khóa sandbox `boa_test_`; khóa cũ có 40 ký tự), `Content-Type: application/json` khi có body.

| Endpoint | Việc |
| --- | --- |
| `GET /v1/account` | Thông tin chủ API key (kiểm tra khóa) |
| `POST /v1/payment-requests` | Tạo yêu cầu thanh toán, trả 201 |
| `GET /v1/payment-requests/:id` | Xem một yêu cầu, kèm mảng `transactions` đã khớp |
| `GET /v1/payment-requests` | Liệt kê, mới nhất trước. Query: `limit` (1–100, mặc định 50), `status` (`open` \| `paid` \| `attention` \| `closed`), `order_id`, `q`, `starting_after` |
| `POST /v1/payment-requests/:id/cancel` | Hủy yêu cầu chưa nhận tiền (`pending`/`expired`); đã nhận tiền thì 409 `CANNOT_CANCEL` |
| `GET /v1/transactions` | Giao dịch vào tài khoản đã liên kết, mới nhất trước. Query: `bank_account`, `type` (`credit` \| `debit`), `from`, `to` (ISO 8601), `limit`, `starting_after` |
| `GET /v1/transactions/:id` | Xem một giao dịch (`txn_…`) |
| `GET /v1/bank-accounts` | Tài khoản ngân hàng đang liên kết (`id` dùng làm `bankAccountId`) |
| `GET /v1/bank-accounts/:id` | Xem một tài khoản (`ba_…`) |

Lệnh POST nhận header `Idempotency-Key` (1–255 ký tự, vd. UUID mới cho mỗi lệnh): gửi lại cùng khóa trong 24 giờ thì nhận đúng phản hồi cũ (header `Idempotent-Replayed: true`), không tạo trùng. Retry sau timeout luôn dùng lại khóa cũ. API key "chỉ đọc" gọi POST bị 403 `INSUFFICIENT_PERMISSION`.

Quản lý webhook endpoint: `/v1/webhook-endpoints` (gọi `POST …/test-url` với `{ url }` trước, rồi tạo kèm `urlTestToken`; signing secret ở `GET …/:id/secret`). Chi tiết: https://bankopenapi.com/tai-lieu/api/webhook-endpoint

Chưa có (đừng gọi): SDK chính thức.

Body tạo yêu cầu:

| Trường | Kiểu | Ghi chú |
| --- | --- | --- |
| `amount` | integer, bắt buộc | VND, 1.000 – 500.000.000 |
| `orderId` | string | Mã đơn của hệ thống, ≤ 64 ký tự `[A-Za-z0-9_.:/#-]`. Không bắt buộc duy nhất |
| `description` | string | ≤ 200 ký tự, hiện cho người trả ở `hostedUrl` |
| `expiresInMinutes` | integer | 5 – 10080, mặc định 1440 |
| `bankAccountId` | string | `ba_…`; bỏ trống = tài khoản liên kết gần nhất |

Đối tượng `payment_request` (rút gọn):

```json
{
  "id": "pr_3fK9mQ2xLwA7nB5vR8tY1cZd",
  "object": "payment_request",
  "amount": 150000,
  "amountReceived": 0,
  "currency": "VND",
  "status": "pending",
  "code": "TT7K3M9Q",
  "orderId": "DH1028",
  "description": "Đơn hàng DH1028",
  "bankAccount": { "id": "ba_…", "bank": { "bin": "970422", "name": "MB Bank" }, "accountNumber": "0123456789", "label": "MB Bank •••• 6789" },
  "transfer": { "bankBin": "970422", "accountNumber": "0123456789", "accountName": "NGUYEN VAN AN", "amount": 150000, "content": "TT7K3M9Q" },
  "qr": { "type": "static", "payload": "00020101021238540010A000000727…6304FE1C", "expiresAt": null },
  "hostedUrl": "https://app.bankopenapi.com/thanh-toan/pr_3fK9mQ2xLwA7nB5vR8tY1cZd",
  "paymentLinkId": null,
  "expiresAt": "2026-11-06T03:20:11.000Z",
  "paidAt": null,
  "canceledAt": null,
  "createdAt": "2026-11-05T03:20:11.000Z"
}
```

Trạng thái: `pending` (chờ) → `paid` (đủ) | `overpaid` (thừa, vẫn gửi `payment_request.paid`) | `underpaid` (thiếu, gửi `payment_request.underpaid`; `transfer.amount` còn số thiếu, người trả chuyển thêm cùng mã) | `expired` (quá `expiresAt` chưa nhận tiền; tiền về sau đó vẫn được khớp) | `canceled`.

Lỗi: HTTP 4xx/5xx kèm `{ "error": { "code": "...", "message": "..." } }`. Rẽ nhánh theo `code`: `INVALID_JSON`, `INVALID_AMOUNT`, `INVALID_ORDER_ID`, `INVALID_EXPIRY`, `INVALID_BANK_ACCOUNT`, `INVALID_CURSOR` (400), `INVALID_API_KEY` (401), `NOT_FOUND` (404), `CANNOT_CANCEL` (409), `NO_BANK_ACCOUNT` (422, chưa liên kết tài khoản), `RATE_LIMITED` (429, có `Retry-After`), `INTERNAL` (500). Chỉ retry 429, 5xx và lỗi mạng. Giới hạn: 100 request/phút mỗi API key.

Tạo yêu cầu bị timeout: trước khi tạo lại, gọi `GET /v1/payment-requests?order_id=<mã đơn>` để tránh một đơn có hai mã.

### Ví dụ: tạo yêu cầu (Node.js 18+)

```js
const API = process.env.BANKOPENAPI_API_BASE ?? "https://app.bankopenapi.com/api/v1";

export async function createPaymentRequest(order) {
  const res = await fetch(`${API}/payment-requests`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${process.env.BANKOPENAPI_API_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({ amount: order.total, orderId: order.id, description: `Đơn hàng ${order.id}` }),
  });
  const data = await res.json();
  if (!res.ok) throw new Error(`${data.error.code}: ${data.error.message}`);
  return data; // lưu data.id vào đơn hàng
}
```

Python: `requests.post(f"{API}/payment-requests", headers={"Authorization": f"Bearer {key}"}, json={...}, timeout=15)`. PHP: cURL với `CURLOPT_HTTPHEADER` và `json_encode`.

## VietQR

Dựng chuỗi VietQR (chuẩn EMVCo của NAPAS) từ `transfer` ngay ở backend, rồi vẽ QR bằng thư viện bất kỳ (npm `qrcode`, PyPI `qrcode[pil]`, composer `chillerlan/php-qrcode`):

```js
const tlv = (id, value) => id + String(value.length).padStart(2, "0") + value;

function crc16(s) {
  let crc = 0xffff;
  for (let i = 0; i < s.length; i++) {
    crc ^= s.charCodeAt(i) << 8;
    for (let j = 0; j < 8; j++) crc = (crc & 0x8000 ? (crc << 1) ^ 0x1021 : crc << 1) & 0xffff;
  }
  return crc.toString(16).toUpperCase().padStart(4, "0");
}

export function vietqr({ bankBin, accountNumber, amount, content }) {
  const merchant = tlv("00", "A000000727") + tlv("01", tlv("00", bankBin) + tlv("01", accountNumber)) + tlv("02", "QRIBFTTA");
  const payload =
    tlv("00", "01") +
    tlv("01", amount > 0 ? "12" : "11") +
    tlv("38", merchant) +
    tlv("53", "704") +
    (amount > 0 ? tlv("54", String(amount)) : "") +
    tlv("58", "VN") +
    (content ? tlv("62", tlv("08", content)) : "") +
    "6304";
  return payload + crc16(payload);
}
// const dataUrl = await QRCode.toDataURL(vietqr(paymentRequest.transfer), { width: 320, margin: 2 });
```

Hiển thị: QR + từng dòng chữ có nút chép (ngân hàng, số tài khoản, tên, số tiền, nội dung), dặn "giữ nguyên nội dung". Trang chờ thanh toán hỏi trạng thái từ backend của người dùng (đã được webhook cập nhật), không gọi BankOpenAPI từ trình duyệt.

Khớp tiền: BankOpenAPI đổi nội dung sang chữ hoa, bỏ ký tự không phải chữ/số, tìm `TT` + 6 ký tự trong `23456789ABCDEFGHJKMNPQRSTUVWXYZ` ở bất kỳ vị trí nào; tiền phải vào đúng tài khoản của yêu cầu. VietinBank không gửi nội dung chuyển khoản nên không tự khớp theo mã.

## Webhook

Người dùng tạo endpoint ở trang Webhook trong app (URL `https://` công khai, chọn sự kiện) và nhận signing secret `whsec_…` → biến môi trường `BANKOPENAPI_WEBHOOK_SECRET`.

Request: `POST`, JSON, header `webhook-id` (`evt_…`, giữ nguyên qua retry), `webhook-timestamp` (giây Unix), `webhook-signature` (`v1,<base64 HMAC-SHA256 của "<id>.<timestamp>.<body>">`, khóa = base64-decode phần sau `whsec_`). Body: `{ "id", "type", "livemode", "timestamp", "data" }`. Vừa đổi secret thì `webhook-signature` có hai chữ ký cách nhau dấu cách (secret mới và cũ); thư viện `standardwebhooks` tự xử lý.

Sự kiện: `transaction.received` (tiền vào, `data` là transaction có `paymentRequestId`), `payment_request.paid`, `payment_request.underpaid` (`data` là payment_request), `bank_account.linked`, `bank_account.removed`, `webhook.test` (nút "Gửi thử"). Bỏ qua loại và trường chưa biết. JSON Schema từng loại (để sinh kiểu dữ liệu hoặc viết test): https://bankopenapi.com/tai-lieu/webhook/schemas/index.json

Lỡ mất webhook: mọi sự kiện lưu 30 ngày, đọc lại bằng `GET /v1/events` (lọc `?type=`), `GET /v1/events/:id`, gửi lại tới endpoint bằng `POST /v1/events/:id/resend` với `{ endpointId }`.

Verify bằng thư viện `standardwebhooks` (npm `standardwebhooks`, PyPI `standardwebhooks`, composer `standard-webhooks/standard-webhooks`):

```js
// Express: cần body thô
import express from "express";
import { Webhook } from "standardwebhooks";

const wh = new Webhook(process.env.BANKOPENAPI_WEBHOOK_SECRET);
app.post("/webhooks/bankopenapi", express.raw({ type: "application/json" }), async (req, res) => {
  let event;
  try {
    event = wh.verify(req.body, req.headers);
  } catch {
    return res.status(400).send("Invalid signature");
  }
  if (await alreadyProcessed(event.id)) return res.sendStatus(200);
  if (event.type === "payment_request.paid") await markOrderPaid(event.data.orderId, event.data);
  await markProcessed(event.id);
  res.sendStatus(200);
});
```

```ts
// Next.js App Router / Hono / Cloudflare Workers: Request chuẩn Web
const body = await req.text();
const event = wh.verify(body, Object.fromEntries(req.headers));
```

```python
# Flask
from standardwebhooks import Webhook, WebhookVerificationError
wh = Webhook(os.environ["BANKOPENAPI_WEBHOOK_SECRET"])
event = wh.verify(request.get_data(), dict(request.headers))  # raise WebhookVerificationError nếu sai
```

```php
// PHP 8: thư viện đọc header bằng tên chữ thường
$wh = new \StandardWebhooks\Webhook(getenv('BANKOPENAPI_WEBHOOK_SECRET'));
$event = $wh->verify(file_get_contents('php://input'), array_change_key_case(getallheaders(), CASE_LOWER));
```

Chống trùng (SQL): bảng `webhook_events(id PRIMARY KEY, type, received_at)`; trong một transaction: `INSERT` id (trùng khóa → bỏ qua, trả 200) rồi `UPDATE orders SET status='paid' WHERE id=? AND status <> 'paid'`.

Kiểm thử ở máy dev: `cloudflared tunnel --url http://localhost:3000` (hoặc ngrok) để có URL https, khai báo làm endpoint, bấm "Gửi thử".

Khi người dùng hết 100 giao dịch miễn phí của tháng và số dư bằng 0, báo có tạm dừng: không có webhook nhưng trạng thái yêu cầu vẫn đúng khi gọi API. Nạp tiền xong, họ bấm "Gửi bù báo có" ở trang Giao dịch.

## MCP server

Agent gọi trực tiếp BankOpenAPI qua MCP (Streamable HTTP): `https://app.bankopenapi.com/api/mcp`, header `Authorization: Bearer <API key>`. Tool: `get_account`, `get_balance`, `create_payment_request`, `get_payment_request`, `list_payment_requests`, `cancel_payment_request`, `list_transactions`.

```bash
claude mcp add --transport http bankopenapi https://app.bankopenapi.com/api/mcp --header "Authorization: Bearer $BANKOPENAPI_API_KEY"
```
