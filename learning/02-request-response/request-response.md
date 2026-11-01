# Request — Response

> Mọi exchange với Web API là một cặp HTTP request/response. Hiểu rõ "shape" của mỗi bên giúp đọc log, debug và design đúng.
> Liên quan: [API & REST](../01-api-and-rest-api/api-and-rest-api.md) · [authentication](../03-api-authentication/api-authentication.md) · [API document](../04-api-document/api-document.md)

## 1. Vòng đời một exchange

```
Client                                              Server
  │  1. resolve DNS → TCP → (TLS nếu https)
  │  2. gửi REQUEST  ──────────────────────────────►  parse, route, validate
  │                                                   auth, business logic, DB
  │  3. nhận RESPONSE ◄──────────────────────────────  serialize, set status
  │  4. (keep-alive: dùng lại connection; hoặc đóng)
```

HTTP là **request-response, half-duplex, stateless**: server không chủ động gửi, và (với REST) không nhớ gì giữa 2 request. Muốn push từ server → WebSocket/SSE.

## 2. Cấu trúc REQUEST — 4 phần

```
POST /v1/orders HTTP/1.1        ← [1] start line: method + path + version
Host: api.shop.example          ← [2] headers (key: value)
Authorization: Bearer eyJhb...
Content-Type: application/json
                                ← [3] dòng trống tách headers và body
{"items": [{"sku": "A1", "qty": 2}]}   ← [4] body (optional)
```

| Phần | Chi tiết |
|---|---|
| Start line | `METHOD /path?query HTTP/x`. Path + query xác định *tài nguyên*; method xác định *thao tác* |
| Headers | metadata bắt buộc/tùy chọn. Phân loại: General (`Date`), Request (`Accept`), Entity (`Content-Type`) |
| Body | `GET`/`DELETE` thường không body; `POST`/`PUT`/`PATCH` có |

**3 cách đưa dữ liệu vào request** — chọn theo bản chất:

| Vị trí | Dùng khi | Ví dụ |
|---|---|---|
| Path parameter | định danh tài nguyên | `/orders/42` |
| Query parameter | filter, sort, phân trang, tham số tùy chọn | `/orders?status=paid&page=2` |
| Body | dữ liệu phức tạp/nhạy cảm khi tạo-sửa | JSON của order |

## 3. Cấu trúc RESPONSE — 3 phần

```
HTTP/1.1 201 Created            ← [1] status line: version + code + reason
Content-Type: application/json  ← [2] headers
Location: /v1/orders/1071
X-RateLimit-Remaining: 42
                                ← [3] body (dạng JSON: resource hoặc error)
{ "id": 1071, "status": "pending" }
```

Response nào cũng có status code; body là tùy chọn (`204` không body, `304` không body).

## 4. Status code — bảng tra theo nhóm

| Nhóm | Nghĩa | Client phải làm gì |
|---|---|---|
| **1xx** | informational (100 Continue, 101 Switching Protocols) | hiếm khi chạm trực tiếp |
| **2xx** | thành công | — |
| **3xx** | redirect | làm theo header `Location` |
| **4xx** | **lỗi do client** (request có vấn đề) | sửa request |
| **5xx** | **lỗi do server** | retry có backoff, hoặc bỏ |

Các code hay gặp nhất:

| Code | Tên | Khi nào dùng |
|---|---|---|
| 200 | OK | GET thành công, PATCH/PUT không tạo tài nguyên mới |
| 201 | Created | POST tạo tài nguyên mới — kèm header `Location` |
| 202 | Accepted | job bất đồng bộ, chưa xong |
| 204 | No Content | DELETE thành công, không trả gì |
| 301/302 | Moved / Found | redirect vĩnh viễn / tạm |
| 304 | Not Modified | cache còn hạn (kèm `ETag`/`If-None-Match`) |
| 400 | Bad Request | payload sai cú pháp/validation fail |
| 401 | Unauthorized | **thiếu/sai danh tính** — chưa đăng ký được (xem [auth](../03-api-authentication/api-authentication.md)) |
| 403 | Forbidden | danh tính hợp lệ nhưng **không được phép** |
| 404 | Not Found | tài nguyên không tồn tại (đừng trả 403 cho resource private — lộ thông tin) |
| 409 | Conflict | trùng lặp (unique), xung đột phiên bản (optimistic lock) |
| 422 | Unprocessable | đúng cú pháp nhưng sai ngữ nghĩa (validation nghiệp vụ) |
| 429 | Too Many Requests | rate limit — đọc `Retry-After` |
| 500 | Internal Server Error | bug/exception không xử lý |
| 502/503/504 | Bad Gateway / Unavailable / Gateway Timeout | hạ tầng, deploy, upstream chết |

