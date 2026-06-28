# [INT-2] FHIR R4 Facade & Integration Engine

> Module INT-2 · FHIR R4 facade read-only + OIE HL7v2/ASTM sidecar + Orthanc DICOM + terminology service · Độ khó: 🥉→🥇 · Prereqs: INT-1 (Interop foundations + BHYT 4750 + e-prescription)

> ⚠️ Toàn bộ module này mô tả **Phase 2 (planned)** — MVP ships ZERO external FHIR/HL7/PACS (ADR-016). Repo CHƯA CÓ CODE; mọi code path đánh dấu *(planned)*. Module này dạy thiết kế MỤC TIÊU để Phase 2 rẻ, vì nền móng đã được bake-in từ MVP.

---

## 1. Vì sao kỹ năng này quan trọng trong HMS

Một bệnh viện không phải ốc đảo. Khi MVP một khoa OPD-BHYT đã chạy, bệnh viện cần **trao đổi dữ liệu lâm sàng** với hệ sinh thái: máy xét nghiệm (analyzer) trả kết quả qua HL7v2/ASTM, máy CĐHA đẩy ảnh DICOM, hệ thống Sở Y tế và (Phase 4) hồ sơ sức khỏe điện tử quốc gia đòi FHIR R4. Nếu mỗi giao tiếp này được nhúng thẳng vào domain lâm sàng, hệ thống sẽ có **hai nguồn sự thật** (OLTP và FHIR resource) — một thảm họa maintainability cho dữ liệu y tế regulated.

HMS chọn **facade-over-embedded-server** (ADR-017): OLTP PostgreSQL là single source of truth, interoperability BC CHỈ **dịch** (map OLTP → FHIR resource khi có ai đó GET), không sở hữu dữ liệu lâm sàng, không persist FHIR. Kỹ năng cần học: thiết kế một lớp dịch read-mostly, đặt integration engine (OIE) đúng chỗ trong K8s, và vận hành terminology service ($lookup/$validate-code/$expand) mà không lock vào một thư viện FHIR đã chết.

Đây là nơi quyết định kiến trúc Phase 2 thành-hay-bại: nếu MVP đã bake-in đúng (cột `(code,system,display)` triplet, Encounter anchor, outbox), facade Phase 2 chỉ là một adapter mỏng. Nếu MVP làm sai (lưu free-text diagnosis, treo dữ liệu vào patient_id), Phase 2 sẽ là rewrite. Module này chỉ ra chính xác cái gì MVP phải bake-in (đã làm ở INT-1) và cái gì Phase 2 mới build.

## 2. Mô hình tư duy (first principles) — từ con số 0

Bắt đầu từ một câu hỏi: *"Hệ thống khác muốn đọc một lần khám của bệnh viện. Họ nói ngôn ngữ gì?"*

1. **Interoperability = dịch giữa các mô hình dữ liệu.** Mỗi hệ thống có schema riêng. Chuẩn (FHIR, HL7v2, DICOM) là "ngôn ngữ chung". Vấn đề không phải truyền byte — mà là **ngữ nghĩa**: "diagnosis" của tôi có cùng nghĩa "Condition" của bạn không?

2. **Ngữ nghĩa cần mã chuẩn hóa, không phải free-text.** "Tăng huyết áp" vô nghĩa với máy. `I10` trong hệ `ICD-10` thì có nghĩa. Vì vậy MVP đã bake-in triplet `(code, system, display)` cho MỌI field lâm sàng (ADR-016, INT-1) — đây là viên gạch đầu tiên, không backfill được.

3. **Facade vs embedded server.** Embedded FHIR server = persist FHIR resource = nguồn sự thật thứ hai → phải sync hai chiều → bug-prone. Facade = OLTP là sự thật duy nhất, FHIR resource được **render on-the-fly** từ OLTP khi có GET. Read-mostly. Không bao giờ là canonical store (ADR-017).

4. **Translation cần một mỏ neo.** Bạn map *cái gì* sang FHIR Encounter? Phải có một aggregate tương ứng 1-1. HMS đã chọn Encounter làm mỏ neo lâm sàng (ADR-004): mọi vitals/diagnosis/order treo vào `encounter_id`. FHIR Encounter ↔ HMS Encounter là ánh xạ tự nhiên — đó là "seam ánh xạ FHIR" đã thiết kế từ MVP.

