# [SEC-1] AppSec & OWASP: Keycloak OIDC + Kong edge + Go authz

> Module SEC-1 · AuthN/AuthZ defense-in-depth cho HMS: Keycloak OIDC issuer, Kong edge-auth coarse-grained, object-level/ABAC enforce trong Go · Độ khó: 🥉→🥇 · Prereqs: BE-2 (REST API với Gin behind Kong)

> Nguồn sự thật: DESIGN_CANON §3 (identity, gateway), ADR-013, ADR-005, ADR-008, ADR-019. Repo CHƯA CÓ CODE — mọi code path đánh dấu *(planned)*. Stack PINNED: Keycloak 26 · Kong Ingress Controller DB-less · Go + Gin behind Kong. KHÔNG bịa IdP/gateway khác.

---

## 1. Vì sao kỹ năng này quan trọng trong HMS

Trong một bệnh viện vừa rời giấy, lỗ hổng authz không phải bug — nó là rò rỉ PHI có thể bị thanh tra NĐ 13/2023 phạt và là vi phạm y đức. Hai lớp rủi ro cụ thể neo vào canon:

- **BOLA/IDOR là lỗ hổng #1 trong y tế** (OWASP API Security Top 10 #1). Gateway không có context lâm sàng nên KHÔNG thể trả lời "bác sĩ X có được xem bệnh nhân Y không" — nó chỉ biết JWT hợp lệ. Nếu để Kong quyết object-level authz, mọi clinician có token hợp lệ đọc được mọi hồ sơ → leak toàn hệ thống (ADR-013).
- **CVE-2026-29413** (Kong auth-bypass, CISA-KEV, đã bị khai thác trên healthcare gateway): nếu Kong bị bypass nhưng mTLS-to-upstream vẫn cho Kong gọi `hms-api`, và Go *mù tin* header `X-Userinfo` Kong inject, thì app sẽ tin identity giả mạo. Đây là [high] risk trong canon §8.

Mục tiêu module: hiểu **tại sao authn ở edge nhưng authz object-level ở Go**, cách Go verify JWT độc lập (defense-in-depth), và cách enforce ABAC (role + branch + quan hệ điều trị + minimum-necessary) khớp với RLS keystone (ADR-003/005). Đây là tầng 1 (Kong) + tầng 2 (Go) trong mô hình 4-tầng defense-in-depth; tầng 3 là Postgres RLS (DATA-1), tầng 4 là K8s NetworkPolicy.

## 2. Mô hình tư duy (first principles) — từ con số 0

Tách bạch ba câu hỏi, KHÔNG gộp:

1. **Authentication (AuthN) — "anh là ai?"** Xác thực danh tính. Keycloak 26 sở hữu password + MFA; Go KHÔNG quản password. Bằng chứng danh tính là JWT (OIDC ID/access token) ký bằng khóa riêng của Keycloak.
2. **Coarse-grained authorization (edge) — "token này có hợp lệ để vào hệ thống không?"** Kong verify chữ ký JWT qua JWKS, reject `alg=none`, rate-limit, TLS. Đây là *lọc thô*, không phải quyết định nghiệp vụ.
3. **Object-level / fine-grained authorization (app) — "user này được làm hành động này lên ĐÚNG resource này không?"** Chỉ Go có đủ context (quan hệ điều trị, branch, mục đích) để trả lời. Đây là nơi chặn BOLA.

First principle nền tảng: **mỗi tầng PHẢI verify độc lập, không tin tầng trước một cách mù quáng (zero-trust nội bộ).** Kong verify JWT là tốt; nhưng Go verify lại signature/claims một lần nữa vì nếu Kong bị bypass (CVE-2026-29413), Go là backstop. Header `X-Userinfo` từ Kong là *gợi ý*, không phải *bằng chứng* — bằng chứng là chữ ký JWT mà Go tự verify qua JWKS.

Nguyên tắc thứ hai: **role KHÔNG nhúng cứng vào logic per-request DB lookup.** Token short-TTL (5–15m) + cached roles (ADR-013) — tránh mỗi request một query DB role, nhưng vẫn revoke nhanh qua TTL ngắn.

## 3. Khái niệm cốt lõi (tăng dần độ khó)

