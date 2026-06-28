# [DATA-1] PostgreSQL RLS & branch_id multi-tenancy (keystone)

> Module DATA-1 · Cô lập dữ liệu PHI đa chi nhánh bằng Row Level Security fail-closed · Độ khó: 🥉→🥇 · Prereqs: BE-3 (pgx/v5 + sqlc + golang-migrate)

> Liên kết: [doc/08-database-schema.md], [doc/09-security.md], ADR-003, ADR-005, ADR-024, ADR-025. Repo HMS **chưa có code** — mọi `path:line` dưới đây là *(planned)* theo layout canon §9.

---

## 1. Vì sao kỹ năng này quan trọng trong HMS

HMS là **shared-schema multi-tenant**: một cụm PostgreSQL 16+, mọi chi nhánh dùng chung bảng, phân biệt bằng cột `branch_id UUID NOT NULL` (ADR-005). Đây là quyết định nền tảng — nhưng nó đặt toàn bộ PHI của mọi bệnh viện vào cùng một bảng `patients`, `encounters`, `prescriptions`. Nếu lớp lọc theo chi nhánh hở **một query**, bác sĩ chi nhánh A đọc được bệnh án chi nhánh B — vi phạm NĐ 13/2023 (bảo vệ dữ liệu cá nhân), TT 13/2025, HIPAA §164.312. Đây là **open risk #1 [critical]** trong canon §8: *"mọi clinician thấy PHI mọi chi nhánh trong khi diagram nói isolated — silent leak, pass review, leak production"*.

Điểm chí tử: lỗi này **không retrofit được**. Một khi đã có data multi-tenant production, bạn không thể "thêm RLS sau" mà chắc chắn không có row nào từng leak. Vì vậy RLS là **keystone Phase-0** (ADR-003), phải nằm trong migration `000001` **trước bất kỳ bảng PHI nào**. WHERE-clause thủ công trong Go là không đủ: chỉ cần một `sqlc` query mới quên `WHERE branch_id = $1` là thủng. RLS đẩy việc lọc xuống **database engine** — hàng phòng thủ cuối cùng không thể bị code application bỏ quên (defense-in-depth tầng Postgres, doc/09).

## 2. Mô hình tư duy (first principles) — từ con số 0

Bắt đầu từ zero: PostgreSQL mặc định, **bất kỳ ai đọc được bảng thì đọc được mọi dòng**. Multi-tenancy cần một bất biến: *"phiên làm việc của user thuộc branch A chỉ thấy dòng có `branch_id = A`"*.

Có 3 cách cài bất biến đó, độ tin cậy tăng dần:

1. **WHERE thủ công ở app** — mọi query thêm `AND branch_id = $branch`. Vấn đề: phụ thuộc kỷ luật con người; quên một chỗ = leak; không ai review được 100% query.
2. **RLS chỉ `ENABLE`** — Postgres tự chèn predicate. Nhưng `ENABLE` **không áp dụng cho table owner** và cho `BYPASSRLS` role. golang-migrate thường chạy bằng chính role app → role app là owner → RLS thành **no-op âm thầm** (ADR-003 "verified").
3. **RLS `ENABLE` + `FORCE` + role tách biệt + `USING` & `WITH CHECK`** — bất biến được engine enforce kể cả với owner, cho cả đọc lẫn ghi. Đây là mô hình HMS chốt.

Ba mảnh ghép phải khớp:
- **`FORCE ROW LEVEL SECURITY`**: ép policy áp dụng cả cho table owner (mặc định owner bypass).
- **Tách role**: `hms_migrator` (owner, chạy DDL, Phase-0) ≠ `hms_app` (`NOSUPERUSER`, `NOBYPASSRLS`, **không** sở hữu bảng). App connect bằng `hms_app` nên luôn chịu RLS.
- **Session variable** `app.current_branch`: Go middleware `SET LOCAL` giá trị này **trong transaction**, trích từ JWT đã verify (KHÔNG lấy từ client). Policy đọc `current_setting('app.current_branch')`.

First-principle then: *RLS chỉ an toàn khi connection role không thể bypass VÀ session variable không thể bị giả mạo VÀ luôn được set trước mọi PHI query*. Ba điều kiện này là ba invariant phải có test (ADR-025).

## 3. Khái niệm cốt lõi (tăng dần độ khó)