5. **Engine boundary ≠ application.** Máy analyzer/PACS nói HL7v2/DICOM, KHÔNG nói REST/JSON của bạn. Cần một **integration engine** (OIE) làm "ổ cắm chuyển đổi" — nhận HL7v2, biến thành domain event, đẩy vào bus. Engine sống ở **namespace riêng (interop)**, KHÔNG sau Kong (ADR-016) vì traffic máy móc không phải traffic người dùng có JWT.

6. **Defer là quyết định, không phải lười.** Tất cả những thứ này là Phase 2/3. MVP ships ZERO external (ADR-016) vì manual lab-result entry chấp nhận được, và một interop layer vận hành kém còn nguy hiểm hơn không có (ADR-002 budget logic).

## 3. Khái niệm cốt lõi (tăng dần độ khó)

| # | Khái niệm | Bản chất trong HMS |
|---|-----------|--------------------|
| 1 | **FHIR R4 Resource** | Đơn vị dữ liệu chuẩn HL7 FHIR (Patient, Encounter, Condition, Observation, MedicationRequest). HMS render từ OLTP, KHÔNG persist. |
| 2 | **Facade read-only** | Endpoint `GET /fhir/R4/Encounter/{id}` đọc OLTP → map → trả FHIR JSON. Không POST/PUT canonical ở Phase 2 (bidirectional là Phase 3). |
| 3 | **Triplet `(code, system, display)`** | Mọi field lâm sàng MVP đã có 3 cột. Map trực tiếp sang FHIR `CodeableConcept.coding[].{code,system,display}`. |
| 4 | **Terminology service** | `$lookup` (mã→nghĩa), `$validate-code` (mã có hợp lệ trong ValueSet?), `$expand` (liệt kê ValueSet). Seeded từ danh mục dùng chung BYT trước, LOINC/RxNorm/SNOMED sau. |
| 5 | **CodeSystem / ValueSet / ConceptMap** | 3 resource quản lý thuật ngữ. ConceptMap dịch giữa hệ mã (vd ICD-10 ↔ SNOMED). Bảng `code_systems`, `value_sets`, `concept_maps` (interoperability BC). |
| 6 | **OIE (Open Integration Engine)** | Sidecar/service nhận HL7v2 (kết quả lab máy), ASTM (analyzer cũ), parse → domain event → outbox/bus. Successor của Mirth Connect. |
| 7 | **HL7v2 message** | Bản tin pipe-delimited (`MSH|^~\&|...`). Vd ORU^R01 = kết quả quan sát. OIE parse, KHÔNG để domain code chạm pipe-syntax. |
| 8 | **Orthanc / DICOM / PACS** | Orthanc là PACS open-source lưu ảnh DICOM (CĐHA). HMS lưu **reference** (study UID) + render link, KHÔNG lưu pixel data trong OLTP. |
| 9 | **SMART-on-FHIR** | OAuth2 launch framework cho app bên thứ ba đọc FHIR. Phase 3, sau khi facade ổn định. |

Thứ tự độ khó: 1→4 là read-side translation (Phase 2 core). 5 là terminology (nền cho validate). 6→8 là engine boundary (machine integration). 9 là external app exchange (Phase 3).

## 4. HMS dùng nó thế nào (bám code path — *(planned)*)

> Layout mục tiêu (canon §9). Interoperability là BC `clean+ddd+cqrs`, KHÔNG sở hữu dữ liệu lâm sàng — chỉ projection/translation. Trong layout planned nó nằm dưới `internal/shared/interop` hoặc `internal/interop` được wiring tại `cmd/hms-api/main.go` (composition root DUY NHẤT).

```
internal/shared/interop/            (planned, Phase 2)
├── fhir/
│   ├── domain/        # FhirResourceView (projection, read-only) — KHÔNG aggregate sở hữu data
│   ├── app/query/     # MapEncounterToFhir, MapPatientToFhir (đọc từ encounter/patient BC qua read port)
│   ├── ports/         # EncounterReadPort, TerminologyPort
│   └── adapters/
│       ├── http/      # GET /fhir/R4/{Resource}/{id}  (read-only)
│       └── postgres/  # code_systems, value_sets, concept_maps
├── terminology/       # $lookup / $validate-code / $expand
└── engine/            # OIE adapter: HL7v2 ORU → domain event → outbox
```

