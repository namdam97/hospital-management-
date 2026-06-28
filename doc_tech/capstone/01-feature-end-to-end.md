# [CAP-1] Capstone — Vertical slice OPD-BHYT trọn vòng

> Module CAP-1 · Xây MỘT use-case OPD-BHYT xuyên mọi tầng (domain → API → FE → deploy → test) như bài tốt nghiệp · Độ khó: 🥉→🥇 · Prereqs: ARCH-1/2/3, BE-1..6, DATA-1/2/3, FE-1/2/3, K8S-1/2, DSO-1/2, SEC-1/2, INT-1, DOM-1/2 (toàn bộ module nền tảng)

Đây là module tổng hợp. Mọi kỹ thuật bạn học rời rạc ở các module trước sẽ hội tụ vào MỘT lát cắt dọc (vertical slice) chạy thật: một người bệnh BHYT đi từ tiếp đón → khám → CLS → kê đơn → viện phí → XML giám định → ký số EMR. Repo HIỆN CHƯA CÓ CODE — bạn xây slice này theo layout planned (canon §9), đánh dấu *(planned)*.

---

## 1. Vì sao kỹ năng này quan trọng trong HMS

Một bệnh viện không "đi vào sản xuất" bằng 14 module CRUD rời rạc — nó đi vào sản xuất khi MỘT người bệnh thật chạy trọn HÀNH TRÌNH và mọi thao tác giấy bị thay bằng một sự kiện số có chủ thể ký số + vết audit (canon §0). MVP của HMS, theo canon §6, chính là **một phòng khám OPD-BHYT chạy trọn vòng end-to-end (thin vertical slice)**. Khả năng cắt một lát dọc xuyên mọi tầng — domain Go, outbox, RLS, API behind Kong, FE persona, K8s deploy, test against real Postgres — là kỹ năng phân biệt giữa "đã đọc tài liệu" và "đã ship được phần mềm y tế".

Vertical slice quan trọng vì các invariant nguy hiểm nhất của HMS chỉ lộ ra khi nhiều tầng gặp nhau:
- **Charge-capture idempotent** (ADR-011) chỉ đúng khi FE idempotency-key và backend idempotency-key là MỘT scheme end-to-end (risk [high] canon §8) — bạn không thấy lỗi này nếu chỉ test một tầng.
- **FORCE RLS** (ADR-003) trông "isolated" trên mọi sơ đồ nhưng leak production nếu PHI query chạy ngoài tx đã `SET LOCAL` (risk [critical] canon §8) — chỉ E2E + testcontainers bắt được.
- **Audit-of-reads fail-closed** (ADR-009) phải commit cùng response — một slice end-to-end là nơi duy nhất chứng minh "không trả PHI nếu audit fail".
- **Ký số EMR** (ADR-004, TT 13/2025) cần synchronous durability — UI chỉ báo "đã ký" sau khi commit confirmed.

Tóm lại: capstone này là bằng chứng MVP đứng vững, và là khuôn mẫu (template) để các slice sau (IPD, imaging) sao chép.

---

## 2. Mô hình tư duy (first principles) — từ con số 0

Bắt đầu từ một câu nghiệp vụ trần trụi: *"Bệnh nhân Nguyễn Văn A có thẻ BHYT đến khám viêm họng, được kê kháng sinh, thanh toán phần tự túc, rồi hồ sơ phải gửi giám định và bệnh án phải ký số."*

Bóc câu này thành các nguyên lý không thể bỏ:

1. **Mọi sự kiện lâm sàng treo vào Encounter, KHÔNG vào patient_id** (ADR-004). Vitals, chẩn đoán, order, đơn, charge đều có `encounter_id`. Encounter là mỏ neo — bỏ nó thì claim và FHIR không ráp được.
2. **Một thao tác giấy = một command + một event + một audit**. Tiếp đón viết sổ → `CheckIn` command + `CheckedIn` event + audit. Không có "ghi tạm rồi nhập sau" (phá audit + CDSS realtime).
3. **Cross-BC không gọi thẳng nhau — chỉ qua transactional outbox in-process** (ADR-001/012). `EncounterClosed` không gọi billing trực tiếp; nó ghi outbox trong cùng `pgx.Tx`, relay đẩy tới subscriber idempotent của billing.
4. **Consistency là tài sản, không phải chi phí**. Charge-capture, claim↔bill↔encounter linkage, FEFO đều cần ACID nội-process → một Postgres, một monolith (ADR-001), không distributed transaction.
5. **Degraded-mode là first-class, không phải nhánh lỗi** (ADR-006). Cổng BHYT lỗi → admit-and-flag, không bao giờ chặn người bệnh.

