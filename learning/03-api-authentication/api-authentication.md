# API Authentication

> Xác thực trong API gần như luôn là **token**, không phải session cookie như web truyền thống — vì REST yêu cầu stateless.
> Liên quan: [request-response](../02-request-response/request-response.md) · [API document](../04-api-document/api-document.md) · [tools](../05-api-tools-and-templates/api-tools-and-templates.md)

## 1. Authentication ≠ Authorization

| | Trả lời câu hỏi | Ví dụ lỗi |
|---|---|---|
| **Authentication** (xác thực) | *Bạn là ai?* | thiếu → `401` |
| **Authorization** (phân quyền) | *Bạn được làm gì?* | sai → `403` |

Sai lầm phổ biến: thiết kế "role" trong lúc chọn loại token. Token chỉ chứng minh danh tính; **quyền** phải kiểm ở server theo từng endpoint.

## 2. Vì sao API dùng token thay session cookie

| | Session + Cookie (web) | Token (API) |
|---|---|---|
| State server | có (RAM/DB phiên) | **không** — đúng ràng buộc stateless |
| Lưu | cookie tự động gửi | header `Authorization` — client tự attach |
| Multi-client (app, CLI, server-to-server) | khó (cookie là của browser) | dễ |
| CSRF | phải chống | gần như miễn nhiễm (không tự động gửi) |
| Scaling | cần sticky session/share store | vô tư |

## 3. Các cơ chế chính, xếp theo mức "nên dùng hôm nay"

| Cơ chế | Dành cho | Tuổi thọ | Ghi chú |
|---|---|---|---|
| **API Key** | server-to-server, đơn giản | dài hạn | một secret duy nhất — lộ là mất cả tài khoản |
| **Basic** (`Authorization: Basic base64(u:p)`) | test nhanh, nội bộ | theo request | base64 **không phải mã hóa**; chỉ chấp nhận được kèm HTTPS |
| **Session cookie** | browser app cùng origin | theo session | phá vỡ stateless nếu server giữ state |
| **JWT (Bearer token)** | chuẩn de-facto API hiện đại | ngắn (5–15 phút) | xem mục 4 |
| **OAuth 2.0 / OIDC** | ủy quyền bên thứ 3, hệ nhiều client | access ngắn + refresh dài | xem mục 5 |
| **mTLS / signature (HMAC)** | fintech, service mesh | phiên kết nối | mỗi request ký bằng secret (AWS SigV4) |

## 4. JWT (JSON Web Token)

Cấu trúc: `header.payload.signature` — 3 phần base64url, ví dụ `eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiIxMjM0In0.dBjgtQ...`

```
Header    {"alg":"RS256","typ":"JWT"}
Payload   {"sub":"user:1071","role":"admin","exp":1730500000,"iat":...}   ← chỉ là base64, AI cũng đọc được!
Signature sign(header + payload, secret/key)                              ← chống sửa đổi
```

| Hiểu đúng | Hệ quả |
|---|---|
| JWT **không mã hóa**, chỉ ký | **không nhét** số điện thoại, email nhạy cảm vào payload |
| Server không giữ state → không "thu hồi" được giữa chừng | access token phải **ngắn hạn**; muốn kick-out thì cần denylist hoặc chờ hết hạn |
| `alg` có thể bị hạ cấp (`none`, HMAC-vs-RSA nhầm) | server **phải chốt algorithm** khi verify |
| Ký đối xứng (HS256, 1 secret) vs bất đối xứng (RS256, keypair) | RS256 cho bên thứ 3 verify được mà không cầm secret (microservices) |

Cặp Access + Refresh:

```
client ──login──► auth server ──► access (JWT, 15') + refresh (opaque, 30 ngày)
client mỗi request: Authorization: Bearer <access>
access hết hạn ──► POST /token/refresh (refresh) ──► access mới
refresh lộ? ──► rotate refresh token + phát hiện reuse
```

## 5. OAuth 2.0 — và một hiểu lầm tai hại

OAuth 2.0 là chuẩn **ủy quyền** (authorization), không phải xác thực. "Đăng nhập bằng Google" thực chất là **OIDC** (OpenID Connect) — lớp danh tính xây trên OAuth 2.0.

Các **flow** chính (OAuth 2.1 hợp nhất khuyến nghị):

| Flow | Ai dùng | Đặc điểm |
|---|---|---|
| **Authorization Code + PKCE** | web app, mobile, SPA | chuẩn hiện đại; `PKCE` chống đánh chặn code; token không qua browser |
| Client Credentials | server-to-server | app gửi `client_id` + `client_secret` lấy token |
| Implicit | (cũ) | bỏ — token lộ trên URL; **không dùng** |
| Resource Owner Password | (cũ) | bỏ — đòi password người dùng |

```
[Authorization Code + PKCE]
1. app sinh code_verifier → challenge = hash(verifier)
2. redirect user → /authorize?client_id=..&code_challenge=..&state=..
3. user đồng ý → redirect về với ?code=..
4. app đổi code: POST /token (code + code_verifier + client_secret nếu confidential)
5. nhận access + refresh token
```

## 6. Secret & truyền dẫn — luật bất biến

1. **HTTPS everywhere.** Token/Basic gửi qua HTTP không mã hóa = cho không. HSTS nên bật.
2. **Không** nhét secret vào query string (rơi vào access log, browser history, Referer).
3. **Không** commit API key vào repo; **không** đưa secret vào app mobile (bundle có thể bị dịch ngược) → mobile dùng token ngắn hạn + PKCE.
4. Storage phía client: `localStorage` dính XSS → cân nhắc cookie `HttpOnly; Secure; SameSite=Lax` cho refresh token.
5. Rotate được key mà không downtime = thiết kế 2 key cùng lúc hợp lệ.

## 7. Bẫy thường gặp

| Triệu chứng | Nguyên nhân | Cách sửa |
|---|---|---|
| App mobile chứa `client_secret` | hiểu nhầm confidential client | mobile là **public client** → PKCE, không secret |
| Logout mà token vẫn dùng được | JWT stateless không thu hồi | token ngắn + refresh revocation / denylist |
| `401` loop khi token hết hạn | client không refresh đúng cách | bắt 401 → thử refresh → retry 1 lần |
| Nhiều service verify JWT bằng chung HS256 secret | secret lan truyền | RS256 + public key (JWKS endpoint) |
| "Đăng nhập Google" tưởng là OAuth | thiếu lớp danh tính | phân biệt OAuth (ủy quyền) vs OIDC (`id_token`) |

## 8. Thuật ngữ cần nhớ

`authentication` vs `authorization` · `credential` · `API key` · `Bearer token` · `JWT` (header/payload/claims, `exp`/`iat`/`sub`/`aud`) · `HS256` vs `RS256` · `JWKS` · `access token` vs `refresh token` · `opaque token` · `OAuth 2.x` · `OIDC` / `id_token` · `PKCE` · `scope` · `confidential` vs `public client` · `rotation` · `HSTS` · `mTLS` · `HMAC signature`.

## Nguồn

- RFC 7519 — JWT: <https://www.rfc-editor.org/rfc/rfc7519>
- RFC 6749 — OAuth 2.0: <https://www.rfc-editor.org/rfc/rfc6749> · OAuth 2.1 draft: <https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1>
- RFC 7636 — PKCE: <https://www.rfc-editor.org/rfc/rfc7636> · OIDC Core: <https://openid.net/specs/openid-connect-core-1_0.html>
- RFC 6750 — Bearer Token Usage: <https://www.rfc-editor.org/rfc/rfc6750>
- OWASP — Authentication Cheat Sheet: <https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html>