## 5. Headers đáng nhớ khi làm API

| Header | Phía | Tác dụng |
|---|---|---|
| `Content-Type` | cả hai | body là gì: `application/json`, `multipart/form-data`, `text/plain` |
| `Accept` | request | client nhận được dạng gì (content negotiation) |
| `Authorization` | request | bằng chứng danh tính |
| `User-Agent` | request | client là gì |
| `Idempotency-Key` | request | UUID để retry `POST` không tạo trùng (Stripe phổ biến hóa) |
| `Cache-Control` | response | `max-age`, `no-store` — quyết định 3 ràng buộc cache của REST |
| `ETag` | response | chữ ký version của resource → `If-Match` cho updates an toàn |
| `Retry-After` | response | đi kèm 429/503: chờ bao lâu |
| `X-RateLimit-*` | response | quota còn lại (không chuẩn hóa nhưng rất phổ biến) |

## 6. JSON response: quy ước shape

Error nên thống nhất một cấu trúc (error envelope) — ví dụ mẫu [RFC 9457 (Problem Details)](https://www.rfc-editor.org/rfc/rfc9457):

```json
{
  "type": "https://api.shop.example/errors/validation",
  "title": "Validation failed",
  "status": 400,
  "detail": "2 field(s) invalid",
  "errors": [{ "field": "items[0].qty", "message": "must be >= 1" }]
}
```

List endpoint nên có pagination rõ ràng thay vì trả mảng trần:

```json
{ "data": [...], "page": 2, "per_page": 20, "total": 137 }
```

## 7. Bẫy thường gặp

| Triệu chứng | Nguyên nhân | Cách sửa |
|---|---|---|
| `POST` form nhưng server nhận rỗng | thiếu `Content-Type: application/x-www-form-urlencoded` | đặt đúng header, encode body |
| Mất chữ tiếng Việt/emoji | gửi sai charset | luôn UTF-8; JSON mặc định UTF-8 |
| Retry khi timeout tạo 2 đơn hàng | `POST` không idempotent + client retry | dùng `Idempotency-Key` hoặc ID do client sinh |
| 401 khi user đăng nhập rồi vẫn báo lỗi | nhầm 401 (chưa xác thực) vs 403 (không có quyền) | sửa theo bảng mục 4 |
| Body GET bị server bỏ qua im lặng | HTTP không cấm nhưng đa số tool/server bỏ | đưa tham số vào query |
| CORS lỗi trong browser nhưng test tool chạy được | thiếu header `Access-Control-Allow-*` ở server | cấu hình CORS — API tool không bị CORS, browser thì có |

## 8. Thuật ngữ cần nhớ

`start line`/`status line` · `header` · `body` · `payload` · `path parameter` vs `query parameter` · `content negotiation` (`Accept`/`Content-Type`) · `idempotent` · `safe method` · `keep-alive` · `ETag` · `Cache-Control` · `Retry-After` · `error envelope` · `pagination` · `CORS` · `RFC 9457 Problem Details`.

## Nguồn

- RFC 9110 — HTTP Semantics (chuẩn hiện hành): <https://www.rfc-editor.org/rfc/rfc9110>
- MDN — HTTP response status codes: <https://developer.mozilla.org/en-US/docs/Web/HTTP/Status>
- MDN — HTTP headers: <https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers>
- RFC 9457 — Problem Details for HTTP APIs: <https://www.rfc-editor.org/rfc/rfc9457>
