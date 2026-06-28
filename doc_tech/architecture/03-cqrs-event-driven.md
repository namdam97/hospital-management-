# CQRS & Event-driven — Transactional Outbox in-process

> Module **ARCH-3** · CQRS read/write split + domain event qua transactional outbox in-process (KHÔNG broker ngoài ở MVP) · Độ khó: 🥉→🥇 · Prereqs: **ARCH-2** (DDD & Bounded Contexts, Encounter anchor)

Neo quyết định: **ADR-012** (một messaging mechanism = in-process outbox, một job framework = River), **ADR-011** (charge-capture idempotent + claim↔bill↔encounter FK + saga), **ADR-001** (modular monolith, cross-BC chỉ qua outbox), **ADR-004** (Encounter là mỏ neo, emit `EMRSigned`/`EncounterClosed`).

---

## 1. Vì sao kỹ năng này quan trọng trong HMS

Trong bệnh viện rời giấy, một thao tác lâm sàng KHÔNG dừng ở một bảng. Khi bác sĩ đóng Encounter và dược sĩ cấp phát thuốc, hệ quả lan ra nhiều bounded context: `billing` phải sinh `ChargeItem`, `insurance` phải dựng claim XML 4750, `audit` phải ghi vết, `pharmacy` phải trừ tồn FEFO. Nếu làm đồng bộ in-line (gọi hàm BC khác ngay trong handler) ta phá layer rule ADR-001 (cross-BC import) và tạo distributed-transaction giả trong monolith. Nếu làm async best-effort (goroutine sau commit) ta mất event khi crash → **charge bị bỏ rơi, claim không khớp bill** — đúng cái gap ADR-011 sửa.

Transactional outbox giải bài toán "đổi DB + phát event phải nguyên tử": event ghi vào bảng `outbox` trong **cùng `pgx.Tx`** với thay đổi nghiệp vụ. Hoặc cả hai commit, hoặc cả hai rollback. Một relay riêng đọc outbox và giao event cho subscriber. Đây là điểm bất biến nhất của thiết kế messaging HMS (ADR-012): "events sống in-process tới khi tách service được chứng minh; lúc đó chỉ swap relay adapter sang Kafka, domain code không đổi".

CQRS bổ trợ: BC phức tạp (`encounter`, `orders`, `lab`, `pharmacy`, `billing`, `insurance`) tách `app/command` (ghi, validate invariant, emit event) khỏi `app/query` (đọc, projection) — vì write-path lâm sàng cần aggregate + invariant chặt, còn read-path (báo cáo, danh sách hàng đợi) cần shape khác, hiệu năng khác.

## 2. Mô hình tư duy (first principles) — từ con số 0

Bắt đầu từ một sự thật vật lý: **bạn không thể commit nguyên tử qua hai hệ thống khác nhau mà không có 2PC** (vốn fragile, ta loại bỏ). Vậy nếu "thay đổi nghiệp vụ" và "phát event" nằm cùng MỘT hệ thống (PostgreSQL), ta được nguyên tử miễn phí qua một transaction.

```
        ┌─────────────── pgx.Tx (một transaction) ───────────────┐
write → │  INSERT charges (...)          ← thay đổi nghiệp vụ     │
        │  INSERT outbox (event_payload) ← event, cùng tx        │
        └──────────────────────── COMMIT ────────────────────────┘
                                    │
              relay (poll): SELECT ... FOR UPDATE SKIP LOCKED
                                    │
                        deliver → subscriber (idempotent qua processed_events)
```

Ba bất biến rút ra từ con số 0:
1. **Atomicity:** event chỉ tồn tại nếu nghiệp vụ commit. Không có "đã trừ tồn nhưng quên phát charge".
2. **At-least-once, không exactly-once:** relay có thể giao trùng (crash sau deliver, trước khi đánh dấu done). Vì vậy subscriber **bắt buộc idempotent** — đây là `processed_events`.
3. **Decoupling thời gian + module:** producer không biết ai consume; consumer chạy sau, có thể retry. Cross-BC không import nhau.

CQRS là first-principle thứ hai: **read và write có ràng buộc đối lập**. Write cần một aggregate nhất quán (Encounter + Diagnosis + Observation). Read cần "hàng đợi tiếp đón hôm nay" — một shape phẳng, nhiều row, không cần invariant. Ép cả hai vào một model = đắt cho cả hai.

## 3. Khái niệm cốt lõi (tăng dần độ khó)

