# API Development Learning Notes

Ghi chú lý thuyết cá nhân về API development — viết bằng tiếng Việt, ưu tiên **khái niệm và cơ chế** trước khi code.

Mỗi chủ đề nằm trong một thư mục riêng với một file markdown: `learning/NN-<topic>/<topic>.md`. Số `NN` chính là **thứ tự học đề xuất** — đọc theo thứ tự thư mục là đúng trình tự.

## Phạm vi

| | |
|---|---|
| Ngữ cảnh | Web API qua HTTP — chuẩn bị cho backend thực chiến (Node/Express, Fastify) |
| Trọng tâm | REST/OpenAPI là trục chính; GraphQL/gRPC/OAuth chỉ ở mức khái niệm |
| Cập nhật ghi chú | 2026-11-01 |

## Mục lục

| # | Chủ đề | Nội dung chính |
|---|---|---|
| 01 | [API & REST API](learning/01-api-and-rest-api/api-and-rest-api.md) | API là hợp đồng · lịch sử REST vs RPC · 6 ràng buộc kiến trúc · uniform interface & HATEOAS · ánh xạ CRUD→method · quy tắc thiết kế URI · idempotent · REST vs GraphQL/gRPC/WebSocket |
| 02 | [Request — Response](learning/02-request-response/request-response.md) | cấu trúc 4 phần của request / 3 phần của response · path vs query vs body · bảng status code theo nhóm · headers đáng nhớ (`ETag`, `Idempotency-Key`, `Retry-After`) · error envelope RFC 9457 · các bẫy charset/idempotency/CORS |
| 03 | [API Authentication](learning/03-api-authentication/api-authentication.md) | authentication vs authorization · vì sao token thay session · API key/Basic/JWT · cấu trúc JWT + HS256 vs RS256 · cặp access/refresh · OAuth 2.x & OIDC & PKCE · luật secret & HTTPS |
| 04 | [API Document](learning/04-api-document/api-document.md) | 3 tầng tài liệu · giải phẫu OpenAPI 3.x (`paths`/`components`/`securitySchemes`) · design-first vs code-first · chống drift (lint, contract test) · AsyncAPI/GraphQL SDL · checklist tài liệu tốt |
| 05 | [API Tools & Templates](learning/05-api-tools-and-templates/api-tools-and-templates.md) | bản đồ công cụ theo việc · so sánh manual client (curl/Bruno/Postman/Insomnia) · mẫu collection file-in-repo · template dự án theo framework · mock server (Prism) · Pact/Schemathesis/k6/mitmproxy |

## Thứ tự đọc

```
01. API & REST          →   tư duy resource + từ vựng, đọc trước để hiểu các phần sau
02. Request–Response    →   "hình dạng" của mọi exchange — nền để hiểu status/header/auth
03. Authentication      →   gắn vào header của request (mục 02) — vì sao token, JWT/OAuth đọc cho đúng
04. API Document        →   đặc tả hóa những gì đã học ở 01–03 thành OpenAPI
05. Tools & Templates   →   công cụ + mẫu để làm việc thật với 01–04
```

Theo nhóm nội dung: 01–02 là khối "khái niệm & giao thức", 03 là khối "bảo mật", 04–05 là khối "thực hành & quy trình" (đặc tả hóa và tool hóa).

## Cấu trúc mỗi file

1. **Định nghĩa** — chủ đề là gì, và hay bị nhầm với cái gì.
2. **Cơ chế** — sơ đồ ASCII / bảng tra, không có ví dụ chạy thật.
3. **Bảng cú pháp hoặc so sánh** — kèm chuẩn (RFC/OpenAPI version) khi hành vi thay đổi.
4. **Bẫy thường gặp** — triệu chứng → nguyên nhân → cách sửa.
5. **Thuật ngữ cần nhớ** — để đọc tài liệu chính thức không bị nghẹn.
6. **Nguồn** — link tới docs chính thức.

## Cấu trúc thư mục

```
APIdevLearn/
├── README.md
└── learning/
    ├── 01-api-and-rest-api/api-and-rest-api.md
    ├── 02-request-response/request-response.md
    ├── 03-api-authentication/api-authentication.md
    ├── 04-api-document/api-document.md
    └── 05-api-tools-and-templates/api-tools-and-templates.md
```

## Không phải tài liệu chính thức

Đây là ghi chú học tập cá nhân, có thể sai hoặc lỗi thời theo phiên bản tooling. Khi cần sự thật, hãy đối chiếu:

- MDN Web Docs — HTTP: <https://developer.mozilla.org/en-US/docs/Web/HTTP>
- RFC 9110 — HTTP Semantics: <https://www.rfc-editor.org/rfc/rfc9110>
- OpenAPI Specification: <https://spec.openapis.org/oas/latest.html>
- REST dissertation (Fielding): <https://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm>
- OWASP API Security Top 10: <https://owasp.org/API-Security/editions/2023/en/0x00-header/>
