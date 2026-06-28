# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

> **Trạng thái repo:** HIỆN CHƯA CÓ CODE — đây là thiết kế MỤC TIÊU. Mọi code path dưới đây đánh dấu *(planned)* theo layout ở `doc/02-backend-architecture.md`. Nguồn sự thật là `doc/` (18 file thiết kế tiếng Việt) + `doc_tech/` (giáo trình learn-by-doing) — KHÔNG được suy diễn ngoài hai thư mục này.

## Project Overview

Hospital Management System (HMS) là hệ thống **production số hóa bệnh viện Việt Nam** — bỏ quản lý thủ công (giấy/Excel), biến mỗi thao tác giấy có người chịu trách nhiệm thành **một sự kiện số có chủ thể ký số + vết audit bất biến**. Thiết kế quanh **HÀNH TRÌNH NGƯỜI BỆNH** với **Encounter làm mỏ neo lâm sàng** (mọi vitals, chẩn đoán ICD-10, order CLS, kết quả, đơn thuốc, charge đều FK tới `encounter_id`) — KHÔNG phải 10 module CRUD hành chính (ADR-004).

Backend áp dụng **2 mô hình kiến trúc** tùy complexity của domain:

- **Clean Architecture** — cho `identity`, `organization`, `patient`, `scheduling` (CRUD-like + policy, domain đơn giản).
- **Clean + DDD + CQRS + transactional-outbox** — cho `encounter`, `orders`, `lab`, `pharmacy`, `billing`, `insurance` (vòng đời thật: charge-capture, claim↔bill linkage, FEFO — cần ACID nội-process, ADR-001).

MVP là **MỘT phòng khám OPD-BHYT** chạy trọn vòng end-to-end: tiếp đón → khám → CLS → kê đơn → viện phí → XML giám định → ký số EMR.

Tài liệu chi tiết: `doc/` — 18 file thiết kế (00-tong-quan → ...), là source of truth.

## Tech Stack (PINNED — không bàn lại, ADR-001/012/013/015/018/019)

- **Backend**: Go modular monolith (`hms-api`, một deployable) — Gin (HTTP behind Kong) + pgx/v5 + sqlc (no ORM) + golang-migrate; River (Postgres-native) là DUY NHẤT job framework (KHÔNG Watermill song song — ADR-012)
- **Frontend**: Vite 6 + React 19 + TypeScript strict, single SPA static behind Kong (KHÔNG Next.js) — Ant Design v6 (ConfigProvider `vi_VN`) + ProTable/ProForm + thin Tailwind layer; TanStack Router + Query v5 + Zustand (global ephemeral only); react-hook-form + zod gen từ OpenAPI (orval/openapi-typescript); barcode/QR (HID + html5-qrcode); KHÔNG PWA write-outbox (ADR-018)
- **Gateway**: Kong Ingress Controller (KIC) DB-less/declarative (Gateway API + KongPlugin CRD), 2+ data-plane + PDB, version-pinned (CVE-2026-29413 là CI/admission gate). Kong CHỈ coarse-grained edge + BFF cho SPA — KHÔNG bao giờ quyết object-level authz (ADR-013/019)
- **Identity**: Keycloak 26 self-hosted onshore (một realm `hospital`, branch là group/attribute — KHÔNG realm-per-branch) là OIDC issuer; Go không quản password; auth-code+PKCE (SPA qua Kong BFF) + client_credentials (service-to-service); MFA bắt buộc + step-up; PKI signing cho EMR/đơn thuốc (ADR-013)
- **Datastore**: PostgreSQL 16+ OLTP shared-schema single cluster — VN-managed PG ưu tiên (VNG/Viettel/FPT), fallback CNPG 1 primary + 1 ASYNC replica (KHÔNG 2 sync); UUID v7 PK; BIGINT IDENTITY cho `audit_log`/`stock_ledger`/`outbox`; `branch_id NOT NULL` + FORCE RLS mọi bảng PHI; NUMERIC(15,2)+CHAR(3) cho tiền (ADR-015)
- **Messaging**: transactional outbox table viết trong cùng `pgx.Tx`, relay tới in-process pub/sub (`SELECT FOR UPDATE SKIP LOCKED`, subscriber idempotent qua `processed_events`). KHÔNG NATS/Kafka/Debezium ở MVP (ADR-012)
- **Deploy**: Kubernetes onshore VN (namespace-per-env day-1, prod cluster tách tại go-live; ≥3 worker ≥2 AZ; NetworkPolicy default-deny; PHI namespace isolated); Argo CD app-of-apps ROLLING deploy + manual promotion (KHÔNG canary MVP); OpenTofu (cloud/cluster) + Helm (operators) + Kustomize (overlays) (ADR-019)
- **Observability**: OpenTelemetry Collector → Prometheus + Loki + Grafana + Alertmanager multi-window burn-rate; SLO API 99.9%, p95<300ms clinical-read. KHÔNG Tempo/tracing ở MVP. Audit log ship riêng tới WORM sink (ADR-009/019)
- **Secrets**: cloud KMS (VNG/Viettel) HOẶC External Secrets Operator; app-side AES-256-GCM envelope encryption với KMS-wrapped DEK cho CHỈ cột siêu nhạy (CCCD, số thẻ BHYT, HIV/tâm thần/di truyền) + blind-index HMAC cho cột tra cứu — MỘT cơ chế crypto (KHÔNG pgcrypto + Vault Transit song song) (ADR-014)