**a. Domain event** — sự kiện đã-xảy-ra trong domain, đặt tên quá khứ: `EncounterClosed`, `EMRSigned`, `DrugDispensed`, `ResultReleased`, `PatientMerged`. Bất biến, mang đủ payload để subscriber hành động mà không query ngược.

**b. Transactional outbox** — bảng `outbox` (BIGINT IDENTITY theo datastore canon) ghi event trong cùng tx với write nghiệp vụ. Cột tối thiểu: `id BIGINT`, `aggregate_type`, `aggregate_id`, `event_type`, `payload JSONB`, `created_at`, `processed_at NULL`, `branch_id`.

**c. Relay (poll-and-deliver)** — vòng lặp đọc outbox chưa xử lý bằng `SELECT ... FOR UPDATE SKIP LOCKED` (nhiều worker không tranh nhau cùng row), giao cho subscriber, rồi set `processed_at`.

**d. Idempotent subscriber + `processed_events`** — subscriber ghi `(event_id, subscriber_name)` UNIQUE; nếu đã có → bỏ qua. Bảo vệ trước at-least-once.

**e. Command vs Query (CQRS)** — `app/command/*` nhận Command, load aggregate, enforce invariant, ghi + emit outbox event trong một tx. `app/query/*` đọc read-model, không chạm aggregate.

**f. Idempotency-Key end-to-end (ADR-011)** — FE và backend charge/claim dùng **MỘT** scheme key (unique-constraint `idempotency_keys`). Khác với `processed_events` (chống trùng KHI replay internal event); idempotency-key chống double-post của LỆNH từ ngoài vào.

**g. Saga** — luồng nhiều bước (quyết toán ra viện: giải phóng tạm ứng → chốt claim) điều phối qua chuỗi event + compensation, không 2PC.

**h. River vs relay** — River (Postgres-native, framework job DUY NHẤT, ADR-012) chạy việc nền có lịch/retry: claim submit/retry, FEFO sweep, outbox cleanup. Relay là cơ chế giao event in-process. KHÔNG ship River + Watermill song song.

## 4. HMS dùng nó thế nào (bám code path — *(planned)*, code chưa viết)

Repo CHƯA CÓ CODE; mọi path dưới là layout MỤC TIÊU (canon §9), đánh dấu *(planned)*.

- `internal/shared/outbox/` *(planned)* — bảng `outbox`, hàm `Enqueue(tx, event)` (ghi trong tx của caller), relay loop `SELECT ... FOR UPDATE SKIP LOCKED`, interface `Relay` để Phase 3 swap sang Kafka mà domain không đổi (ADR-012 trigger).
- `internal/billing/app/command/` *(planned)* — handler `CaptureCharge` consume `DrugDispensed`/`ResultReleased`/`OrderPlaced`, sinh `ChargeItem` vào Invoice của Encounter với Idempotency-Key (ADR-011).
- `processed_events` *(planned, owned bởi shared)* — UNIQUE`(event_id, subscriber)` đảm bảo charge-capture idempotent dưới at-least-once.
- `internal/encounter/app/command/` *(planned)* — `CloseEncounter`/`SignEMR` emit `EncounterClosed`/`EMRSigned` qua outbox (ADR-004), drive charge + claim + interop.
- `internal/insurance/` *(planned)* — consume charge events dựng `InsuranceClaim` từ chính `ChargeItem` (claim↔bill nhất quán), submit qua River saga + idempotency + retry (ADR-011, ADR-023 rejection-code state machine).
- `cmd/worker/main.go` *(planned)* — composition root chạy relay + River queues.

Sơ đồ luồng charge-capture (MVP, in-process):

```mermaid
sequenceDiagram
    participant Ph as pharmacy.command
    participant DB as PostgreSQL (outbox)
    participant Re as shared/outbox relay
    participant Bi as billing.command
    participant In as insurance (River saga)
    Ph->>DB: tx{ INSERT dispense + stock_ledger + outbox(DrugDispensed) } COMMIT
    Re->>DB: SELECT FOR UPDATE SKIP LOCKED
    Re->>Bi: deliver DrugDispensed
    Bi->>DB: tx{ check processed_events; INSERT ChargeItem (idem-key) } COMMIT
    Bi->>DB: emit ChargeCaptured (outbox)
    Re->>In: deliver ChargeCaptured
    In->>DB: build claim XML 4750 (River retry)
```

Code minh hoạ enqueue trong cùng tx *(planned, Go + pgx/v5)*:

