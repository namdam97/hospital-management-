# [INT-1] Interop foundations + BHYT 4750 + e-prescription liên thông

> Module INT-1 · Nền móng interoperability (coded data) + hai external regulatory bắt buộc trong MVP (BHYT XML 4750 & đơn thuốc liên thông donthuocquocgia.vn) · Độ khó: 🥉→🥇 · Prereqs: DATA-2 (Clinical data model — Encounter anchor, terminology triplets)

Module này dạy kỹ năng "làm cho dữ liệu HMS NÓI CHUYỆN được với hệ thống bên ngoài" mà không phá kiến trúc monolith. Khác với INT-2 (FHIR facade Phase 2), INT-1 tập trung vào hai thứ KHÔNG deferrable: nền móng coded-data rẻ tiền (bake-in từ MVP) và hai tích hợp regulatory mà pháp luật VN bắt buộc ngay — BHYT (QĐ 4750) và liên thông đơn thuốc (TT 26/2025).

---

## 1. Vì sao kỹ năng này quan trọng trong HMS

Interoperability trong bệnh viện không phải "tính năng đẹp" — nó là **điều kiện pháp lý để được thanh toán và được phép kê đơn**. Hai sự thật nghiệp vụ định hình module này:

- **BHYT là dòng tiền sống còn**: ~80–90% lượt khám OPD ở bệnh viện công VN thanh toán qua BHYT. Nếu bộ XML giám định sai format hoặc claim không khớp bill, cơ quan BHXH **từ chối thanh toán** — bệnh viện mất doanh thu thật. Đây là lý do `ChargeItem` phải là nguồn DUY NHẤT sinh ra cả Invoice lẫn claim (ADR-011): claim↔bill phải nhất quán tuyệt đối.
- **Liên thông đơn thuốc là bắt buộc, đã quá hạn**: TT 26/2025 + QĐ 808 buộc cơ sở khám chữa bệnh đẩy đơn ngoại trú lên `donthuocquocgia.vn` ngay sau khám. MVP của HMS LÀ một khoa OPD-kê-đơn → đây là **MVP-mandatory, không phải Phase 2** (ADR-007). In một tờ đơn không có mã đơn quốc gia = "digitization giả", staff vẫn phải giữ sổ cũ.

Nền móng coded-data (cột `code` + `system` + `display` cho mọi field lâm sàng) là phần rẻ nhất nhưng không-retrofit-được: nếu MVP lưu chẩn đoán là free-text "viêm phổi" thay vì `(J18.9, ICD-10, "Viêm phổi, không đặc hiệu")`, thì Phase 2 dựng FHIR facade hay xuất XML 4750 sẽ phải re-code thủ công toàn bộ lịch sử — cực đắt. ADR-016 chốt: bake-in foundations rẻ ở MVP để Phase 2 rẻ.

Bài học từ degraded-mode (ADR-006): mọi external dependency LIVE đều có thể down. Kỹ năng cốt lõi của interop engineer ở đây là **thiết kế để không bao giờ chặn người bệnh khi cổng lỗi** — admit-and-flag, queue-and-retry, reconcile-later.

---

## 2. Mô hình tư duy (first principles) — từ con số 0

Bắt đầu từ định nghĩa trần trụi: **interoperability = hai hệ thống đồng ý về (a) ý nghĩa của dữ liệu và (b) cách trao đổi nó.**

1. **Ý nghĩa cần một từ điển chung (terminology).** "Viêm phổi" với người là rõ; với máy thì vô nghĩa cho tới khi gắn vào một mã trong một hệ thống mã hoá: `J18.9` trong `ICD-10` (QĐ 4469). Một mã trần (`J18.9`) cũng chưa đủ — phải biết nó thuộc **system** nào (cùng chuỗi `308` mang nghĩa khác nhau giữa ICD-10 và mã DVKT). Vì vậy đơn vị nguyên tử của coded data là **triplet `(code, system, display)`**, không phải string.

2. **Trao đổi cần một hợp đồng (contract).** Hai bên phải thống nhất schema (XML 4750 định nghĩa XML1..XML15), giao thức (HTTPS/JSON cho card-check, SOAP/REST + mTLS cho cổng giám định), và ngữ nghĩa lỗi (rejection-code).