- **Facade mapping** *(planned)*: `internal/shared/interop/fhir/app/query/map_encounter.go` đọc `encounters` + FK lâm sàng (`diagnoses`, `observations`) qua read port của encounter BC. KHÔNG import chéo domain (ADR-001 depguard) — chỉ qua port. Triplet `(code,system,display)` map thẳng sang `CodeableConcept`. **KHÔNG lock samply/golang-fhir-models** (chết Dec 2022, ADR-017) — Phase 2 đánh giá active forks (fastenhealth/gofhir-models) hoặc generate R4 structs in-house.
- **Terminology** *(planned)*: bảng `code_systems`, `value_sets`, `concept_maps` do interoperability BC sở hữu (canon §4). Lưu ý MVP: terminology catalog `terminology_concepts` thuộc **patient BC** (ICD-10/LOINC/RxNorm/DMDC dùng chung) — Phase 2 mới promote thành CodeSystem/ValueSet đầy đủ.
- **OIE placement** *(planned)*: `deploy/` đặt OIE ở **namespace `interop` riêng, KHÔNG sau Kong** (ADR-016). Máy analyzer push HL7v2 ORU^R01 vào OIE → OIE parse → emit `ResultReleased` domain event vào outbox → lab BC subscribe (idempotent qua `processed_events`). Điều này thay thế **manual lab-result entry** của MVP (canon §6 mục 4).
- **Orthanc/DICOM** *(planned)*: Orthanc deploy như Helm operator trong namespace `interop`; OLTP lưu study reference (UID), FE render viewer link. Pixel data KHÔNG vào Postgres.
- **Event bus dependency**: OIE feed bus **chỉ khi bus tồn tại** (ADR-016 hệ quả). MVP outbox là in-process (ADR-012); khi tách BC ra service (trigger ADR-001) thì swap relay adapter sang Kafka — OIE feed vào đó.

```go
// internal/shared/interop/fhir/app/query/map_encounter.go  (planned, illustrative)
// Read-only facade: OLTP là single source of truth (ADR-017), KHÔNG persist FHIR.
func (q *FhirQuery) MapEncounter(ctx context.Context, id uuid.UUID) (*FhirEncounter, error) {
    // Chạy trong tx đã SET LOCAL app.current_branch (RLS invariant, ADR-003/005)
    enc, err := q.encounterRead.GetByID(ctx, id) // qua port, KHÔNG import encounter/domain
    if err != nil {
        return nil, fmt.Errorf("read encounter: %w", err) // wrap, không nuốt lỗi
    }
    diags := make([]FhirCoding, 0, len(enc.Diagnoses))
    for _, d := range enc.Diagnoses {
        // Triplet bake-in từ MVP → CodeableConcept (ADR-016)
        diags = append(diags, FhirCoding{System: d.System, Code: d.Code, Display: d.Display})
    }
    return &FhirEncounter{ResourceType: "Encounter", ID: enc.ID.String(), Diagnosis: diags}, nil
}
```

```yaml
# deploy/kustomize/base/oie/  (planned) — OIE ở namespace interop, KHÔNG sau Kong (ADR-016)
apiVersion: apps/v1
kind: Deployment
metadata: { name: oie, namespace: interop }   # tách khỏi PHI app namespace
spec:
  replicas: 2
  template:
    spec:
      securityContext: { runAsNonRoot: true, seccompProfile: { type: RuntimeDefault } }
      # NetworkPolicy default-deny; chỉ allow analyzer subnet → OIE:2575 (MLLP) (canon §3 deploy)
```

## 5. Best practices (mỗi mục có nguồn)