| Khái niệm | 🥉 | Mô tả |
|---|---|---|
| OIDC issuer + JWKS | 🥉 | Keycloak phát hành JWT ký bằng private key; JWKS endpoint công bố public key để verify. Realm = `hospital` (một realm, branch là group/attribute, KHÔNG realm-per-branch). |
| Auth-code + PKCE qua Kong BFF | 🥉 | SPA không bao giờ thấy token; Kong làm BFF, lưu token trong cookie `HttpOnly+Secure+SameSite=Strict`, inject identity tới upstream. |
| JWT verification (alg pinning) | 🥈 | Verify `RS256` từ JWKS; **reject `alg=none`** và **reject HS256-confusion** (không bao giờ verify bằng public key như HMAC secret). |
| Claims: `branch_id`, `roles`, `sub`, `exp` | 🥈 | Go đọc `branch_id` từ claim đã verify (KHÔNG từ client) → dùng cho `SET LOCAL app.current_branch` (RLS). |
| RBAC personas | 🥈 | `bac_si / dieu_duong / duoc_si / le_tan / thu_ngan / giam_dinh / quan_tri` qua Keycloak group. |
| ABAC object-level | 🥇 | Quyết định = f(role, branch, quan hệ điều trị clinician↔patient/encounter, minimum-necessary). Enforce trong `app` layer. |
| Step-up auth | 🥇 | Hành động nhạy cảm (ký EMR, break-the-glass) yêu cầu re-auth/MFA mạnh hơn (`acr`/`amr` claim). |
| 404-not-403 cross-branch | 🥇 | Resource khác branch trả **404** (không 403) — không tiết lộ sự tồn tại (ADR-003). |
| Fail-closed | 🥇 | Authz check error/timeout → deny. CDSS hard-stop cũng fail-closed (ADR-008). |

## 4. HMS dùng nó thế nào (bám code path — *(planned)*)

Repo chưa có code. Layout mục tiêu (canon §9):

```mermaid
sequenceDiagram
    participant SPA as React SPA
    participant Kong as Kong KIC (edge)
    participant Go as hms-api (Gin)
    participant KC as Keycloak 26 (JWKS)
    participant PG as Postgres (FORCE RLS)
    SPA->>Kong: request + HttpOnly cookie (token)
    Kong->>KC: verify JWT via JWKS (cache), reject alg=none
    Kong->>Go: forward + X-Userinfo (mTLS upstream)
    Note over Go: KHÔNG mù tin header
    Go->>KC: verify JWT signature/claims độc lập (cached JWKS)
    Go->>Go: ABAC: role+branch+quan hệ điều trị+min-necessary
    Go->>PG: BEGIN; SET LOCAL app.current_branch=<claim>; query
    PG-->>Go: chỉ rows của branch (RLS USING/WITH CHECK)
```

- `deploy/kong/` *(planned)* — KongPlugin CRD DB-less (canon §9): `jwt` plugin verify JWKS, `rate-limiting` (Redis), `ip-restriction` cho admin path. Kong CHỈ coarse-grained — KHÔNG object-level (ADR-013/019).
- `internal/shared/auth/` *(planned)* — Go verify JWT độc lập: fetch + cache JWKS từ Keycloak, verify `RS256`, validate `iss/aud/exp`, reject `alg=none`. Đây là mitigation CVE-2026-29413.
- `internal/shared/middleware/` *(planned)* — Gin middleware: (1) extract+verify JWT → `Identity{sub, branchID, roles}`; (2) mở tx + `SET LOCAL app.current_branch` (DATA-1 contract); (3) gọi audit-of-reads commit-with-response (SEC-2).
- `internal/<bc>/app/` *(planned)* — ABAC policy nằm ở app layer (vd `internal/encounter/app/`), KHÔNG ở adapter HTTP. Quyết định object-level dùng quan hệ điều trị từ domain.
- `internal/identity/` *(planned)* — BC IAM: đồng bộ role từ Keycloak, BranchMembership, MfaFactor, BreakGlassGrant (canon §4).

Pattern Go verify JWT độc lập *(planned, minh hoạ — chưa phải code repo)*:

```go
// internal/shared/auth/verifier.go (planned)
type Identity struct {
    Subject  string
    BranchID uuid.UUID
    Roles    []string
    ACR      string // step-up level
}

// VerifyFromRequest verify JWT từ Kong-forwarded token ĐỘC LẬP (CVE-2026-29413 backstop).
// KHÔNG đọc X-Userinfo như nguồn tin cậy.
func (v *Verifier) VerifyFromRequest(r *http.Request) (Identity, error) {
    raw := extractBearer(r) // token Kong forward, KHÔNG phải header userinfo
    tok, err := jwt.Parse(raw, v.keyfunc, jwt.WithValidMethods([]string{"RS256"})) // reject alg=none/HS256
    if err != nil || !tok.Valid {
        return Identity{}, ErrUnauthenticated // fail-closed
    }
    claims := tok.Claims.(jwt.MapClaims)
    bid, err := uuid.Parse(claims["branch_id"].(string)) // branch TỪ CLAIM, không từ client
    if err != nil {
        return Identity{}, ErrUnauthenticated
    }
    return Identity{Subject: claims["sub"].(string), BranchID: bid, Roles: toRoles(claims), ACR: acr(claims)}, nil
}
```