| Khái niệm | Mô tả | Tại sao quan trọng trong HMS |
|-----------|-------|------------------------------|
| `ENABLE RLS` | Bật policy filter cho bảng | Điều kiện cần, KHÔNG đủ (owner vẫn bypass) |
| `FORCE RLS` | Áp policy cả cho table owner | Bịt lỗ owner-bypass — keystone ADR-003 |
| `USING (expr)` | Predicate cho **đọc** (SELECT/UPDATE/DELETE thấy dòng nào) | Lọc đọc theo branch |
| `WITH CHECK (expr)` | Predicate cho **ghi** (INSERT/UPDATE dòng nào hợp lệ) | Chặn ghi cross-tenant (poisoning) — thiếu = write vector |
| `current_setting('x', true)` | Đọc session GUC, `true`=missing-ok | Nguồn `branch_id` cho policy |
| `SET LOCAL` | Set GUC **chỉ trong tx hiện tại** | Sống cùng tx; pool reuse connection an toàn |
| Role separation | `migrator` (owner) ≠ `app` (NOBYPASSRLS) | App không bao giờ bypass được RLS |
| `cross_branch_reader` | Policy escalation cho vai trò liên-chi-nhánh | Giám định/quản lý vùng (ADR-005) |

**Leo thang độ khó:** (a) policy đơn `branch_id = current_setting(...)`; (b) tách `USING` vs `WITH CHECK` cho INSERT/UPDATE; (c) policy đa điều kiện cho `cross_branch_reader` + 404-not-403; (d) MPI `patients` dùng chung xuyên chi nhánh nên KHÔNG lọc branch như bảng khác — cần policy riêng (ADR-005); (e) bất biến SET-LOCAL-trong-tx trên pgx pool.

## 4. HMS dùng nó thế nào (bám code path — *(planned)*)

Tất cả đường dẫn dưới đây là **planned** (repo chưa có code), theo layout canon §9.

**Migration 000001** *(planned)* `backend/migrations/000001_phase0_compliance.up.sql` — tạo extensions (`pgcrypto`, `pg_trgm`, uuid), role separation, và RLS keystone TRƯỚC mọi bảng PHI (ADR-024):

```sql
-- 1. Role separation (ADR-003)
CREATE ROLE hms_migrator NOLOGIN;                 -- owner, chạy DDL
CREATE ROLE hms_app LOGIN NOSUPERUSER NOBYPASSRLS;-- app — KHÔNG sở hữu bảng
-- Bảng do hms_migrator tạo; cấp DML (không DDL, không OWNER) cho hms_app:
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO hms_app;

-- 2. Ví dụ bảng PHI: encounters (anchor lâm sàng, ADR-004)
CREATE TABLE encounters (
  id          UUID PRIMARY KEY,            -- UUID v7
  branch_id   UUID NOT NULL,               -- cột dẫn đầu index (ADR-003)
  patient_id  UUID NOT NULL,
  status      TEXT NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_encounters_branch ON encounters (branch_id, patient_id);

-- 3. KEYSTONE: ENABLE + FORCE + USING & WITH CHECK
ALTER TABLE encounters ENABLE ROW LEVEL SECURITY;
ALTER TABLE encounters FORCE  ROW LEVEL SECURITY;     -- áp cả cho owner
CREATE POLICY branch_isolation ON encounters
  USING      (branch_id = current_setting('app.current_branch')::uuid)
  WITH CHECK (branch_id = current_setting('app.current_branch')::uuid);
```

**Session-var contract** *(planned)* `internal/shared/rls/` + `internal/shared/middleware/` — middleware mở tx, `SET LOCAL` GUC từ JWT-claim đã verify, rồi mọi PHI query của BC chạy trong tx đó (ADR-005, open risk §8 SET-LOCAL):

```go
// internal/shared/rls/tx.go (planned)
func WithBranchTx(ctx context.Context, pool *pgxpool.Pool, branchID uuid.UUID,
    fn func(pgx.Tx) error) error {
    tx, err := pool.Begin(ctx)
    if err != nil { return err }
    defer tx.Rollback(ctx)
    // SET LOCAL: chỉ sống trong tx này; an toàn với pool reuse connection.
    // dùng set_config (tham số hoá) — KHÔNG nối chuỗi (chống SQL injection).
    if _, err = tx.Exec(ctx,
        "SELECT set_config('app.current_branch', $1, true)", branchID.String()); err != nil {
        return err
    }
    if err = fn(tx); err != nil { return err }
    return tx.Commit(ctx)
}
```

`branchID` đến từ JWT `branch_id` claim Go verify độc lập (ADR-013, KHÔNG mù tin header Kong) — **không bao giờ** từ body/query client (chống tenant spoofing).

