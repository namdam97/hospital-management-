# [BE-4] Background jobs với River (Postgres-native)

> Module BE-4 · Background jobs trên Postgres bằng River — claim retry, FEFO sweep, outbox cleanup, audit retention · Độ khó: 🥉→🥇 · Prereqs: BE-3 (pgx/v5 + sqlc + golang-migrate), DATA-1 (RLS), ARCH-3 (outbox)

Tài liệu liên quan: [BE-3 PostgreSQL pgx/migrations](./03-postgres-pgx-migrations.md) · [ARCH-3 CQRS & Transactional Outbox](../architecture/03-cqrs-event-driven.md) · [DOM-2 CDSS/FEFO/charge-capture](../domain-clinical/02-cdss-fefo-charge-capture.md) · [INT-1 BHYT 4750 + e-prescription](../interoperability/01-coded-data-bhyt-eprescription.md) · `doc/02-backend-architecture.md` · `doc/10-deployment-operations.md`

---

## 1. Vì sao kỹ năng này quan trọng trong HMS

Một bệnh viện rời giấy không thể để mọi việc xảy ra **đồng bộ trong request của người bệnh**. Khi thu ngân quyết toán, không thể bắt người bệnh đứng chờ trong khi hệ thống gọi cổng giám định BHYT đang quá tải; khi bác sĩ kê đơn, không thể block màn hình chờ donthuocquocgia.vn trả mã đơn quốc gia. Những việc *chậm, có thể fail, cần retry, cần chạy định kỳ* phải tách ra **background job**.

Trong HMS đây là các job **load-bearing** (neo theo canon):
- **Claim submission + retry** (ADR-011, ADR-006, ADR-023): đẩy bộ XML1–XML15 QĐ 4750 lên cổng giám định BHYT qua saga, **at-least-once + idempotent** (dedupe trên claim reference). Cổng chậm/quá tải là điều bình thường → retry với backoff, không double-submit.
- **donthuoc submission + retry** (ADR-007): đẩy đơn ngoại trú lên donthuocquocgia.vn sau khám lấy mã đơn quốc gia (TT 26/2025); national system down → retry idempotent.
- **FEFO expiry sweep** (ADR-021): quét `medication_lots` cận hạn/hết hạn, sinh `StockExpiring` / `ReorderAlert`.
- **Outbox cleanup + relay health** (ADR-012): dọn outbox đã xử lý, theo dõi backlog.
- **Audit retention** (ADR-009): partition rotation + ship batch tới WORM sink, **không bao giờ xóa** trong cửa sổ giữ luật.
- **Reminders** (scheduling-reception): nhắc lịch hẹn sau `AppointmentBooked`.

Quyết định kiến trúc cứng: **River (Postgres-native) là DUY NHẤT framework background job — KHÔNG ship River + Watermill song song** (ADR-012). Lý do: monolith một process với zero cross-process consumer không cần hai cơ chế Postgres-native chồng nhau.

---

## 2. Mô hình tư duy (first principles) — từ con số 0

Một background job system chỉ cần trả lời 4 câu hỏi:

1. **Lưu việc cần làm ở đâu?** → một hàng trong bảng (job = một row, có state, args, attempt count).
2. **Ai lấy việc ra chạy?** → worker poll bảng, claim job bằng `SELECT ... FOR UPDATE SKIP LOCKED` (đúng pattern outbox ở ARCH-3) để N worker không giành cùng một job.
3. **Fail thì sao?** → tăng `attempt`, tính `scheduled_at = now() + backoff`, trả về queue; quá `max_attempts` → đẩy vào trạng thái `discarded` (dead-letter).
4. **Chạy lại có an toàn không?** → **at-least-once** nghĩa là job CÓ THỂ chạy >1 lần (crash sau side-effect, trước khi đánh dấu done). Vì vậy **mọi job handler phải idempotent**.

River chính là bảng-job + worker-loop + backoff + leader election + cron, **đặt ngay trong Postgres của bạn** — cùng database với domain data. Điều này cho một thứ vô giá: **enqueue job trong CÙNG `pgx.Tx` với thay đổi domain**. Charge được ghi và claim-submit job được enqueue *atomically* — hoặc cả hai commit, hoặc cả hai rollback. Không có "ghi DB xong nhưng mất job" như khi dùng broker ngoài.