1. **Facade, không embedded server — OLTP là single source of truth.** Persist FHIR = second source of truth = sync hell. Render on-read. Nguồn: HL7 FHIR — Architecture & Designing Resources, https://www.hl7.org/fhir/R4/overview-arch.html
2. **Đừng lock vào thư viện FHIR chết.** samply/golang-fhir-models last release Dec 2022. Đánh giá lại ở Phase 2 (active forks hoặc generate). Nguồn ADR-017; FHIR R4 spec để generate structs: https://www.hl7.org/fhir/R4/downloads.html
3. **Terminology là first-class: $validate-code trước khi nhận mã ngoài.** Đừng tin mã từ hệ thống khác. Nguồn: HL7 FHIR Terminology Service, https://www.hl7.org/fhir/R4/terminology-service.html
4. **Đặt integration engine ở biên riêng, không qua API gateway người-dùng.** Machine traffic (MLLP/HL7v2) khác HTTP+JWT. Nguồn: Open Integration Engine (OIE, successor Mirth) docs, https://openintegrationengine.org/
5. **HL7v2 parse ở engine, domain chỉ thấy event sạch.** Đừng để pipe-syntax rò vào aggregate. Nguồn: HL7v2 Messaging Standard overview, https://www.hl7.org/implement/standards/product_brief.cfm?product_id=185
6. **PACS lưu reference, không lưu pixel trong OLTP.** Orthanc/DICOMweb giữ ảnh; HMS giữ study UID. Nguồn: Orthanc Book — DICOMweb & integration, https://orthanc.uclouvain.be/book/
7. **Map qua port, không import chéo BC** (giữ extraction path mở, ADR-001). Nguồn: canon §9 layer rule + depguard golangci-lint, https://golangci-lint.run/usage/linters/

## 6. Lỗi thường gặp & anti-patterns

- ❌ **Build FHIR facade trong MVP.** Vi phạm ADR-016 (MVP ships ZERO external). Manual lab entry là chấp nhận được. Scaffolding sớm = budget violation (ADR-002).
- ❌ **Persist FHIR resource thành canonical store.** Tạo nguồn sự thật thứ hai → phải sync → bug. Facade read-only render từ OLTP (ADR-017).
- ❌ **Lock samply/golang-fhir-models bây giờ.** Dependency chết 3.5 năm, R4-only, 4 open issue → maintainability/security hazard cho layer regulated (ADR-017). Đánh giá lại Phase 2.
- ❌ **Đặt OIE sau Kong.** Máy analyzer không có JWT; MLLP không phải HTTP. OIE ở namespace `interop` riêng (ADR-016).
- ❌ **Để HL7v2 pipe-syntax / DICOM tag rò vào domain.** Engine phải parse sạch thành domain event trước outbox.
- ❌ **Free-text diagnosis ở MVP** (giết Phase 2). Triplet `(code,system,display)` là hard requirement MVP (ADR-016) — không có thì facade không map được.
- ❌ **OIE feed bus khi chưa có bus.** MVP outbox in-process (ADR-012); engine→bus async chỉ Phase 2+ khi bus tồn tại.
- ❌ **Bỏ RLS khi facade query.** FHIR read vẫn chạy trong tx đã `SET LOCAL app.current_branch` (ADR-003/005) — facade không phải đường vòng qua RLS.
- ❌ **Nhận mã ngoài không $validate-code.** Mã sai/lạ vào OLTP làm hỏng claim/báo cáo. Validate trước.

## 7. Lộ trình luyện tập NGAY trong repo (🥉→🥈→🥇)

> Repo CHƯA CÓ CODE — bài tập là thiết kế/scaffold layout planned + viết spec, KHÔNG implement production.

- 🥉 **Cơ bản — đọc & truy vết foundation.** Mở `doc_tech/interoperability/01-coded-data-bhyt-eprescription.md` (INT-1) và `doc/17-interoperability.md`. Liệt kê chính xác 4 thứ MVP bake-in để Phase 2 rẻ (triplet, Encounter anchor, outbox, terminology catalog). Vẽ bảng "MVP có sẵn → Phase 2 build thêm". Xác nhận terminology_concepts thuộc patient BC (canon §4) còn code_systems/value_sets/concept_maps thuộc interoperability BC.

- 🥈 **Trung cấp — thiết kế mapping & terminology.** Tạo skeleton `internal/shared/interop/fhir/{domain,app/query,ports,adapters}/` *(planned)* với interface (chưa impl): `EncounterReadPort`, `MapEncounterToFhir`. Viết một bảng ánh xạ HMS→FHIR R4: `encounters`→Encounter, `diagnoses`→Condition, `observations`→Observation, `prescriptions`→MedicationRequest, `patients`→Patient. Cho mỗi field, chỉ rõ triplet nào map sang `CodeableConcept.coding`. Soạn 3 ví dụ terminology op: `$lookup` mã ICD-10 I10, `$validate-code` trong ValueSet chẩn đoán, `$expand` danh mục dùng chung BYT.

