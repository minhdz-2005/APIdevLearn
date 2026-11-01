# API Document

> Tài liệu API là *hợp đồng* giữa người xây và người dùng API. Không có nó, mỗi client tích hợp là một cuộc phỏng vấn với tác giả.
> Liên quan: [request-response](../02-request-response/request-response.md) · [tools](../05-api-tools-and-templates/api-tools-and-templates.md)

## 1. Ba tầng tài liệu

| Tầng | Là gì | Sản phẩm |
|---|---|---|
| **Specification** (máy đọc) | mô tả machine-readable đầy đủ endpoint/kiểu dữ liệu | **OpenAPI**, AsyncAPI, GraphQL SDL |
| **Reference doc** (người đọc) | bản render từ spec + giải thích | Swagger UI, Redoc, Scalar |
| **Guide/Tutorial** | luồng "start here", quickstart, use-case | Docusaurus, Markdown site |

Nguyên tắc: **spec là nguồn sự thật**, hai tầng kia dẫn xuất từ spec — đừng viết tay reference doc.

## 2. OpenAPI Specification (OAS)

Chuẩn de-facto cho REST/HTTP (Linux Foundation, kế thừa Swagger). Phiên bản: 3.0.x (rộng rãi nhất về tooling) và 3.1 (align với JSON Schema chính xác hơn).

Cấu trúc một file `openapi.yaml`:

```yaml
openapi: 3.1.0
info:
  title: Shop API
  version: 1.4.0            # theo SemVer của API, không phải của app
servers:
  - url: https://api.shop.example/v1
  - url: https://staging-api.shop.example/v1
paths:
  /orders:
    post:                   # operation
      operationId: createOrder
      tags: [Orders]
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/OrderInput' }
      responses:
        '201':
          description: Created
          headers:
            Location: { schema: { type: string } }
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Order' }
        '400': { $ref: '#/components/responses/ValidationError' }
components:
  securitySchemes:
    bearerAuth: { type: http, scheme: bearer, bearerFormat: JWT }
  schemas:
    Order:
      type: object
      required: [id, status]
      properties:
        id: { type: integer, examples: [1071] }
        status: { type: string, enum: [pending, paid, shipped] }
security:
  - bearerAuth: []
```

| Khối | Vai trò |
|---|---|
| `info` / `servers` | danh tính + các môi trường |
| `paths` → operation | endpoint + method + request/response + status codes |
| `components.schemas` | **data model** tái sử dụng (`$ref`) — phần giá trị nhất |
| `components.securitySchemes` + `security` | khai báo auth — khớp [auth](../03-api-authentication/api-authentication.md) đã cài thật |
| `operationId` | tên máy đọc — codegen đặt tên hàm theo nó |

## 3. Design-first vs Code-first

| | Design-first (spec → code) | Code-first (code → spec) |
|---|---|---|
| Workflow | review YAML trước khi implement | annotate controller, tool sinh spec (swagger-jsdoc, Swashbuckle, fastapi) |
| Chất lượng API | cao: contract chốt trước, frontend không chờ backend | dễ trôi: spec phản ánh bug của code |
| Rủi ro drift | vẫn drift nếu không enforce | spec ít được ai thiết kế |
| Phù hợp | team nhiều bên, API công khai | team nhỏ, iterate nhanh |

Kháng drift: lint spec (`spectral`), test tự động từ spec (contract testing — Pact), CI so spec với behavior thật.

## 4. Ngoài OpenAPI

| Chuẩn | Khi nào |
|---|---|
| **AsyncAPI** | event-driven: Kafka/websocket, "OpenAPI cho message" |
| **GraphQL SDL** (`type Query {...}`) | API có schema là sự thật sẵn — docs render từ introspection |
| **API Blueprint (APIs.json)** | đã gần như chết, gặp ở hệ legacy |
| Postman Collection v2.1 | bản thân collection là tài liệu chạy được — xem [tools](../05-api-tools-and-templates/api-tools-and-templates.md) |

## 5. Một tài liệu API tốt có gì (checklist)

- ☐ Every endpoint: method, path, **tất cả** status codes (đặc biệt các 4xx hay gặp)
- ☐ Data model với **kiểu, bắt buộc, enum, range** — không chỉ "object"
- ☐ Ví dụ request + response **thật** (`examples`), kể cả ví dụ lỗi
- ☐ Authentication: cách lấy token, header nào, scope nào cho endpoint nào
- ☐ Pagination / filtering / sorting: tham số + cách đọc tổng số trang
- ☐ Rate limit: con số, header trả về, hành xử khi vượt
- ☐ Versioning policy + deprecation timeline (thông báo `Deprecation`/`Sunset` header)
- ☐ Quickstart copy-paste chạy được (curl + 1 ngôn ngữ)
- ☐ Change log giữa các version

## 6. Bẫy thường gặp

| Triệu chứng | Nguyên nhân | Cách sửa |
|---|---|---|
| Docs nói một đằng, API làm một nẻo | viết docs tay, sửa code không sửa docs | design-first + CI validate từ spec |
| Client hỏi "400 này nghĩa là gì" | chỉ khai báo `200` trong spec | liệt kê 4xx/5xx + error schema |
| 2 team hiểu `status` khác nhau | không ghi enum/đơn vị | schema chặt: enum, `format: date-time`, đơn vị tiền |
| Swagger UI sinh ra không ai đọc | thiếu guide, không có ví dụ | thêm `examples` + trang quickstart riêng |
| Đổi field thành `nullable` làm client crash | coi docs chỉ là "tài liệu tham khảo" | contract = hợp đồng; thay đổi lớn = version mới |

## 7. Thuật ngữ cần nhớ

`OpenAPI Specification` · `operation` / `operationId` · `schema` / `$ref` / `components` · `security scheme` · `example` vs `schema` · `Swagger` (công cụ) vs `OpenAPI` (chuẩn) · `Redoc` / `Scalar` · `design-first` / `code-first` · `contract testing` · `Spectral` (lint) · `AsyncAPI` · `introspection` (GraphQL) · `deprecation policy`.

## Nguồn

- OpenAPI Specification 3.1: <https://spec.openapis.org/oas/latest.html>
- OpenAPI Guide & mẫu: <https://www.openapis.org/>
- Swagger Editor (thử viết spec online): <https://editor.swagger.io/>
- AsyncAPI: <https://www.asyncapi.com/docs> · Redoc style guide: <https://redocly.com/redoc-style-guide/>