Mô hình tư duy của một slice: **command đi xuống (write), event lan ngang (outbox), query đi lên (read), audit ghi bên cạnh mọi PHI-touch**. Vẽ được sơ đồ này trong đầu cho một use-case = bạn đã "nghĩ như HMS".

---

## 3. Khái niệm cốt lõi (tăng dần độ khó)

**Mức 1 — Use-case là đơn vị slice.** Một use-case (vd "kê đơn ngoại trú") cắt qua: handler HTTP (adapters) → command (app) → aggregate (domain) → repo + outbox INSERT (adapters/postgres). Đây là chiều dọc trong MỘT BC.

**Mức 2 — Slice xuyên BC qua event.** Use-case lớn nối nhiều BC: `encounter` đóng → emit `EncounterClosed` → `billing` charge-capture → emit `ChargeCaptured` → `insurance` build claim. Mỗi mũi tên là một outbox event + subscriber idempotent (`processed_events`), KHÔNG import chéo (depguard chặn — ADR-001).

**Mức 3 — Saga điều phối multi-step.** Quyết toán ra viện (ADR-011) là orchestration saga: giải phóng tạm ứng → chốt invoice → sinh claim → ký số → submit cổng. Mỗi bước là River job at-least-once + idempotent; bù trừ (compensation) khi bước sau lỗi.

**Mức 4 — Invariant xuyên tầng.** Ba invariant phải đúng đồng thời ở nhiều tầng:
- *Idempotency end-to-end*: `Idempotency-Key` từ FE → header API → unique-constraint `idempotency_keys` (ADR-011).
- *RLS session-var*: middleware `SET LOCAL app.current_branch` trong tx bao trọn mọi PHI query (ADR-003/005).
- *Fail-closed*: audit-write fail → rollback → không trả PHI; CDSS timeout → reject command (ADR-008/009).

**Mức 5 — Slice là contract test surface.** Một slice hoàn chỉnh cho phép viết E2E critical flow (ADR-025): check-in + BHYT card-check → OPD order + CDSS hard-stop → dispense FEFO → cashier receipt → claim submit + reject handling. Đây là merge-blocking proof MVP còn sống.

---

## 4. HMS dùng nó thế nào (bám code path — *(planned)*)

Repo chưa có code; dưới đây là layout MỤC TIÊU (canon §9) cho slice OPD-BHYT.

Luồng slice end-to-end:

```mermaid
sequenceDiagram
  participant FE as SPA persona (FE)
  participant Kong as Kong BFF (edge auth)
  participant API as hms-api (Gin)
  participant DOM as aggregate (domain)
  participant PG as Postgres (RLS + outbox)
  participant RLY as outbox relay (in-process)
  participant RIV as River worker
  FE->>Kong: POST /encounters/:id/close (cookie, Idem-Key)
  Kong->>API: inject JWT identity (X-Userinfo)
  API->>API: verify JWT độc lập (CVE-2026-29413)
  API->>PG: BEGIN; SET LOCAL app.current_branch
  API->>DOM: Encounter.Close() → EMRDocument signed
  API->>PG: persist + outbox INSERT (cùng tx) + audit
  API->>PG: COMMIT (sync durability — UI mới báo 'signed')
  RLY-->>PG: SELECT FOR UPDATE SKIP LOCKED
  RLY->>API: deliver EncounterClosed → billing charge-capture
  API->>PG: ChargeItem + ChargeCaptured (idempotent)
  RLY->>API: ChargeCaptured → insurance build XML4750
  RIV->>API: claim submit job (saga, retry, idempotent)
```

Neo code path *(planned)*:

| Tầng | Path *(planned)* | Trách nhiệm slice |
|------|------------------|-------------------|
| FE | `frontend/src/features/bac-si/CloseEncounter.tsx` | Form ký số, gọi API với `Idempotency-Key` |
| Edge | `deploy/kong/` (Gateway API + KongPlugin) | Verify JWT/JWKS, BFF cookie→header |
| HTTP | `internal/encounter/adapters/http/close_handler.go` | Verify JWT độc lập, decode, map command |
| App | `internal/encounter/app/command/close_encounter.go` | Orchestrate tx, gọi aggregate, outbox |
| Domain | `internal/encounter/domain/encounter.go` | State machine, `EMRDocument` immutable |
| Postgres | `internal/encounter/adapters/postgres/repo.go` | Persist + outbox INSERT cùng `pgx.Tx` |
| Shared | `internal/shared/{rls,outbox,auth,audit}` | `SET LOCAL`, relay, JWT verify, audit |
| Billing | `internal/billing/app/command/capture_charge.go` | Subscriber idempotent của `EncounterClosed` |
| Insurance | `internal/insurance/app/command/build_claim.go` | Sinh XML1–XML15 QĐ 4750 từ ChargeItem |
| Job | `cmd/worker/` + `internal/shared/jobs` (River) | Claim submit/retry, saga quyết toán |
| Migration | `backend/migrations/000001_phase0_compliance.up.sql` | FORCE RLS + audit_log + role separation |

Minh họa command app-layer *(planned)* — orchestrate tx + outbox + audit fail-closed:

```go
// internal/encounter/app/command/close_encounter.go  (planned)
func (h *CloseEncounterHandler) Handle(ctx context.Context, c CloseEncounterCmd) error {
    return h.tx.WithinTx(ctx, c.BranchID, func(tx pgx.Tx) error { // SET LOCAL app.current_branch
        enc, err := h.repo.Load(ctx, tx, c.EncounterID)
        if err != nil { return err }
        doc, err := enc.Close(c.Actor, h.clock.Now()) // domain: → EMRDocument signed, immutable
        if err != nil { return err }                  // state-machine guard
        if err := h.repo.SaveSigned(ctx, tx, enc, doc); err != nil { return err }
        if err := h.outbox.Enqueue(ctx, tx, event.EncounterClosed{...}); err != nil { return err }
        return h.audit.WriteWithResponse(ctx, tx, audit.Event{Action: "update", ...}) // fail-closed
    }) // COMMIT confirmed TRƯỚC khi handler trả 'signed' (ADR-004 sync durability)
}
```

Slice này hiện thực hóa: ADR-001 (monolith), ADR-003/005 (RLS), ADR-004 (EMR ký số), ADR-008/009 (fail-closed), ADR-011 (charge/claim idempotent + saga), ADR-012 (outbox + River).

---

## 5. Best practices (mỗi mục kèm 1 nguồn đã research)

- **Vertical slice trước, hoàn thiện chiều rộng sau.** Ship một use-case xuyên mọi tầng để lộ rủi ro tích hợp sớm, thay vì xây từng layer "đầy đủ" rồi mới ráp. Nguồn: Jimmy Bogard, "Vertical Slice Architecture" — https://www.jimmybogard.com/vertical-slice-architecture/
- **Transactional outbox để event nhất quán với state.** Ghi event trong cùng transaction với business state, relay đẩy sau — chống mất event / dual-write. Nguồn: microservices.io, "Pattern: Transactional outbox" — https://microservices.io/patterns/data/transactional-outbox.html
- **Saga cho luồng multi-step không thể 2PC.** Quyết toán ra viện dùng orchestration saga + compensation. Nguồn: microservices.io, "Pattern: Saga" — https://microservices.io/patterns/data/saga.html
- **Idempotency-Key chuẩn hóa cho POST không an toàn lặp.** Một key end-to-end, unique-constraint phía server. Nguồn: IETF draft "The Idempotency-Key HTTP Header Field" — https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/
- **RLS phải test bằng integration test against real Postgres.** RLS không mock được; chứng minh branch-B invisible. Nguồn: PostgreSQL Docs, "Row Security Policies" — https://www.postgresql.org/docs/16/ddl-rowsecurity.html
- **E2E critical flow chạy như user thật qua trình duyệt.** Test slice qua Kong BFF auth bằng Playwright. Nguồn: Playwright Docs, "Best Practices" — https://playwright.dev/docs/best-practices
- **Test bằng container ephemeral, không DB chia sẻ.** Mỗi run dựng Postgres riêng để RLS/FEFO/outbox test cô lập. Nguồn: Testcontainers for Go — https://golang.testcontainers.org/