**Resource cross-branch trả 404 không 403** *(planned)* `internal/shared/errors/` — vì 403 tiết lộ "row tồn tại nhưng bạn không được xem" (information leak). Dưới RLS, row branch khác đơn giản **không tồn tại** với phiên hiện tại → `pgx.ErrNoRows` → map sang 404 (ADR-003).

**MPI patients** *(planned)* `internal/patient/adapters/postgres/` — `patients` dùng chung xuyên chi nhánh (một `patient_id`, ADR-005) nên policy cho phép đọc khi có quan hệ điều trị hợp lệ, KHÔNG lọc cứng branch như `encounters`. Đây là exception phải document rõ trong doc/08.

## 5. Best practices (mỗi mục kèm nguồn)

1. **Luôn dùng cả `USING` và `WITH CHECK`.** `USING`-only cho phép INSERT/UPDATE dòng sang branch khác (cross-tenant poisoning). — PostgreSQL docs, *CREATE POLICY*: https://www.postgresql.org/docs/16/sql-createpolicy.html
2. **`FORCE ROW LEVEL SECURITY` cho bảng PHI** vì owner mặc định bypass; và đảm bảo connection role **không** là owner và **không** `BYPASSRLS`. — PostgreSQL docs, *Row Security Policies §5.9*: https://www.postgresql.org/docs/16/ddl-rowsecurity.html
3. **`SET LOCAL` / `set_config(..., is_local=true)` trong transaction**, không dùng `SET SESSION` trên connection pool — GUC session-level rò rỉ sang request kế tiếp dùng lại connection. — pgx FAQ / Crunchy Data, *Row Level Security for Tenants*: https://www.crunchydata.com/blog/row-level-security-for-tenants-in-postgres
4. **Tham số hoá giá trị GUC** (`set_config($1)`), không nội suy chuỗi `SET LOCAL app.current_branch = '...'` — tránh SQL injection qua biến cấu hình. — Supabase RLS guide: https://supabase.com/docs/guides/database/postgres/row-level-security
5. **Index dẫn đầu bằng `branch_id`** (`(branch_id, ...)`) — RLS thêm predicate `branch_id = ...` vào mọi query; thiếu index → seq scan trên bảng PHI lớn. — Citus / multi-tenant Postgres pattern: https://www.citusdata.com/blog/2018/01/03/making-postgres-tenant-aware/
6. **Test RLS bằng PostgreSQL thật, không mock** — RLS là behavior của engine, không thể mock; chạy testcontainers (ADR-025). — testcontainers-go docs: https://golang.testcontainers.org/modules/postgres/

## 6. Lỗi thường gặp & anti-patterns

- **App role là table owner.** golang-migrate chạy bằng role app → role app sở hữu bảng → RLS no-op âm thầm. Diagram nói "isolated", production leak. **Fix:** role `hms_migrator` riêng (ADR-003).
- **Quên `FORCE`.** Chỉ `ENABLE` → owner và superuser bypass. **Fix:** `FORCE` mọi bảng PHI.
- **`USING`-only, thiếu `WITH CHECK`.** Đọc bị lọc nhưng ghi cross-tenant lọt → poisoning.
- **`SET SESSION` trên pool.** GUC dính sang request kế tiếp trên cùng connection → một bệnh nhân thấy branch của người khác. **Fix:** `SET LOCAL` trong tx (ADR-005, risk §8).
- **Query PHI ngoài tx wrapper.** Sau khi tx commit/rollback, GUC mất → query trên pooled connection revert no-filter → leak. **Fix:** invariant "mọi PHI query trong `WithBranchTx`" + lint/review.
- **Trả 403 cho row branch khác.** Tiết lộ sự tồn tại của row. **Fix:** 404.
- **`current_setting('app.current_branch')`** không truyền `missing_ok=true` ở chỗ chấp nhận thiếu → panic; ngược lại, ở policy PHI nên để THIẾU = không match (fail-closed), KHÔNG default về branch nào.
- **Lấy `branch_id` từ client.** Tenant spoofing. **Fix:** chỉ từ JWT verify (ADR-013).

## 7. Lộ trình luyện tập NGAY trong repo (🥉→🥇)

> Thao tác trên repo HMS. Vì code chưa tồn tại, bài tập là **viết migration + test thật**, là chính phần Phase-0 phải build (ADR-024/025).

