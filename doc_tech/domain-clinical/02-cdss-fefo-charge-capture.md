# [DOM-2] CDSS, FEFO & Charge-capture — an toàn người bệnh + viện phí không sai sót

> Module DOM-2 · CDSS hard-stop fail-closed, cấp phát FEFO theo lô/hạn, và charge-capture idempotent claim↔bill↔encounter · Độ khó: 🥈→🥇 · Prereqs: DOM-1 (patient journey), ARCH-3 (CQRS & transactional outbox)

Module này hợp nhất ba cơ chế bậc nhất của vòng OPD-BHYT mà nếu làm sai sẽ giết người bệnh hoặc làm thất thoát tiền: **CDSS** (Clinical Decision Support — chặn dị ứng/tương tác thuốc), **FEFO** (First-Expired-First-Out — cấp phát đúng lô/hạn), và **charge-capture** (tự động sinh viện phí + claim BHYT từ mọi order/dispense). Ba thứ này gặp nhau tại `internal/pharmacy` và `internal/billing` *(planned)* và đều neo vào `Encounter`. Neo: ADR-008 (CDSS), ADR-021 (FEFO + PCI scope), ADR-011 (charge-capture idempotent + saga).

---

## 1. Vì sao kỹ năng này quan trọng trong HMS

Trên giấy, dược sĩ tự nhớ dị ứng của bệnh nhân, tự nhìn hạn dùng trên hộp thuốc, và thu ngân gõ tay từng dòng tiền vào Excel rồi đối soát cuối ngày. Cả ba đều là **single point of human failure**:

- **CDSS** — Một bệnh nhân có dị ứng penicillin đã ghi trong hồ sơ, bác sĩ kê amoxicillin lúc 11h trưa đông bệnh: nếu hệ thống không **chặn ở backend**, modal cảnh báo trên React có thể bị bỏ qua (devtools, gọi thẳng API, mạng rớt modal không hiện). Người bệnh sốc phản vệ. ADR-008 biến CDSS từ "popup UX" thành **control fail-closed enforce ở aggregate**: command bị reject trừ khi có override record (lý do + người duyệt) ghi audit. Khi check lỗi/timeout → **KHÔNG BAO GIỜ** xác nhận "no known interaction".
- **FEFO** — Thuốc cận hạn không cấp trước thì hết hạn phải hủy (lãng phí tiền BHYT + ngân sách), hoặc tệ hơn cấp nhầm thuốc đã hết hạn. ADR-021 buộc cấp phát `ORDER BY expiry_date ASC` trong transaction có `FOR UPDATE SKIP LOCKED` để hai dược sĩ cùng quầy không trừ trùng một lô.
- **Charge-capture** — Excel đối soát cuối ngày là thứ chuyển đổi số phải xóa sổ. Mỗi order/dispense/dịch vụ phải **tự động** sinh `ChargeItem` vào `Invoice` của Encounter, idempotent (retry không double-post), và `InsuranceClaim` sinh từ **chính** `ChargeItem` đó để claim↔bill↔encounter luôn nhất quán (ADR-011). Sai linkage = giám định BHYT từ chối = bệnh viện mất tiền.

Đây là nơi *patient-safety lens* và *revenue-integrity* gặp nhau — và cũng là nơi council đánh dấu **critical risk** (xem canon §8: CDSS/audit fail-open, double-post idempotency).

---

## 2. Mô hình tư duy (first principles)

Xuất phát từ con số 0, ba câu hỏi nền tảng:

1. **Ai là người ra quyết định cuối cùng — UI hay server?** Nếu một control bảo vệ tính mạng, nó **phải** sống ở nơi không bỏ qua được: backend aggregate, trong cùng transaction với hành động nó bảo vệ. UI chỉ là tiện nghi. Đây là nguyên lý *enforcement at the trust boundary*.
2. **Khi không biết, mặc định là gì — an toàn hay nguy hiểm?** Fail-closed nghĩa là: khi không chắc chắn (CDSS timeout, chưa có allergy data), hệ thống chọn trạng thái **ngăn chặn**, không phải cho qua. "Allergy unknown" ≠ "safe" — đây là phân biệt sống còn (ADR-008).
3. **Một hành động lặp lại có được tính nhiều lần không?** Mạng chập chờn → client retry → cùng một dispense gửi 2 lần. Nếu mỗi lần đều sinh charge mới, bệnh nhân bị tính tiền 2 lần. **Idempotency** = cùng một khóa logic → đúng một hiệu ứng, dù gọi bao nhiêu lần (ADR-011).

Ba nguyên lý này quy về một meta-nguyên lý: **trong y tế, mặc định phải nghiêng về an toàn và đúng đắn, chi phí là thêm latency/phức tạp — và chi phí đó luôn đáng**.

---

## 3. Khái niệm cốt lõi (tăng dần độ khó)

### 3.1 CDSS hard-stop (🥈)
- **Aggregate invariant**: Lệnh `PrescribeMedication` / `DispenseMedication` không thể hoàn tất nếu vi phạm dị ứng/tương tác đã biết.
- **Override record**: cách hợp lệ DUY NHẤT để vượt hard-stop — gồm `reason`, `authorizer`, ghi vào `interaction_overrides` + audit. Không có "tắt cảnh báo".
- **Allergy-unknown là first-class state**: ba trạng thái phân biệt — `safe` / `conflict` / `unknown`. `unknown` không bao giờ collapse thành `safe`.

### 3.2 Fail-closed semantics (🥈)
- Khi engine CDSS error/timeout: trả `check_failed`, command bị reject (không phải "pass").
- Hard-online gate cho dispense: nếu không gọi được CDSS thì không cấp phát (ADR-018 cắt PWA write-outbox đúng vì lý do này).

### 3.3 FEFO + lot/expiry + stock ledger (🥈→🥇)
- `medication_lots(lot_no, expiry_date, qty)`: chọn `ORDER BY expiry_date ASC`.
- Concurrency: `SELECT ... FOR UPDATE SKIP LOCKED` để hai phiên cấp phát song song không khóa nhau và không trừ trùng.
- `stock_ledger` **append-only** (`BIGINT IDENTITY`, `movement_type`, `qty_delta`, `balance_after`) — audit trail kho, không UPDATE/DELETE.

### 3.4 Charge-capture idempotent (🥇)
- Mọi order/dispense emit domain event qua **transactional outbox** (ARCH-3) → billing subscriber sinh `ChargeItem`.
- `Idempotency-Key` persisted unique-constraint (`idempotency_keys`): replay an toàn.
- **Price snapshot** tại thời điểm charge (giá từ `service_catalog` chargemaster — không tham chiếu live, giá có thể đổi sau).

### 3.5 Claim↔bill↔encounter linkage + saga (🥇)
- `InsuranceClaim` sinh từ chính `ChargeItem` → claim & bill nhất quán by construction.
- FK cứng: `charges.encounter_id`, `bills`↔`charges`, `insurance_claims`↔`bills`.
- Quyết toán ra viện = saga (giải phóng advance + chốt claim) qua outbox; claim submission là River job at-least-once + idempotent external call.

---

## 4. HMS dùng nó thế nào (bám code path — *(planned)*)

Repo HIỆN CHƯA CÓ CODE; dưới đây là layout mục tiêu (canon §9). Cross-BC chỉ qua outbox in-process, KHÔNG import chéo BC.