## Build & Run Commands *(planned — Makefile chưa tồn tại)*

> Vòng lặp local thống nhất qua **root `Makefile`**. Migrations dùng **golang-migrate** (ADR-024); migration `000001` thiết lập nền compliance (RLS keystone) chạy TRƯỚC bất kỳ bảng PHI nào. CI cần Docker-in-CI cho testcontainers (RLS/outbox/FEFO/idempotency — ADR-025).

```bash
# Local dev (khuyến nghị — root Makefile, planned)
make up              # postgres + migrate + hms-api + worker (River)
make dev             # DB + hot-reload; make dev-be / make dev-fe
make migrate         # áp migrations (golang-migrate up); make migrate-down để rollback
make seed            # nạp reference data (terminology ICD-10/LOINC, chargemaster) idempotent
make test            # unit BE (go -race, ≥80%) + integration (testcontainers) + FE (Vitest+RTL+axe)
make e2e             # critical flow Playwright qua Kong BFF (tự dựng + teardown)
make down            # dừng (down-v để xoá volume)

# Trực tiếp (khi cần)
cd backend && go build ./... && go test -race ./... && golangci-lint run
cd frontend && pnpm install && pnpm build && pnpm test
```

**E2E critical flow (ADR-025):** check-in + BHYT card-check → OPD order + CDSS hard-stop → dispense FEFO → cashier receipt+print → claim submit + reject handling.

## Architecture Rules (MUST FOLLOW)

### Layer Direction (strict, one-way — bất khả xâm phạm, ADR-001)

```
adapters → ports ← app → domain
```

