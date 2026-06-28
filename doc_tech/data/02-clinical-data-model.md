# [DATA-2] Clinical data model: Encounter anchor, terminology triplets, versioning

> Module DATA-2 · Mô hình dữ liệu lâm sàng quanh Encounter anchor, coded-triplet, và bất biến hóa EMR ký số · Độ khó: 🥉→🥇 · Prereqs: DATA-1 (PostgreSQL RLS & branch_id multi-tenancy), ARCH-2 (DDD & Bounded Contexts)

Module này dạy bạn thiết kế schema lâm sàng cho HMS sao cho phản ánh đúng nghiệp vụ "một lượt khám/đợt điều trị", liên thông được (FHIR/BHYT), và bất biến pháp lý (TT 13/2025). Neo vào **ADR-004** (Encounter anchor + EMR ký số), **ADR-016** (coded foundations), **ADR-024** (migration 000001), và phụ thuộc keystone **ADR-003/005** (RLS branch_id) đã học ở DATA-1.

---

## 1. Vì sao kỹ năng này quan trọng trong HMS

Một bệnh viện rời giấy không chỉ cần "lưu được dữ liệu" — nó cần một system-of-record mà mỗi vitals, mỗi chẩn đoán ICD-10, mỗi order CLS, mỗi đơn thuốc, mỗi charge đều **treo vào đúng một lượt khám** và có thể tái dựng lại y nguyên 10 năm sau khi bị thanh tra. Đây là khác biệt sống còn giữa "10 module CRUD hành chính" và một EMR thật.

Nếu data model sai từ gốc, mọi thứ phía trên sụp: claim BHYT không link được về encounter để giám định; bác sĩ không thấy "đợt khám này gồm những gì"; EMR ký số sửa được sau khi ký (vi phạm tính pháp lý TT 13/2025); và Phase 2 muốn xuất FHIR thì phát hiện không có `system`/`code` để map. Ba quyết định gốc của module này — **(1) Encounter là anchor (FK tới `encounter_id`, KHÔNG `patient_id` trực tiếp)**, **(2) mọi field lâm sàng mang triplet `(code, system, display)`**, **(3) signed → amendment-only + `*_history`** — đều là loại không-backfill-được: phải đúng TRƯỚC khi có data thật (ADR-024, Phase-0).

Cụ thể với MVP một khoa OPD-BHYT: `EncounterClosed`/`Diagnosed`/`EMRSigned` là event drive charge-capture (billing), claim XML 4750 (insurance), và Phase-2 interop — nếu encounter aggregate không phát đúng event với đúng dữ liệu coded, cả chuỗi downstream hỏng theo.

---

## 2. Mô hình tư duy (first principles) — từ con số 0

Bắt đầu từ câu hỏi: **"Một sự thật lâm sàng (clinical fact) sống ở đâu và bất biến đến mức nào?"**

1. **Sự thật lâm sàng luôn có ngữ cảnh thời điểm.** "Bệnh nhân X bị tăng huyết áp" là vô nghĩa nếu không gắn "đo lúc nào, trong lượt khám nào, ai ghi". Vì thế đơn vị neo không phải con người (`patient`) mà là **một lượt tiếp xúc y tế** = Encounter. Patient là danh tính liên tục (MPI, xuyên chi nhánh); Encounter là một sự kiện có biên giới thời gian.

2. **Dữ liệu cần được máy hiểu, không chỉ người đọc.** "Tăng huyết áp vô căn" với người là đủ; với BHYT/FHIR cần `code=I10, system=ICD-10`. Vì vậy mọi field lâm sàng = triplet **`code` (mã chuẩn) + `system` (hệ mã) + `display` (nhãn người đọc tại thời điểm ghi)**. Lưu cả `display` snapshot vì nhãn của catalog có thể đổi, nhưng record lịch sử phải giữ nhãn lúc bấy giờ.

3. **Một số sự thật, sau khi xác nhận, KHÔNG được phép biến đổi.** EMR ký số là tài liệu pháp lý: sau khi bác sĩ ký PKI, nó đóng băng. Sửa = phát hành addendum/amendment mới tham chiếu bản gốc, KHÔNG UPDATE in-place. Đây là mô hình "append-only / event-sourcing-lite" cho clinical record.