| Cơ chế | Path *(planned)* | BC · arch style |
|---|---|---|
| CDSS hard-stop | `internal/pharmacy/domain/` (invariant), `internal/pharmacy/app/command/` (PrescribeMedication) | pharmacy · clean+ddd+cqrs |
| Allergy feed | `internal/patient/` → `patient_allergies` (đọc qua port, không import chéo) | patient · clean |
| FEFO dispense | `internal/pharmacy/adapters/postgres/` (FOR UPDATE SKIP LOCKED) | pharmacy |
| Stock ledger | `medication_lots` + `stock_ledger` (append-only) | pharmacy/inventory chung |
| Outbox emit | `internal/shared/outbox/` (SKIP LOCKED relay) | shared |
| Charge-capture | `internal/billing/app/command/` subscriber `DrugDispensed`/`OrderPlaced` | billing · clean+ddd+cqrs |
| Idempotency | `idempotency_keys` (unique-constraint) | billing |
| Claim từ ChargeItem | `internal/insurance/app/command/` | insurance · clean+ddd+cqrs |
| Saga quyết toán | `internal/billing/app/command/` orchestrator + River | billing |

**CDSS aggregate (planned, Go)** — enforce ở `app/command`, không ở handler HTTP:

```go
// internal/pharmacy/app/command/prescribe.go  (planned)
func (h *PrescribeHandler) Handle(ctx context.Context, cmd PrescribeCmd) (PrescriptionID, error) {
    allergyStatus, err := h.cdss.CheckAllergy(ctx, cmd.PatientID, cmd.DrugCode) // patient BC qua port
    if err != nil {
        // FAIL-CLOSED: timeout/error KHÔNG được hiểu là "an toàn" (ADR-008)
        return "", ErrCdssUnavailable
    }
    switch allergyStatus {
    case AllergyConflict:
        if cmd.Override == nil { // chặn trừ khi có override hợp lệ
            return "", ErrAllergyHardStop
        }
        h.audit.RecordOverride(ctx, cmd.Override) // reason + authorizer → audit + interaction_overrides
    case AllergyUnknown:
        // KHÔNG render là 'safe'; gắn cờ unknown lên Prescription, hiển thị rõ ở UI
    }
    // ... persist + emit PrescriptionCreated qua outbox trong CÙNG tx
}
```

**FEFO dispense (planned, SQL)** — trong tx đã `SET LOCAL app.current_branch` (RLS, DATA-1):

```sql
-- internal/pharmacy/adapters/postgres/dispense.sql  (planned)
SELECT id, lot_no, expiry_date, qty
FROM medication_lots
WHERE drug_id = $1 AND branch_id = current_setting('app.current_branch')::uuid
  AND qty > 0 AND expiry_date >= CURRENT_DATE
ORDER BY expiry_date ASC          -- FEFO (ADR-021)
FOR UPDATE SKIP LOCKED            -- hai dược sĩ song song không trừ trùng/không deadlock
LIMIT 1;
-- sau đó: UPDATE qty; INSERT stock_ledger (append-only, balance_after)
```

**Charge-capture idempotent (planned)** — billing là subscriber outbox, idempotent qua `processed_events` + `idempotency_keys`:

```go
// internal/billing/app/command/capture_charge.go  (planned)
func (h *CaptureHandler) OnDrugDispensed(ctx context.Context, e DrugDispensedEvent) error {
    // dedupe: nếu idempotency_key đã tồn tại → no-op (ADR-011)
    if h.repo.ChargeExists(ctx, e.IdempotencyKey) {
        return nil
    }
    price := h.chargemaster.SnapshotPrice(ctx, e.ServiceCode) // price snapshot, không live
    return h.repo.InsertCharge(ctx, Charge{
        EncounterID: e.EncounterID, IdempotencyKey: e.IdempotencyKey,
        Amount: price, BhytPortion: split.Bhyt, SelfPay: split.Self,
    })
}
```

`InsuranceClaim` (insurance BC) đọc các `ChargeItem` của bill để sinh XML1–XML15 (QĐ 4750, INT-1) — KHÔNG dựng số liệu riêng, nên claim luôn khớp bill.

---

## 5. Best practices (mỗi mục kèm nguồn đã research)