3. **External call là I/O không tin cậy.** Khác với hàm nội bộ, gọi cổng BHYT/donthuoc có thể: chậm (timeout), down, trả lỗi tạm thời (retry được) hay lỗi vĩnh viễn (rejection — không retry). First principle: **tách biệt "lưu trong nhà" khỏi "gửi ra ngoài"**. Lưu phải synchronous + ACID; gửi phải async + idempotent + retry. Đây chính là vì sao outbox + River tồn tại (ADR-012).

4. **Idempotency là nền của at-least-once.** Mạng không đảm bảo "đúng một lần". Nếu retry gửi claim 3 lần, cổng BHXH không được tạo 3 claim. Giải pháp: dedupe key (claim reference) để external call lặp lại không gây side-effect kép (ADR-011).

Tư duy chốt: **MVP ships ZERO embedded FHIR server**, nhưng bake-in coded columns để mọi field lâm sàng "sẵn sàng dịch". Dịch (FHIR/HL7) là Phase 2 (INT-2); module này chỉ dựng nền + hai regulatory externals.

---

## 3. Khái niệm cốt lõi (tăng dần độ khó)

| # | Khái niệm | Bản chất |
|---|-----------|----------|
| 1 | **Coded triplet** | `(code, system, display)` — `system` định danh hệ mã (ICD-10, LOINC, RxNorm, mã DVKT BHYT, danh mục dùng chung BYT) |
| 2 | **terminology_concepts catalog** | Bảng tra cứu nội bộ (patient BC sở hữu) seed từ danh mục dùng chung BYT; nguồn validate `code` hợp lệ |
| 3 | **Encounter anchor** | Mọi dữ liệu lâm sàng FK tới `encounter_id` → đơn vị xuất XML/claim/FHIR là một Encounter, không phải patient (ADR-004) |
| 4 | **BHYT two-touch** | Touch-1: LIVE card-check JSON tại tiếp đón. Touch-2: XML1–XML15 đẩy cổng giám định lúc quyết toán (ADR-006) |
| 5 | **XML1–XML15 (QĐ 4750)** | Bộ 15 bảng XML giám định BHYT (sửa QĐ 3176, hiệu lực 01/01/2025): XML1 tổng hợp, XML2 thuốc, XML3 DVKT, ... XML15 |
| 6 | **claim↔bill↔encounter FK** | InsuranceClaim sinh từ chính `ChargeItem` → đảm bảo số liệu claim = số liệu bill (ADR-011) |
| 7 | **National Rx link** | Mã đơn quốc gia (semantics C/N/H/Y) lưu trên prescription sau khi đẩy donthuocquocgia.vn (ADR-007) |
| 8 | **Degraded-mode** | admit-and-flag / queue-and-retry / reconcile-later khi external down — không bao giờ chặn người bệnh (ADR-006) |
| 9 | **Idempotent external call** | Dedupe trên claim reference / Idempotency-Key → at-least-once không gây double-submit (ADR-011) |
| 10 | **Saga + state machine** | Submission qua River job + saga; rejection-code là first-class state, không phải log dòng text (ADR-006/011/023) |

Phân biệt sống còn: **outbound submission (async, retry, idempotent)** vs **synchronous card-check tại tiếp đón (sync với timeout + fallback)**. Card-check là sync vì lễ tân cần verdict ngay; nhưng vẫn phải có degraded-mode khi cổng chậm.

---

## 4. HMS dùng nó thế nào (bám code path — *(planned)*, code chưa viết)

> Mọi đường dẫn dưới đây theo layout repo MỤC TIÊU (canon §9). Repo HIỆN CHƯA CÓ CODE — đánh dấu *(planned)*.

**(a) Coded triplet bake-in** — `internal/encounter/domain` *(planned)*: mỗi `Diagnosis`/`Observation` mang triplet thay vì string. Validate `code` qua catalog `terminology_concepts` (patient BC sở hữu, canon §4).

```go
// internal/shared/coded/coded.go (planned)
type Coded struct {
    Code    string // "J18.9"
    System  string // "ICD-10" | "LOINC" | "RxNorm" | "DMDC" | "BYT-DUNGCHUNG"
    Display string // "Viêm phổi, không đặc hiệu"
}
```

