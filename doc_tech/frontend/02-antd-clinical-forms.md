# [FE-2] Ant Design v6 + RHF/zod clinical forms (vi_VN)

> Module FE-2 · AntD v6 ProForm/ProTable + react-hook-form/zod cho form lâm sàng dày tiếng Việt, một schema = form + API type · Độ khó: 🥉→🥇 · Prereqs: FE-1 (Vite + React 19 + TS SPA, TanStack Router/Query, per-persona surfaces)

Liên quan: `doc/14-frontend-architecture.md`, `doc_tech/frontend/01-vite-react-spa.md` (FE-1), `doc_tech/frontend/03-clinical-safety-ux.md` (FE-3), `doc_tech/backend-go/06-api-design-openapi.md` (BE-6 — OpenAPI là nguồn codegen). Neo: **ADR-018** (Vite SPA + AntD v6 + Kong BFF), **ADR-022** (print phiếu pháp lý), **ADR-008** (CDSS allergy-unknown ≠ safe), **ADR-011** (idempotency-key end-to-end), **ADR-025** (Vitest+RTL+axe gate).

---

## 1. Vì sao kỹ năng này quan trọng trong HMS

Bệnh viện rời giấy nghĩa là **mọi tờ phiếu viết tay biến thành một form số có validation, có chủ thể ký, có vết audit**. Form không phải UI trang trí — nó là cổng nhập liệu cho dữ liệu pháp lý và an toàn người bệnh: vitals (LOINC), chẩn đoán ICD-10 (QĐ 4469), đơn thuốc liên thông (TT 26/2025), phiếu thanh toán bảng 4750. Một form lỏng lẻo cho phép nhập sai liều, sai mã thẻ BHYT, hay bỏ trống dị ứng → hậu quả là charge sai, claim bị từ chối, hoặc bệnh nhân nhận thuốc gây dị ứng đã ghi.

**ADR-018** chốt Ant Design v6 (ConfigProvider `vi_VN`) + ProForm/ProTable + thin Tailwind chính vì AntD là workhorse cho **UI lâm sàng dày**: bảng nhập nhiều cột, form nhiều field điều kiện, date/time tiếng Việt, currency `NUMERIC(15,2)`. AntD thắng shadcn/ui (logmon dùng) cho data-entry mật độ cao — ta chấp nhận có chủ đích hai design system across org (ADR-018 hệ quả, risk [low] line 336).

Điểm cốt lõi của FE-2: **một zod schema = vừa validate form vừa là kiểu TypeScript của API request**, gen từ OpenAPI (BE-6) qua `orval`/`openapi-typescript`. Đây là cách chống FE↔BE drift (ADR-018 hệ quả) — backend đổi field, codegen fail build ngay, không để lỗi rơi vào production lâm sàng.

---

## 2. Mô hình tư duy (first principles) — từ con số 0

Một form, bóc đến tận gốc, là một **hàm thuần**: `(input người dùng) → (giá trị hợp lệ | tập lỗi)`. Ba câu hỏi gốc:

1. **Nguồn sự thật của "shape" là gì?** — Không phải JSX. Là **zod schema**. Schema định nghĩa field, kiểu, ràng buộc. Từ schema suy ra: (a) `type` TS cho request, (b) resolver cho react-hook-form, (c) thông báo lỗi tiếng Việt.
2. **Ai sở hữu state form?** — `react-hook-form` (RHF) giữ state *uncontrolled* (ref-based) để re-render tối thiểu — quan trọng với form 40+ field như bệnh án. AntD chỉ là lớp *trình bày* (Input, Select, DatePicker) được `Controller` của RHF nối vào.
3. **Validation chạy mấy lần?** — Hai lần, hai nơi. FE validate để UX tốt (báo lỗi tức thì), backend validate lại vì **client không bao giờ đáng tin** (coding-style: validate at boundaries). FE zod ≠ control an toàn — đặc biệt CDSS hard-stop là *backend* (ADR-008), modal FE chỉ là UX.

Tư duy đúng: **đừng viết hai sự thật** (một zod cho form, một type tay cho API). Sinh type API từ OpenAPI, rồi hoặc dùng zod gen sẵn (`orval` với zod client), hoặc viết zod tham chiếu type đó. Mọi field lâm sàng mang triplet `(code, system, display)` (ADR-016) — form chọn `display`, gửi `code`+`system`.

---

## 3. Khái niệm cốt lõi (tăng dần độ khó)

