# PostgreSQL: pgx/v5 + sqlc + golang-migrate

> Module **BE-3** · Tầng truy cập dữ liệu của hms-api: pgx/v5 pool, sqlc type-safe queries, golang-migrate + Phase-0 FORCE RLS, outbox INSERT cùng `pgx.Tx` · Độ khó: 🥉→🥇 · Prereqs: **BE-1** (Go production-grade, composition root, layout `internal/<bc>/{domain,app,ports,adapters}`)

Module này dạy tầng persistence của HMS — nơi mọi quyết định compliance bậc nhất (ADR-003 FORCE RLS, ADR-009 audit fail-closed, ADR-011 idempotency, ADR-012 outbox) trở thành code thật. Repo CHƯA CÓ CODE; mọi code path đánh dấu *(planned)* theo layout canon section 9.

---

## 1. Vì sao kỹ năng này quan trọng trong HMS

HMS là một Go modular monolith (ADR-001) đặt trên **một** PostgreSQL 16 OLTP shared-schema (ADR-015). Toàn bộ an toàn bệnh nhân và hợp lệ pháp lý đều kết tinh ở tầng DB này:

- **Cách ly dữ liệu đa chi nhánh sống ở DB, không ở Go.** RLS (Row Level Security) lọc theo `branch_id` (ADR-003/005). Nếu tầng pgx không SET đúng session-var trong tx, một bác sĩ chi nhánh A có thể đọc PHI chi nhánh B — leak âm thầm pass review (open risk #1). Tầng pgx là nơi enforce invariant "mọi PHI query chạy trong tx đã `SET LOCAL`".
- **Charge-capture, claim↔bill↔encounter linkage, FEFO cần ACID nội-process** (ADR-001/011/021). Vì không có broker ngoài ở MVP (ADR-012), tính nhất quán cross-BC dựa hoàn toàn vào việc ghi domain change + outbox event trong **cùng một `pgx.Tx`**. Sai một dòng tx ở đây = double-post viện phí hoặc claim mất.
- **EMR ký số (TT 13/2025) cần synchronous durability** (ADR-004/015): commit phải confirmed TRƯỚC khi UI báo "signed". Đây là cấu hình mức pool/tx, không phải tùy chọn.
- **Audit-of-reads fail-closed** (ADR-009): không trả PHI nếu chưa ghi được audit — audit INSERT phải nằm trong cùng tx với (hoặc trước) response.
- **Schema không-backfill-được phải đúng từ migration 000001** (ADR-024): extensions, role separation, FORCE RLS phải có TRƯỚC bất kỳ bảng PHI nào. golang-migrate là org-standard, sqlc đọc cùng file schema.

Hiểu sai pgx/sqlc/migrate ở đây không phải bug thường — là PHI leak hoặc record pháp lý mất.

---

## 2. Mô hình tư duy (first principles) — từ con số 0

Bắt đầu từ câu hỏi gốc: *"Go nói chuyện với Postgres như thế nào, và ai giữ những bất biến an toàn?"*

1. **Một query = một câu lệnh gửi qua TCP tới Postgres trên một connection.** Mở connection mỗi request thì quá đắt → ta dùng **connection pool**. pgx/v5 (`pgxpool`) quản pool này.
2. **Connection được tái sử dụng** (pool reuse). Đây là first principle cốt tử: nếu bạn `SET app.current_branch = 'A'` trên một connection rồi trả về pool, request sau mượn lại connection đó vẫn thấy `'A'`. → Phải dùng `SET LOCAL` (chỉ sống trong tx) thay vì `SET`, và phải bọc query PHI trong tx.
3. **Transaction là đơn vị atomic + đơn vị scope của `SET LOCAL`.** Mọi thứ phải-nhất-quán (domain row + outbox row + audit row) đi vào CÙNG một `tx`. Commit = tất cả, rollback = không gì cả.
4. **Viết SQL bằng tay nhưng có type safety.** ORM giấu SQL → ta không kiểm soát được RLS/index/plan. Viết SQL thuần nhưng để **sqlc** sinh ra hàm Go typed từ chính SQL đó (compile-time check, no reflection). ADR-003 pinned: `pgx/v5 + sqlc (no ORM)`.
5. **Schema tiến hóa qua các bước có thứ tự, version-controlled, reversible.** Đó là **migrations** (golang-migrate): `NNNNNN_name.up.sql` / `.down.sql`. Migration 000001 đặt nền compliance (ADR-024).
6. **Quyền DB phân tách theo nguyên tắc least-privilege.** Role chạy DDL (sở hữu bảng) ≠ role chạy app. Vì PostgreSQL table OWNER **bypass RLS kể cả khi NOBYPASSRLS** — nếu app-role là owner, RLS thành no-op (ADR-003). Đây là điều phản trực giác nhất và là lý do migration owner phải tách.

Mô hình rút gọn: **pool → tx (SET LOCAL branch) → {sqlc query + outbox INSERT + audit INSERT} → commit**. Mọi thứ khác là chi tiết.

---

## 3. Khái niệm cốt lõi (tăng dần độ khó)

🥉 **Cơ bản**
- `pgxpool.Pool`: khởi tạo từ DSN, cấu hình `MaxConns`, `MinConns`, `MaxConnLifetime`, healthcheck. Một pool cho cả app, inject từ composition root *(planned: `cmd/hms-api/main.go`)*.
- `Query`/`QueryRow`/`Exec`; quét kết quả bằng `pgx.CollectRows` + `pgx.RowToStructByName`.
- sqlc workflow: viết `.sql` query → `sqlc generate` → dùng `Queries` struct typed.

🥈 **Trung cấp**
- Transaction: `pool.Begin(ctx)` → `tx.Commit/Rollback`; helper `WithTx` đảm bảo rollback-on-error + `defer`.
- `SET LOCAL` GUC trong tx: `SELECT set_config('app.current_branch', $1, true)` (param `true` = local-to-tx) — nền của RLS contract (ADR-005).
- Outbox cùng tx (ADR-012): domain repo và `outbox` repo nhận **cùng** `tx`, ghi domain row + outbox row atomic.
- Idempotency (ADR-011): bảng `idempotency_keys` unique-constraint; INSERT ... ON CONFLICT để dedupe charge/claim.

🥇 **Nâng cao**
- FORCE RLS + policy `USING` **và** `WITH CHECK` (ADR-003): `USING` chặn đọc cross-branch, `WITH CHECK` chặn ghi cross-branch (poisoning).
- Migration owner vs app-role separation: app-role `NOSUPERUSER NOBYPASSRLS`, KHÔNG own bảng.
- `FOR UPDATE SKIP LOCKED`: dùng cho outbox relay và FEFO dispense (ADR-012/021) — concurrent worker không block nhau.
- Zero-downtime migration (ADR-024): add nullable/DEFAULT, add→backfill→switch→drop (không rename), `CREATE INDEX CONCURRENTLY` tách tx.
- Cross-branch reader escalation (ADR-005): policy cho giám định/quản lý vùng.

---

## 4. HMS dùng nó thế nào (bám code path — *(planned)*)

Layout đích (canon section 9). Tất cả `internal/...` dưới đây CHƯA tồn tại — đây là thiết kế mục tiêu.

**Pool + WithTx helper** — *(planned)* `internal/shared/rls/` và `internal/shared/config/`:

```go
// internal/shared/rls/tx.go (planned)
// WithBranchTx mở tx, SET LOCAL branch_id (lấy từ JWT đã verify, KHÔNG từ client),
// chạy fn, commit/rollback. MỌI PHI query PHẢI đi qua đây (ADR-003/005).
func WithBranchTx(ctx context.Context, pool *pgxpool.Pool, branchID uuid.UUID,
    fn func(tx pgx.Tx) error) error {
    tx, err := pool.Begin(ctx)
    if err != nil { return fmt.Errorf("begin tx: %w", err) }
    defer tx.Rollback(ctx) // no-op sau Commit
    // local=true => GUC chỉ sống trong tx này, không leak qua pool reuse
    if _, err := tx.Exec(ctx,
        "SELECT set_config('app.current_branch', $1, true)", branchID.String()); err != nil {
        return fmt.Errorf("set branch guc: %w", err)
    }
    if err := fn(tx); err != nil { return err }
    return tx.Commit(ctx)
}
```

**Repo + outbox cùng tx** — *(planned)* `internal/billing/adapters/postgres/` (charge-capture, ADR-011/012):

```go
// internal/billing/adapters/postgres/charge_repo.go (planned)
func (r *ChargeRepo) CaptureCharge(ctx context.Context, tx pgx.Tx, c domain.ChargeItem) error {
    q := sqlcgen.New(tx) // sqlc Queries chạy trên tx (KHÔNG pool)
    // 1) idempotency guard (ADR-011): unique-constraint trên idempotency_key
    if err := q.InsertIdempotencyKey(ctx, c.IdempotencyKey); err != nil {
        if isUniqueViolation(err) { return domain.ErrDuplicateCharge } // dedupe
        return err
    }
    // 2) ghi charge row
    if err := q.InsertCharge(ctx, toInsertParams(c)); err != nil { return err }
    // 3) outbox event CÙNG tx (ADR-012) — relay sau bằng SKIP LOCKED
    return q.InsertOutboxEvent(ctx, sqlcgen.InsertOutboxEventParams{
        AggregateType: "charge", AggregateID: c.ID, EventType: "ChargeCaptured",
        Payload: mustJSON(c),
    })
}
```

App layer gọi `WithBranchTx(...)` rồi truyền `tx` xuống repo — đảm bảo charge + outbox + (audit) atomic.

**Outbox relay** — *(planned)* `internal/shared/outbox/relay.go` (ADR-012):

```sql
-- relay poll, một worker giữ batch, worker khác bỏ qua (không block)
SELECT id, aggregate_type, event_type, payload
FROM outbox_events
WHERE processed_at IS NULL
ORDER BY id
FOR UPDATE SKIP LOCKED
LIMIT 100;
```

**Migration 000001** — *(planned)* `backend/migrations/000001_phase0_compliance.up.sql` (ADR-024, keystone ADR-003). Phải tạo theo đúng thứ tự:

```sql
-- 1) extensions
CREATE EXTENSION IF NOT EXISTS pgcrypto;
CREATE EXTENSION IF NOT EXISTS pg_trgm;
-- (uuid v7 sinh ở app-side bằng Go, BIGINT IDENTITY cho audit_log/outbox/stock_ledger)

-- 2) role separation (ADR-003) — owner chạy DDL, app-role KHÔNG own bảng
CREATE ROLE hms_migration_owner;                          -- chạy migrations, sở hữu bảng
CREATE ROLE hms_app NOSUPERUSER NOBYPASSRLS NOCREATEDB;    -- app runtime, KHÔNG bypass RLS

-- 3) bảng nền: branches, accounts/roles/permissions, audit_log (INSERT-only, partition tháng)

-- 4) RLS keystone trên MỌI bảng PHI — ENABLE + FORCE (ADR-003)
ALTER TABLE patients ENABLE ROW LEVEL SECURITY;
ALTER TABLE patients FORCE ROW LEVEL SECURITY;            -- FORCE: kể cả owner cũng bị lọc
CREATE POLICY branch_isolation ON patients
    USING      (branch_id = current_setting('app.current_branch')::uuid)   -- chặn đọc
    WITH CHECK (branch_id = current_setting('app.current_branch')::uuid);  -- chặn ghi
GRANT SELECT, INSERT, UPDATE, DELETE ON patients TO hms_app;               -- app-role, không own
```

`down.sql` reverse đúng thứ tự (drop policy → disable RLS → drop tables → drop roles → drop extensions). **sqlc** đọc cùng thư mục `migrations/` làm schema *(planned: `backend/sqlc.yaml`)*.

**Audit fail-closed** — *(planned)* `internal/audit/adapters/postgres/`: cho PHI **read**, audit INSERT commit cùng/trước response (ADR-009) — nếu INSERT fail thì rollback và KHÔNG trả PHI.

---

## 5. Best practices (mỗi mục kèm 1 nguồn đã research)

1. **Dùng `pgxpool` + context-aware, đặt `MaxConns`/`MaxConnLifetime` hợp lý; không mở connection thủ công.** Pool tuning sai gây connection storm tới managed PG.
   — pgx v5 docs & FAQ: https://github.com/jackc/pgx/wiki/Getting-started-with-pgx
2. **Luôn dùng parameterized query ($1,$2…); pgx không nội suy chuỗi.** Chống SQL injection ở system boundary (security.md).
   — OWASP SQL Injection Prevention Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html
3. **Đặt RLS policy có CẢ `USING` và `WITH CHECK`, và bật `FORCE`.** `USING`-only vẫn cho ghi cross-tenant; thiếu `FORCE` thì owner bypass.
   — PostgreSQL 16 docs, CREATE POLICY / Row Security Policies: https://www.postgresql.org/docs/16/ddl-rowsecurity.html
4. **Tách role sở hữu schema khỏi role app, app-role `NOBYPASSRLS` và KHÔNG là owner.** Multi-tenant-safe RLS yêu cầu điều này.
   — Supabase RLS performance & ownership guide: https://supabase.com/docs/guides/database/postgres/row-level-security
5. **Viết SQL thuần + sinh code bằng sqlc thay vì ORM; review SQL như review code.** Type-safe, không reflection, plan minh bạch.
   — sqlc official docs: https://docs.sqlc.dev/en/latest/
6. **Outbox row ghi trong cùng tx với domain change; relay bằng `FOR UPDATE SKIP LOCKED`.** Đảm bảo at-least-once không mất event mà không cần broker.
   — microservices.io — Transactional Outbox pattern: https://microservices.io/patterns/data/transactional-outbox.html
7. **Migration phải reversible + zero-downtime (expand/contract, `CREATE INDEX CONCURRENTLY` tách tx).** Tránh table lock trên bảng PHI đang chạy.
   — golang-migrate docs + PostgreSQL CREATE INDEX CONCURRENTLY: https://github.com/golang-migrate/migrate / https://www.postgresql.org/docs/16/sql-createindex.html

---

## 6. Lỗi thường gặp & anti-patterns

| Anti-pattern | Hậu quả | Cách đúng |
|---|---|---|
| `SET app.current_branch` (không `LOCAL`/`set_config(...,true)`) | GUC dính qua pool reuse → request sau leak branch khác (open risk #2) | `set_config(..., true)` trong tx, qua `WithBranchTx` |
| Query PHI ngoài tx (dùng `pool.Query` trực tiếp) | Không có branch GUC → policy thấy NULL → leak hoặc lỗi | Lint/review chặn `pool.Query` cho bảng PHI; chỉ query qua tx |
| App-role là owner bảng (golang-migrate chạy bằng app-role) | Owner **bypass RLS** kể cả NOBYPASSRLS → RLS thành no-op âm thầm | Migrations chạy bằng `hms_migration_owner`; app dùng `hms_app` |
| Policy chỉ có `USING`, thiếu `WITH CHECK` | Ghi được row branch khác (cross-tenant poisoning) | Luôn cả `USING` + `WITH CHECK` |
| Outbox INSERT ở tx riêng sau commit domain | Crash giữa hai commit → mất event / charge không sinh claim | Domain + outbox + audit cùng một `pgx.Tx` |
| Charge/claim không có idempotency key unique-constraint | Retry/replay → double-post viện phí (ADR-011) | `idempotency_keys` unique + ON CONFLICT |
| Trả PHI rồi mới ghi audit best-effort async | Crash mất audit → vi phạm ADR-009/NĐ13/HIPAA | Audit-read commit-with-response, fail-closed |
| Resource khác branch trả 403 | Lộ sự tồn tại của record cross-branch | Trả **404** (ADR-003) |
| `CREATE INDEX` (không CONCURRENTLY) trên bảng PHI lớn | Table lock, downtime | `CONCURRENTLY`, tách khỏi tx migration |

---

## 7. Lộ trình luyện tập NGAY trong repo

> Repo chưa có code — các bài tập sau **tạo** scaffold *(planned)* theo layout canon. Dùng testcontainers-go + Docker (ADR-025).

🥉 **Cơ bản — pool + sqlc + một migration**
1. Scaffold `backend/` (`go mod init hms`), thêm `migrations/000001_phase0_compliance.up.sql` chỉ với extensions + bảng `branches` + một bảng `patients` tối thiểu.
2. Cấu hình `sqlc.yaml`, viết query `GetPatientByID`, chạy `sqlc generate`.
3. Viết `pgxpool` init trong `internal/shared/config` *(planned)* và một test `pool.Ping`.

🥈 **Trung cấp — RLS + WithBranchTx + outbox cùng tx**
1. Thêm `branch_id` + `ENABLE`+`FORCE RLS` + policy `USING`/`WITH CHECK` vào `patients` trong migration 000001.
2. Viết `WithBranchTx` *(planned)* `internal/shared/rls/tx.go` đúng như Mục 4.
3. Thêm bảng `outbox_events` + repo `InsertOutboxEvent`; viết một `CaptureCharge`-style flow ghi domain row + outbox row trong cùng tx, integration test (testcontainers) xác nhận cả hai cùng commit/rollback.

🥇 **Nâng cao — branch-isolation gate + role separation + idempotency**
1. Viết integration test merge-blocking (ADR-003/025): seed row branch A và B, mở tx với `app.current_branch=A`, assert query KHÔNG thấy row B (branch-B invisible). Đây là gate bắt buộc.
2. Tạo `hms_migration_owner` vs `hms_app`; chạy migration bằng owner, chạy app query bằng app-role; test chứng minh app-role **không** bypass RLS dù tắt policy bằng tay là không thể.
3. Thêm `idempotency_keys` unique + `INSERT ... ON CONFLICT`; test replay cùng key chỉ tạo 1 charge (ADR-011).

---

## 8. Skill/agent ECC nên dùng khi luyện

- **`ecc:postgres-patterns`** — query optimization, schema design, indexing, RLS (Supabase best practices); dùng khi thiết kế migration 000001 và policy.
- **`ecc:database-migrations`** — review expand/contract, reversibility, zero-downtime cho golang-migrate.
- **`ecc:go-review`** (go-reviewer agent) — review tầng adapters/postgres: tx handling, error wrapping, không leak `pool.Query` cho PHI.
- **`ecc:go-test`** — enforce TDD table-driven + verify coverage ≥80% (testing.md) cho repo + WithBranchTx.
- **`ecc:security-review`** / **`ecc:security-scan`** — kiểm SQL injection boundary, RLS bypass, role privilege; chạy trước commit (security.md).
- **`ecc:go-build`** (go-build-resolver) — gỡ lỗi build/sqlc-gen mismatch incrementally.
- **`ecc:healthcare-phi-compliance`** — đối chiếu audit-of-reads, encryption scope, leak vector cho tầng DB PHI.

---

## 9. Tài nguyên học thêm (2024–2026)

- pgx/v5 repository + wiki (jackc/pgx): https://github.com/jackc/pgx — pool, tx, `CollectRows`, `RowToStructByName`.
- sqlc documentation: https://docs.sqlc.dev/en/latest/ — config, query annotations, pgx/v5 driver.
- golang-migrate: https://github.com/golang-migrate/migrate — CLI, file format, postgres driver.
- PostgreSQL 16 — Row Security Policies: https://www.postgresql.org/docs/16/ddl-rowsecurity.html
- PostgreSQL 16 — CREATE POLICY / ALTER TABLE … FORCE ROW LEVEL SECURITY: https://www.postgresql.org/docs/16/sql-createpolicy.html
- Transactional Outbox pattern (microservices.io): https://microservices.io/patterns/data/transactional-outbox.html
- "SELECT … FOR UPDATE SKIP LOCKED" queue pattern (PostgreSQL docs §13.3.2): https://www.postgresql.org/docs/16/explicit-locking.html
- OWASP SQL Injection Prevention Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html
- Supabase RLS guide (ownership + performance): https://supabase.com/docs/guides/database/postgres/row-level-security

---

## 10. Checklist "đã hiểu"

- [ ] Giải thích được vì sao `SET LOCAL`/`set_config(...,true)` bắt buộc thay vì `SET` (pool reuse, open risk #2).
- [ ] Nêu được vì sao app-role KHÔNG được là owner bảng (owner bypass RLS kể cả NOBYPASSRLS — ADR-003).
- [ ] Viết được policy có CẢ `USING` và `WITH CHECK`, và biết `FORCE` khác `ENABLE` chỗ nào.
- [ ] Hiểu vì sao domain row + outbox row + audit row phải cùng một `pgx.Tx` (ADR-009/011/012).
- [ ] Biết outbox relay dùng `FOR UPDATE SKIP LOCKED` và idempotency dùng unique-constraint + ON CONFLICT.
- [ ] Liệt kê được thứ tự bắt buộc của migration 000001 (extensions → roles → bảng nền → FORCE RLS) — ADR-024.
- [ ] Biết resource khác branch trả **404** không phải 403.
- [ ] Viết được branch-isolation integration test (testcontainers) chứng minh branch-B invisible — gate merge-blocking (ADR-003/025).
- [ ] Hiểu signed-EMR write path cần synchronous durability (commit confirmed trước UI báo "signed") — ADR-004/015.
- [ ] Biết zero-downtime migration: expand/contract, `CREATE INDEX CONCURRENTLY` tách tx, không rename trực tiếp (ADR-024).