```sql
-- migration cho encounters/diagnoses (planned, sau Phase-0 000001)
CREATE TABLE diagnoses (
  id            UUID PRIMARY KEY,           -- UUID v7
  branch_id     UUID NOT NULL,              -- FORCE RLS (ADR-003)
  encounter_id  UUID NOT NULL REFERENCES encounters(id),  -- Encounter anchor (ADR-004)
  code          TEXT NOT NULL,              -- "J18.9"
  code_system   TEXT NOT NULL DEFAULT 'ICD-10',
  code_display  TEXT NOT NULL,
  is_primary    BOOLEAN NOT NULL DEFAULT false
);
```

**(b) BHYT touch-1 — LIVE card-check tại tiếp đón** — `internal/scheduling/...` gọi adapter, BC `scheduling-reception` sở hữu bảng `bhyt_eligibility_checks` (canon §4). Sync với timeout + fallback:

```go
// internal/scheduling/app/check_eligibility.go (planned)
verdict, err := h.bhytClient.CheckCard(ctx, cardNo, cccd)
if err != nil { // cổng/mạng lỗi → KHÔNG chặn người bệnh (ADR-006)
    return admitAndFlag(ctx, patientID) // thẻ provisionally-unverified + queue retry
}
// verdict: eligible | ineligible | co-pay (+ 6 lần khám gần nhất)
```

**(c) BHYT touch-2 — XML1–XML15 đẩy cổng giám định** — `internal/insurance/{domain,app,adapters}` *(planned)*, BC `insurance` sở hữu `insurance_claims`, `claim_xml_records`, `claim_responses`, `bhyt_submission_log` (canon §4). Claim sinh từ `ChargeItem` (ADR-011), ký số trước khi gửi, đẩy qua mTLS + saga + River retry:

```go
// internal/insurance/app/command/submit_claim.go (planned)
// 1. build XML1..XML15 từ ChargeItem (claim==bill nhất quán)
// 2. ký số client cert (secret store, ADR-021)
// 3. enqueue River job (at-least-once) — dedupe trên claim reference (ADR-011)
```

**(d) E-prescription liên thông** — `internal/pharmacy/adapters/donthuoc` *(planned)*, BC `pharmacy` sở hữu `national_rx_links` (canon §4). Đẩy qua outbox + River retry idempotent, lưu mã đơn quốc gia (ADR-007):

```go
// internal/pharmacy/adapters/donthuoc/client.go (planned)
// app-name/app-key + mã liên-thông cơ sở (organization BC: facility_external_codes)
//                   + mã liên-thông bác sĩ → POST donthuocquocgia.vn
// → lưu national_rx_code (semantics C/N/H/Y) lên prescription
```

Lưu ý kiến trúc: cross-BC (encounter→billing→insurance) chỉ qua **domain event + transactional outbox in-process** (ADR-012), KHÔNG import chéo BC. `facility_external_codes` (mã liên-thông per branch) thuộc `organization` BC, không phải insurance/pharmacy.

---

## 5. Best practices (mỗi mục kèm 1 nguồn đã research)

1. **Coded data dùng triplet, không string** — luôn lưu `(code, system, display)`; HL7 FHIR coding pattern là chuẩn quốc tế cho việc này. Nguồn: HL7 FHIR R4 — Coding datatype, <https://www.hl7.org/fhir/R4/datatypes.html#Coding>
2. **Idempotency cho mọi external write** — dùng Idempotency-Key/dedupe reference để retry an toàn. Nguồn: Stripe — Idempotent requests, <https://docs.stripe.com/api/idempotent_requests>
3. **Tách "lưu trong DB" khỏi "gửi ra ngoài" bằng transactional outbox** — không gọi external trong tx DB. Nguồn: microservices.io — Transactional Outbox pattern, <https://microservices.io/patterns/data/transactional-outbox.html>
4. **Saga cho quy trình nhiều bước có external** (quyết toán + claim submit) với compensating action. Nguồn: microservices.io — Saga pattern, <https://microservices.io/patterns/data/saga.html>
5. **Retry với exponential backoff + jitter** cho cổng quá tải/chậm; phân biệt lỗi tạm thời (retry) vs vĩnh viễn (reject). Nguồn: AWS — Timeouts, retries, and backoff with jitter, <https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/>
6. **mTLS + ký số cho kênh regulatory** (BHYT outbound). Nguồn: Cloudflare — What is mutual TLS (mTLS)?, <https://www.cloudflare.com/learning/access-management/what-is-mutual-tls/>
7. **Validate code dựa trên terminology catalog**, không trust free-text từ FE. Nguồn: OWASP — Input Validation Cheat Sheet, <https://cheatsheetseries.owasp.org/cheatsheets/Input_Validation_Cheat_Sheet.html>
8. **Bám đúng văn bản pháp lý hiện hành** — QĐ 4750/QĐ-BYT (sửa QĐ 3176) bộ XML giám định BHYT. Nguồn: Cổng giám định BHYT, <https://gdbhyt.baohiemxahoi.gov.vn/>