| Mức | Khái niệm | Vai trò trong HMS |
|-----|-----------|-------------------|
| 🥉 | `ConfigProvider locale={viVN}` + theme token | Toàn bộ AntD nói tiếng Việt; theme token = thin layer, KHÔNG ghi đè bằng inline style |
| 🥉 | RHF `useForm` + `Controller` | Nối AntD component (uncontrolled) vào RHF; `zodResolver` chạy validation |
| 🥈 | `z.infer<typeof schema>` | Schema → type; type này == API request type (gen OpenAPI) |
| 🥈 | ProForm / ProTable | Form/table cấp cao: layout, search, pagination, column render cho danh sách bệnh nhân/order |
| 🥈 | House components (thin layer) | `AllergyBanner`, `VitalsGrid`, `PatientHeader` — giữ nhất quán nội bộ HMS (ADR-018 hệ quả) |
| 🥇 | `discriminatedUnion` cho form điều kiện | Order lab vs medication vs imaging khác field; allergy-unknown ≠ allergy-empty |
| 🥇 | Print phiếu pháp lý (print-stylesheet) | Đơn thuốc QR/mã đơn + block chữ ký số, phiếu 4750 (ADR-022) |
| 🥇 | Idempotency-Key trên submit | FE sinh key, gửi header — cùng scheme end-to-end với backend (ADR-011) |

Mối quan hệ: `OpenAPI → codegen (orval) → {apiTypes.ts, zodSchemas.ts, hooks.ts}` → form import schema → RHF resolver → submit qua TanStack Query mutation (FE-1) → Kong BFF inject token (FE-3).

---

## 4. HMS dùng nó thế nào (bám code path — *(planned)*, code chưa viết)

Repo HIỆN CHƯA CÓ CODE; layout mục tiêu theo canon §9. Frontend nằm dưới `frontend/src/{app,features/<persona>,shared,api(orval-gen)}` *(planned)*.

```
frontend/                                         # ADR-018 — Vite 6 + React 19 + TS strict
├── src/
│   ├── api/                 (orval-gen)           # gen từ OpenAPI (BE-6): apiTypes, zod, hooks — KHÔNG sửa tay
│   ├── shared/
│   │   ├── ui/AllergyBanner.tsx                   # house component — allergy-unknown ≠ safe (ADR-008)
│   │   ├── ui/VitalsGrid.tsx                      # nhập vitals LOINC
│   │   ├── ui/PatientHeader.tsx                   # MPI header dùng chung mọi persona
│   │   ├── form/zodResolver.ts                    # cầu RHF ↔ zod
│   │   └── print/legal/                           # print-stylesheet phiếu pháp lý (ADR-022)
│   └── features/
│       ├── bac_si/EncounterForm.tsx               # bệnh sử/khám/ICD-10 → Encounter
│       ├── duoc_si/PrescriptionForm.tsx           # kê đơn + CDSS modal (control ở BE, ADR-008)
│       ├── le_tan/ReceptionForm.tsx               # tra cứu MPI + verdict BHYT
│       └── thu_ngan/InvoiceTable.tsx              # ProTable charge + receipt print
└── vite.config.ts  package.json
```

Quy ước nối AntD vào RHF với zod sinh từ OpenAPI *(planned)*:

```tsx
// frontend/src/features/bac_si/EncounterForm.tsx  (planned)
import { useForm, Controller } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { ConfigProvider, Select, DatePicker, Button } from 'antd'
import viVN from 'antd/locale/vi_VN'
import { diagnosisSchema, type DiagnosisRequest } from '@/api/zod' // ADR-018: gen, không viết tay

export function DiagnosisField() {
  const { control, handleSubmit } = useForm<DiagnosisRequest>({
    resolver: zodResolver(diagnosisSchema),       // 1 schema = form + API type
    defaultValues: { icd10Code: '', system: 'ICD-10' }, // (code,system,display) triplet — ADR-016
  })
  const onSubmit = handleSubmit((data) => mutate(data, {
    headers: { 'Idempotency-Key': crypto.randomUUID() }, // ADR-011 — cùng scheme với BE
  }))
  return (
    <ConfigProvider locale={viVN}>
      <form onSubmit={onSubmit}>
        <Controller name="icd10Code" control={control}
          render={({ field, fieldState }) => (
            <Select {...field} showSearch placeholder="Chẩn đoán ICD-10 (QĐ 4469)"
              status={fieldState.error ? 'error' : undefined} />
          )} />
        <Button htmlType="submit" type="primary">Lưu chẩn đoán</Button>
      </form>
    </ConfigProvider>
  )
}
```