---

## 6. Lỗi thường gặp & anti-patterns

- **Hai idempotency scheme rời (FE vs BE).** Queued-then-replayed dispense/charge double-post khi mạng về (risk [high] canon §8). Đúng: MỘT scheme; MVP cắt PWA write-outbox (ADR-018), hard-online gate cho dispense/cashier.
- **PHI query chạy ngoài tx đã `SET LOCAL`.** Trên pgx pool reuse connection, query revert về no-branch-filter → leak (risk [critical] canon §8). Đúng: mọi PHI query trong `WithinTx`; invariant CÓ TEST, không chỉ prose.
- **Gọi thẳng BC khác trong handler.** Phá ADR-001, depguard chặn import chéo. Đúng: emit outbox event, subscriber idempotent xử lý.
- **UI báo "đã ký/đã thu" trước khi COMMIT confirmed.** Crash mất EMR ký số / charge (ADR-004/015). Đúng: synchronous durability, chỉ báo sau commit.
- **Audit-write best-effort async cho READ.** Window mất audit khi crash → vi phạm HIPAA §164.312(b)/NĐ13 (ADR-009). Đúng: commit-with-response, fail-closed.
- **CDSS chỉ ở FE modal.** Bypassable qua devtools/API (ADR-008). Đúng: hard-stop ở aggregate; modal chỉ UX; `allergy-unknown` ≠ `safe`.
- **Mù tin `X-Userinfo` header từ Kong.** CVE-2026-29413 auth-bypass → forged identity (ADR-013). Đúng: Go verify JWT signature/claims độc lập.
- **Claim sinh tách rời ChargeItem.** Claim↔bill lệch nhau (gap ADR-011 sửa). Đúng: claim sinh TỪ ChargeItem, FK cứng claim↔bill↔encounter.
- **Chặn người bệnh khi cổng BHYT lỗi.** Trigger #1 staff quay về giấy (risk [high]). Đúng: admit-and-flag, queue-and-retry, reconcile-later (ADR-006).

---

## 7. Lộ trình luyện tập NGAY trong repo (🥉 → 🥇)

> Repo chưa có code — luyện bằng cách scaffold theo layout planned (§9), viết test trước (TDD red-green-refactor, ADR-025).

**🥉 Cơ bản — một use-case trong MỘT BC.**
Scaffold `internal/encounter/` (domain + app + ports + adapters). Hiện thực `Encounter.Close()` state-machine guard (`in-progress → finished`), viết unit test table-driven cho mọi chuyển trạng thái hợp lệ/không hợp lệ. Mục tiêu: command → aggregate → repo (chưa cross-BC).

**🥈 Trung cấp — nối hai BC qua outbox.**
Thêm outbox INSERT trong cùng tx của `CloseEncounter`; viết subscriber `billing/capture_charge.go` idempotent (`processed_events`). Integration test với testcontainers: đóng encounter → assert đúng 1 `ChargeItem` sinh ra dù relay deliver event 2 lần (idempotency). Thêm test RLS: branch-B đóng encounter của branch-A phải trả 404.

**🥇 Nâng cao — slice trọn vòng + deploy + E2E.**
Hoàn tất chuỗi: charge → `insurance/build_claim.go` sinh XML1–XML15 (QĐ 4750) → River job submit cổng (sandbox/contract test) → ký số EMR (sync durability). Viết Playwright E2E qua Kong BFF: persona bác-sĩ đóng encounter → persona thu-ngân thu phí (degraded-mode khi cổng lỗi) → persona giám-định thấy claim. Deploy slice lên namespace `dev` qua Argo CD rolling (Kustomize overlay), verify `/readyz` + RLS branch-isolation test merge-blocking xanh.

---

## 8. Skill/agent ECC nên dùng khi luyện