---

## 6. Lỗi thường gặp & anti-patterns

- **Lưu chẩn đoán/thuốc là free-text** → không xuất được XML 4750/FHIR, phải re-code thủ công. Sửa: triplet ngay từ MVP (ADR-016).
- **Gọi external API trực tiếp trong HTTP handler hoặc trong tx DB** → tx giữ lock chờ network, hoặc mất event khi crash sau commit DB nhưng trước khi gửi. Sửa: outbox + River (ADR-012).
- **Chặn check-in khi cổng BHYT down** → trigger #1 staff quay về giấy (canon §8). Sửa: admit-and-flag (ADR-006).
- **Claim sinh độc lập với bill** → số liệu lệch, BHXH từ chối. Sửa: claim từ chính `ChargeItem`, FK 1-1/1-n claim↔bill↔encounter (ADR-011).
- **Submission không idempotent** → retry tạo claim trùng. Sửa: dedupe trên claim reference (ADR-011).
- **Rejection-code xử lý bằng log text/điện thoại** → không theo dõi được. Sửa: rejection-code là state machine first-class trong insurance BC (ADR-023).
- **In đơn thuốc thiếu QR/mã đơn quốc gia + block chữ ký số** → không hợp lệ pháp lý, digitization giả (ADR-007/022).
- **Lock thư viện FHIR chết** (samply/golang-fhir-models, release cuối 12/2022) cho regulatory layer → hazard. Sửa: KHÔNG cam kết thư viện ở MVP, đánh giá lại Phase 2 (ADR-017). *(thuộc INT-2)*
- **Trust mã liên-thông/branch code từ client** thay vì từ `organization` BC + JWT branch → tenant spoofing. Sửa: lấy từ `facility_external_codes` theo branch đã verify.

---

## 7. Lộ trình luyện tập NGAY trong repo (🥉 → 🥈 → 🥇)

> Repo chưa có code; bài tập là dựng phần *(planned)* theo layout canon §9, có test.

- 🥉 **Cơ bản — Coded triplet + validate**: Tạo `internal/shared/coded/coded.go` *(planned)* với type `Coded`. Viết migration `diagnoses` có `code/code_system/code_display` + `encounter_id` FK + `branch_id`. Viết unit test (table-driven) `ValidateCode()` reject mã không có trong `terminology_concepts` seed (ICD-10 mẫu QĐ 4469). RED→GREEN→refactor.

- 🥈 **Trung cấp — Card-check degraded-mode**: Định nghĩa interface `BhytCardClient` trong `internal/scheduling/ports` *(planned)* và một fake adapter trả timeout. Viết app service `CheckEligibility` đi qua `admit-and-flag` khi client lỗi (ghi `bhyt_eligibility_checks` với trạng thái provisionally-unverified + enqueue retry). Unit test cả ba verdict (eligible/ineligible/co-pay) + nhánh degraded. Đảm bảo KHÔNG có return-path nào chặn người bệnh.

