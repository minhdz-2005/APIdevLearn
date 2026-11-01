# API & REST API

> Nền tảng: API là gì, REST là tập nguyên tắc chứ không phải giao thức, và một "RESTful API" được thiết kế quanh tài nguyên.
> Liên quan: [request-response](../02-request-response/request-response.md) · [authentication](../03-api-authentication/api-authentication.md) · [API document](../04-api-document/api-document.md)

## 1. API là gì — định nghĩa tối thiểu

**API (Application Programming Interface)**: một *hợp đồng* cho phép hai phần mềm nói chuyện với nhau mà không cần biết nội bộ nhau.

| Tầng | Ví dụ | "Ngôn ngữ" |
|---|---|---|
| API hệ điều hành | `read()`, `CreateFile()` | syscall, theo từng máy |
| API thư viện | `Array.map()`, `requests.get()` | hàm/object trong cùng process |
| **Web API** | `GET https://api.example.com/users` | HTTP + dữ liệu (thường JSON) |

Ghi chú này nói về **Web API**: API qua mạng, client và server là hai process tách biệt.

## 2. Lịch sử ngắn: vì sao REST sinh ra

| Thời kỳ | Cách làm | Vấn đề |
|---|---|---|
| 1990s | RPC-style: URL là *động từ* (`/getUser?id=5`, `/deleteOrder`) | mỗi thao tác một endpoint, khó mở rộng |
| 2000 | **REST** — Roy Fielding định nghĩa trong luận án tiến sĩ (kiến trúc, không phải chuẩn) | — |
| 2000s | SOAP/XML nặng nề → JSON thay thế XML ở đa số API công khai | — |
| Nay | REST phổ biến nhất; GraphQL, gRPC, WebSocket cho các nhu cầu riêng | — |

RPC vẫn hợp lệ (gRPC chính là RPC hiện đại). "REST thay RPC" là xu hướng, không là chân lý.

## 3. REST: 6 ràng buộc kiến trúc

Fielding định nghĩa REST bằng 6 ràng buộc; thiếu ràng buộc 1–5 thì **không phải REST**, dù người ta vẫn gọi bừa:

| # | Ràng buộc | Ý nghĩa thực tế |
|---|---|---|
| 1 | **Client–Server** | tách trách nhiệm: server quản lý dữ liệu, client quản lý giao diện |
| 2 | **Stateless** | mỗi request chứa đủ thông tin; server **không** nhớ session giữa hai request |
| 3 | **Cacheable** | response phải đánh dấu được cache hay không (qua header) |
| 4 | **Layered System** | client không biết mình nói chuyện trực tiếp với server hay qua proxy/gateway |
| 5 | **Uniform Interface** | *trái tim của REST* — xem mục 4 |
| 6 | **Code on Demand** (tùy chọn) | server có thể gửi code về client (hiếm dùng) |

## 4. Uniform Interface — dịch ra thực hành

Bốn quy ước tạo nên "giao diện thống nhất":

1. **Resources**: mọi thứ là *danh từ* xác định bằng URI — `/users`, `/users/42`, `/orders/7/items`. Không có `/getAllActiveUsersAndSortThem`.
2. **Representations**: client nhận JSON/XML... mô tả tài nguyên, kèm `Content-Type`; không nhận object nội bộ của server.
3. **Self-descriptive messages**: bản thân message đủ để hiểu cách xử lý (method + status + header) — xem [request-response](../02-request-response/request-response.md).
4. **HATEOAS** (Hypermedia As The Engine Of Application State): response chứa link chỉ dẫn bước tiếp theo. **Đây là phần bị bỏ qua nhiều nhất** — hầu hết API "REST" thực tế chỉ đạt mức 1–3 (người ta gọi vui là "REST-ish").

## 5. Ánh xạ CRUD → HTTP Method

| Operation | Method | URI | Idempotent? |
|---|---|---|---|
| Create | `POST` | `/users` | Không |
| Read | `GET` | `/users`, `/users/42` | Có |
| Update (full) | `PUT` | `/users/42` | Có |
| Update (partial) | `PATCH` | `/users/42` | Không bắt buộc |
| Delete | `DELETE` | `/users/42` | Có |

**Idempotent** = gọi N lần cho kết quả cuối cùng như gọi 1 lần. `POST /orders` 3 lần → 3 đơn hàng; `DELETE /orders/7` 3 lần → vẫn chỉ một lần xóa thật.

## 6. URI design: quy ước thực hành

| Quy ước | Đúng | Sai | Vì sao |
|---|---|---|---|
| Danh từ, số nhiều | `/users` | `/getUser` | method đã là động từ |
| Quan hệ bằng nesting | `/users/42/orders` | `/getOrdersOfUser42` | đọc được như sơ đồ |
| Số ít cho collection con | `/users/42/orders` | `/users/42/orders/` | trailing slash gây nhiễu cache |
| Version trong path/header | `/v1/users` | `/version1.2.3-final/users` | dễ migration |
| Query cho filter/phân trang | `/users?city=Hanoi&page=2` | `/users/city/Hanoi` | không phải tài nguyên |
| Kebab-case | `/order-items` | `/orderItems`, `/order_items` | URL chuẩn là lowercase |

## 7. REST vs các lựa chọn khác

| | REST (HTTP+JSON) | GraphQL | gRPC | WebSocket |
|---|---|---|---|---|
|granularity | server định nghĩa endpoint | **client** hỏi đúng field cần | message có schema | 2 chiều push |
| Nhiều tài nguyên 1 round-trip | nhiều request | 1 query | stream | — |
| Over-fetching (dư dữ liệu) | hay gặp | **không gặp** | không (schema chặt) | — |
| Cache HTTP tự nhiên | mạnh | yếu (chung 1 endpoint) | không | không |
| Phù hợp | CRUD chuẩn, API công khai | app nhiều màn hình dữ liệu khác nhau | service-to-service hiệu năng cao | chat, realtime, bidding |

## 8. Bẫy thường gặp

| Triệu chứng | Nguyên nhân | Cách sửa |
|---|---|---|
| API đầy `/getUser`, `/postUser` | đang viết RPC đội vỏ REST | chuyển sang resource + method |
| `POST` làm luôn việc của `PUT` | không phân biệt idempotent | client sinh ID hoặc dùng `PUT /users/{id}` |
| Server "nhớ" state giữa 2 request | vi phạm stateless | đưa state vào client (token) hoặc store có khóa, không vào session RAM |
| Lỗi trả về `200` kèm `{success: false}` | status code bị bỏ | dùng 4xx/5xx đúng nghĩa — xem [request-response](../02-request-response/request-response.md) |
| Mọi thứ là một mega-endpoint trả toàn bộ object | over-fetching | tách resource, dùng query param/sparselist |

## 9. Thuật ngữ cần nhớ

`API` · `Web API` · `endpoint` · `resource` · `representation` · `stateless` · `idempotent` · `safe method` · `HATEOAS` · `uniform interface` · `URI` vs `URL` · `collection` vs `singleton` · `versioning` · `over-fetching`/`under-fetching` · `RPC` · `GraphQL` · `gRPC`.

## Nguồn

- R. Fielding, *Architectural Styles and the Design of Network-based Software Architectures* (luận án 2000, khai sinh REST): <https://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm>
- MDN — HTTP request methods: <https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods>
- Microsoft — REST API requirements and considerations: <https://learn.microsoft.com/en-us/azure/architecture/best-practices/api-design>
- GraphQL spec: <https://spec.graphql.org/> · gRPC: <https://grpc.io/docs/>