- **`ecc:feature-dev`** — guided feature development, bám codebase understanding & architecture cho cả slice dọc.
- **`ecc:plan` / `ecc:plan-orchestrate`** — chia slice thành bước có dependency trước khi code (DAG use-case → cross-BC → saga).
- **`ecc:go-test`** — TDD Go table-driven, verify coverage 80%+ (ADR-025) cho domain/app.
- **`ecc:go-review`** + **`ecc:security-review`** — review idiomatic Go, fail-closed, RLS, CVE-2026-29413 trust-but-verify.
- **`ecc:postgres-patterns`** + **`ecc:database-migrations`** — RLS policy USING&WITH-CHECK, migration 000001, idempotency unique-constraint.
- **`ecc:healthcare-emr-patterns`** + **`ecc:healthcare-phi-compliance`** — encounter workflow, audit-of-reads, PHI leak vectors.
- **`ecc:react-test`** + **`ecc:react-review`** — Playwright/RTL E2E persona, AntD clinical form.
- **`ecc:e2e-testing`** + **`ecc:browser-qa`** — critical flow qua Kong BFF auth.
- **`ecc:kubernetes-patterns`** + **`ecc:deployment-patterns`** — deploy slice lên namespace dev, probes/PDB/HPA.
- **`ecc:code-review`** (built-in `/code-review`) — quét diff toàn slice trước commit.

---

## 9. Tài nguyên học thêm (2024–2026)

- microservices.io — Transactional Outbox, Saga, Idempotent Consumer patterns (Chris Richardson, cập nhật liên tục): https://microservices.io/patterns/data/transactional-outbox.html
- Jimmy Bogard — Vertical Slice Architecture: https://www.jimmybogard.com/vertical-slice-architecture/
- PostgreSQL 16 — Row Security Policies: https://www.postgresql.org/docs/16/ddl-rowsecurity.html
- Testcontainers for Go (real-PG integration test): https://golang.testcontainers.org/
- Playwright — Best Practices & Auth (E2E qua BFF): https://playwright.dev/docs/best-practices
- River — Postgres-native background jobs (Go): https://riverqueue.com/docs
- IETF — Idempotency-Key HTTP Header (draft): https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/
- Cổng giám định BHXH (4750 XML + card-check, sandbox Phase-0): tham chiếu QĐ 4750/QĐ 3176 + ADR-023 trong `doc/13-adr.md`.
- TT 13/2025/TT-BYT (EMR ký số), TT 26/2025 + QĐ 808 (đơn thuốc liên thông): neo trong `doc/05-billing-insurance-bhyt.md` và `doc/03-clinical-encounter-emr.md`.

---

## 10. Checklist "đã hiểu"

- [ ] Tôi vẽ được sequence của slice OPD-BHYT: command xuống → event lan ngang (outbox) → query lên → audit bên cạnh.
- [ ] Tôi biết mọi sự kiện lâm sàng FK tới `encounter_id`, KHÔNG `patient_id` (ADR-004).
- [ ] Tôi giải thích được vì sao cross-BC chỉ qua outbox in-process và depguard chặn import chéo (ADR-001/012).
- [ ] Tôi chứng minh được idempotency end-to-end là MỘT scheme (FE key = API header = unique-constraint) (ADR-011).
- [ ] Tôi viết được integration test (testcontainers) chứng minh branch-B invisible dưới `app.current_branch=A` (ADR-003).
- [ ] Tôi biết PHI query phải nằm trong tx đã `SET LOCAL`, và đây là invariant CÓ TEST.
- [ ] Tôi hiểu UI chỉ báo "đã ký/đã thu" sau khi COMMIT confirmed (synchronous durability — ADR-004/015).
- [ ] Tôi hiểu audit-of-reads commit-with-response fail-closed và CDSS hard-stop fail-closed (ADR-008/009).
- [ ] Tôi biết Go verify JWT độc lập, không mù tin `X-Userinfo` (CVE-2026-29413, ADR-013).
- [ ] Tôi triển khai được slice lên namespace dev qua Argo CD rolling và chạy E2E critical flow qua Kong BFF (ADR-019/025).
- [ ] Tôi biết degraded-mode (admit-and-flag / reconcile-later) là first-class, không bao giờ chặn người bệnh (ADR-006).
