# API Tools & Templates

> Công cụ để gọi, test, mock, và sinh code từ API; template/mẫu để bắt đầu nhanh và giữ chuẩn.
> Liên quan: [request-response](../02-request-response/request-response.md) · [API document](../04-api-document/api-document.md)

## 1. Bản đồ: công cụ theo việc

| Việc | Công cụ tiêu biểu |
|---|---|
| Gọi thử 1 request (manual client) | **curl**, HTTPie, Bruno, Postman, Insomnia, VS Code REST Client |
|Docs tương tác từ OpenAPI | Swagger UI, Redoc, **Scalar** |
| Soạn/spec-first + mock server | Stoplight Studio, **Prism** (mock từ spec), Postman |
| Test tự động (integration) | REST Assured (Java), Supertest (Node), **pytest + requests/respx**, Dredd, Schemathesis |
| Test hợp đồng giữa 2 team | **Pact** (consumer-driven), Microcosm |
| Load/stress | k6, Locust, JMeter, artillery |
| Debug proxy (nhìn toàn bộ traffic) | **mitmproxy**, Charles, Fiddler, HTTP Toolkit |
| Sinh code từ OpenAPI | **openapi-generator**, `swagger-codegen`, `openapi-typescript`, `orval`, `kiota` |
| Management/lifecycle | Postman Flows, Kong/Gravitee (gateway), Readme.io (docs portal) |

## 2. Manual client — chọn một cái là đủ

| | curl | Bruno | Postman | Insomnia | VS Code REST Client |
|---|---|---|---|---|---|
| Dạng | CLI | app desktop | app + cloud | app desktop | extension |
| Collection lưu | file shell/script | **file `.bru` trong repo** | cloud (proprietary) | trong app/database | `.http` text trong repo |
| Offline được | có | có | hạn chế | có | có |
|Biến môi trường | `-w`, script | có | mạnh | có | có |
| Sinh code (codegen) | lệnh mẫu | — | có | — | — |
| Điểm mạnh | có mặt mọi nơi, dùng trong CI | Git-friendly, nhẹ, không lock-in | share, mock, monitor | UI đẹp, plugin | cạnh code đang viết |

**Khuyến nghị cá nhân**: `curl` để hiểu cơ chế cấp thấp + **Bruno** hoặc `.http` files để lưu collection **cùng repo** (collection là code, review được, không lock-in vào cloud của vendor).

## 3. curl & HTTPie — hai lệnh phủ 80% nhu cầu

```bash
# GET kèm header auth + pretty JSON
curl -s -H "Authorization: Bearer $TOKEN" https://api.shop.example/v1/orders | jq

# POST JSON
curl -si -X POST https://api.shop.example/v1/orders \
  -H "Content-Type: application/json" \
  -d '{"items":[{"sku":"A1","qty":2}]}'

# Xem timing + status (debug)
curl -s -o /dev/null -w "%{http_code} %{time_total}s\n" https://api.example.com/health
```

```bash
# HTTPie: cú pháp ngắn, format sẵn — hợp khi explore
http POST https://api.shop.example/v1/orders items:='[{"sku":"A1","qty":2}]' \
     Authorization:"Bearer $TOKEN"
http GET :8080/health --print=h     # chỉ xem headers
```

## 4. Mẫu request để lưu trong repo (template kiểu Bruno/.http)

```http
### List orders — auth + phân trang
GET {{host}}/v1/orders?page=1&per_page=20
Authorization: Bearer {{$dotenv API_TOKEN}}

> {% client.test("200", () => assert(response.status === 200)); %}

### Create order — ví dụ 400
POST {{host}}/v1/orders
Content-Type: application/json
Authorization: Bearer {{$dotenv API_TOKEN}}

{
  "items": [ { "sku": "A1", "qty": 2 } ]
}
```

Kèm **env template** (`bruno.env` / `.env.example`) — chỉ khai tên biến, **không commit giá trị secret**:

```
API_BASE_URL=https://staging-api.shop.example
API_TOKEN=            # mỗi dev tự điền, lấy từ vault/1Password
```

## 5. Template khởi tạo dự án API

| Ngăn kéo | Khi nào |
|---|---|
| **Fastify + `@fastify/swagger`** (Node) | API nhỏ–vừa; route khai báo schema → tự sinh OpenAPI |
| **NestJS** (Node) | app enterprise, module/DI/IoC sẵn, `@nestjs/swagger` code-first |
| **Express + swagger-jsdoc** (Node) | đơn giản, nhưng spec viết tay dễ drift |
| **FastAPI** (Python) | type hints → OpenAPI chuẩn xác "miễn phí"; kèm Pydantic validation |
| Spring Boot + springdoc | Java ecosystem |
| `.NET minimal API` + Swashbuckle | Microsoft stack |

Chuẩn nên có **trong mọi template** bất kể framework — đây mới là phần "template API dev" thật sự đáng học:

```
project/
├── openapi.yaml          # contract (design-first)
├── src/routes/...        # implement ĐÚNG openapi.yaml
├── tests/contract        # test sinh/đối chiếu từ spec
├── postman|bruno/        # collection chia sẻ cho team
├── .env.example          # không bao giờ có secret thật
└── docs/quickstart.md    # curl chạy được trong 5 phút
```

## 6. Mock server & contract testing

**Mock từ spec** (frontend không chờ backend):

```bash
npx @stoplight/prism-cli mock openapi.yaml -p 4010
# Prism trả response ĐÚNG schema + ví dụ trong spec, validate request client gửi lên
```

**Contract testing (Pact)** — consumer mô tả kỳ vọng, provider chứng minh thỏa mãn trong CI; thay cho "integration test khổng lồ" mà không ai chạy:

```
Team Mobile viết pact (kỳ vọng GET /users/1 → {name}) 
        → CI của Backend replay mọi pact → merge blocker nếu vỡ
```

**Schemathesis**: đọc OpenAPI rồi *fuzz* mọi endpoint với ca lỗi biên — tìm 500 mà test tay bỏ sót.

## 7. Bẫy thường gặp

| Triệu chứng | Nguyên nhân | Cách sửa |
|---|---|---|
| "Chạy trên Postman được, vào app thì 401" | hidden state: cookie/biến môi trường trong app bị quên | export env kèm collection; test bằng curl sạch |
| Collection nằm trong cloud vendor, nhân viên nghỉ là mất | lock-in | dùng format file-in-repo (Bruno/`.http`/collection.json commit) |
| Secret lộ trong screenshot/Postman echo | hardcode token vào collection | biến tham chiếu (`{{$dotenv}}`), masked vars |
| Load test bằng 1 user ảo kết luận "API nhanh" | sai kịch bản | k6: nhiều VU, think time, đo p95/p99 không chỉ avg |
| Docs Swagger UI đỏ lỗi validate |spec sinh từ code chứa rác | lint `spectral` trong CI |

## 8. Thuật ngữ cần nhớ

`manual client` · `collection` · `environment variables` · `pre-request/test script` · `codegen` · `mock server` (`Prism`) · `contract testing` / `consumer-driven contract` (Pact) · `fuzzing` (`Schemathesis`) · `intercepting proxy` (mitmproxy) · `k6/Locust` · `p50/p95/p99` · `design-first tooling` · `lock-in`.

## Nguồn

- curl docs: <https://curl.se/docs/> · HTTPie: <https://httpie.io/docs>
- Bruno: <https://www.usebruno.com/> · Insomnia: <https://insomnia.rest/> · VS Code REST Client: <https://marketplace.visualstudio.com/items?itemName=humao.rest-client>
- Prism (mock from OpenAPI): <https://stoplight.io/open-source/prism> · Pact: <https://docs.pact.io/> · Schemathesis: <https://schemathesis.readthedocs.io/>
- k6: <https://k6.io/docs/> · mitmproxy: <https://docs.mitmproxy.org/>