Tư duy nền: **transactional outbox và River là hai mặt của một đồng tiền** — cả hai khai thác ACID của Postgres để thay thế message broker. Outbox = pub/sub in-process (ARCH-3); River = scheduled/retried work. Chúng dùng chung database, chung pattern `SKIP LOCKED`, chung triết lý "đừng thêm stateful system khi Postgres đủ" (ADR-002).

---

## 3. Khái niệm cốt lõi (tăng dần độ khó)

**(a) Job args & worker.** Một job = struct args (JSON-serialized) + `Worker` xử lý. River dùng generics: `river.Worker[T]` với `Work(ctx, *river.Job[T]) error`. `Kind()` định danh loại job.

**(b) Insert trong transaction.** `riverpgxv5.New(pool)` + `client.InsertTx(ctx, tx, args, opts)` — enqueue cùng tx với domain write. Đây là tính năng quyết định việc chọn River cho HMS.

**(c) Queue & priority.** Job vào queue đặt tên (`claim`, `donthuoc`, `maintenance`); mỗi queue có `MaxWorkers` riêng. Tách queue để claim-submit không bị reminder làm nghẽn.

**(d) Retry & backoff.** Mặc định exponential backoff, `MaxAttempts` cấu hình per-job. Override `NextRetry(job)` để tùy chỉnh (vd cổng BHYT trả 429 → backoff dài hơn). Hết attempt → `discarded` (dead-letter), không mất.

**(e) Uniqueness / idempotency-at-enqueue.** `UniqueOpts` (theo args/queue/period) chống enqueue trùng. Nhưng đây CHỈ là lớp một; lớp hai là **handler idempotent** (dedupe trên claim reference) vì at-least-once vẫn có thể chạy lại.

**(f) Periodic jobs (cron).** `PeriodicJobs` chạy theo lịch với **leader election** (chỉ một process trong cluster chạy) — đúng cho FEFO sweep, outbox cleanup, audit retention khi có ≥3 replica (canon deploy).

**(g) Snooze / Cancel / Pause.** `river.JobSnooze` hoãn job (vd cổng đang bảo trì); `JobCancel` dừng vĩnh viễn. Hỗ trợ degraded-mode (ADR-006).

**(h) Worker process tách biệt.** Canon planned: `cmd/worker/` *(planned)* là process River riêng, scale độc lập với `cmd/hms-api/`. Cả hai dùng chung `internal/shared/jobs` *(planned)*.

---

## 4. HMS dùng nó thế nào (bám code path:line — *(planned)*, code chưa viết)

Layout mục tiêu (canon §9), tất cả *(planned)*:

```
backend/
├── cmd/worker/main.go                       # River worker process (planned)
├── cmd/hms-api/main.go                       # enqueue qua client trong handler/usecase (planned)
├── migrations/000xxx_river_tables.up.sql     # river migrate (planned)
└── internal/
    ├── shared/jobs/                          # River client, registry, wiring (planned)
    │   ├── client.go       # riverpgxv5 + Config + queues
    │   └── workers.go      # AddWorker registry
    ├── insurance/adapters/jobs/claim_submit.go   # ClaimSubmitWorker (planned)
    ├── pharmacy/adapters/jobs/donthuoc_submit.go # DonthuocSubmitWorker (planned)
    ├── pharmacy/adapters/jobs/fefo_sweep.go      # FefoExpirySweepWorker (planned)
    └── audit/adapters/jobs/retention.go          # AuditRetentionWorker (planned)
```

**Wiring River client trong shared/jobs/client.go *(planned)*** — neo ADR-012, ADR-015 (cùng pool Postgres):

```go
// internal/shared/jobs/client.go (planned)
func NewClient(pool *pgxpool.Pool, w *river.Workers) (*river.Client[pgx.Tx], error) {
    return river.NewClient(riverpgxv5.New(pool), &river.Config{
        Queues: map[string]river.QueueConfig{
            "claim":       {MaxWorkers: 4},  // BHYT 4750 submit/retry
            "donthuoc":    {MaxWorkers: 4},  // donthuocquocgia.vn
            "maintenance": {MaxWorkers: 2},  // FEFO sweep, outbox cleanup, audit retention
            "reminders":   {MaxWorkers: 2},
        },
        Workers:     w,
        PeriodicJobs: periodicJobs(), // cron: FEFO sweep, outbox cleanup, audit retention
    })
}
```