```go
// internal/shared/outbox/outbox.go (planned)
func Enqueue(ctx context.Context, tx pgx.Tx, e Event) error {
    _, err := tx.Exec(ctx, `
        INSERT INTO outbox (aggregate_type, aggregate_id, event_type, payload, branch_id)
        VALUES ($1, $2, $3, $4, current_setting('app.current_branch')::uuid)`, // RLS GUC, ADR-005
        e.AggregateType, e.AggregateID, e.EventType, e.Payload)
    return err
}
```

Relay poll an toàn nhiều worker *(planned, SQL)*:

```sql
-- relay: chỉ một worker chiếm mỗi batch, không block worker khác (ADR-012)
WITH batch AS (
    SELECT id FROM outbox
    WHERE processed_at IS NULL
    ORDER BY id
    FOR UPDATE SKIP LOCKED
    LIMIT 100
)
UPDATE outbox o SET processed_at = now()
FROM batch WHERE o.id = batch.id
RETURNING o.id, o.event_type, o.payload;
```

## 5. Best practices (mỗi mục kèm nguồn đã research)

1. **Outbox ghi trong CÙNG transaction với write nghiệp vụ** — đây là toàn bộ giá trị của pattern; relay tách riêng. Nguồn: microservices.io — Transactional Outbox pattern, https://microservices.io/patterns/data/transactional-outbox.html
2. **Subscriber idempotent (xử lý at-least-once)** — outbox/relay là at-least-once, exactly-once delivery là ảo tưởng; khử trùng ở consumer. Nguồn: Microsoft Azure Architecture — Idempotent message processing, https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/containers/aks-mqtt-broker/idempotent-message-processing
3. **`FOR UPDATE SKIP LOCKED` cho queue/relay trên Postgres** — cho phép nhiều worker xử lý song song không tranh chấp, không lock-wait. Nguồn: PostgreSQL 16 docs — The Locking Clause (SKIP LOCKED), https://www.postgresql.org/docs/16/sql-select.html#SQL-FOR-UPDATE-SHARE
4. **River cho background job Postgres-native, transaction-aware insert** — chỉ một job framework; River insert job trong cùng tx như outbox để không "phát job rồi rollback DB". Nguồn: River docs — Transactional enqueueing, https://riverqueue.com/docs/transactional-enqueueing
5. **Saga thay 2PC cho luồng nhiều bước/nhiều BC** — quyết toán ra viện, claim submit là chuỗi local-tx + compensation. Nguồn: microservices.io — Saga pattern, https://microservices.io/patterns/data/saga.html
6. **CQRS chỉ tách khi read/write thực sự lệch nhu cầu** — đừng CQRS-hoá BC `clean` đơn giản (identity/organization); chỉ BC `clean+ddd+cqrs` (canon §4). Nguồn: Microsoft — CQRS pattern, https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs

## 6. Lỗi thường gặp & anti-patterns

- **"Dual write" (anti-pattern chí mạng):** ghi DB rồi gọi event bus ngoài tx → crash giữa chừng = state lệch event. Đây CHÍNH là lý do outbox tồn tại; ADR-012 cấm broker ngoài ở MVP nên không có cám dỗ này — nhưng đừng tự dựng goroutine "phát event sau commit".
- **Subscriber không idempotent:** charge-capture chạy hai lần khi relay retry → **double charge** người bệnh. Bắt buộc check `processed_events` (ADR-011).
- **Nhầm `processed_events` với `idempotency_keys`:** cái đầu chống trùng event NỘI BỘ (at-least-once); cái sau chống double-post LỆNH NGOÀI (FE replay). Phải tồn tại CẢ HAI, cùng một scheme key end-to-end (ADR-011, risk §8).
- **Cross-BC import thay vì event:** gọi thẳng `insurance` từ `billing` phá ADR-001; depguard (golangci-lint) chặn ở CI.
- **Outbox không mang `branch_id` / event xử lý ngoài tx có SET LOCAL GUC:** mất RLS context (ADR-005, risk critical §8) → leak/insert sai branch. Relay phải SET LOCAL `app.current_branch` từ event trước khi subscriber chạy query PHI.
- **CQRS read-model realtime bằng CDC ở MVP:** ADR-012 loại Debezium/Kafka cho analytics MVP — dùng scheduled SQL→read-table. CDC là Phase 3.
- **Ship River + Watermill song song:** overlap, ADR-012 cấm — một job framework duy nhất.
- **Event "anemic" thiếu payload:** subscriber phải query ngược aggregate → coupling + N+1. Mang đủ dữ liệu trong payload.

## 7. Lộ trình luyện tập NGAY trong repo