🥉 **Cơ bản** — Viết `backend/migrations/000001_phase0_compliance.up.sql` *(planned)*: tạo role `hms_migrator`/`hms_app`, bảng `branches`, một bảng PHI (`encounters`) với `branch_id NOT NULL` + `ENABLE`+`FORCE` RLS + policy `USING`&`WITH CHECK`. Chạy `golang-migrate up` trên một Postgres 16 local, dùng `\d encounters` xác nhận "Policies" + "row security enabled (forced)".

🥈 **Trung cấp** — Viết integration test `internal/shared/rls/rls_test.go` *(planned)* bằng **testcontainers-go**: connect bằng `hms_app`, `SET LOCAL app.current_branch=A`, INSERT 2 row branch A và B (qua migrator), rồi SELECT — assert chỉ thấy row A. Thêm case: thử INSERT row branch B khi GUC=A → expect lỗi `WITH CHECK` violation. Đây là **merge-blocking gate** (ADR-003/025).

🥇 **Nâng cao** — Implement `WithBranchTx` *(planned)* `internal/shared/rls/tx.go` + một bài test chứng minh **invariant SET-LOCAL-trong-tx trên pool**: tạo `pgxpool` `max_conns=1`, chạy request branch A trong tx, rồi request branch B reuse cùng connection — assert request B KHÔNG thấy data A và KHÔNG kế thừa GUC của A. Thêm policy `cross_branch_reader` cho persona giám định và test escalation. Map `pgx.ErrNoRows`→404 ở `internal/shared/errors`.

## 8. Skill/agent ECC nên dùng khi luyện

- **ecc:postgres-patterns** — pattern schema/index/RLS Postgres (Supabase best practices), kiểm policy `USING`/`WITH CHECK` và index `branch_id`-leading.
- **ecc:database-migrations** — review migration 000001 zero-downtime, thứ tự DDL, role separation.
- **ecc:go-review** / agent **go-reviewer** — review `WithBranchTx`, pgx pool usage, không nối chuỗi GUC.
- **ecc:security-review** / **security-reviewer** — kiểm tenant-isolation, tenant spoofing, 404-vs-403, fail-closed.
- **ecc:healthcare-phi-compliance** — data classification PHI, leak vector, audit theo NĐ13/HIPAA.
- **ecc:test-coverage** / **ecc:go-test** — dựng testcontainers RLS branch-isolation test đạt gate merge-blocking 80%.

## 9. Tài nguyên học thêm (2024–2026)

- PostgreSQL 16 — *Row Security Policies (§5.9)*: https://www.postgresql.org/docs/16/ddl-rowsecurity.html
- PostgreSQL 16 — *CREATE POLICY*: https://www.postgresql.org/docs/16/sql-createpolicy.html
- Crunchy Data — *Row Level Security for Tenants in Postgres*: https://www.crunchydata.com/blog/row-level-security-for-tenants-in-postgres
- Supabase — *Row Level Security*: https://supabase.com/docs/guides/database/postgres/row-level-security
- AWS Database Blog — *Multi-tenant data isolation with PostgreSQL RLS*: https://aws.amazon.com/blogs/database/multi-tenant-data-isolation-with-postgresql-row-level-security/
- testcontainers-go — *Postgres module*: https://golang.testcontainers.org/modules/postgres/
- pgx/v5 — *pool & transactions*: https://pkg.go.dev/github.com/jackc/pgx/v5/pgxpool
- NĐ 13/2023/NĐ-CP — Bảo vệ dữ liệu cá nhân (data residency onshore): tham chiếu canon §0.

## 10. Checklist "đã hiểu"

- [ ] Giải thích được vì sao `ENABLE` RLS một mình KHÔNG đủ (owner bypass) và `FORCE` sửa gì.
- [ ] Phân biệt được role `hms_migrator` (owner) vs `hms_app` (NOBYPASSRLS, không owner) và lý do tách (ADR-003).
- [ ] Viết được policy có cả `USING` và `WITH CHECK`, giải thích thiếu `WITH CHECK` mở write vector.
- [ ] Hiểu `SET LOCAL` chỉ sống trong tx và vì sao `SET SESSION` trên pool gây leak (risk §8).
- [ ] Biết `branch_id` lấy từ JWT verify, KHÔNG từ client; resource branch khác trả 404 không 403.
- [ ] Giải thích MPI `patients` dùng chung xuyên chi nhánh là exception policy (ADR-005).
- [ ] Viết được integration test branch-isolation bằng testcontainers và biết nó là merge-blocking gate (ADR-025).
- [ ] Hiểu vì sao RLS keystone phải ở migration 000001, không retrofit được (ADR-024, risk §8).
