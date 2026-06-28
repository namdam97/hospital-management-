# 13 — Architecture Decision Records (ADR-001 … ADR-025)

> Sổ đăng ký quyết định kiến trúc của HMS (Hospital Management System). Đây là nguồn sự thật cho MỌI quyết định lớn: tên bounded context, lựa chọn công nghệ pinned, tên bảng. Mọi file `doc/` và `doc_tech/` khi nói "vì sao" PHẢI neo về một ADR-XXX dưới đây.
>
> Liên quan: [`doc/00-tong-quan.md`](00-tong-quan.md) (vision), [`doc/01-kien-truc-tong-the.md`](01-kien-truc-tong-the.md) (4 tầng defense-in-depth), [`doc/02-backend-architecture.md`](02-backend-architecture.md) (BC + outbox), [`doc/08-database-schema.md`](08-database-schema.md) (RLS/encryption), [`doc/10-deployment-operations.md`](10-deployment-operations.md) (platform budget + earn-in triggers), [`doc/12-roadmap.md`](12-roadmap.md) (phasing).
>
> Repo HIỆN CHƯA CÓ CODE — các ADR mô tả THIẾT KẾ MỤC TIÊU; code path tham chiếu theo layout *(planned)* tại [`doc/02-backend-architecture.md`](02-backend-architecture.md).

## Quy ước ADR

Mỗi record gồm: **Context** (bối cảnh/áp lực), **Decision** (quyết định binding), **Rationale** (lý do, neo vào lens council: patient-safety / workflow / data / security), **Alternatives** (phương án đã loại + lý do từ chối), **Consequences** (hệ quả ràng buộc downstream). Tất cả 25 ADR đều `status: accepted` (bản chốt hội đồng kiến trúc). Khi một ADR bị thay thế trong tương lai, đổi `status: superseded by ADR-NNN` chứ KHÔNG xóa.

## Mục lục