> Repo chưa có code — bài tập là dựng scaffold MỤC TIÊU theo canon §9, hoặc thiết kế trên giấy/migration nếu chưa tới Phase build.

- 🥉 **Cơ bản:** Viết migration `outbox` + `processed_events` (BIGINT IDENTITY, branch_id, FORCE RLS) như một phần `backend/migrations/` *(planned)*. Vẽ lại sơ đồ sequence charge-capture cho luồng `ResultReleased → ChargeCaptured`. Liệt kê 5 domain event MVP và payload tối thiểu của từng cái.
- 🥈 **Trung cấp:** Scaffold `internal/shared/outbox/` *(planned)* với `Enqueue(tx, event)` + relay `SELECT FOR UPDATE SKIP LOCKED`. Viết một subscriber `billing` idempotent qua `processed_events`. Chứng minh bằng table-driven unit test: deliver cùng event 2 lần → chỉ 1 `ChargeItem`.
- 🥇 **Nâng cao:** Viết integration test (testcontainers, ADR-025) chạy relay thật trên Postgres: (1) crash giữa deliver và `processed_at` → event được giao lại, charge KHÔNG nhân đôi; (2) hai relay worker song song không xử lý trùng row (SKIP LOCKED); (3) outbox event mang `branch_id`, subscriber query dưới RLS GUC đúng branch (ADR-003/005). Thêm một saga quyết toán ra viện 2 bước với compensation.

## 8. Skill/agent ECC nên dùng khi luyện

- `ecc:go-review` (go-reviewer agent) — review concurrency của relay (race trên `processed_at`, goroutine leak), error handling fail-closed.
- `ecc:go-test` / `ecc:test-coverage` — TDD table-driven cho subscriber idempotent, gate 80% (testing rule).
- `ecc:postgres-patterns` — kiểm `FOR UPDATE SKIP LOCKED`, index `(processed_at, id)` cho relay, JSONB payload.
- `ecc:hexagonal-architecture` — giữ `Relay`/`EventBus` là port để Phase 3 swap Kafka mà domain không đổi (ADR-012).
- `ecc:database-migrations` — migration outbox/processed_events zero-downtime (ADR-024).
- `ecc:security-review` / `ecc:security-scan` — đảm bảo event PHI mang branch_id, không leak qua relay ngoài RLS context.

## 9. Tài nguyên học thêm (2024–2026)

- microservices.io — Transactional Outbox & Saga & CQRS (Chris Richardson): https://microservices.io/patterns/data/transactional-outbox.html
- River (Postgres-native jobs, Go) docs: https://riverqueue.com/docs
- PostgreSQL 16 — SELECT … FOR UPDATE … SKIP LOCKED: https://www.postgresql.org/docs/16/sql-select.html
- Microsoft Azure Architecture Center — CQRS & Idempotent processing: https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs
- "Implementing the Outbox Pattern" — Debezium blog (khái niệm, KHÔNG dùng Debezium ở MVP per ADR-012): https://debezium.io/blog/2019/02/19/reliable-microservices-data-exchange-with-the-outbox-pattern/
- pgx/v5 (PostgreSQL driver Go) docs: https://pkg.go.dev/github.com/jackc/pgx/v5

## 10. Checklist "đã hiểu"

- [ ] Giải thích được vì sao outbox event PHẢI ghi trong cùng `pgx.Tx` với write nghiệp vụ (atomicity, chống dual-write).
- [ ] Phân biệt rõ `processed_events` (idempotent subscriber, at-least-once) vs `idempotency_keys` (chống double-post lệnh ngoài, ADR-011) và biết cần CẢ HAI.
- [ ] Hiểu `FOR UPDATE SKIP LOCKED` cho phép nhiều relay worker song song không tranh chấp.
- [ ] Biết relay phải SET LOCAL `app.current_branch` từ `branch_id` của event trước khi subscriber query PHI (ADR-003/005).
- [ ] Phân biệt relay (giao event in-process) vs River (job có lịch/retry) — một job framework duy nhất (ADR-012).
- [ ] Biết khi nào CQRS (chỉ BC `clean+ddd+cqrs`), khi nào KHÔNG (BC `clean` đơn giản).
- [ ] Hiểu trigger swap relay→Kafka (ADR-012) và vì sao domain code không đổi (port interface).
- [ ] Mô tả được saga quyết toán ra viện thay 2PC (local-tx + compensation, ADR-011).
- [ ] Biết vì sao analytics MVP dùng scheduled SQL→read-table, KHÔNG CDC/Kafka (ADR-012, Phase 3).