**Enqueue claim-submit CÙNG tx với charge-capture *(planned)*** — neo ADR-011 (claim sinh từ ChargeItem, at-least-once + idempotent):

```go
// internal/insurance/app/command/submit_claim.go (planned)
func (h *SubmitClaimHandler) Handle(ctx context.Context, cmd SubmitClaim) error {
    return pgx.BeginTxFunc(ctx, h.pool, func(tx pgx.Tx) error {
        // SET LOCAL app.current_branch (DATA-1 RLS contract) đã do middleware đặt
        ref, err := h.repo.PersistClaimXml(ctx, tx, cmd.ClaimID) // claim_xml_records (XML1..15)
        if err != nil { return err }
        // enqueue trong CÙNG tx — claim ghi và job enqueue atomic
        _, err = h.river.InsertTx(ctx, tx, ClaimSubmitArgs{
            ClaimID: cmd.ClaimID, ClaimRef: ref, // ClaimRef = dedupe key idempotent
        }, &river.InsertOpts{
            Queue:       "claim",
            MaxAttempts: 12, // cổng giám định quá tải/chậm — retry kiên nhẫn (ADR-006)
            UniqueOpts:  river.UniqueOpts{ByArgs: true}, // chống double-enqueue
        })
        return err
    })
}
```

**Worker idempotent + degraded-aware *(planned)*** — neo ADR-006 (degraded-mode), ADR-023 (rejection-code state machine):

```go
// internal/insurance/adapters/jobs/claim_submit.go (planned)
func (w *ClaimSubmitWorker) Work(ctx context.Context, job *river.Job[ClaimSubmitArgs]) error {
    // 1) idempotent: nếu cổng đã nhận claimRef này → coi như done (at-least-once)
    if done, _ := w.gateway.AlreadyAccepted(ctx, job.Args.ClaimRef); done {
        return nil
    }
    resp, err := w.gateway.Submit(ctx, job.Args.ClaimID) // mTLS + chữ ký số (ADR-021)
    if err != nil {
        if errors.Is(err, gateway.ErrOverloaded) {
            return river.JobSnooze(2 * time.Minute) // degraded: hoãn, không đốt attempt
        }
        return err // tăng attempt + backoff; hết → discarded (dead-letter)
    }
    // rejection-code là state machine first-class, KHÔNG nuốt lỗi (ADR-023)
    return w.repo.RecordResponse(ctx, job.Args.ClaimID, resp) // claim_responses
}
```

**Periodic FEFO sweep *(planned)*** — neo ADR-021 (FEFO, stock_ledger append-only):

```go
// internal/pharmacy/adapters/jobs/fefo_sweep.go (planned) — chạy 1 leader trong cluster
func (w *FefoExpirySweepWorker) Work(ctx context.Context, _ *river.Job[FefoSweepArgs]) error {
    // quét medication_lots ORDER BY expiry_date ASC, sinh StockExpiring/ReorderAlert qua outbox
    return w.svc.SweepExpiringLots(ctx, time.Now())
}
```

> Lưu ý RLS (DATA-1, ADR-003): periodic maintenance job chạy NGOÀI request của user → KHÔNG có `app.current_branch` từ JWT. Job phải hoặc chạy dưới role có policy `cross_branch_reader` được kiểm soát, hoặc lặp tường minh theo từng `branch_id` và `SET LOCAL` cho mỗi vòng. **Không** vô tình chạy job dưới role bypass-RLS.

---

## 5. Best practices (mỗi mục kèm 1 nguồn đã research)