`AllergyBanner` phải render ba trạng thái tách bạch (ADR-008): có dị ứng → đỏ; không có dị ứng đã xác nhận → xanh; **chưa biết** → vàng "Chưa rõ tình trạng dị ứng" — KHÔNG BAO GIỜ hiển thị "an toàn" khi unknown.

---

## 5. Best practices (mỗi mục kèm 1 nguồn đã research)

1. **Schema-first: zod là nguồn sự thật, infer ra type** — không khai báo type tay song song với schema. Nguồn: react-hook-form Advanced + zod resolver — https://react-hook-form.com/get-started#SchemaValidation
2. **Controller cho mọi component không phải input native** — AntD `Select`/`DatePicker` là controlled, phải bọc `Controller`, không dùng `register` trực tiếp. Nguồn: RHF `Controller` API — https://react-hook-form.com/docs/usecontroller/controller
3. **Đặt `ConfigProvider locale={vi_VN}` ở gốc app một lần** — không lặp; phủ DatePicker/Pagination/Modal. Nguồn: Ant Design Internationalization — https://ant.design/docs/react/i18n
4. **Sinh client từ OpenAPI bằng orval (mode zod)** — một lệnh gen ra types + zod + TanStack Query hooks, chống FE↔BE drift. Nguồn: orval docs — https://orval.dev/guides/zod
5. **`discriminatedUnion` cho form đa biến thể** — order lab vs medication; cho lỗi chính xác hơn `union`. Nguồn: Zod discriminated unions — https://zod.dev/?id=discriminated-unions
6. **Test form theo hành vi + a11y (RTL + jest-axe)** — query theo role/label tiếng Việt, không theo class; axe gate mỗi PR (ADR-025). Nguồn: Testing Library guiding principles — https://testing-library.com/docs/guiding-principles
7. **Tiền tệ dùng AntD `InputNumber` formatter + `NUMERIC(15,2)`** — không float JS cho tiền. Nguồn: AntD InputNumber formatter — https://ant.design/components/input-number#components-input-number-demo-formatter
8. **Print phiếu pháp lý bằng print-stylesheet `@media print`** — bố cục cố định cho đơn thuốc QR + block ký số (ADR-022). Nguồn: MDN print stylesheets — https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_media_queries/Printing

---

## 6. Lỗi thường gặp & anti-patterns

- **Coi modal CDSS là control an toàn.** Modal bypass được qua devtools/API (ADR-008). Hard-stop PHẢI ở backend aggregate; FE chỉ hiển thị. Render allergy-unknown thành "safe" là lỗi chết người.
- **Hai nguồn sự thật type.** Viết zod tay + type tay → drift âm thầm. Luôn gen từ OpenAPI (ADR-018 hệ quả).
- **Dùng `register` cho AntD component.** AntD là controlled → mất giá trị; phải `Controller`.
- **Quên Idempotency-Key trên submit charge/dispense.** Double-click → double-post charge (risk [high] line 328). FE + BE phải MỘT scheme (ADR-011).
- **Hardcode chuỗi tiếng Anh / format ngày en-US.** Bỏ `ConfigProvider vi_VN` → DatePicker, Pagination ra tiếng Anh; phiếu in sai chuẩn pháp lý.
- **Inline style đè AntD token.** Thin Tailwind/theme token only (ADR-018); inline style phá nhất quán + a11y.
- **Form lớn controlled toàn bộ (mỗi field re-render cả form).** Bệnh án 40+ field → lag; RHF uncontrolled mới đúng.
- **Validate float cho tiền** thay vì `NUMERIC(15,2)` semantics → lệch xu khi đối soát BHYT.
- **Test theo CSS class/testid** thay vì role/label → test giòn, bỏ sót a11y (ADR-025 axe gate).

---

## 7. Lộ trình luyện tập NGAY trong repo (🥉 → 🥈 → 🥇)

> Repo chưa có code — đây là bài tập scaffold trong `frontend/` *(planned)* theo layout §9.

- 🥉 **Cơ bản — VitalsGrid + vi_VN.** Tạo `frontend/src/shared/ui/VitalsGrid.tsx`: bọc app trong `ConfigProvider locale={viVN}`, render form nhập 3 vitals (mạch/nhiệt độ/huyết áp) bằng AntD `InputNumber`, nối RHF `Controller`. Viết zod schema thủ công, `z.infer` ra type. Kiểm DatePicker hiển thị tiếng Việt.