- **domain/** chỉ import **Go standard library** — không gì khác
- **app/** chỉ import `domain` + `ports`
- **ports/** chỉ chứa interfaces, không implementation
- **adapters/** implement interfaces từ `ports`
- `cmd/hms-api/main.go` là nơi **DUY NHẤT** wiring (DI / composition root)
- **KHÔNG cross-BC imports** — BC giao tiếp CHỈ qua **domain event + transactional outbox in-process**. `depguard` (golangci-lint) cấm cross-BC import như merge-blocking gate
- Tách BC ra service riêng = swap relay adapter sang Kafka, domain code KHÔNG đổi (chỉ khi có trigger proven scaling/compliance — ADR-001/012)

### Bounded Contexts (14) — dùng ĐÚNG tên này

| BC | Pattern | Bảng sở hữu (trích) |
|----|---------|---------------------|
| `internal/identity/` *(MVP)* | Clean | accounts, staff_profiles, roles, permissions, role_permissions, user_roles, branch_memberships, mfa_factors, break_glass_grants, sessions |
| `internal/organization/` *(MVP)* | Clean | branches, departments, rooms, beds, service_catalog, price_lists, facility_external_codes |
| `internal/patient/` *(MVP, MPI)* | Clean | patients, patient_identifiers (encrypted+blind_index), patient_allergies, consents, patient_merge_links, terminology_concepts |
| `internal/scheduling/` *(MVP)* | Clean | appointments, queue_tickets, check_ins, bhyt_eligibility_checks |
| `internal/encounter/` *(MVP, EMR core)* | Clean+DDD+CQRS | encounters, admissions, bed_assignments, diagnoses, observations, clinical_notes, clinical_notes_history, emr_documents, emr_signatures |
| `internal/orders/` *(MVP, CPOE)* | Clean+DDD+CQRS | service_orders, order_items, order_status_history |
| `internal/lab/` *(MVP, LIS-lite)* | Clean+DDD+CQRS | specimens, lab_results, lab_reference_ranges |
| `internal/pharmacy/` *(MVP)* | Clean+DDD+CQRS | drugs, prescriptions, prescription_items, medication_dispenses, medication_lots, national_rx_links, interaction_overrides |
| `internal/inventory/` *(Phase 2)* | Clean | warehouses, stock_ledger, goods_receipts, suppliers, reorder_rules |
| `internal/billing/` *(MVP)* | Clean+DDD+CQRS | charges, bills, payments, adjustments, advances, idempotency_keys |
| `internal/insurance/` *(MVP, BHYT)* | Clean+DDD+CQRS | insurance_claims, claim_xml_records, claim_responses, bhyt_submission_log |
| `internal/audit/` *(MVP)* | Clean+DDD+CQRS | audit_log (partitioned, FORCE RLS, INSERT-only), data_subject_requests, dpia_records, data_access_log |
| `analytics-reporting` *(Phase 3)* | Clean+DDD+CQRS | none (projections only; scheduled SQL→read-table) |
| `interoperability` *(Phase 2)* | Clean+DDD+CQRS | code_systems, value_sets, concept_maps (MVP: terminology owned by `patient` catalog) |
| `internal/shared/` | Shared Kernel | `{auth,outbox,middleware,metrics,errors,httpx,crypto,rls,config}` — không sở hữu bảng PHI |

### Clean Architecture Layers (identity, organization, patient, scheduling)

```
domain/       — entities, value objects, domain errors (stdlib only)
app/          — use cases (application services)
ports/        — interfaces (repository, cache, external services)
adapters/     — implementations (Gin handlers, pgx/sqlc repos, KMS crypto, Keycloak sync)
```

### DDD + CQRS Layers (encounter, orders, lab, pharmacy, billing, insurance)

```
domain/       — aggregates, value objects, domain events, domain services (CDSS hard-stop, FEFO, state machine)
app/command/  — write side: state-changing use cases (place order, dispense, charge-capture)
app/query/    — read side: read models tối ưu đọc
ports/        — interfaces + EventPublisher (outbox) + ReadModel
adapters/     — pgx/sqlc repos, outbox relay, BHYT client (mTLS), donthuoc client, River jobs
```

## Bất khả xâm phạm — 4 control PHI/an toàn fail-closed (KHÔNG được làm yếu)

Đây là keystone — `code-reviewer`/`security-reviewer` PHẢI block mọi PR làm suy yếu:

1. **RLS keystone (ADR-003/005, Phase-0, không retrofit được)** — Mọi bảng PHI `ENABLE + FORCE ROW LEVEL SECURITY` trong migration `000001`. Tách `role-migration-owner` (sở hữu bảng, DDL) khỏi `app-role` (`NOSUPERUSER`, `NOBYPASSRLS`, KHÔNG sở hữu bảng — vì table OWNER bypass RLS kể cả NOBYPASSRLS). Policy có CẢ `USING` VÀ `WITH CHECK`: `branch_id = current_setting('app.current_branch')::uuid`. Go middleware `SET LOCAL app.current_branch` trong tx (trích từ JWT đã verify — KHÔNG từ client). **Mọi PHI query PHẢI chạy trong tx đã SET LOCAL GUC** (pgx pool reuse connection → query ngoài tx mất filter = leak). Resource khác branch trả **404** (không 403). CI testcontainers test "branch-B invisible dưới `app.current_branch=A`" là merge-blocking.
2. **CDSS hard-stop fail-closed (ADR-008)** — Hard-stop dị ứng/tương tác enforce ở **order/dispense aggregate (server-side)**, KHÔNG ở React modal. CDSS error/timeout → **fail-closed** (KHÔNG confirm "no known interaction"). Bệnh nhân không có allergy data → trạng thái **"allergy status unknown"** tách bạch, KHÔNG BAO GIỜ render là "safe". Override luôn cần `reason + authorizer` ghi audit.
3. **Audit-of-reads fail-closed (ADR-009)** — Audit PHI-read commit cùng/trước response: nếu audit write fail thì **KHÔNG trả PHI**. `audit_log` INSERT-only (policy chặn UPDATE/DELETE `USING false` kể cả app-role), partition theo tháng, FORCE RLS, **hash-chain** (mỗi record chứa hash record trước), stream **WORM sink** ngoài Postgres. State-change audit qua outbox.
4. **Break-the-glass closed loop (ADR-010)** — Time-boxed (auto-expire) + scoped (patient/encounter cụ thể) + named reviewer + hard SLA + consequence path. Áp dụng CẢ access VÀ creation/ordering (ED register-first-identify-later).

## Domain Events (DDD BCs — qua transactional outbox in-process, KHÔNG direct import)

```
EncounterClosed / EMRSigned → ChargeCapture (billing) + ClaimGeneration (insurance) + interop
OrderPlaced                 → Lab/Pharmacy fulfilment routing
ResultReleased / CriticalValue (lab) → Encounter review + CDSS
DrugDispensed (pharmacy)    → ChargeItem (billing) idempotent
ChargeItem (billing)        → InsuranceClaim XML1–XML15 (claim↔bill↔encounter FK)
PatientRegistered / Merged / AllergyRecorded (patient) → CDSS allergy list
```

River jobs *(planned)*: reminders, BHYT claim submit/retry (saga, idempotent), donthuoc submission retry, FEFO sweep (expiry/min-stock), audit retention, outbox cleanup.

## Ràng buộc pháp lý VN (day-1, không deferrable)

- **TT 13/2025/TT-BYT** — bệnh án điện tử ký số PKI (hạn 30/9/2025, đã lapsed → remediation, ADR-004)
- **QĐ 4750** (sửa QĐ 3176/2024, hiệu lực 01/01/2025) — bộ XML giám định BHYT 1–15 (ADR-006/011)
- **TT 26/2025 + QĐ 808** — đơn thuốc điện tử liên thông `donthuocquocgia.vn` (hạn 1/10/2025, MVP-mandatory, ADR-007)
- **NĐ 13/2023 + Luật Bảo vệ DLCN 2026** — DPIA (nộp A05 trong 60 ngày từ go-live) + consent + data-subject-rights là Phase-0 legal blocker (ADR-020). PHI onshore VN (NĐ 53/2022)
- **QĐ 4469** — ICD-10. **BHYT degraded-mode bắt buộc** (admit-and-flag / thu + reconcile-later — không bao giờ chặn người bệnh, ADR-006)

## Security (OWASP + PHI)

- **Input validation**: validator cho mọi request struct; zod schema FE gen từ OpenAPI chống FE↔BE drift
- **Parameterized queries**: LUÔN `$1, $2` (pgx) — KHÔNG string concatenation
- **Auth**: Go verify JWT signature/claims độc lập — KHÔNG mù tin `X-Userinfo` header (defense-in-depth chống CVE-2026-29413, ADR-013). Object-level/ABAC (role+branch+quan hệ điều trị+minimum-necessary) enforce ở Go (chống BOLA/IDOR)
- **Secrets**: KMS/ESO + app-side envelope encryption (ADR-014), KHÔNG hardcode, KHÔNG commit secret. Client cert BHYT mTLS trong secret store
- **Payment**: cổng thứ ba (VNPay/Momo/napas) redirect/tokenization — HMS KHÔNG lưu số thẻ thật (ngoài PCI scope, ADR-021)
- **Migrations**: golang-migrate zero-downtime (add nullable→backfill→switch→drop, CREATE INDEX CONCURRENTLY tách tx, ADR-024)
- **CI security gates merge-blocking**: Gitleaks, govulncheck, golangci-lint, Trivy (SLSA/Cosign/ZAP follow sau, ADR-019)

## Key Documentation (source of truth)

- `doc/` — 18 file thiết kế tiếng Việt: 00-tong-quan · 01-kien-truc-tong-the · 02-backend-architecture · 03-clinical-encounter-emr · 04-orders-lab-pharmacy · 05-billing-insurance-bhyt · 06-identity-rbac-audit · ... (database-schema, api-spec, deployment-operations, roadmap, ADR, frontend, devsecops, IaC). **MỌI file nhất quán tuyệt đối với `DESIGN_CANON.md`** (cùng tên BC, công nghệ pinned, số ADR, tên bảng)
- `doc_tech/` — giáo trình learn-by-doing (mỗi module đủ 10 mục cố định)
- `README.md` — tổng quan + quick start

## Dev Workflow (ECC skills/agents)

- **Backend Go**: `ecc:go-review`, `ecc:go-test` (table-driven, ≥80%, TDD red-green-refactor), `ecc:go-build`
- **Frontend**: `ecc:react-review` + `ecc:react-test` (RTL+axe gate mỗi PR), `ecc:design-system` cho UI lâm sàng dày (AntD)
- **PHI/an toàn**: `ecc:security-review`, `ecc:healthcare-phi-compliance`, `ecc:healthcare-emr-patterns`, `ecc:healthcare-cdss-patterns`, `ecc:hipaa-compliance` — BẮT BUỘC trước commit code chạm auth/PHI/CDSS/audit
- **DB/Infra**: `ecc:postgres-patterns` (RLS/index), `ecc:kubernetes-patterns`, `ecc:database-migrations`
- **Code review**: `code-reviewer` ngay sau khi viết code; block CRITICAL (4 control bất khả xâm phạm) + HIGH trước merge

## Language

Project documentation is in Vietnamese. Respond and interact in Vietnamese; giữ thuật ngữ kỹ thuật bằng tiếng Anh.