4. **Đa tầng bất biến.** Không phải mọi thứ đều immutable như nhau: clinical_notes có thể sửa khi *chưa* ký (giữ `*_history`); emr_documents *sau* ký là tuyệt đối bất biến; audit_log/stock_ledger là INSERT-only ngay từ đầu. Phân biệt đúng "mức độ đông cứng" cho từng bảng là kỹ năng cốt lõi.

Tư duy nền: **schema lâm sàng = đồ thị có gốc là Encounter, các nút lá là coded facts, một số nút bị niêm phong (sealed) bằng chữ ký và hash.**

---

## 3. Khái niệm cốt lõi (tăng dần độ khó)

**3.1. Encounter anchor & FK discipline (🥉).** Mọi bảng lâm sàng có cột `encounter_id UUID NOT NULL REFERENCES encounters(id)`. Patient được tìm gián tiếp qua encounter. Ngoại lệ hợp lệ: `patient_allergies` treo vào `patient_id` (dị ứng là sự thật xuyên lượt khám, feed CDSS) — nhưng diagnoses/observations/orders/charges thì KHÔNG.

**3.2. UUID v7 làm PK (🥉).** PK lâm sàng dùng UUID v7 (time-ordered) thay vì v4 — giảm B-tree fragmentation, index locality tốt cho insert-heavy. `audit_log`/`stock_ledger`/`outbox` dùng `BIGINT GENERATED ALWAYS AS IDENTITY` (cần thứ tự tuyệt đối + hash-chain). Tham chiếu ADR-005 (UUID v7 PK) + ADR-009 (BIGINT cho audit).

**3.3. Coded triplet & terminology catalog (🥈).** Cột lâm sàng có ý nghĩa mã hóa lưu `*_code`, `*_system`, `*_display`. `terminology_concepts` (thuộc patient BC, ADR-016) là catalog seeded từ danh mục dùng chung BYT (ICD-10 QĐ 4469, LOINC, RxNorm/DMDC). FK mềm (validate qua app/CHECK, không hard-FK chặn) để không khóa cứng vào version catalog.

**3.4. Encounter state machine (🥈).** `planned → arrived → triaged → in-progress → finished → billed → closed`. Transition hợp lệ enforce ở domain layer (encounter aggregate), cột `status` + bảng transition log. OPD/ED/IPD là `encounter_class`; IPD có `admissions`/`bed_assignments` con.

**3.5. Signed → amendment-only + versioning (🥇).** Trước ký: `clinical_notes` mutable, mỗi sửa snapshot sang `clinical_notes_history`. Khi đóng encounter: kết tinh `emr_documents` (immutable) + `emr_signatures` (PKI). Sau ký: chỉ tạo addendum (document mới `supersedes` bản cũ), KHÔNG sửa. Synchronous durability: commit confirmed TRƯỚC khi UI báo "đã ký" (ADR-004/015).

**3.6. Hash-chain & PITR survival (🥇).** EMR document có `content_hash`; signature có `prev_hash` tạo chuỗi tamper-evident. Chuỗi phải sống sót PITR restore — nghĩa là hash tính trên nội dung canonical, không phụ thuộc ID tự sinh runtime.

---

## 4. HMS dùng nó thế nào (bám code path — *(planned)*, code chưa viết)

Theo layout repo mục tiêu (canon §9), các artifact sống ở:

- Migration: `backend/migrations/00000X_encounter.up.sql` *(planned)* — tạo `encounters`, `diagnoses`, `observations`, `clinical_notes(_history)`, `emr_documents`, `emr_signatures` (sau migration 000001 đã bật FORCE RLS — ADR-024).
- Domain: `backend/internal/encounter/domain/` *(planned)* — `Encounter` aggregate (state machine), `EMRDocument` (sealed), value object `CodedConcept{Code, System, Display}`.
- App/command: `backend/internal/encounter/app/command/` *(planned)* — `SignEMRDocument`, `AmendClinicalNote`; emit `EMRSigned`/`Diagnosed` qua outbox (ADR-012).
- Catalog: `terminology_concepts` thuộc patient BC (`backend/internal/patient/...`) *(planned)*.
- Shared: `backend/internal/shared/rls/` — SET LOCAL `app.current_branch` GUC trong tx (DATA-1).

Bảng lõi (đúng tên canon §4 encounter BC) *(MVP, planned)*:

```sql
-- 00000X_encounter.up.sql (chạy SAU 000001 đã ENABLE+FORCE RLS — ADR-024)
CREATE TABLE encounters (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v7(),   -- UUID v7, ADR-005
  branch_id       UUID NOT NULL,                                  -- RLS leading column, ADR-003/005
  patient_id      UUID NOT NULL REFERENCES patients(id),
  encounter_class TEXT NOT NULL CHECK (encounter_class IN ('OPD','ED','IPD')),
  status          TEXT NOT NULL DEFAULT 'planned'
                  CHECK (status IN ('planned','arrived','triaged',
                                    'in-progress','finished','billed','closed')),
  opened_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  closed_at       TIMESTAMPTZ,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
ALTER TABLE encounters ENABLE ROW LEVEL SECURITY;
ALTER TABLE encounters FORCE ROW LEVEL SECURITY;                  -- ADR-003 keystone
CREATE POLICY enc_isolation ON encounters
  USING      (branch_id = current_setting('app.current_branch')::uuid)
  WITH CHECK (branch_id = current_setting('app.current_branch')::uuid);

-- Mọi fact lâm sàng FK tới encounter_id, KHÔNG patient_id (ADR-004)
CREATE TABLE diagnoses (
  id            UUID PRIMARY KEY DEFAULT uuid_generate_v7(),
  branch_id     UUID NOT NULL,
  encounter_id  UUID NOT NULL REFERENCES encounters(id),
  dx_code       TEXT NOT NULL,            -- triplet, ADR-016
  dx_system     TEXT NOT NULL DEFAULT 'ICD-10',   -- QĐ 4469
  dx_display    TEXT NOT NULL,            -- snapshot nhãn lúc ghi
  rank          SMALLINT NOT NULL DEFAULT 1,       -- 1 = chính
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE observations (             -- vitals/labs, LOINC triplet
  id            UUID PRIMARY KEY DEFAULT uuid_generate_v7(),
  branch_id     UUID NOT NULL,
  encounter_id  UUID NOT NULL REFERENCES encounters(id),
  obs_code      TEXT NOT NULL, obs_system TEXT NOT NULL DEFAULT 'LOINC',
  obs_display   TEXT NOT NULL,
  value_num     NUMERIC, value_text TEXT, unit TEXT,   -- code+system cho unit (UCUM) ở Phase 2
  observed_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Versioning & signing *(planned)*:

```sql
CREATE TABLE clinical_notes (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v7(),
  branch_id UUID NOT NULL, encounter_id UUID NOT NULL REFERENCES encounters(id),
  body TEXT NOT NULL, version INT NOT NULL DEFAULT 1,
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- Mỗi UPDATE (khi chưa ký) snapshot bản cũ sang *_history (trigger hoặc app-layer)
CREATE TABLE clinical_notes_history (LIKE clinical_notes INCLUDING ALL);

CREATE TABLE emr_documents (            -- bất biến SAU ký, ADR-004
  id UUID PRIMARY KEY DEFAULT uuid_generate_v7(),
  branch_id UUID NOT NULL, encounter_id UUID NOT NULL REFERENCES encounters(id),
  doc_type TEXT NOT NULL,               -- bệnh án, giấy ra viện...
  content_canonical JSONB NOT NULL,     -- nội dung kết tinh
  content_hash TEXT NOT NULL,           -- sha-256, base cho hash-chain
  supersedes_id UUID REFERENCES emr_documents(id),  -- addendum, KHÔNG sửa in-place
  signed_at TIMESTAMPTZ
);
CREATE TABLE emr_signatures (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,  -- thứ tự tuyệt đối
  document_id UUID NOT NULL REFERENCES emr_documents(id),
  signed_by UUID NOT NULL, signature_blob BYTEA NOT NULL,
  prev_hash TEXT, this_hash TEXT NOT NULL,             -- hash-chain
  signed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Domain enforce immutability (KHÔNG dựa DB-only) *(planned)*:

```go
// backend/internal/encounter/domain/emr_document.go (planned)
type CodedConcept struct{ Code, System, Display string }

func (e *EMRDocument) Amend(newContent CanonicalContent, by StaffID) (*EMRDocument, error) {
    if e.SignedAt.IsZero() {
        return nil, ErrNotYetSigned // chưa ký thì sửa note, không amend
    }
    // KHÔNG mutate e — tạo bản mới supersedes (immutable pattern, coding-style rule)
    return newAmendment(e.ID, newContent, by), nil
}
```

---

## 5. Best practices (mỗi mục kèm 1 nguồn)

1. **Lưu coded value dạng triplet, snapshot `display` cùng record.** Nhãn catalog đổi theo thời gian; record lịch sử phải giữ nhãn lúc ghi để tái dựng đúng. FHIR R4 `Coding` chính là pattern này (`system`+`code`+`display`). Nguồn: HL7 FHIR R4 Coding datatype — https://hl7.org/fhir/R4/datatypes.html#Coding

2. **Dùng UUID v7 (time-ordered) thay v4 cho PK insert-heavy.** Giảm index fragmentation, cải thiện locality. Nguồn: RFC 9562 (UUID v7) — https://www.rfc-editor.org/rfc/rfc9562.html#name-uuid-version-7

3. **Bất biến hóa qua temporal/append-only thay vì UPDATE in-place cho dữ liệu pháp lý.** Mô hình "non-destructive change". Nguồn: Martin Fowler — Temporal Patterns / "Audit Log" — https://martinfowler.com/eaaDev/timeNarrative.html

4. **Snapshot giá/nhãn tại thời điểm giao dịch, không tham chiếu live.** Đảm bảo tái dựng đúng (áp cho diagnosis display lẫn charge price — liên kết ADR-011). Nguồn: PostgreSQL docs — Constraints/CHECK & data integrity — https://www.postgresql.org/docs/16/ddl-constraints.html

5. **Tách dữ liệu nhận diện (catalog) khỏi dữ liệu giao dịch; FK mềm tới catalog version.** Tránh khóa cứng schema vào version terminology. Nguồn: HL7 FHIR — Terminology Module (CodeSystem/ValueSet) — https://hl7.org/fhir/R4/terminologies.html

6. **JSONB cho nội dung canonical kết tinh nhưng cột quan hệ cho field truy vấn.** Hybrid: query/index trên cột; freeze toàn văn trong JSONB để hash. Nguồn: PostgreSQL JSONB best practices — https://www.postgresql.org/docs/16/datatype-json.html

---

## 6. Lỗi thường gặp & anti-patterns

- **Treo fact lâm sàng vào `patient_id` trực tiếp** (bỏ qua Encounter): mất ngữ cảnh lượt khám, không claim/giám định được, khó map FHIR Encounter. Vi phạm ADR-004. (Ngoại lệ đúng: `patient_allergies`.)
- **Lưu `display` mà bỏ `code`/`system`** (hoặc ngược lại): không liên thông, Phase-2 FHIR phải backfill — không khả thi sau khi có data (ADR-016).
- **Cho UPDATE `emr_documents` sau ký** ("chỉ sửa typo tí"): phá tính pháp lý bất biến TT 13/2025. Phải addendum/`supersedes`.
- **Quên `branch_id` + FORCE RLS trên bảng lâm sàng mới**: leak cross-branch âm thầm — diagram nói "isolated" nhưng production rò (open risk [critical], ADR-003). Bảng lâm sàng tạo SAU 000001 vẫn phải tự ENABLE+FORCE.
- **Dùng UUID v4 cho PK insert-heavy**: write amplification, B-tree bloat (chọn v7).
- **Hard-FK chặn cứng tới `terminology_concepts`**: nâng version catalog làm fail insert dữ liệu cũ — dùng validate mềm.
- **Hash-chain phụ thuộc serial ID runtime**: không sống sót PITR restore (ADR-004 hệ quả) — hash trên nội dung canonical.
- **Trùng nguồn sự thật** (signed note vừa ở `clinical_notes` vừa `emr_documents` mutable): kết tinh một chiều, note→document, document seal.

---

## 7. Lộ trình luyện tập NGAY trong repo

> Repo chưa có code — bài tập tạo file *(planned)* theo layout §9 và chứng minh invariant bằng test.

- 🥉 **Cơ bản — vẽ & viết DDL.** Trong `backend/migrations/` tạo bản nháp `00000X_encounter.up.sql` cho `encounters` + `diagnoses` + `observations` với: UUID v7 PK, `branch_id NOT NULL`, ENABLE+FORCE RLS + policy USING&WITH-CHECK, triplet `(code,system,display)`. Vẽ mermaid ERD encounter→fact. Tự kiểm: mọi fact FK `encounter_id`?

- 🥈 **Trung cấp — state machine + versioning.** Viết `backend/internal/encounter/domain/encounter.go` *(planned)*: enum status + hàm `Transition(to)` reject bước không hợp lệ (vd `planned→closed`). Thêm `clinical_notes_history` + logic snapshot khi sửa-trước-ký. Viết table-driven unit test cho mọi transition hợp lệ/bất hợp lệ (≥80% per testing rule).

- 🥇 **Nâng cao — seal + hash-chain + branch-isolation.** Hiện thực `EMRDocument.Sign()`/`Amend()` immutable (không mutate, trả bản mới), tính `content_hash` + nối `prev_hash`. Viết integration test testcontainers chứng minh: (a) UPDATE emr_documents sau ký bị domain từ chối; (b) record branch-B vô hình dưới `app.current_branch=A` (kế thừa DATA-1, ADR-025); (c) hash-chain verify lại được sau khi dump/restore. Đây là loại invariant không mock được — phải real Postgres.

---

## 8. Skill/agent ECC nên dùng khi luyện

- **`ecc:postgres-patterns`** — review schema/index/constraint, JSONB vs cột quan hệ, RLS policy cho bảng lâm sàng mới.
- **`ecc:database-migrations`** — kiểm migration zero-downtime (add nullable→backfill→switch), thứ tự chạy sau 000001 (ADR-024).
- **`ecc:healthcare-emr-patterns`** — pattern encounter/EMR/immutability, clinical safety cho data model.
- **`ecc:go-review`** (go-reviewer) — review immutability của aggregate (không mutate `EMRDocument`), value object `CodedConcept`, coding-style rule.
- **`ecc:go-test`** — TDD table-driven cho state machine + testcontainers cho RLS/hash-chain invariant.
- **`ecc:security-review` / `ecc:healthcare-phi-compliance`** — xác minh branch-isolation + audit cho PHI read trên bảng mới.

---

## 9. Tài nguyên học thêm (2024–2026)

- HL7 FHIR R4 — Encounter, Condition, Observation resources (mapping seam Phase 2): https://hl7.org/fhir/R4/encounter.html
- RFC 9562 — UUID version 6/7/8 (chuẩn hóa 2024): https://www.rfc-editor.org/rfc/rfc9562.html
- PostgreSQL 16 — Row Security Policies & DDL constraints: https://www.postgresql.org/docs/16/ddl-rowsecurity.html
- QĐ 4469/QĐ-BYT — danh mục ICD-10 dùng tại VN (tra cứu Cổng TTĐT Bộ Y tế): https://moh.gov.vn
- TT 13/2025/TT-BYT — bệnh án điện tử ký số (tra cứu hệ thống VBPL): https://vbpl.vn
- sqlc docs — query-first codegen từ schema (đọc cùng migrations, ADR-024): https://docs.sqlc.dev
- Martin Fowler — Event Sourcing & Temporal Patterns (nền cho append-only EMR): https://martinfowler.com/eaaDev/EventSourcing.html

---

## 10. Checklist "đã hiểu"

- [ ] Giải thích được vì sao fact lâm sàng FK tới `encounter_id` chứ không `patient_id` (ADR-004), và ngoại lệ `patient_allergies`.
- [ ] Viết được DDL bảng lâm sàng mới với UUID v7 PK + `branch_id NOT NULL` + ENABLE & FORCE RLS + USING&WITH-CHECK.
- [ ] Phân biệt được tại sao audit_log/stock_ledger dùng BIGINT IDENTITY còn encounter/diagnosis dùng UUID v7.
- [ ] Lưu được coded field dạng triplet `(code, system, display)` và biết vì sao snapshot `display` (ADR-016).
- [ ] Liệt kê đúng 7 trạng thái Encounter state machine và biết transition nào hợp lệ.
- [ ] Phân biệt mức bất biến: note mutable-trước-ký (+history) vs emr_documents sealed-sau-ký (addendum-only).
- [ ] Giải thích hash-chain EMR phải sống sót PITR restore và synchronous durability cho signed write (ADR-004/015).
- [ ] Chỉ ra anti-pattern hard-FK cứng vào `terminology_concepts` và cách validate mềm.
- [ ] Viết được integration test (testcontainers) chứng minh branch-B invisible + sửa-sau-ký bị reject (ADR-025).
- [ ] Biết encounter BC phát `EncounterClosed/Diagnosed/EMRSigned` qua outbox drive charge/claim/interop (ADR-011/012).