1. **Enqueue trong transaction với domain write** — dùng `InsertTx`/`InsertManyTx` để job và data commit nguyên tử; đây là lý do gốc chọn River trên Postgres. Nguồn: River docs — Transactional enqueueing, https://riverqueue.com/docs/transactional-enqueueing
2. **Mọi handler idempotent** — at-least-once nghĩa là job chạy ≥1 lần; thiết kế `Work` an toàn khi re-run (dedupe trên claim reference / external id). Nguồn: River docs — Unique jobs & idempotency, https://riverqueue.com/docs/unique-jobs
3. **Tách queue + giới hạn MaxWorkers** để công việc nặng (claim submit) không bỏ đói công việc nhẹ; cấu hình concurrency per-queue. Nguồn: River docs — Queues, https://riverqueue.com/docs/queues
4. **Periodic jobs dùng leader election** — chỉ một instance chạy cron job, an toàn với ≥3 replica HMS. Nguồn: River docs — Periodic & cron jobs, https://riverqueue.com/docs/periodic-jobs
5. **Backoff + giới hạn attempt + dead-letter** — đặt `MaxAttempts`, override `NextRetry` cho external gateway 429/5xx; job hết attempt vào `discarded`, không mất. Nguồn: River docs — Job retries & error handling, https://riverqueue.com/docs/error-handling
6. **Đặt timeout cho external call trong job** — gói gateway call bằng `context.WithTimeout`; cổng BHYT/donthuoc chậm phải bị cắt để worker không treo. Nguồn: Go blog — Context, https://go.dev/blog/context
7. **Quan sát hàng đợi (queue depth, retry rate)** — export metric backlog/attempt để Alertmanager burn-rate (canon observability) cảnh báo cổng down. Nguồn: River docs — Observability/monitoring, https://riverqueue.com/docs/monitoring

---

## 6. Lỗi thường gặp & anti-patterns

- **Chạy River + Watermill (hoặc thêm broker) song song.** Vi phạm trực tiếp ADR-012 — hai cơ chế Postgres-native chồng nhau, zero giá trị cho monolith một process. Một framework job duy nhất.
- **Enqueue NGOÀI transaction của domain write.** Charge commit nhưng claim-submit job mất (hoặc ngược lại) khi crash giữa hai bước. Luôn `InsertTx` trong cùng `pgx.Tx`.
- **Handler không idempotent.** Retry/replay → double-submit claim lên cổng giám định hoặc double-post donthuoc. Phải dedupe trên claim reference (ADR-011) — đây là cùng-một-scheme idempotency end-to-end với FE+backend (Open Risk: queued-then-replayed double-post).
- **Nuốt lỗi rejection BHYT.** `return nil` khi cổng từ chối → mất state machine rejection-code (ADR-023). Lỗi nghiệp vụ phải thành state, lỗi hạ tầng mới retry.
- **Fail-open trên việc an toàn.** Nếu một job liên quan an toàn (vd kiểm tra trước cấp phát) — nhưng nhắc lại: CDSS hard-stop là **synchronous server-side**, KHÔNG đẩy ra background (ADR-008). Đừng biến safety-check thành job async.
- **Periodic job chạy dưới role bypass-RLS** → leak/ghi cross-branch (ADR-003). Job maintenance phải tôn trọng session-var contract hoặc lặp theo branch tường minh.
- **Không timeout external call** → một cổng treo làm cạn worker pool, kéo cả queue claim chết.
- **Xóa audit_log trong job retention.** ADR-009: audit INSERT-only, ship tới WORM; retention job archive/rotate partition, **không DELETE** trong cửa sổ giữ luật.

---

## 7. Lộ trình luyện tập NGAY trong repo (🥉 → 🥈 → 🥇)

> Repo chưa có code — bài tập thao tác trên layout *(planned)* canon §9. Mỗi bài: viết test trước (TDD red-green-refactor, testing rule 80%).

**🥉 Cơ bản — chạy River trên Postgres testcontainer.**
1. Tạo `backend/migrations/000xxx_river_tables.up.sql` *(planned)* bằng `river migrate-get --up`.
2. Định nghĩa `ReminderArgs` + `ReminderWorker` (log "nhắc lịch hẹn") trong `internal/scheduling/adapters/jobs/` *(planned)*.
3. Viết integration test (testcontainers-go, TEST-1) enqueue + assert worker chạy đúng một lần.

**🥈 Trung cấp — claim-submit idempotent + retry.**
1. Viết `ClaimSubmitWorker` *(planned)* với fake gateway trả `ErrOverloaded` 2 lần rồi success; assert job retry rồi done.
2. Thêm test **idempotency**: gọi `Work` 2 lần với cùng `ClaimRef` → cổng chỉ nhận đúng 1 lần (`AlreadyAccepted`).
3. Test enqueue trong tx: rollback tx → assert KHÔNG có job nào được tạo (atomic với domain write).

