# [BE-2] REST API với Gin behind Kong

> Module **BE-2** · Xây HTTP adapter layer của hms-api bằng Gin, đứng sau Kong, trust-but-verify identity · Độ khó: 🥉→🥇 · Prereqs: **BE-1** (Go production-grade, composition root, `internal/shared/{errors,config,httpx}`)

Liên quan: `doc/07-api-specification.md` (response envelope, Idempotency-Key contract), `doc/09-security.md` (Kong edge vs Go object-level authz, CVE-2026-29413), `doc_tech/architecture/01-clean-architecture.md` (ARCH-1, layer rule), `doc_tech/backend-go/06-api-design-openapi.md` (BE-6, OpenAPI là nguồn FE codegen).

---

## 1. Vì sao kỹ năng này quan trọng trong HMS

HTTP adapter là **mặt tiền duy nhất** mà tiếp đón, bác sĩ, dược sĩ, thu ngân, giám định chạm vào hms-api. Mỗi request là một thao tác từng-làm-trên-giấy: check-in BHYT, ký EMR, kê đơn kích hoạt CDSS, cấp phát FEFO, sinh XML 4750. Nếu tầng HTTP làm sai, ba thứ vỡ ngay:

- **An toàn người bệnh**: handler để command đi qua mà không validate → order/dispense rác xuống aggregate, hoặc bỏ qua CDSS hard-stop (ADR-008). Validate ở boundary là lớp phòng thủ đầu tiên.
- **Cô lập PHI**: handler phải mở **một tx có `SET LOCAL app.current_branch`** trước mọi PHI query — nếu để app/domain tự mở connection từ pool, RLS revert về no-filter và leak chi nhánh (ADR-003, ADR-005, open risk critical #2).
- **Identity không giả mạo**: Kong đứng trước nhưng **Kong KHÔNG quyết object-level authz** (ADR-013). Gin handler phải tự verify JWT, không mù tin header `X-Userinfo` — đây chính là backstop chống CVE-2026-29413 (Kong auth-bypass, CISA-KEV, đã bị khai thác trên healthcare gateway).

Gin được pin làm HTTP framework của modular monolith (ADR-001, mục 3 canon: *Gin (HTTP behind Kong) + pgx/v5 + sqlc*). Đây không phải "học framework" — đây là học cách dựng **một biên giới fail-closed** cho hệ thống PHI.

---

## 2. Mô hình tư duy (first principles) — từ con số 0

Một HTTP request đi vào hms-api là một chuỗi biến đổi có thể suy ra từ nguyên lý, không cần thuộc lòng API Gin:

```
[ TCP/TLS ] → Kong (edge: TLS terminate, JWT verify JWKS, rate-limit, inject identity)
   → [ Go net/http Server ] → Gin Engine → middleware chain → handler
      → decode body → validate → map sang Command/Query (DTO, KHÔNG phải domain type)
      → app service (use case) → domain → ports → adapters(pgx tx + RLS)
      → map kết quả/err → response envelope → ghi status + body
```

Bốn chân lý nền:

1. **HTTP là cơ chế vận chuyển, không phải nơi chứa business logic.** Handler chỉ làm: parse → validate → ủy quyền cho `app/` → map kết quả. Mọi quyết định nghiệp vụ nằm trong `domain/`+`app/` (layer rule ARCH-1: `adapters → ports ← app → domain`). Handler thuộc `adapters/http`, **không được import BC khác**.
2. **Mọi dữ liệu vào từ ngoài là không tin cậy** — kể cả identity header từ Kong. Verify, đừng tin.
3. **Lỗi là dữ liệu**: domain error phải map xác định sang HTTP status + envelope, không panic, không rò chi tiết nội bộ.
4. **Request có vòng đời và ngân sách**: `context.Context` mang deadline, cancellation, request-id, identity, branch_id — chảy xuyên mọi tầng tới pgx.

---

## 3. Khái niệm cốt lõi (tăng dần độ khó)

| # | Khái niệm | Trong HMS |
|---|-----------|-----------|
| 1 | `gin.Engine`, `RouterGroup`, route versioning | `/api/v1/<bc>/...`, group theo BC |
| 2 | `gin.HandlerFunc` & middleware chain | request-id → recover → identity → audit → handler |
| 3 | Binding & validation (`ShouldBindJSON` + `go-playground/validator`) | DTO có tag `binding:"required,..."`, fail-fast 422 |
| 4 | Response envelope nhất quán | `{success, data, error, meta}` (patterns.md) |
| 5 | Error mapping tập trung | domain error → HTTP status, không leak |
| 6 | Context propagation | `c.Request.Context()` → app → pgx tx |
| 7 | Identity middleware trust-but-verify | parse+verify JWT độc lập (ADR-013) |
| 8 | RLS tx middleware | mở tx + `SET LOCAL app.current_branch` |
| 9 | Idempotency-Key middleware | charge/dispense double-post guard (ADR-011) |
| 10 | Graceful shutdown, timeouts, `/livez` `/readyz` | K8s probe (ADR-deploy) |

Độ khó tăng dần: 🥉 route+envelope → 🥈 validation+error mapping+context → 🥇 identity-verify + RLS-tx + idempotency fail-closed.

---

## 4. HMS dùng nó thế nào (bám code path — *(planned)*, code chưa viết)

Layout neo (canon §9). Mọi đường dẫn dưới đây là **(planned)**:

```
backend/
├── cmd/hms-api/main.go                     # composition root DUY NHẤT: build Engine, wire middleware + handler
├── internal/shared/
│   ├── httpx/        # envelope, error-mapper, binder helper, request-id      (planned)
│   ├── middleware/   # identity, rls-tx, idempotency, audit, recover, reqid   (planned)
│   ├── auth/         # JWT verify qua JWKS Keycloak (reject alg=none)         (planned)
│   ├── errors/       # AppError sentinel + code → HTTP status                 (planned)
│   └── rls/          # WithBranchTx(ctx, branchID, fn)                        (planned)
└── internal/<bc>/adapters/http/            # handler MỎNG mỗi BC               (planned)
    ├── encounter/adapters/http/encounter_handler.go
    ├── pharmacy/adapters/http/prescription_handler.go
    └── billing/adapters/http/charge_handler.go
```

**Composition root** `cmd/hms-api/main.go` *(planned)* — nơi DUY NHẤT wiring; handler nhận app-service qua interface (port), không tự `new` dependency:

```go
// cmd/hms-api/main.go (planned)
r := gin.New()
r.Use(middleware.RequestID(), middleware.Recover(), middleware.AccessLog())
r.GET("/livez", httpx.Live); r.GET("/readyz", httpx.Ready(db)) // KHÔNG qua auth

api := r.Group("/api/v1")
api.Use(middleware.Identity(jwks))        // verify JWT độc lập (ADR-013) — KHÔNG mù tin X-Userinfo
api.Use(middleware.RLSTx(pool))           // mở pgx.Tx + SET LOCAL app.current_branch (ADR-003/005)
api.Use(middleware.AuditRead(auditSvc))   // audit-of-reads fail-closed (ADR-009)

enc := encounterhttp.New(encounterApp)    // app-service inject qua port
enc.Register(api.Group("/encounters"))
```

**Handler mỏng** *(planned)* — chỉ parse → validate → gọi app → map. KHÔNG có business logic:

```go
// internal/pharmacy/adapters/http/prescription_handler.go (planned)
func (h *Handler) CreatePrescription(c *gin.Context) {
    var req CreatePrescriptionRequest                       // DTO của adapter, KHÔNG phải domain type
    if err := c.ShouldBindJSON(&req); err != nil {
        httpx.WriteValidationError(c, err); return          // 422 + field errors
    }
    cmd := req.ToCommand(httpx.Identity(c))                 // map DTO → Command
    res, err := h.app.PrescribeMedication(c.Request.Context(), cmd) // ctx mang tx+branch+identity
    if err != nil {
        httpx.WriteError(c, err); return                    // CDSS hard-stop → 409, allergy-unknown rõ ràng
    }
    httpx.WriteCreated(c, res)                              // envelope {success,data}
}
```

**Trust-but-verify identity** *(planned)* — backstop CVE-2026-29413 (open risk high):

```go
// internal/shared/middleware/identity.go (planned)
func Identity(jwks auth.KeySet) gin.HandlerFunc {
    return func(c *gin.Context) {
        raw := auth.BearerFromCookieOrHeader(c)             // Kong BFF set HttpOnly cookie
        claims, err := jwks.Verify(raw)                     // verify signature+iss+aud+exp; reject alg=none
        if err != nil { httpx.WriteError(c, errors.ErrUnauthenticated); c.Abort(); return }
        c.Set(httpx.CtxIdentity, auth.Identity{
            AccountID: claims.Sub, BranchID: claims.BranchID, Roles: claims.Roles,
        })                                                  // KHÔNG đọc branch_id từ X-Userinfo/body
        c.Next()
    }
}
```

**RLS-tx middleware** *(planned)* — invariant chống open-risk critical #2 (SET LOCAL chỉ sống trong tx):

```go
// internal/shared/middleware/rls_tx.go (planned)
func RLSTx(pool *pgxpool.Pool) gin.HandlerFunc {
    return func(c *gin.Context) {
        id := httpx.Identity(c)
        rls.WithBranchTx(c.Request.Context(), pool, id.BranchID, func(ctx context.Context) error {
            c.Request = c.Request.WithContext(ctx)          // tx + GUC nằm trong ctx
            c.Next()                                        // MỌI PHI query phải dùng ctx này
            return httpx.CommitDecision(c)                  // commit nếu 2xx, rollback nếu lỗi
        })
    }
}
// rls.WithBranchTx chạy: SET LOCAL app.current_branch = $1 trong tx; pgx pool reuse conn nên KHÔNG SET ngoài tx
```

Object-level/ABAC authz (role+branch+quan-hệ-điều-trị+minimum-necessary) enforce trong `app/`, **không** ở Gin middleware (ADR-013) — Gin chỉ chặn authn + coarse role guard.

---

## 5. Best practices (mỗi mục kèm 1 nguồn)

1. **Dùng `gin.New()` + middleware tường minh, KHÔNG `gin.Default()`** — `Default()` gắn Logger ghi mọi request ra stdout (có thể lộ query string chứa định danh) và Recovery mặc định; ta cần access-log có cấu trúc + recover map về envelope. Nguồn: [Gin docs — Using middleware](https://gin-gonic.com/docs/examples/using-middleware/).
2. **Validate ở boundary bằng `go-playground/validator` qua tag `binding`, fail-fast 422** — không để dữ liệu rác chạm aggregate. Nguồn: [go-playground/validator README](https://github.com/go-playground/validator).
3. **Set `ReadHeaderTimeout`/`ReadTimeout`/`WriteTimeout` trên `http.Server`** — chống slow-loris và treo goroutine; mặc định Go là vô hạn. Nguồn: [Cloudflare — The complete guide to Go net/http timeouts](https://blog.cloudflare.com/the-complete-guide-to-golang-net-http-timeouts/).
4. **Graceful shutdown bằng `srv.Shutdown(ctx)` khi nhận SIGTERM** — để K8s rolling deploy (maxUnavailable=0) không cắt request đang ký EMR/charge. Nguồn: [Gin docs — Graceful shutdown/restart](https://gin-gonic.com/docs/examples/graceful-restart-or-stop/).
5. **Luôn propagate `c.Request.Context()` xuống pgx/app; không dùng `context.Background()` trong handler** — để deadline/cancellation và tx+branch GUC chảy đúng. Nguồn: [Go Blog — Contexts and structs](https://go.dev/blog/context-and-structs).
6. **Phòng thủ ở Go là backstop, không tin gateway một mình** — verify token độc lập, authz object-level ở app. Nguồn: [OWASP API Security Top 10 — API1:2023 Broken Object Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/).
7. **Idempotency-Key cho mọi mutation non-idempotent (charge/dispense)** — header + unique-constraint persist, trả lại response cũ khi key trùng. Nguồn: [IETF draft — The Idempotency-Key HTTP Header Field](https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/).

---

## 6. Lỗi thường gặp & anti-patterns

- **Business logic trong handler.** Tính BHYT co-pay, gọi CDSS, build XML ngay trong Gin handler → không test được, vi phạm layer rule. Đúng: handler ủy quyền cho `app/`.
- **Mù tin `X-Userinfo` / lấy `branch_id` từ body hoặc query.** Đây chính là vector CVE-2026-29413 và tenant-spoofing (ADR-005). Branch_id CHỈ từ JWT đã verify.
- **PHI query ngoài tx đã `SET LOCAL`.** Gọi repository với `context.Background()` hay pool trực tiếp → pgx tái dùng connection, GUC revert, RLS no-op, leak branch (open-risk critical). Đúng: mọi PHI query đi qua ctx của `RLSTx`.
- **`gin.Default()` ở production** — logger lộ định danh trong URL, không structured. Dùng `gin.New()`.
- **Trả lỗi không nhất quán** — chỗ trả string, chỗ trả JSON khác nhau, leak stacktrace/SQL. Đúng: một `WriteError` map sentinel → status, log chi tiết server-side, trả message an toàn (coding-style.md error handling).
- **`c.JSON(200, ...)` rồi vẫn `c.Next()`** hoặc quên `c.Abort()` sau khi reject auth → handler vẫn chạy. Luôn `Abort()` khi short-circuit.
- **Audit-read best-effort async.** Trả PHI rồi mới log → mất audit khi crash (vi phạm ADR-009). Audit phải commit-with-response, fail-closed.
- **Bind vào domain struct trực tiếp.** Lộ field nội bộ, mass-assignment. Đúng: DTO riêng ở adapter + `ToCommand()`.

---

## 7. Lộ trình luyện tập NGAY trong repo

> Repo chưa có code — các bài tập là dựng *(planned)* layout theo §9 rồi tự kiểm bằng test (ADR-025: testcontainers cho RLS/idempotency).

- 🥉 **Cơ bản** — Trong `internal/shared/httpx/` *(planned)* viết `WriteSuccess`/`WriteCreated`/`WriteError`/`WriteValidationError` theo envelope `{success,data,error,meta}` (patterns.md). Thêm `GET /livez`, `GET /readyz` vào `cmd/hms-api/main.go`. Viết unit test: lỗi validation → 422 + danh sách field error.
- 🥈 **Trung cấp** — Dựng `internal/organization/adapters/http/branch_handler.go` *(planned)* với `ShouldBindJSON` + tag `binding`, map 3 domain sentinel error → 404/409/422 qua một `WriteError` tập trung. Thêm `middleware.Identity` đọc claim giả-lập từ header test, set vào `gin.Context`, viết test reject `alg=none` và missing token → 401.
- 🥇 **Nâng cao** — Implement `middleware.RLSTx` + `rls.WithBranchTx` *(planned)* và `middleware.Idempotency`. Viết **integration test với testcontainers-go** (real Postgres): (a) request với `branch=A` KHÔNG thấy row của `branch=B` (merge-blocking gate, ADR-003); (b) replay cùng `Idempotency-Key` trên endpoint charge chỉ tạo 1 ChargeItem (ADR-011). Cả hai phải xanh trước khi coi handler "xong".

---

## 8. Skill/agent ECC nên dùng khi luyện

- **`ecc:go-review`** — review handler/middleware: idiomatic Go, error handling, context propagation, không leak. Chạy ngay sau khi viết (code-review.md trigger).
- **`ecc:go-test`** — ép TDD table-driven cho validation/error-mapping; verify coverage ≥80% (testing.md).
- **`ecc:security-review`** / **`ecc:security-scan`** — soi trust-but-verify identity, CVE-2026-29413, BOLA/IDOR ở object-level (security.md, ADR-013).
- **`ecc:golang-patterns`** — tham chiếu pattern Gin/net/http production.
- **`ecc:api-design`** — kiểm envelope, status code, pagination, idempotency contract (đồng bộ với BE-6/`doc/07`).
- **`ecc:postgres-patterns`** — khi viết RLS-tx middleware, kiểm `SET LOCAL` + pool reuse invariant.

---

## 9. Tài nguyên học thêm (2024–2026)

- Gin chính thức — [gin-gonic.com/docs](https://gin-gonic.com/docs/) (middleware, binding, graceful shutdown).
- go-playground/validator — [github.com/go-playground/validator](https://github.com/go-playground/validator) (tag reference, custom validator).
- Go Blog — [Contexts and structs](https://go.dev/blog/context-and-structs) & [pkg.go.dev/context](https://pkg.go.dev/context).
- Cloudflare — [The complete guide to Go net/http timeouts](https://blog.cloudflare.com/the-complete-guide-to-golang-net-http-timeouts/).
- OWASP API Security Top 10 (2023) — [owasp.org/API-Security](https://owasp.org/API-Security/editions/2023/en/0x11-t10/) (BOLA, broken auth).
- IETF — [Idempotency-Key HTTP Header Field draft](https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/).
- testcontainers-go — [golang.testcontainers.org](https://golang.testcontainers.org/) (integration test real Postgres, ADR-025).

---

## 10. Checklist "đã hiểu"

- [ ] Tôi giải thích được vì sao handler thuộc `adapters/http` và **không chứa business logic** (layer rule ARCH-1).
- [ ] Tôi dùng `gin.New()` + middleware tường minh, biết vì sao tránh `gin.Default()` ở production.
- [ ] Tôi validate body bằng `ShouldBindJSON` + tag `binding`, trả 422 với field errors qua envelope chung.
- [ ] Tôi map domain error → HTTP status ở MỘT chỗ, không leak SQL/stacktrace, `Abort()` khi short-circuit.
- [ ] Tôi verify JWT độc lập trong Go, **không mù tin** `X-Userinfo`, lấy `branch_id` chỉ từ token (ADR-013, CVE-2026-29413).
- [ ] Tôi đảm bảo mọi PHI query chạy trong tx đã `SET LOCAL app.current_branch` qua `RLSTx` (ADR-003/005, open-risk #2).
- [ ] Tôi áp Idempotency-Key cho charge/dispense và hiểu nó là một scheme end-to-end với FE (ADR-011).
- [ ] Tôi propagate `c.Request.Context()` xuống app/pgx và cấu hình timeout + graceful shutdown cho K8s.
- [ ] Tôi viết được integration test testcontainers chứng minh branch-B invisible + idempotency replay (ADR-025).