| # | ADR | Chủ đề | Lens |
|---|-----|--------|------|
| 001 | [ADR-001](#adr-001--go-modular-monolith-tách-service-chỉ-khi-có-trigger-proven) | Go modular monolith, tách service theo trigger | Architecture |
| 002 | [ADR-002](#adr-002--mvp-component-budget-cứng--named-operating-model) | MVP component budget + operating model trước stack | Ops/Risk |
| 003 | [ADR-003](#adr-003--force-row-level-security--migration-owner-tách-app-role-keystone-phi) | FORCE RLS + migration-owner role (keystone PHI) | Patient-safety |
| 004 | [ADR-004](#adr-004--encounter-là-mỏ-neo-lâm-sàng-emr-ký-số-bất-biến-tt-132025) | Encounter anchor + EMR ký số bất biến | Clinical/Legal |
| 005 | [ADR-005](#adr-005--multi-tenancy-branch_id--rls-mpi-dùng-chung) | Multi-tenancy branch_id + RLS, MPI dùng chung | Data |
| 006 | [ADR-006](#adr-006--bhyt-là-phụ-thuộc-live-hai-chạm--degraded-mode) | BHYT live hai chạm + degraded-mode | Workflow |
| 007 | [ADR-007](#adr-007--liên-thông-đơn-thuốc-quốc-gia-trong-mvp-tt-262025) | Liên thông đơn thuốc quốc gia trong MVP | Workflow/Legal |
| 008 | [ADR-008](#adr-008--cdss-hard-stop-enforce-backend-fail-closed) | CDSS hard-stop enforce backend, fail-closed | Patient-safety |
| 009 | [ADR-009](#adr-009--audit-of-reads-commit-with-response-hash-chain--worm) | Audit-of-reads commit-with-response + WORM | Security/Legal |
| 010 | [ADR-010](#adr-010--break-the-glass-time-boxed--scoped--closed-review-loop) | Break-the-glass time-boxed + scoped + review | Patient-safety |
| 011 | [ADR-011](#adr-011--charge-capture-idempotent--claimbillencounter-fk--saga) | Charge-capture idempotent + claim↔bill FK + saga | Data |
| 012 | [ADR-012](#adr-012--một-messaging-in-process-outbox-một-job-framework-river) | Một messaging (outbox) + một job framework (River) | Architecture |
| 013 | [ADR-013](#adr-013--keycloak-oidc--kong-edge-auth--object-level-authz-ở-go) | Keycloak OIDC + Kong edge-auth + authz ở Go | Security |
| 014 | [ADR-014](#adr-014--field-level-encryption-app-side-envelope--blind-index-hmac) | Field-level encryption app-side + blind-index | Security |
| 015 | [ADR-015](#adr-015--postgres-mvp-vn-managed-ưu-tiên-cnpg-1-primary--1-async) | Postgres MVP: VN-managed / CNPG async | Ops |
| 016 | [ADR-016](#adr-016--interop-phasing-mvp-foundations--hai-regulatory-externals) | Interop phasing: foundations + hai externals | Architecture |
| 017 | [ADR-017](#adr-017--không-cam-kết-samplygolang-fhir-models-đánh-giá-lại-phase-2) | Không lock thư viện FHIR chết | Maintainability |
| 018 | [ADR-018](#adr-018--frontend-vite-spa--ant-design-v6--kong-bff-không-pwa-write-outbox) | Frontend Vite SPA + AntD v6 + Kong BFF | Frontend |
| 019 | [ADR-019](#adr-019--kong-kic-db-less--argo-cd-rolling-không-canarytempo-mvp) | Kong KIC DB-less + Argo CD rolling | Deploy |
| 020 | [ADR-020](#adr-020--dpia--consent--data-subject-rights-là-phase-0-legal-deliverable) | DPIA + consent + DSR là Phase-0 legal | Legal |
| 021 | [ADR-021](#adr-021--fefo--lotexpiry--stock_ledger-append-only-tránh-pci-scope) | FEFO + stock_ledger append-only, tránh PCI | Pharmacy/Compliance |
| 022 | [ADR-022](#adr-022--dual-run--print-phiếu-pháp-lý-là-high-priority-với-kpi-owner) | Dual-run + print phiếu pháp lý HIGH | Change-mgmt |
| 023 | [ADR-023](#adr-023--bhxh-sandbox--rejection-code-handling-là-phase-0-blocker) | BHXH sandbox + rejection-code là Phase-0 | Workflow |
| 024 | [ADR-024](#adr-024--migrations-golang-migrate--phase-0-migration-000001) | golang-migrate + Phase-0 migration 000001 | Data |
| 025 | [ADR-025](#adr-025--testing-strategy-testcontainers--e2e-critical-flows) | Testing: testcontainers + E2E critical flows | Quality |

---

### ADR-001 — Go modular monolith, tách service chỉ khi có trigger proven
*status: accepted*

- **Context**: Bệnh viện vừa rời giấy/Excel, đội IT nhỏ kiêm ops. Các luồng cốt lõi (charge-capture, claim↔bill linkage, FEFO stock movement) cần tính nhất quán mạnh.
- **Decision**: Xây HMS là một **Go modular monolith** (`hms-api`, một container, một deployable) — mỗi BC là một domain folder `internal/<bc>/{domain,app,ports,adapters}` + `shared/` kernel. Tách BC ra service riêng CHỈ khi có nhu cầu scaling độc lập hoặc cô lập compliance được chứng minh; vì cross-BC đã đi qua outbox nên tách = swap relay adapter sang Kafka, domain code không đổi.
- **Rationale**: Cần ACID nội-process cho charge-capture, claim↔bill, FEFO — microservices chỉ làm phức tạp. Một deployable vận hành được bởi đội nhỏ. Council xác nhận không over-engineered.
- **Alternatives**: Microservices-per-BC từ đầu (loại: vận hành quá tải, mất ACID, distributed transaction phức tạp); SOA 3–4 service (loại: chưa có trigger).
- **Consequences**: `depguard` (golangci-lint) cấm cross-BC import; cross-BC chỉ qua outbox; kỷ luật layer `adapters→ports←app→domain`; extraction là đường tiến hóa đã thiết kế sẵn (xem [ADR-012](#adr-012--một-messaging-in-process-outbox-một-job-framework-river)).

### ADR-002 — MVP component budget cứng + named operating model
*status: accepted*

- **Context**: Council liệt kê 15+ hệ thống stateful độc lập muốn vào MVP cho một khoa OPD — collectively unrunnable bởi đội nhỏ.
- **Decision**: Chốt **mô hình vận hành** (dev team kiêm ops ở MVP, dedicated SRE khi mở rộng) TRƯỚC khi chọn stack. MVP CHỈ chạy: managed/CNPG-async Postgres + Go monolith + Kong KIC DB-less + KMS/ESO secrets + Argo CD rolling deploy + Prometheus+Loki — **không hệ thống stateful nào khác**. Mỗi hệ thống defer (Vault-đầy-đủ, NATS/Kafka, Debezium, OIE, Orthanc, FHIR facade, service mesh, canary) gắn một **earn-in trigger** viết sẵn trong [`doc/10-deployment-operations.md`](10-deployment-operations.md) + [`doc/12-roadmap.md`](12-roadmap.md).
- **Rationale**: Vault/HA-Postgres/Kafka vận hành kém còn nguy hiểm cho PHI hơn managed đơn giản. Biến "Phase N on proven need" thành gate cưỡng chế.
- **Alternatives**: Front-load full stack ngày 1 (loại: operational risk = PHI risk); không có budget (loại: build team âm thầm scaffold sớm).
- **Consequences**: Mọi đề xuất thêm stateful system phải dẫn chứng trigger đã đạt; review pipeline kiểm budget; tốc độ Phase 2+ phụ thuộc năng lực ops thực tế.

### ADR-003 — FORCE ROW LEVEL SECURITY + migration-owner tách app-role (keystone PHI)
*status: accepted*

- **Context**: Shared-schema multi-tenant. PostgreSQL table OWNER **bypass RLS kể cả** `NOBYPASSRLS`; nếu `golang-migrate` chạy bằng app-role → app-role thành owner → RLS thành no-op âm thầm.
- **Decision**: Mọi bảng PHI `ENABLE` + **`FORCE ROW LEVEL SECURITY`** trong migration `000001`; tách **role-migration-owner** (sở hữu bảng, chạy DDL) khỏi **app-role** (`NOSUPERUSER`, `NOBYPASSRLS`, KHÔNG sở hữu bảng); mọi policy có CẢ `USING` VÀ `WITH CHECK` (`branch_id = current_setting('app.current_branch')::uuid`); CI integration test (testcontainers) chứng minh dữ liệu branch-B vô hình dưới `app.current_branch=A` là **merge-blocking gate**. Resource khác branch trả **404** (không 403).
- **Rationale**: Patient-safety lens `MOST IMPORTANT`. `USING`-only cho phép write cross-tenant (poisoning). Không retrofit được sau khi có data multi-tenant.
- **Alternatives**: Chỉ `ENABLE` RLS + `NOBYPASSRLS` (loại: owner vẫn bypass); chỉ `USING` (loại: thiếu `WITH CHECK` = write vector); schema-per-branch (defer: chỉ khi branch cực lớn).
- **Consequences**: Phải có Phase-0 migration; mọi PHI query chạy trong tx đã `SET LOCAL` GUC (invariant có test — pgx pool reuse connection, query ngoài tx mất filter); `branch_id` là cột dẫn đầu index. Xem [ADR-024](#adr-024--migrations-golang-migrate--phase-0-migration-000001).

### ADR-004 — Encounter là mỏ neo lâm sàng, EMR ký số bất biến (TT 13/2025)
*status: accepted*

- **Context**: Nghiệp vụ thực là "một lần khám/đợt điều trị". TT 13/2025/TT-BYT (hạn 30/9/2025, đã lapsed) bắt buộc EMR ký số cho bệnh viện đã cấp phép.
- **Decision**: Mọi sự kiện lâm sàng (vitals, chẩn đoán ICD-10 QĐ 4469, order, kết quả, đơn thuốc, charge) FK tới **`encounter_id`** (KHÔNG `patient_id` trực tiếp). `Encounter` có state machine `planned→arrived→triaged→in-progress→finished→billed→closed`, loại OPD/ED/IPD (IPD có `Admission` con). Khi đóng, kết tinh **`EMRDocument` bất biến ký số PKI** (bác sĩ + chữ ký tổ chức); signed→amendment-only + `*_history` versioning; field `signedBy/signedAt/signatureBlob/hash` từ đầu.
- **Rationale**: EMR ký số = remediation-of-non-compliance, không deferrable. Encounter là FHIR-mappable seam.
- **Alternatives**: Treo dữ liệu vào `patient_id` (loại: không phản ánh lượt khám, khó claim/FHIR); cho sửa EMR sau ký (loại: vi phạm tính pháp lý bất biến).
- **Consequences**: Signed-EMR write path cần synchronous durability (commit trước khi UI báo 'signed', xem [ADR-015](#adr-015--postgres-mvp-vn-managed-ưu-tiên-cnpg-1-primary--1-async)); hash-chain phải sống sót PITR restore; mọi BC lâm sàng phụ thuộc encounter contract.

### ADR-005 — Multi-tenancy branch_id + RLS, MPI dùng chung
*status: accepted*

- **Context**: Một người bệnh — một hồ sơ liên tục xuyên chi nhánh; nhưng dữ liệu lâm sàng phải cách ly theo chi nhánh.
- **Decision**: Shared-schema: **`branch_id UUID NOT NULL`** trên mọi bảng PHI, RLS lọc theo `app.current_branch` do Go middleware `SET LOCAL` trong tx (trích từ JWT đã verify, **KHÔNG lấy từ client**). Patient/MPI dùng chung toàn hệ thống (một `patient_id` xuyên chi nhánh); vai trò liên-chi-nhánh (giám định/quản lý vùng) qua policy escalation `cross_branch_reader`.
- **Rationale**: `branch_id` không bao giờ từ client (chống tenant spoofing). MPI dùng chung là yêu cầu nghiệp vụ.
- **Alternatives**: DB-per-branch (defer: chỉ khi cô lập pháp lý/hiệu năng đặc biệt — Phase 4); patient cục bộ theo branch + link (loại: phá liên tục hồ sơ, khó FHIR Patient).
- **Consequences**: `patient_identifiers` cần xử lý cross-branch cẩn trọng; RLS contract (session-var) là chuẩn org-wide mọi BC dùng. Nền tảng cho [ADR-003](#adr-003--force-row-level-security--migration-owner-tách-app-role-keystone-phi).

### ADR-006 — BHYT là phụ thuộc LIVE hai chạm + degraded-mode
*status: accepted*

- **Context**: Cổng giám định BHYT có API JSON card-check (verified). Council ban đầu chỉ mô hình XML export cuối encounter → để tiếp đón trên giấy. "Cổng/mạng lỗi" là trigger #1 staff quay về giấy.
- **Decision**: BHYT hai chạm: **(touch 1)** LIVE web service JSON tại tiếp đón trả giá trị thẻ + miễn cùng chi-trả + cờ thu hồi/tạm khóa + 6 lần khám gần nhất → verdict `eligible/ineligible/co-pay`; **(touch 2)** XML1–XML15 QĐ 4750 (sửa 3176, hiệu lực 01/01/2025) lúc quyết toán đẩy cổng giám định qua mTLS + saga + idempotency. **Degraded-mode bắt buộc**: cổng/mạng lỗi → admit-and-flag (thẻ provisionally-unverified, queue retry), không bao giờ chặn người bệnh; cashier thu + reconcile-later; UI hiện "đã lưu, chờ gửi cổng".
- **Rationale**: Workflow lens `MOST IMPORTANT`. Block khi cổng lỗi = đẩy staff về giấy vĩnh viễn.
- **Alternatives**: BHYT chỉ là batch XML cuối encounter (loại: không số hóa tiếp đón); block khi cổng lỗi (loại: permanent-paper).
- **Consequences**: Cần BHXH sandbox/test endpoint ([ADR-023](#adr-023--bhxh-sandbox--rejection-code-handling-là-phase-0-blocker)); claim↔invoice FK 1-1 ([ADR-011](#adr-011--charge-capture-idempotent--claimbillencounter-fk--saga)); rejection-code handling là state machine first-class trong `insurance` BC.

### ADR-007 — Liên thông đơn thuốc quốc gia trong MVP (TT 26/2025)
*status: accepted*

- **Context**: TT 26/2025 + QĐ 808 (hạn bệnh viện 1/10/2025, đã lapsed) bắt buộc gửi đơn ngoại trú lên `donthuocquocgia.vn` ngay sau khám. MVP IS một khoa OPD-kê-đơn.
- **Decision**: Adapter `donthuoc-quocgia` trong `pharmacy` BC đẩy đơn ngoại trú lên `donthuocquocgia.vn` ngay sau khám, dùng app-name/app-key + mã liên-thông cơ sở + mã liên-thông bác sĩ, lưu **mã đơn quốc gia** (semantics C/N/H/Y) trên `prescriptions`. Đẩy qua outbox + River retry idempotent.
- **Rationale**: Legally MVP-mandatory, không phải Phase 2. In đơn giấy không có mã quốc gia = digitization giả.
- **Alternatives**: Defer e-prescription tới Phase 2 (loại: tự mâu thuẫn với MVP department + đã quá hạn pháp lý).
- **Consequences**: Đơn thuốc in phải có QR/mã đơn + block chữ ký số ([ADR-022](#adr-022--dual-run--print-phiếu-pháp-lý-là-high-priority-với-kpi-owner)); cần mã liên-thông per branch (`organization` BC, bảng `facility_external_codes`); retry khi national system down.

### ADR-008 — CDSS hard-stop enforce backend, fail-closed
*status: accepted*

- **Context**: React modal bypassable (devtools/API/network). Silent pass khi CDSS check fail = bệnh nhân chết vì dị ứng đã ghi.
- **Decision**: Hard-stop CDSS enforce ở **order/dispense aggregate (server-side)**: command bị reject trừ khi có override record (reason + authorizer) ghi audit. Khi CDSS error/timeout: **fail-closed** — KHÔNG confirm "no known interaction". Khi bệnh nhân không có allergy data: trạng thái **"allergy status unknown" tách bạch**, hiển thị rõ, KHÔNG BAO GIỜ render là "safe". React modal chỉ là UX, không phải control.
- **Rationale**: Patient-safety. "Unknown" ≠ "safe" là phân biệt sống còn.
- **Alternatives**: CDSS chỉ ở FE modal (loại: không phải control); fail-open khi CDSS down (loại: safety hazard).
- **Consequences**: Order/dispense path phụ thuộc CDSS availability (hard-online cho dispense, xem [ADR-018](#adr-018--frontend-vite-spa--ant-design-v6--kong-bff-không-pwa-write-outbox)); override luôn ghi audit; allergy-unknown là first-class UI state.

### ADR-009 — Audit-of-reads commit-with-response, hash-chain + WORM
*status: accepted*

- **Context**: Reconcile mâu thuẫn Data-lens (audit in-tx) vs Security-lens (audit qua outbox async). Postgres-policy "immutable" KHÔNG tamper-evident với DBA/superuser.
- **Decision**: Audit PHI-read ghi **commit cùng/trước response** (KHÔNG best-effort async): nếu audit write fail thì KHÔNG trả PHI (**fail-closed**). State-change audit qua outbox. Bảng `audit_log` INSERT-only (policy chặn UPDATE/DELETE `USING false` kể cả app-role), partition theo tháng, FORCE RLS, **hash-chain** (mỗi record chứa hash record trước), stream **WORM sink** ngoài Postgres (object-lock). Ghi `who/when/action(read|create|update|delete|print|export)/resource/patient_id/before-after/ip/session/branch`.
- **Rationale**: Cho READ phải durable trước response (rollback/crash mất audit = vi phạm HIPAA §164.312(b)/NĐ13/TT13). Hash-chain + WORM external bắt buộc, không optional (chống DBA).
- **Alternatives**: Audit read async qua outbox (loại: window mất audit khi crash); chỉ Postgres immutable policy (loại: không chống DBA).
- **Consequences**: Thêm latency mỗi PHI read (chấp nhận cho compliance); `audit_log` không chịu RLS branch-leak; cần WORM storage onshore.

### ADR-010 — Break-the-glass time-boxed + scoped + closed review loop
*status: accepted*

- **Context**: Break-the-glass không có closed loop = authz bypass có log side-effect, auditor NĐ13 coi là finding. Council ban đầu chỉ có break-the-glass cho ACCESS, không cho CREATION/ordering — chỗ giấy thắng ở ED.
- **Decision**: Nhân viên khai lý do → cấp quyền tạm **time-boxed** (auto-expire N giờ), **scoped** tới patient/encounter cụ thể, sinh audit cờ đỏ + thông báo security officer. Có **named reviewer role, hard review SLA, consequence path** khi lạm dụng. Áp dụng cho CẢ access VÀ creation/ordering flow trong cấp cứu (ED register-first-identify-later).
- **Rationale**: Patient-safety lens; control gap nếu chỉ log không review.
- **Alternatives**: Break-the-glass chỉ logging không review (loại: control gap); không time-box/scope (loại: over-broad grant).
- **Consequences**: Cần quy trình tổ chức (reviewer, SLA) đi kèm kỹ thuật; ED flow phải tạo Encounter không cần appointment + chưa confirm MPI (merge sau). Aggregate `BreakGlassGrant` trong `identity-access` BC.

### ADR-011 — Charge-capture idempotent + claim↔bill↔encounter FK + saga
*status: accepted*

- **Context**: Prior-analysis gap: claim không link bill. Retry/replay gây double-post charge.
- **Decision**: Mọi order/dispense/dịch vụ sinh **`ChargeItem`** tự động vào Invoice của Encounter qua outbox event với **Idempotency-Key** (persisted unique-constraint, bảng `idempotency_keys`). `InsuranceClaim` sinh từ chính `ChargeItem` (claim & bill nhất quán), giữ FK cứng **claim↔bill↔encounter**. Quyết toán ra viện là **orchestration saga** (giải phóng tạm ứng + chốt claim). Claim submission là River job at-least-once + idempotent external call (dedupe trên claim reference).
- **Rationale**: Idempotency chống double-post khi retry/replay; saga cho luồng quyết toán multi-step.
- **Alternatives**: Charge thủ công (loại: Excel đối soát không biến mất); claim không link bill (loại: chính gap cần sửa).
- **Consequences**: FE offline-outbox idempotency key và backend charge/claim idempotency key PHẢI là MỘT scheme end-to-end (chống double-post khi replay); price snapshot tại thời điểm charge. Xem [ADR-018](#adr-018--frontend-vite-spa--ant-design-v6--kong-bff-không-pwa-write-outbox).

### ADR-012 — Một messaging (in-process outbox), một job framework (River)
*status: accepted*

- **Context**: Hai messaging system cho một monolith với zero cross-process consumer là vô nghĩa; River + Watermill overlap (cùng Postgres-native).
- **Decision**: MVP: **transactional outbox table** viết trong cùng `pgx.Tx`, relay tới in-process pub/sub (`SELECT FOR UPDATE SKIP LOCKED`, subscriber idempotent qua `processed_events`). KHÔNG NATS/Kafka/Debezium ở MVP. **River** (Postgres-native) là DUY NHẤT framework background job (reminders, claim submit/retry, FEFO sweep, audit retention, outbox cleanup) — KHÔNG ship River + Watermill song song. Trigger swap sang Kafka: tách BC ra service.
- **Rationale**: Outbox in-process là điểm chính của pattern — events ở in-process tới khi extraction proven ([ADR-001](#adr-001--go-modular-monolith-tách-service-chỉ-khi-có-trigger-proven)).
- **Alternatives**: NATS JetStream ngày 1 (loại: stateful broker zero consumer); Debezium+Kafka CDC cho analytics MVP (loại: pipeline 3-component cho báo cáo một khoa).
- **Consequences**: Analytics MVP qua scheduled SQL→read-table ([ADR-016](#adr-016--interop-phasing-mvp-foundations--hai-regulatory-externals)); CDC/Kafka là Phase 3; relay adapter viết sẵn interface để swap.

### ADR-013 — Keycloak OIDC + Kong edge-auth + object-level authz ở Go
*status: accepted*

- **Context**: Data residency NĐ13/TT46 → không IdP SaaS nước ngoài. BOLA/IDOR là lỗ hổng #1 y tế; gateway không có context lâm sàng. CVE-2026-29413 (CISA-KEV, exploited healthcare gateways) cho phép Kong auth-bypass.
- **Decision**: **Keycloak 26 self-hosted onshore** (một realm `hospital`, branch là group/attribute — KHÔNG realm-per-branch) là OIDC issuer; Go không quản password. Auth-code+PKCE cho SPA qua Kong BFF; client_credentials cho service-to-service. Kong verify JWT qua JWKS (**reject `alg=none`**) + mTLS-to-upstream. **Object-level/ABAC authz** (role+branch+quan hệ điều trị+minimum-necessary) enforce trong Go — Kong KHÔNG bao giờ quyết "bác sĩ này xem bệnh nhân kia". Go **KHÔNG mù tin** header `X-Userinfo` — verify JWT signature/claims độc lập (defense-in-depth chống CVE-2026-29413).
- **Rationale**: Gateway không có context lâm sàng = BOLA. Nếu auth bypass nhưng mTLS vẫn cho Kong tới upstream, app sẽ tin identity giả nếu mù tin header.
- **Alternatives**: IdP SaaS nước ngoài (loại: residency); authz ở Kong (loại: không context lâm sàng, BOLA); app mù tin Kong header (loại: forged identity khi Kong bypass).
- **Consequences**: Role không nhúng cứng token; dùng short-TTL token (5–15m) + cached roles (KHÔNG per-request DB lookup); Kong version-pin + patch là CI/admission gate ([ADR-019](#adr-019--kong-kic-db-less--argo-cd-rolling-không-canarytempo-mvp)).

### ADR-014 — Field-level encryption: app-side envelope + blind-index HMAC
*status: accepted*

- **Context**: Vault 3-job khó vận hành; broken Vault = app không decrypt PHI (availability = safety risk). Encrypted column không searchable → reception không tìm được bệnh nhân hoặc team lưu plaintext tạm (leak).
- **Decision**: **MỘT cơ chế**: app-side **AES-256-GCM envelope encryption** với KMS-wrapped DEK cho CHỈ cột siêu nhạy (CCCD, số thẻ BHYT, HIV/tâm thần/di truyền) — KHÔNG đồng thời pgcrypto DB-side + Vault Transit. Cột cần tra cứu exact-match (CCCD/số thẻ tại tiếp đón) thêm cột **blind-index HMAC deterministic**. Tách bảng `patient_identifiers` giới hạn bề mặt giải mã. Scope cột nhạy cảm phải CHỐT trong [`doc/08-database-schema.md`](08-database-schema.md) TRƯỚC khi có data.
- **Rationale**: Reconcile mâu thuẫn council (pgcrypto VÀ Vault Transit). Over-scope phá RLS-indexed query + search.
- **Alternatives**: Vault Transit + pgcrypto cả hai (loại: hai cơ chế, DB giữ key); mã hóa rộng mọi PHI column (loại: phá query/search); plaintext lookup (loại: leak).
- **Consequences**: Blind-index leak một số thông tin tần suất (chấp nhận cho exact-match); DEK rotation qua KMS; full Vault earn-in khi cần PKI signing/dynamic creds ([ADR-002](#adr-002--mvp-component-budget-cứng--named-operating-model)).

### ADR-015 — Postgres MVP: VN-managed ưu tiên, CNPG 1 primary + 1 async
*status: accepted*

- **Context**: Self-run HA Postgres sync = full-time DBA. 2 sync replica gấp đôi write-latency trên charge/claim path + risk write-stall khi replica lag.
- **Decision**: MVP ưu tiên **VN-managed PG** (VNG/Viettel/FPT, đạt residency NĐ13/53). Nếu không có managed onshore chấp nhận được thì **CNPG 1 primary + 1 ASYNC replica** (KHÔNG 2 sync) + WAL archive + base backup onshore S3-compatible, PITR RPO≤5min platform, RTO≤1h. **Quarterly restore drill**. **Signed-EMR/MAR write path cần synchronous durability riêng** (commit confirmed trước UI báo 'signed'/'administered'), survive PITR.
- **Rationale**: Managed reliable hơn hand-run CNPG. Platform RPO≤5min không đủ cho signed EMR/MAR (mất record pháp lý/an toàn).
- **Alternatives**: CNPG 1 primary + 2 sync replica (loại: latency + stall risk + ops burden); foreign managed RDS (loại: residency).
- **Consequences**: Cần xác nhận VN provider managed PG meets residency; signed-EMR commit path tách khỏi RPO chung ([ADR-004](#adr-004--encounter-là-mỏ-neo-lâm-sàng-emr-ký-số-bất-biến-tt-132025)); restore drill đo RTO thực vào ephemeral namespace.

### ADR-016 — Interop phasing: MVP foundations + hai regulatory externals
*status: accepted*

- **Context**: OIE/Orthanc/FHIR facade đã đúng phase ở council nhưng BC list trình bày như designed-in now → mời build team scaffold sớm. Manual lab-result entry chấp nhận được per domain lead.
- **Decision**: MVP ships **ZERO** external FHIR/HL7/PACS, chỉ bake-in nền móng rẻ: **(code,system,display) triplet** mọi field lâm sàng, **Encounter anchor**, outbox, **terminology catalog** seeded từ danh mục dùng chung BYT. **HAI external REGULATORY trong MVP**: BHYT 4750 ([ADR-006](#adr-006--bhyt-là-phụ-thuộc-live-hai-chạm--degraded-mode)) + e-prescription ([ADR-007](#adr-007--liên-thông-đơn-thuốc-quốc-gia-trong-mvp-tt-262025)). Phase 2: FHIR R4 facade read-only (KHÔNG lock thư viện cụ thể), OIE HL7v2 sidecar (namespace interop, KHÔNG sau Kong), Orthanc DICOM. Phase 3: bidirectional FHIR, SMART apps, SNOMED, CDC/Kafka.
- **Rationale**: Foundations rẻ làm Phase 2 rẻ; manual entry OK cho MVP một khoa.
- **Alternatives**: FHIR/OIE/Orthanc trong MVP (loại: over-engineering); zero foundation (loại: Phase 2 đắt).
- **Consequences**: Coded columns + Encounter anchor là hard requirement MVP; clinical event contract owner là `encounter` BC; OIE feed bus chỉ khi bus tồn tại (Phase 2+).

### ADR-017 — KHÔNG cam kết samply/golang-fhir-models, đánh giá lại Phase 2
*status: accepted*

- **Context**: `samply/golang-fhir-models` last release v0.3.2 Dec 2022 (~3.5 năm chết, R4-only, 4 open issue). Pin interop y tế regulated vào dependency chết = maintainability/security hazard.
- **Decision**: Giữ quyết định **FACADE-over-embedded-server** (reasoning sound) nhưng **GỠ cam kết thư viện** samply/golang-fhir-models + Google FhirProto khỏi accepted plan. Tại Phase 2 đánh giá lại: active forks (`fastenhealth/gofhir-models`) hoặc generate R4 structs in-house.
- **Rationale**: Patient-safety lens verified dead dependency. Không go-live blocker (FHIR Phase 2).
- **Alternatives**: Lock samply ngay (loại: dead dependency); embedded HAPI/full FHIR server (loại: second source of truth, chỉ cần khi persist FHIR người khác).
- **Consequences**: Facade decision ổn định; lựa chọn thư viện hoãn tới Phase 2 research; OLTP là single source of truth.

### ADR-018 — Frontend: Vite SPA + Ant Design v6 + Kong BFF, KHÔNG PWA write-outbox
*status: accepted*

- **Context**: Offline write-queue conflict-replay cho clinical write là highest-bug-density + patient-safety hazard (mis-replay dispense/order), YAGNI cho một khoa OPD trên LAN.
- **Decision**: **Vite 6 + React 19 + TS strict** single SPA static behind Kong (KHÔNG Next.js — không SEO, server/client boundary là cost thuần). **Ant Design v6** (`ConfigProvider vi_VN`) + ProTable/ProForm + thin Tailwind cho UI lâm sàng dày. Kong làm BFF (token trong HttpOnly+Secure+SameSite=Strict cookie, SPA không thấy token). TanStack Router+Query v5+Zustand; react-hook-form+zod gen từ OpenAPI (orval/openapi-typescript). MVP: read-only cached reference data + **hard-online gate** cho dispense/cashier/BHYT — **CẮT PWA write-outbox**.
- **Rationale**: AntD là workhorse cho form lâm sàng dày + vi_VN. SPA không cần SSR.
- **Alternatives**: Next.js (loại: SSR cost không giá trị); shadcn/ui như logmon (loại: AntD tốt hơn cho clinical data-entry); PWA clinical write-outbox (loại: safety hazard).
- **Consequences**: Hai design system across org reference (logmon shadcn vs HMS AntD) — chấp nhận có chủ đích; nếu ward WiFi yếu thì read-only cached + block write khi offline; OpenAPI codegen chống FE↔BE drift; thin house-component (AllergyBanner/VitalsGrid) giữ nhất quán.

### ADR-019 — Kong KIC DB-less + Argo CD rolling, KHÔNG canary/Tempo MVP
*status: accepted*

- **Context**: Canary + SLO-burn-rate-rollback là maturity-L4 cho team chưa ship v1; full LGTM (4 stateful) nặng; Tempo vô nghĩa khi một service.
- **Decision**: **Kong Ingress Controller DB-less/declarative** (Gateway API + KongPlugin CRD, no Kong Postgres, 2+ data-plane + PDB) reconciled bởi **Argo CD app-of-apps ROLLING deploy** + manual promotion. KHÔNG Argo Rollouts canary/SLO-auto-rollback ở MVP. Observability MVP: OTel→Prometheus+Loki+Grafana+Alertmanager multi-window burn-rate; KHÔNG Tempo/distributed-tracing. Audit log ship riêng WORM ([ADR-009](#adr-009--audit-of-reads-commit-with-response-hash-chain--worm)). Security gates merge-blocking rẻ giữ (Gitleaks, govulncheck, golangci-lint, Trivy); SLSA/Cosign/ZAP follow sau.
- **Rationale**: Plain Argo CD rolling là production-real. Security gates rẻ high-value giữ.
- **Alternatives**: Argo Rollouts canary + full LGTM ngày 1 (loại: over-engineering); Kong DB-mode (loại: thêm Postgres vận hành).
- **Consequences**: Canary/Tempo/SLSA/Cosign earn-in khi multi-service; Kong config GitOps-versioned YAML; KGO (Gateway Operator) là Phase 3. Kong version-pin + patch là CI/admission gate ([ADR-013](#adr-013--keycloak-oidc--kong-edge-auth--object-level-authz-ở-go), CVE-2026-29413).

### ADR-020 — DPIA + consent + data-subject-rights là Phase-0 legal deliverable
*status: accepted*

- **Context**: NĐ13: nộp DPIA cho Bộ Công an/A05 trong 60 ngày kể từ BẮT ĐẦU xử lý (= go-live). Council list DPIA như control nhưng treat như docs. Luật Bảo vệ DLCN (hiệu lực 2026) đang superseding NĐ13.
- **Decision**: **DPIA** (hồ sơ đánh giá tác động xử lý DLCN) cho dữ liệu sức khỏe là deliverable **Phase-0 có deadline**. **Consent-records design** + **data-subject-rights** (truy cập/xóa/rút đồng ý) là blocking legal artifact TRƯỚC production data. Xác nhận nghĩa vụ hiện hành theo Luật Bảo vệ DLCN 2026, không chỉ Decree.
- **Rationale**: DPIA là nghĩa vụ pháp lý, không phải documentation; clock bắt đầu go-live.
- **Alternatives**: DPIA phase sau (loại: legal blocker, clock chạy từ go-live); chỉ dựa NĐ13 (loại: Luật 2026 superseding).
- **Consequences**: Cần legal/compliance owner Phase-0; consent enforcement ở Go trước xử lý ngoài mục đích điều trị; data residency onshore bắt buộc. Bảng `data_subject_requests`, `dpia_records`, `consents` (xem `audit-compliance` + `patient` BC).

### ADR-021 — FEFO + lot/expiry + stock_ledger append-only, tránh PCI scope
*status: accepted*

- **Context**: Thuốc cận hạn lãng phí + an toàn; thanh toán viện phí không được kéo HMS vào PCI-DSS scope.
- **Decision**: Cấp phát chọn lot **`ORDER BY expiry_date ASC` (FEFO)** trong tx **`FOR UPDATE SKIP LOCKED`**; `medication_lots(lot_no, expiry, qty)` + **`stock_ledger`** (BIGINT IDENTITY, `movement_type`, `qty_delta`, `balance_after`) **append-only**. Thanh toán qua cổng thứ ba (VNPay/Momo/napas) **redirect/tokenization** — HMS KHÔNG chạm/lưu số thẻ thật (ngoài PCI-DSS scope), chỉ lưu token + transaction id. BHYT outbound qua mTLS + chữ ký số, client cert trong secret store.
- **Rationale**: FEFO chống lãng phí + an toàn; append-only ledger là audit trail kho; tokenization giữ HMS ngoài PCI scope.
- **Alternatives**: FIFO (loại: không tối ưu hạn dùng); HMS lưu số thẻ (loại: PCI scope creep); BHYT không ký số (loại: TT46).
- **Consequences**: `stock_ledger` dùng chung `pharmacy`+`inventory`; payment reconciliation qua transaction id; client cert quản lý trong secret store ([ADR-014](#adr-014--field-level-encryption-app-side-envelope--blind-index-hmac)).

### ADR-022 — Dual-run + print phiếu pháp lý là HIGH priority với KPI-owner
*status: accepted*

- **Context**: Council ban đầu đánh 'med' nhưng change-management là cốt lõi (removing paper là mục tiêu cả dự án). In phiếu không hợp lệ pháp lý hoặc dual-run không có owner KPI → permanent-paper.
- **Decision**: **Dual-run 2–4 tuần/khoa** với super-user tại khoa, checklist đối chiếu giấy↔số, đào tạo theo vai trò — nâng lên **HIGH priority**. **Print templates phải là form pháp lý** (đơn thuốc TT 27/26-2025 có QR/mã đơn + block chữ ký số, phiếu thanh toán theo bảng 4750, giấy ra viện, bệnh án). **Feature-flag-per-department "tắt giấy tại KPI"** có named owner + defined adoption KPI.
- **Rationale**: In phiếu không hợp lệ pháp lý hoặc dual-run không owner → permanent-paper (staff giữ sổ cũ).
- **Alternatives**: Dual-run med-priority không owner (loại: permanent-paper); không in phiếu pháp lý (loại: staff giữ sổ).
- **Consequences**: Cần signing service backend cho PDF ký số ([ADR-004](#adr-004--encounter-là-mỏ-neo-lâm-sàng-emr-ký-số-bất-biến-tt-132025)); KPI owner định nghĩa ngưỡng adoption; print-stylesheet + server-PDF cho legal docs.

### ADR-023 — BHXH sandbox + rejection-code handling là Phase-0 blocker
*status: accepted*

- **Context**: Workflow-load-bearing — không có test endpoint thì build phần số-hóa-nhất (reception check + claim submit) untested, discover format/rejection mismatch chỉ ở production = moment staff abandon system.
- **Decision**: Promote "xác nhận môi trường kiểm thử BHXH (4750 XML + card-check API) + rejection-code handling" từ open-question thành **Phase-0 BLOCKER**. Test card-check + claim-submit + reject-code mapping against real/sandbox gateway TRƯỚC khi build production flow.
- **Rationale**: Build untested = production surprise tại đúng điểm staff abandon system.
- **Alternatives**: Để như open question (loại: build untested, production surprise).
- **Consequences**: Cần liên hệ BHXH lấy sandbox/spec sớm; contract test cho BHYT client ([ADR-025](#adr-025--testing-strategy-testcontainers--e2e-critical-flows)); rejection-code là state machine trong `insurance` BC ([ADR-006](#adr-006--bhyt-là-phụ-thuộc-live-hai-chạm--degraded-mode)).

### ADR-024 — Migrations golang-migrate + Phase-0 migration 000001
*status: accepted*

- **Context**: Thay đổi không backfill rẻ (branch_id, encounter_id, audit, RLS, encryption) phải Phase-0 trước data thật. FORCE RLS keystone phải ở `000001`.
- **Decision**: **golang-migrate** (`NNNNNN_name.up/down.sql` trong `backend/migrations/`, sqlc đọc cùng schema). Zero-downtime: add nullable/DEFAULT, add→backfill→switch→drop (không rename trực tiếp), `CREATE INDEX CONCURRENTLY` tách tx. **Migration `000001` PHẢI** tạo extensions (pgcrypto, pg_trgm, uuid), `branches`, accounts/roles/permissions, `audit_log`, migration-owner-vs-app-role separation, và **BẬT ENABLE+FORCE RLS TRƯỚC bất kỳ bảng PHI nào**. Schema theo phase, chỉ migrate khi BC build.
- **Rationale**: golang-migrate sqlc-friendly org-standard; FORCE RLS keystone phải ở 000001.
- **Alternatives**: Atlas declarative diff / goose (loại: golang-migrate org-standard, sqlc-friendly); RLS sau khi có data (loại: không retrofit được).
- **Consequences**: RLS/audit trigger/partition sống trong migrations; index lớn CONCURRENTLY ngoài tx; org-wide single migration standard. Thực thi keystone [ADR-003](#adr-003--force-row-level-security--migration-owner-tách-app-role-keystone-phi).

### ADR-025 — Testing strategy: testcontainers + E2E critical flows
*status: accepted*

- **Context**: RLS/outbox/FEFO/idempotency là invariant ở DB layer — không thể mock. RLS branch-B-invisible test là merge-blocking ([ADR-003](#adr-003--force-row-level-security--migration-owner-tách-app-role-keystone-phi)). Testing rule: 80% coverage.
- **Decision**: Table-driven unit test domain/app (≥80% per testing rule); **integration test against real Postgres qua testcontainers-go** (RLS branch-isolation, outbox, FEFO, idempotency, audit-fail-closed PHẢI test against real PG); **contract test** cho BHYT + donthuoc client; **E2E critical flow** (check-in với BHYT card-check → OPD order với CDSS hard-stop → dispense FEFO → cashier receipt+print → claim submit + reject handling). TDD red-green-refactor. FE: Vitest+RTL+axe gate mỗi PR, Playwright E2E.
- **Rationale**: Invariant không mock được — phải test real PG. RLS branch-B-invisible test là merge-blocking.
- **Alternatives**: Mock DB cho RLS test (loại: RLS không mock được); chỉ unit test (loại: invariant ở DB layer).
- **Consequences**: CI cần testcontainers (Docker-in-CI); E2E qua Kong BFF auth; coverage gate 80% merge-blocking.

---

## Phụ lục — chỉ số liên kết ADR ↔ tài liệu

| ADR | Doc liên quan chính | BC ảnh hưởng |
|-----|---------------------|--------------|
| 001, 012 | `02-backend-architecture.md` | tất cả |
| 002, 015, 019 | `10-deployment-operations.md` | platform |
| 003, 005, 024 | `08-database-schema.md` | tất cả (PHI) |
| 004, 022 | `03-clinical-encounter-emr.md` | encounter |
| 006, 011, 023 | `05-billing-insurance-bhyt.md` | billing, insurance, scheduling-reception |
| 007, 008, 021 | `04-orders-lab-pharmacy.md` | orders, lab, pharmacy |
| 009, 010, 013, 020 | `06-identity-rbac-audit.md` | identity-access, audit-compliance |
| 014 | `08-database-schema.md` | patient |
| 016, 017 | `01-kien-truc-tong-the.md` | interoperability |
| 018 | `07-frontend-architecture.md` | frontend (tất cả persona) |
| 025 | `09-testing-strategy.md` | tất cả |