**🥇 Nâng cao — periodic FEFO sweep + RLS-safe + dead-letter.**
1. Viết `FefoExpirySweepWorker` periodic *(planned)*; seed `medication_lots` nhiều branch; assert sweep tôn trọng RLS (không quét nhầm branch) — chạy under real PG (DATA-1 branch-isolation pattern).
2. Cấu hình `MaxAttempts` + override `NextRetry`; ép gateway luôn fail; assert job vào `discarded` và metric backlog phản ánh đúng.
3. Viết audit-retention job *(planned)* rotate partition tháng cũ + assert KHÔNG có DELETE record (ADR-009), chỉ archive ship WORM.

---

## 8. Skill/agent ECC nên dùng khi luyện

- **`ecc:go-test`** — enforce TDD table-driven cho worker logic + verify coverage 80% (testing rule).
- **`ecc:go-review`** — review idempotency, context-timeout, error-vs-state-machine handling trong worker.
- **`ecc:golang-patterns`** + **`ecc:golang-testing`** — pattern Go production + testing background worker.
- **`ecc:postgres-patterns`** — kiểm `SKIP LOCKED`, index trên bảng River, RLS interaction cho periodic job.
- **`ecc:database-migrations`** — viết migration River tables zero-downtime (ADR-024 CONCURRENTLY ngoài tx).
- **`ecc:go-build`** — nếu build/vet River wiring fail (generics worker registry).
- **`ecc:security-review`** — periodic job role không bypass-RLS; external client cert/secret (ADR-021).
- **`ecc:healthcare-emr-patterns`** / **`ecc:healthcare-phi-compliance`** — audit-retention WORM + claim flow đúng compliance.

---

## 9. Tài nguyên học thêm (2024–2026)

- River — Documentation (transactional enqueue, unique jobs, periodic, error handling): https://riverqueue.com/docs
- River — GitHub repo (`riverqueue/river`, `riverpgxv5` driver): https://github.com/riverqueue/river
- Brandur Leach — "Building River, a robust high-performance job queue in Go" (thiết kế River, SKIP LOCKED): https://brandur.org/river
- "Transactional outbox pattern" — Microservices.io: https://microservices.io/patterns/data/transactional-outbox.html
- pgx/v5 — Transactions & pgxpool: https://pkg.go.dev/github.com/jackc/pgx/v5
- Go blog — Context (timeout/cancel cho external call): https://go.dev/blog/context
- QĐ 4750/QĐ-BYT (sửa QĐ 3176) — bộ XML giám định BHYT 1–15 (bối cảnh claim-submit job): tra cứu Cổng giám định BHYT (giamdinhbhyt.baohiemxahoi.gov.vn).

---

## 10. Checklist "đã hiểu"

- [ ] Giải thích được vì sao River (Postgres-native) là **framework job DUY NHẤT** trong HMS và vì sao KHÔNG ship Watermill song song (ADR-012).
- [ ] Enqueue được job CÙNG `pgx.Tx` với domain write bằng `InsertTx` và hiểu vì sao điều này thay thế broker ngoài.
- [ ] Hiểu **at-least-once → handler phải idempotent**; viết được dedupe trên claim reference (ADR-011).
- [ ] Biết tách queue + `MaxWorkers` để claim-submit không bỏ đói reminder; đặt `MaxAttempts` + backoff cho cổng quá tải.
- [ ] Phân biệt **lỗi hạ tầng (retry)** vs **rejection nghiệp vụ (state machine, ADR-023)** trong worker BHYT.
- [ ] Dùng `JobSnooze` cho degraded-mode (ADR-006) và biết job hết attempt vào `discarded` (dead-letter), không mất.
- [ ] Hiểu periodic job dùng **leader election** và phải **RLS-safe** (không chạy dưới role bypass-RLS, ADR-003).
- [ ] Biết NÊN giữ CDSS hard-stop **synchronous** (ADR-008), KHÔNG đẩy safety-check ra background.
- [ ] Audit-retention job **archive/rotate + ship WORM**, KHÔNG DELETE trong cửa sổ giữ luật (ADR-009).
- [ ] Đặt `context.WithTimeout` cho mọi external call trong job và export metric queue depth cho Alertmanager.