- 🥇 **Nâng cao — engine boundary + deploy topology.** Viết một manifest Kustomize `deploy/kustomize/base/oie/` và `orthanc/` *(planned)* đặt cả hai ở namespace `interop` (KHÔNG sau Kong), kèm NetworkPolicy default-deny chỉ allow analyzer subnet → OIE MLLP:2575. Vẽ mermaid sequence: analyzer → OIE (parse HL7v2 ORU^R01) → domain event → bus/outbox → lab BC `ResultReleased` (idempotent). Viết ADR ngắn (proposed) "FHIR library selection Phase 2" so sánh fastenhealth/gofhir-models vs generate-in-house, neo vào ADR-017.

## 8. Skill/agent ECC nên dùng khi luyện

- **ecc:go-review** (go-reviewer) — review facade mapping code (port boundary, error wrapping, không import chéo BC).
- **ecc:healthcare-emr-patterns** — pattern EMR/encounter, đảm bảo facade map đúng ngữ nghĩa lâm sàng.
- **ecc:healthcare-phi-compliance** — kiểm facade read vẫn fail-closed audit + RLS, không leak PHI cross-branch.
- **ecc:kubernetes-patterns** — review OIE/Orthanc placement (namespace isolation, NetworkPolicy, securityContext).
- **ecc:security-review** — biên engine: machine traffic không qua JWT, mTLS/NetworkPolicy đúng.
- **ecc:architecture-decision-records** — viết ADR "FHIR library Phase 2" theo chuẩn, neo ADR-017.
- **ecc:documentation-lookup** / **deep-research** — tra FHIR R4 spec, OIE docs, Orthanc Book khi cần fact chính xác.

## 9. Tài nguyên học thêm (2024–2026)

- HL7 FHIR R4 (4.0.1) — Resource Index & Architecture: https://www.hl7.org/fhir/R4/
- FHIR R4 Terminology Service ($lookup/$validate-code/$expand): https://www.hl7.org/fhir/R4/terminology-service.html
- FHIR R4 downloads (generate structs in-house): https://www.hl7.org/fhir/R4/downloads.html
- Open Integration Engine (OIE — successor của Mirth Connect): https://openintegrationengine.org/
- Orthanc Book (DICOM/DICOMweb, PACS open-source): https://orthanc.uclouvain.be/book/
- HL7v2 Messaging Standard product brief: https://www.hl7.org/implement/standards/product_brief.cfm?product_id=185
- SMART App Launch (SMART-on-FHIR): https://hl7.org/fhir/smart-app-launch/
- DICOMweb standard (PS3.18 REST): https://www.dicomstandard.org/using/dicomweb
- Cổng dữ liệu y tế / danh mục dùng chung BYT (seed terminology): https://kcb.vn/ (Cục Quản lý Khám chữa bệnh)

## 10. Checklist "đã hiểu"

- [ ] Giải thích được vì sao FHIR/HL7/PACS bị defer khỏi MVP (ADR-016) và 4 foundation nào MVP bake-in để Phase 2 rẻ.
- [ ] Phân biệt facade read-only vs embedded FHIR server, và vì sao OLTP là single source of truth (ADR-017).
- [ ] Chỉ ra triplet `(code,system,display)` map sang FHIR `CodeableConcept.coding` thế nào, và vì sao free-text giết Phase 2.
- [ ] Nêu được 3 terminology op ($lookup/$validate-code/$expand) và bảng nào sở hữu chúng (code_systems/value_sets/concept_maps, interoperability BC).
- [ ] Vẽ được nơi đặt OIE và Orthanc trong K8s (namespace `interop`, KHÔNG sau Kong) và lý do.
- [ ] Mô tả luồng HL7v2 ORU^R01 → OIE parse → domain event → bus/outbox → lab BC, thay manual lab entry MVP.
- [ ] Giải thích vì sao KHÔNG lock samply/golang-fhir-models và phương án Phase 2 (active forks hoặc generate in-house).
- [ ] Xác nhận facade query vẫn chạy trong tx đã SET LOCAL app.current_branch (RLS không bị bypass) và đi qua port (không import chéo BC).
- [ ] Biết OIE feed bus chỉ khi bus tồn tại (liên hệ ADR-012 outbox in-process → swap Kafka khi tách service).