1. **Enforce CDSS ở server, modal chỉ là UX**. CDS phải là rào chắn không bỏ qua được, không phải interruptive alert dễ "click-through". — CDS Five Rights / interruptive-alert fatigue: https://www.ncbi.nlm.nih.gov/pmc/articles/PMC3128404/
2. **Phân biệt "no known allergy" với "not assessed"**. Thiếu dữ liệu ≠ an toàn; phải render trạng thái unknown tách bạch. — ISMP guidance on allergy documentation: https://www.ismp.org/resources/allergy-fields-electronic-health-records
3. **`FOR UPDATE SKIP LOCKED` cho hàng đợi/cấp phát đồng thời** thay vì khóa toàn bảng — chuẩn Postgres cho work-queue/lot picking. — PostgreSQL docs (Locking Clause): https://www.postgresql.org/docs/16/sql-select.html#SQL-FOR-UPDATE-SHARE
4. **Transactional outbox cho charge-capture** đảm bảo state change và event atomically — không mất charge, không double-emit. — microservices.io Transactional Outbox: https://microservices.io/patterns/data/transactional-outbox.html
5. **Idempotency key bằng unique-constraint persisted** để retry an toàn trên payment/charge path. — Stripe idempotency design: https://docs.stripe.com/api/idempotent_requests
6. **Append-only ledger cho tồn kho/tài chính** (immutable movement, balance_after) thay vì UPDATE số tồn trực tiếp. — Martin Fowler, Event Sourcing / accounting ledger: https://martinfowler.com/eaaDev/EventSourcing.html
7. **Saga cho quy trình quyết toán nhiều bước** (giải phóng advance + chốt claim) với compensating actions. — microservices.io Saga pattern: https://microservices.io/patterns/data/saga.html

---

## 6. Lỗi thường gặp & anti-patterns

- **CDSS chỉ ở React modal** → bypass bằng devtools/API trực tiếp. Modal KHÔNG phải control (ADR-008).
- **Fail-open khi CDSS timeout** → silent "no interaction". Đây là critical risk canon §8 — bệnh nhân chết vì dị ứng đã ghi.
- **Collapse "unknown" → "safe"** trong UI hoặc backend → mất phân biệt sống còn.
- **FIFO thay FEFO** → thuốc cận hạn ứ đọng phải hủy (ADR-021 từ chối FIFO).
- **Quên `SKIP LOCKED`** → dùng `FOR UPDATE` trơn → hai phiên cấp phát chặn nhau (latency) hoặc thiếu `FOR UPDATE` → trừ trùng lô (oversell).
- **Charge-capture không idempotent** → FE retry / outbox replay → double-post (critical risk canon §8). FE idempotency key và backend idempotency key PHẢI là MỘT scheme end-to-end.
- **Live price thay vì snapshot** → giá đổi sau khi charge → bill không khớp số đã thu.
- **Claim dựng số liệu riêng tách khỏi ChargeItem** → claim↔bill lệch → giám định BHYT từ chối (chính gap ADR-011 sửa).
- **UPDATE/DELETE trên stock_ledger** → phá audit trail kho.
- **Query FEFO ngoài tx đã `SET LOCAL app.current_branch`** → RLS revert no-filter trên pooled connection → leak/sai branch (critical risk canon §8).

---

## 7. Lộ trình luyện tập NGAY trong repo

> Repo chưa có code — bài tập tạo skeleton *(planned)* theo layout canon §9. Mục tiêu là tập tư duy enforcement + concurrency + idempotency.

- 🥉 **Cơ bản** — Viết test bảng (table-driven) cho hàm thuần `evaluateAllergy(status, override) → decision` với 3 nhánh: `conflict+no-override→reject`, `conflict+override→allow+audit`, `unknown→flag-not-safe`. Tạo file `internal/pharmacy/domain/cdss_test.go` *(planned)*, chạy TDD red-green (chưa cần DB).
- 🥈 **Trung cấp** — Viết integration test FEFO với testcontainers-go + Postgres thật: seed 3 lô cùng drug khác `expiry_date`, chạy 2 goroutine dispense song song, assert (a) lô hết hạn sớm nhất bị trừ trước, (b) không lô nào bị trừ quá `qty`, (c) `stock_ledger` có đúng N bản ghi append. Đặt tại `internal/pharmacy/adapters/postgres/dispense_test.go` *(planned)*.
- 🥇 **Nâng cao** — Mô phỏng charge-capture end-to-end: dispense emit `DrugDispensed` qua outbox stub → billing subscriber sinh `ChargeItem` với `Idempotency-Key`; **gọi subscriber 2 lần cùng key** và assert chỉ 1 charge tồn tại; rồi assert `InsuranceClaim` đọc đúng các ChargeItem của bill (claim↔bill linkage). Bao gồm test fail-closed: CDSS trả error → command reject. Path `internal/billing/app/command/capture_charge_test.go` *(planned)*.