ABAC object-level (chặn BOLA), trả 404 cross-branch *(planned)*:

```go
// internal/encounter/app/authz.go (planned)
func (a *App) authorizeEncounterRead(id Identity, enc Encounter) error {
    if enc.BranchID != id.BranchID && !id.HasRole("giam_dinh", "quan_tri") {
        return ErrNotFound // 404 không 403 (ADR-003) — không lộ tồn tại
    }
    if !id.HasRole("bac_si", "dieu_duong") || !a.hasTreatmentRelation(id.Subject, enc.ID) {
        return ErrForbidden // minimum-necessary: chỉ clinician có quan hệ điều trị
    }
    return nil
}
```

## 5. Best practices (mỗi mục kèm 1 nguồn đã research)

- **Verify JWT độc lập ở mọi service, pin algorithm, reject `alg=none`.** Go không mù tin gateway header (ADR-013). Nguồn: OWASP JWT for Java/JSON Web Token Cheat Sheet — https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html
- **Object-level authorization ở mỗi endpoint nhận object id** — chống BOLA (#1 API risk). Nguồn: OWASP API Security Top 10 2023, API1:2023 BOLA — https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/
- **Auth-code + PKCE, token KHÔNG để SPA chạm** (BFF pattern, token trong HttpOnly cookie). Nguồn: IETF OAuth 2.0 Security BCP (RFC 9700) — https://datatracker.ietf.org/doc/rfc9700/
- **Keycloak làm OIDC issuer self-hosted, dùng group/role mapper, JWKS rotation.** Nguồn: Keycloak Server Admin Guide (securing apps / token) — https://www.keycloak.org/docs/latest/server_admin/index.html
- **Kong JWT plugin verify JWKS + version-pin Kong là CI/admission gate** (CVE-2026-29413). Nguồn: Kong JWT plugin docs — https://docs.konghq.com/hub/kong-inc/jwt/ ; CISA Known Exploited Vulnerabilities Catalog — https://www.cisa.gov/known-exploited-vulnerabilities-catalog
- **MFA bắt buộc + step-up cho hành động nhạy cảm** (ký EMR/break-glass), dựa `acr/amr`. Nguồn: NIST SP 800-63B Digital Identity Guidelines — https://pages.nist.gov/800-63-3/sp800-63b.html
- **Multi-tenancy: branch_id từ claim đã verify, KHÔNG từ client; ghép RLS.** Nguồn: PostgreSQL Row Security Policies — https://www.postgresql.org/docs/16/ddl-rowsecurity.html

## 6. Lỗi thường gặp & anti-patterns

- **Để Kong quyết object-level authz.** Gateway không có context lâm sàng → mọi token hợp lệ đọc mọi PHI. Authz object-level PHẢI ở Go (ADR-013).
- **Mù tin `X-Userinfo` header.** Nếu Kong bypass (CVE-2026-29413) → forged identity. Go phải verify JWT signature độc lập.
- **Không pin algorithm.** Để verifier chấp nhận `alg=none` hoặc HS256-confusion → forge token. Luôn `WithValidMethods(["RS256"])`.
- **Lấy `branch_id` từ request body/query.** Tenant spoofing → đọc branch khác. Luôn lấy từ claim đã verify (ADR-005).
- **Trả 403 cho resource cross-branch.** Lộ sự tồn tại của resource → enumeration. Trả **404** (ADR-003).
- **Fail-open khi authz/JWKS lỗi.** Authz check timeout → KHÔNG được "cho qua". Fail-closed (như CDSS, ADR-008).
- **Per-request DB lookup role.** Chậm + coupling. Dùng short-TTL token + cached roles (ADR-013).
- **Nhúng role cứng vào token rồi không revoke.** Long-lived token → không thu hồi kịp. Dùng TTL 5–15m.
- **Query PHI ngoài tx đã `SET LOCAL`.** pgx pool reuse connection → mất branch filter, leak (canon §8 [critical]). Mọi PHI query trong tx wrapper.

## 7. Lộ trình luyện tập NGAY trong repo (🥉→🥇)

> Repo chưa có code; bài tập là *thiết kế + scaffold + viết test trước* (TDD theo testing rule 80%). Dùng layout *(planned)* canon §9.

- 🥉 **Cơ bản — JWKS verifier có test.** Tạo `internal/shared/auth/verifier_test.go` *(planned)*: viết table-driven test RED chứng minh (a) token `alg=none` bị reject, (b) token sai `iss` bị reject, (c) token hợp lệ trả đúng `branch_id`. Sau đó implement `Verifier` để xanh. Neo ADR-013.
- 🥈 **Trung cấp — Gin middleware AuthN→tx→branch.** Scaffold `internal/shared/middleware/auth.go` *(planned)*: middleware verify JWT → mở pgx.Tx → `SET LOCAL app.current_branch`. Viết integration test (testcontainers real PG, ADR-025) chứng minh request branch-A KHÔNG đọc được row branch-B. Đây là RLS keystone test (DATA-1) ghép authz.
- 🥇 **Nâng cao — ABAC object-level + 404 cross-branch + step-up.** Implement `authorizeEncounterRead` trong `internal/encounter/app/authz.go` *(planned)*: (a) cross-branch → 404; (b) clinician không có quan hệ điều trị → 403; (c) ký EMR yêu cầu `acr` step-up. Viết E2E qua Kong BFF (Playwright/contract) cho luồng login PKCE → đọc encounter → ký EMR step-up. Bonus: viết admission/CI check Kong version-pin (CVE-2026-29413 gate, ADR-019).

## 8. Skill/agent ECC nên dùng khi luyện

- `ecc:security-review` / agent **security-reviewer** — soát OWASP API Top 10 (BOLA/BFLA), JWT misconfig, fail-open. Chạy TRƯỚC mọi commit auth code (security rule).
- `ecc:go-review` / agent **go-reviewer** — review idiomatic Go, error handling fail-closed, immutability của `Identity`.
- `ecc:hipaa-compliance` + `ecc:healthcare-phi-compliance` — đối chiếu minimum-necessary, access control, audit-of-reads (ghép SEC-2).
- `ecc:security-scan` (AgentShield) — quét secret/permission surface; bảo đảm Keycloak client-secret không hardcode (KMS/ESO, ADR-014).
- `ecc:postgres-patterns` — verify RLS USING & WITH CHECK ghép branch claim (DATA-1).
- `ecc:go-test` — enforce TDD red-green + 80% coverage cho verifier/middleware (ADR-025).
- `ecc:kubernetes-patterns` — review Kong KIC DB-less + NetworkPolicy default-deny (tầng 1 & 4 defense-in-depth).

## 9. Tài nguyên học thêm (2024–2026)

- OWASP API Security Top 10 (2023): https://owasp.org/API-Security/editions/2023/en/0x11-t10/
- OWASP Authentication Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- OAuth 2.0 Security Best Current Practice — RFC 9700 (2025): https://datatracker.ietf.org/doc/rfc9700/
- OAuth 2.0 for Browser-Based Apps (BFF pattern): https://datatracker.ietf.org/doc/draft-ietf-oauth-browser-based-apps/
- Keycloak 26 docs (Server Admin / securing apps): https://www.keycloak.org/documentation
- Kong Gateway JWT plugin & Ingress Controller (DB-less): https://docs.konghq.com/kubernetes-ingress-controller/latest/
- CISA Known Exploited Vulnerabilities Catalog (theo dõi CVE Kong): https://www.cisa.gov/known-exploited-vulnerabilities-catalog
- NIST SP 800-63B (Digital Identity / MFA): https://pages.nist.gov/800-63-3/sp800-63b.html
- PostgreSQL 16 Row Security Policies: https://www.postgresql.org/docs/16/ddl-rowsecurity.html
- NĐ 13/2023 (bảo vệ dữ liệu cá nhân) + TT 13/2025 (EMR ký số) — bối cảnh pháp lý VN, đối chiếu canon §0.

## 10. Checklist "đã hiểu"

- [ ] Phân biệt được AuthN (Keycloak), edge-authz (Kong coarse-grained), object-level authz (Go) — và vì sao Kong KHÔNG quyết object-level (ADR-013).
- [ ] Giải thích được CVE-2026-29413 và tại sao Go phải verify JWT độc lập, không mù tin `X-Userinfo`.
- [ ] Biết reject `alg=none`/HS256-confusion và pin `RS256` qua JWKS.
- [ ] Hiểu `branch_id` lấy từ claim đã verify (không từ client) rồi `SET LOCAL app.current_branch` ghép RLS (ADR-005/003).
- [ ] Giải thích BOLA và cách ABAC (role+branch+quan hệ điều trị+minimum-necessary) chặn nó.
- [ ] Biết vì sao cross-branch trả 404 không 403.
- [ ] Hiểu fail-closed: authz/JWKS lỗi → deny; ghép với CDSS hard-stop (ADR-008).
- [ ] Nắm Auth-code+PKCE qua Kong BFF, token trong HttpOnly cookie, SPA không thấy token (ADR-018).
- [ ] Biết MFA bắt buộc + step-up cho ký EMR/break-glass (`acr/amr`).
- [ ] Viết được test RED chứng minh token độc hại bị reject và branch-B vô hình (testcontainers, ADR-025).
- [ ] Biết Kong version-pin là CI/admission gate (ADR-019) và short-TTL token + cached roles (không per-request DB lookup).