- 🥇 **Nâng cao — XML 4750 + idempotent submission saga**: Trong `internal/insurance` *(planned)* dựng builder sinh XML1 + XML2 (thuốc) + XML3 (DVKT) từ một tập `ChargeItem` mẫu; assert tổng tiền XML == tổng `bill`. Wire submission thành River job với dedupe trên claim reference; viết **integration test với testcontainers Postgres** chứng minh submit 2 lần cùng reference chỉ tạo 1 `bhyt_submission_log` entry (idempotency), và một `claim_responses` rejection chuyển claim sang state `rejected` (state machine). Tham chiếu ADR-011/023/025.

---

## 8. Skill/agent ECC nên dùng khi luyện

- **`ecc:go-review`** — review adapter BHYT/donthuoc: error handling, context timeout, không gọi external trong tx.
- **`ecc:go-test`** — TDD table-driven cho XML builder + verdict mapping; coverage ≥80% (testing rule).
- **`ecc:postgres-patterns`** — schema coded columns, FK encounter anchor, indexing branch_id-leading.
- **`ecc:api-design`** + **`ecc:api-connector-builder`** — thiết kế external client contract (card-check JSON, cổng giám định) theo một pattern nhất quán.
- **`ecc:security-review`** / **`ecc:security-scan`** — mTLS, client cert trong secret store (ADR-021), không leak app-key/mã liên-thông.
- **`ecc:healthcare-emr-patterns`** + **`ecc:healthcare-phi-compliance`** — coded clinical data, prescription generation, PHI khi gửi external.
- **`ecc:database-migrations`** — migration cho coded columns không-backfill (zero-downtime add→backfill→switch, ADR-024).
- **`ecc:tdd-workflow`** — enforce red-green-refactor cho idempotency/saga test.

---

## 9. Tài nguyên học thêm (2024–2026)

- QĐ 4750/QĐ-BYT (sửa QĐ 3176) — bộ XML giám định BHYT XML1–XML15: Cổng giám định BHYT <https://gdbhyt.baohiemxahoi.gov.vn/>
- TT 26/2025/TT-BYT + QĐ 808 — đơn thuốc điện tử liên thông: Hệ thống đơn thuốc quốc gia <https://donthuocquocgia.vn/>
- TT 13/2025/TT-BYT — bệnh án điện tử (ký số): Cổng thông tin Bộ Y tế <https://moh.gov.vn/>
- ICD-10 (QĐ 4469/QĐ-BYT) — danh mục mã bệnh dùng tại VN.
- HL7 FHIR R4 — Coding / CodeableConcept datatypes: <https://www.hl7.org/fhir/R4/datatypes.html>
- microservices.io — Transactional Outbox & Saga: <https://microservices.io/patterns/data/transactional-outbox.html>
- AWS Builders' Library — Timeouts, retries, backoff with jitter: <https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/>
- River (Postgres-native job queue) docs: <https://riverqueue.com/docs>
- NĐ 13/2023/NĐ-CP — bảo vệ dữ liệu cá nhân (PHI residency): <https://datafiles.chinhphu.vn/>

---

## 10. Checklist "đã hiểu"

- [ ] Giải thích được vì sao mọi field lâm sàng phải là triplet `(code, system, display)` thay vì string, và vì sao không-retrofit-được (ADR-016).
- [ ] Phân biệt được BHYT touch-1 (sync card-check) vs touch-2 (async XML submission) và mode degraded của từng touch (ADR-006).
- [ ] Liệt kê được các bảng `insurance` BC sở hữu và vì sao claim sinh từ `ChargeItem` (ADR-011).
- [ ] Hiểu vì sao external call đi qua outbox + River, KHÔNG trong tx DB / không trong HTTP handler (ADR-012).
- [ ] Mô tả được idempotency scheme end-to-end để retry submission không tạo claim/đơn trùng.
- [ ] Biết rejection-code phải là state machine first-class, không phải log text (ADR-023).
- [ ] Giải thích được vì sao e-prescription donthuocquocgia.vn là MVP-mandatory chứ không Phase 2 (ADR-007).
- [ ] Biết mã liên-thông per branch nằm ở `organization` BC (`facility_external_codes`), lấy theo JWT branch đã verify — không trust client.
- [ ] Biết FHIR facade & lock thư viện là Phase 2 (INT-2), KHÔNG thuộc MVP (ADR-016/017).
- [ ] Viết được integration test (testcontainers) chứng minh idempotency + state transition cho claim submission (ADR-025).