---

## 8. Skill/agent ECC nên dùng khi luyện

- **`ecc:healthcare-cdss-patterns`** — pattern CDSS clinical decision support, alert design, fail-closed enforcement.
- **`ecc:healthcare-emr-patterns`** — encounter workflow, prescription generation, clinical safety.
- **`ecc:go-review`** (go-reviewer) — review aggregate invariant, error handling fail-closed, concurrency.
- **`ecc:go-test`** — TDD table-driven + testcontainers, ép write-test-first + 80% coverage (canon ADR-025).
- **`ecc:postgres-patterns`** — `FOR UPDATE SKIP LOCKED`, append-only ledger, index branch_id-leading.
- **`ecc:security-review`** — kiểm fail-open, bypass control, idempotency double-post.
- **`ecc:tdd-workflow`** + **tdd-guide** agent — enforce red→green→refactor cho CDSS/FEFO invariant.

---

## 9. Tài nguyên học thêm (2024–2026)

- PostgreSQL 16 — `SELECT ... FOR UPDATE SKIP LOCKED`: https://www.postgresql.org/docs/16/sql-select.html
- River (Postgres-native job queue, Go) — claim retry / FEFO sweep: https://riverqueue.com/docs
- microservices.io — Transactional Outbox & Saga: https://microservices.io/patterns/data/transactional-outbox.html
- Stripe — Idempotent requests: https://docs.stripe.com/api/idempotent_requests
- ISMP — Allergy documentation & alert fatigue: https://www.ismp.org/resources
- QĐ 4750/QĐ-BYT (sửa QĐ 3176/2024, hiệu lực 01/01/2025) — bộ XML1–XML15 giám định BHYT (cổng giám định BHXH).
- TT 26/2025/TT-BYT + QĐ 808 — đơn thuốc điện tử liên thông donthuocquocgia.vn.
- sqlc + pgx/v5 — type-safe query Go: https://docs.sqlc.dev

---

## 10. Checklist "đã hiểu"

- [ ] Giải thích được vì sao CDSS modal trên React KHÔNG phải là control, và đâu là nơi enforce thật (ADR-008).
- [ ] Phân biệt rõ ba trạng thái `safe` / `conflict` / `unknown` và vì sao `unknown` không bao giờ = `safe`.
- [ ] Mô tả được fail-closed: khi CDSS timeout/error thì command bị reject, không "pass".
- [ ] Viết được query FEFO `ORDER BY expiry_date ASC ... FOR UPDATE SKIP LOCKED` và giải thích từng mệnh đề (ADR-021).
- [ ] Hiểu vì sao `stock_ledger` append-only và `balance_after` thay vì UPDATE số tồn.
- [ ] Giải thích charge-capture idempotent qua outbox + `idempotency_keys` unique-constraint chống double-post (ADR-011).
- [ ] Hiểu vì sao `InsuranceClaim` sinh từ chính `ChargeItem` để claim↔bill↔encounter nhất quán.
- [ ] Biết price snapshot tại thời điểm charge thay vì live price.
- [ ] Hiểu mọi FEFO/charge query phải chạy trong tx đã `SET LOCAL app.current_branch` (RLS, DATA-1).
- [ ] Liệt kê được 3 anti-pattern critical (fail-open CDSS, FIFO thay FEFO, charge không idempotent) và mitigation.