- 🥈 **Trung cấp — PrescriptionForm với codegen.** Cấu hình `orval` đọc OpenAPI (BE-6, planned) sinh `src/api/zod` + hooks. Dựng `features/duoc_si/PrescriptionForm.tsx` dùng schema gen sẵn, `discriminatedUnion` cho loại thuốc, gắn `AllergyBanner` ba trạng thái (ADR-008). Thêm `Idempotency-Key` header vào mutation (ADR-011). Viết RTL test: submit hợp lệ gọi mutation đúng payload; submit thiếu liều hiện lỗi tiếng Việt.

- 🥇 **Nâng cao — In đơn thuốc pháp lý.** Tạo `shared/print/legal/PrescriptionPrint.tsx` + print-stylesheet `@media print`: block chữ ký số + ô QR/mã đơn quốc gia (ADR-022, TT 26/2025). Render từ Encounter đã ký. Thêm jest-axe gate (ADR-025) cho cả form lẫn layout in. Bonus: ProTable `thu_ngan/InvoiceTable.tsx` tách BHYT vs tự túc, `InputNumber` formatter currency, nút in biên lai bảng 4750.

---

## 8. Skill/agent ECC nên dùng khi luyện

- **`ecc:react-review`** — review hook correctness, render performance, server/client boundary; chạy sau khi viết EncounterForm/PrescriptionForm.
- **`ecc:react-test`** — enforce TDD React: viết RTL test (behavior + a11y first) TRƯỚC component, dò Vitest/Jest, verify coverage (ADR-025, testing rule 80%).
- **`ecc:react-build`** — fix lỗi build Vite/TSX, type mismatch sau codegen orval.
- **`ecc:frontend-a11y`** / **`ecc:accessibility`** — audit WCAG 2.2 AA, ARIA cho form lâm sàng (label tiếng Việt, focus order, error announce).
- **`ecc:react-patterns`** + **`ecc:frontend-patterns`** — pattern component/form, state management.
- **`ecc:healthcare-emr-patterns`** — accessibility-first UI cho medical data entry, prescription generation, allergy-unknown state.
- **`ecc:update-docs`** — sync tài liệu này khi OpenAPI/schema đổi.

---

## 9. Tài nguyên học thêm (2024–2026)

- Ant Design v6 — Components & ConfigProvider: https://ant.design/components/overview
- Ant Design ProComponents (ProForm/ProTable): https://procomponents.ant.design/en-US/components
- react-hook-form v7 docs: https://react-hook-form.com/docs
- Zod (schema validation, discriminatedUnion): https://zod.dev
- orval — OpenAPI → TanStack Query + zod codegen: https://orval.dev
- TanStack Query v5 (mutation, optimistic): https://tanstack.com/query/latest
- Testing Library + jest-axe: https://testing-library.com/docs · https://github.com/nickcolley/jest-axe
- WCAG 2.2 Quick Ref (forms, error identification): https://www.w3.org/WAI/WCAG22/quickref/
- TT 26/2025/TT-BYT (đơn thuốc điện tử liên thông) + QĐ 808 — nội dung in pháp lý đơn thuốc.
- QĐ 4750 (sửa QĐ 3176) — bố cục phiếu thanh toán/giám định BHYT cho print template.

---

## 10. Checklist "đã hiểu"

- [ ] Giải thích được vì sao zod schema là nguồn sự thật và `z.infer` cho ra type API (một sự thật, không viết tay đôi).
- [ ] Nối được AntD `Select`/`DatePicker` vào RHF qua `Controller` (không dùng `register`).
- [ ] Đặt `ConfigProvider locale={vi_VN}` đúng một lần ở gốc và biết nó phủ component nào.
- [ ] Cấu hình orval gen `src/api/{types,zod,hooks}` từ OpenAPI và hiểu nó chống FE↔BE drift (ADR-018).
- [ ] Trình bày được vì sao modal CDSS KHÔNG phải control, và allergy-unknown ≠ safe (ADR-008).
- [ ] Gắn `Idempotency-Key` lên submit charge/dispense, cùng scheme với backend (ADR-011).
- [ ] Dựng print-stylesheet phiếu pháp lý (đơn thuốc QR + block ký số) theo ADR-022.
- [ ] Viết RTL test theo role/label tiếng Việt + jest-axe gate, hiểu coverage 80% merge-blocking (ADR-025).
- [ ] Phân biệt thin theme/Tailwind layer vs inline style đè AntD token, và biết house components AllergyBanner/VitalsGrid/PatientHeader.
- [ ] Dùng `discriminatedUnion` cho form đa biến thể (lab/medication/imaging) và `InputNumber` formatter cho tiền `NUMERIC(15,2)`.
