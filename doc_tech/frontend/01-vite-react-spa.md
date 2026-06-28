# [FE-1] Vite 6 + React 19 + TypeScript SPA (per-persona)

> Module **FE-1** · Dựng nền SPA tĩnh sau Kong BFF, typed routing & per-persona surface cho HMS · Độ khó: 🥉→🥇 · Prereqs: — (khuyến nghị đọc trước `doc/14-frontend-architecture.md`, ADR-018, ADR-013)

Module này dạy bạn dựng **bộ khung frontend** của Hospital Management System: một single SPA tĩnh (Vite 6 + React 19 + TypeScript strict) đứng sau Kong BFF, chia bề mặt theo persona (tiếp đón / bác sĩ / dược sĩ / thu ngân / giám định), routing typed bằng TanStack Router, state ephemeral bằng Zustand. Mọi quyết định neo vào **ADR-018** (Vite SPA + AntD + Kong BFF, KHÔNG Next.js, KHÔNG PWA write-outbox) và **ADR-013** (token trong HttpOnly cookie, SPA không thấy token). Repo CHƯA CÓ CODE — mọi code path đánh dấu *(planned)* theo layout canon section 9.

---

## 1. Vì sao kỹ năng này quan trọng trong HMS

Frontend HMS không phải "web app marketing" — nó là **bề mặt thao tác lâm sàng** mà lễ tân, bác sĩ, dược sĩ, thu ngân, giám định viên dùng thay sổ giấy suốt ca trực. Một SPA dựng sai nền dẫn tới ba hậu quả trực tiếp cho bệnh viện:

- **Sai persona = lộ PHI hoặc thao tác sai vai trò.** Nếu nav và route không gate theo role (Keycloak group `bac_si/dieu_duong/duoc_si/le_tan/thu_ngan/giam_dinh/quan_tri`), thu ngân có thể mở màn hình kê đơn — vừa là lỗi UX vừa là vi phạm minimum-necessary (NĐ 13/2023). Lưu ý: RBAC nav chỉ là **UX gating**, object-level authz luôn enforce ở Go (ADR-013) — FE không bao giờ là control.
- **Token rò rỉ = chiếm phiên truy cập PHI.** Nếu SPA giữ access token trong `localStorage`, một lỗ XSS bất kỳ là một vụ rò token. ADR-013/ADR-018 chốt: Kong làm BFF, token nằm trong cookie `HttpOnly + Secure + SameSite=Strict`, **JS không đọc được token**. FE chỉ gọi API tương đối (`/api/...`) và dựa vào browser tự đính cookie.
- **Nền dựng cẩu thả = nợ kỹ thuật trên đường patient-journey.** SPA này sẽ phình tới ~5 persona × nhiều màn lâm sàng dày. Không có code-splitting theo route + typed router + một nguồn type duy nhất từ OpenAPI thì màn hình kê đơn và charge-capture sẽ drift khỏi backend, dội bug ngay trên luồng an toàn người bệnh.

MVP là MỘT phòng khám OPD-BHYT chạy trọn vòng tiếp đón→khám→CLS→kê đơn→viện phí→giám định→ký số EMR. Bộ khung SPA bạn dựng ở module này là khung chứa toàn bộ luồng đó.

---

## 2. Mô hình tư duy (first principles) — từ con số 0

Bắt đầu từ câu hỏi gốc: *trình duyệt cần gì để hiển thị một màn hình lâm sàng và gọi API an toàn?*

1. **Một file HTML + một bundle JS + CSS.** Đó là toàn bộ "SPA tĩnh". Server (Kong) chỉ phục vụ file tĩnh và proxy `/api`. Không có server-render — vì HMS sau đăng nhập, không cần SEO, không cần SSR (ADR-018 loại Next.js: "use-client/server boundary là cost thuần").
2. **Vite là dev-server + bundler.** Dev: ESM native + HMR cực nhanh (esbuild pre-bundle). Build: Rollup ra static assets. Bạn không cần hiểu webpack — Vite 6 là default hiện đại.
3. **React 19 là cây UI khai báo.** Bạn mô tả "trạng thái → giao diện", React reconcile. Điều mới ở 19: `use()` hook, Actions/`useActionState`, ref-as-prop (bỏ `forwardRef`), cải thiện Suspense.
4. **Routing là ánh xạ URL → cây component.** URL phải là **single source of truth** cho "đang ở màn nào, lọc gì" — nên search-params nằm trên URL (chia sẻ link, refresh không mất ngữ cảnh). TanStack Router cho điều này typed end-to-end.
5. **State có ba loại — đừng trộn.** (a) *Server state* (dữ liệu từ API) → TanStack Query, có cache + invalidation. (b) *URL state* (filter, tab, id) → router search-params. (c) *Global ephemeral* (sidebar mở/đóng, branch đang chọn, theme) → Zustand. Form state → RHF (module FE-2). Trộn ba loại vào một store là anti-pattern #1.
6. **Auth không nằm ở SPA.** SPA không có token, không có refresh-loop. Nó hỏi Kong "tôi là ai?" qua `/auth/me`, nhận identity (roles, branch_id, tên) và render theo đó. Mọi 401 → redirect tới Kong login (auth-code + PKCE).

Tư duy nền: **SPA là một client mỏng, không tin được; mọi quyết định bảo mật thật nằm ở Kong + Go + Postgres RLS.**

---

## 3. Khái niệm cốt lõi (tăng dần độ khó)

🥉 **Vite project & TS strict.** `vite.config.ts`, `tsconfig.json` với `"strict": true`, alias `@/` → `src/`. Env qua `import.meta.env` (chỉ biến `VITE_` mới lộ ra client — đừng nhét secret vào).

🥉 **React 19 component & hook.** Function component, `useState/useEffect`, và đặc biệt React 19: `use(promise)`, `useActionState`, ref truyền thẳng như prop.

🥈 **TanStack Router typed routes.** File/route tree khai báo, `Route` có `validateSearch` (zod) → search-params được type. `loader` prefetch dữ liệu. `beforeLoad` để chặn route theo role.

🥈 **TanStack Query v5.** `useQuery({ queryKey, queryFn })` cho read; `useMutation` cho write; `queryClient.invalidateQueries` sau mutation. `staleTime` cho reference data (danh mục ICD-10, chargemaster) cache lâu.

🥈 **Zustand store.** `create()` ra hook, chỉ giữ **global ephemeral** (branch hiện tại lấy từ `/auth/me`, UI flags). Không giữ server data ở đây.

🥇 **Per-persona surface + RBAC nav.** `features/<persona>/` tách theo vai trò; route group có `beforeLoad` kiểm role; nav menu dựng từ roles trả về bởi `/auth/me`. Lazy-load mỗi persona bundle (`React.lazy` + route-based code splitting).

🥇 **BFF auth integration.** Không lưu token; cookie tự đính kèm vì cùng origin sau Kong. Interceptor: 401 → `window.location = '/auth/login'`. CSRF: vì `SameSite=Strict` + cookie, cần double-submit/header token cho mutation (Kong BFF cấp).

🥇 **Read-only cached reference + hard-online gate.** ADR-018: KHÔNG PWA write-outbox. Reference data (chargemaster, ICD-10, danh mục thuốc) được cache đọc; nhưng **dispense / cashier / BHYT** có *hard-online gate* — nếu offline thì chặn write và hiển thị rõ, không queue.

---

## 4. HMS dùng nó thế nào (bám code path — *(planned)*)

Layout mục tiêu (canon section 9), tất cả *(planned)* vì repo chưa có code:

```
frontend/                                  # Vite 6 + React 19 + TS + AntD v6
├── package.json   vite.config.ts   tsconfig.json
└── src/
    ├── app/                # bootstrap: router, QueryClient, AntD ConfigProvider(vi_VN), Zustand providers
    │   ├── main.tsx        # createRoot — composition root của FE (planned)
    │   ├── router.tsx      # TanStack Router tree + beforeLoad RBAC (planned)
    │   └── providers.tsx   # QueryClientProvider + ConfigProvider locale=vi_VN (planned)
    ├── features/<persona>/ # reception | doctor | pharmacy | cashier | insurance (planned)
    ├── shared/             # auth (useAuth/<auth/me>), components (AllergyBanner...), hooks, lib (planned)
    └── api/                # orval-gen: types + hooks sinh từ OpenAPI — KHÔNG sửa tay (planned)
```

- **Composition root**: `src/app/main.tsx` *(planned)* — nơi DUY NHẤT `createRoot()` và bọc `<RouterProvider>` + `<QueryClientProvider>` + AntD `<ConfigProvider locale={viVN}>`. Tương tự `cmd/hms-api/main.go` là composition root backend (ADR-001), FE có một điểm wiring.
- **Typed router + RBAC gate**: `src/app/router.tsx` *(planned)* — route group `/reception`, `/encounter/$encounterId`, `/pharmacy`, `/cashier`, `/insurance`; mỗi group `beforeLoad` đọc roles từ auth store, throw `redirect` nếu thiếu quyền. Search-params (vd `?status=in-progress`) validate bằng zod.
- **Auth surface**: `src/shared/auth/useAuth.ts` *(planned)* — `useQuery(['me'], fetchMe)` gọi `GET /auth/me` qua Kong BFF; trả `{ accountId, roles, branchId, fullName }`. Branch hiện tại nuôi RLS context phía backend (ADR-005: branch_id từ JWT, KHÔNG từ client — FE chỉ hiển thị, không gửi branch_id lên).
- **Per-persona ví dụ**: `src/features/reception/RoutesReception.tsx` *(planned)* — màn tra cứu MPI + BHYT card-check verdict (eligible/ineligible/co-pay), degraded-mode "đã lưu, chờ gửi cổng" (ADR-006). `src/features/pharmacy/` *(planned)* nơi CDSS blocking modal sống — nhưng nhắc lại ADR-008: **modal chỉ là UX, hard-stop enforce ở Go**.
- **Generated API client**: `src/api/` *(planned)* — orval/openapi-typescript sinh type + TanStack Query hooks từ OpenAPI của backend (`doc/07-api-specification.md`), chống FE↔BE drift (ADR-018 hệ quả). Không bao giờ sửa tay file trong `api/`.

Minh hoạ wiring composition root *(planned)*:

```tsx
// src/app/main.tsx (planned)
import { createRoot } from 'react-dom/client'
import { RouterProvider } from '@tanstack/react-router'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { ConfigProvider } from 'antd'
import viVN from 'antd/locale/vi_VN'
import { router } from './router'

const queryClient = new QueryClient({
  defaultOptions: { queries: { retry: 1, refetchOnWindowFocus: false } },
})

createRoot(document.getElementById('root')!).render(
  <QueryClientProvider client={queryClient}>
    <ConfigProvider locale={viVN}>
      <RouterProvider router={router} />
    </ConfigProvider>
  </QueryClientProvider>,
)
```

```tsx
// src/app/router.tsx — RBAC beforeLoad gate (planned)
import { createRoute, redirect } from '@tanstack/react-router'
import { authStore } from '@/shared/auth/store'

export const pharmacyRoute = createRoute({
  path: '/pharmacy',
  beforeLoad: () => {
    const { roles } = authStore.getState()
    if (!roles.includes('duoc_si')) {
      // FE chỉ gating UX — object-level authz vẫn enforce ở Go (ADR-013)
      throw redirect({ to: '/forbidden' })
    }
  },
})
```

---

## 5. Best practices (mỗi mục kèm 1 nguồn đã research)

1. **Bật TypeScript `strict` ngay từ đầu, không bật dần.** Strict mode bắt null/undefined trên dữ liệu lâm sàng (vd `allergy: AllergyStatus | undefined` ép phân biệt allergy-unknown ≠ safe). Nguồn: TypeScript Handbook — Strictness <https://www.typescriptlang.org/tsconfig/#strict>.
2. **Tách ba loại state, để server-state cho TanStack Query.** Không đổ data API vào Zustand. Nguồn: TanStack Query — Important Defaults & philosophy <https://tanstack.com/query/latest/docs/framework/react/guides/important-defaults>.
3. **Đặt search-params trên URL, validate bằng schema.** Filter trạng thái encounter, id bệnh nhân nằm trên URL để share/refresh không mất ngữ cảnh. Nguồn: TanStack Router — Search Params <https://tanstack.com/router/latest/docs/framework/react/guide/search-params>.
4. **Route-based code splitting (lazy) cho mỗi persona.** Giảm bundle ban đầu; thu ngân không tải code kê đơn. Nguồn: React — `lazy` <https://react.dev/reference/react/lazy>.
5. **KHÔNG lưu token JWT trong `localStorage`/`sessionStorage`; dùng HttpOnly cookie + BFF.** Token trong JS-readable storage = XSS exfiltration. Nguồn: OWASP — JWT for Java Cheat Sheet / token storage guidance <https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html> và OWASP HTML5 Security Cheat Sheet (Local Storage) <https://cheatsheetseries.owasp.org/cheatsheets/HTML5_Security_Cheat_Sheet.html>.
6. **Cố định Zustand cho global ephemeral, dùng selector tránh re-render thừa.** Nguồn: Zustand docs — Recipes/selectors <https://zustand.docs.pmnd.rs/guides/updating-state>.
7. **Dùng `import.meta.env` đúng cách, chỉ expose biến `VITE_`.** Đừng để base URL/secret rò ra bundle. Nguồn: Vite — Env Variables and Modes <https://vite.dev/guide/env-and-mode>.
8. **Generate client từ OpenAPI thay vì viết fetch tay.** Một schema = type + hook, chống FE↔BE drift. Nguồn: orval — Getting Started <https://orval.dev/overview>.

---

## 6. Lỗi thường gặp & anti-patterns

- **Lưu access token trong `localStorage`.** Vi phạm ADR-013. Hậu quả: XSS → rò token → truy cập PHI. Đúng: cookie HttpOnly, SPA gọi `/auth/me`.
- **Gửi `branch_id` lên backend từ client để "chọn chi nhánh".** Vi phạm ADR-005 (tenant spoofing). branch_id luôn trích từ JWT đã verify ở Go. FE chỉ hiển thị tên chi nhánh.
- **Coi React CDSS modal là control an toàn.** Vi phạm ADR-008 — modal bypassable qua devtools/API. Hard-stop phải ở Go aggregate.
- **Đổ server data vào Zustand rồi tự đồng bộ.** Tự dựng cache không có stale/invalidation → dữ liệu lâm sàng cũ. Để TanStack Query lo.
- **Dựng PWA write-outbox cho dispense/cashier.** Vi phạm ADR-018 (mis-replay = safety hazard). MVP: read-only cached reference + hard-online gate, chặn write khi offline.
- **Một bundle khổng lồ không code-split.** Tải toàn bộ 5 persona ngay lần đầu → chậm trên máy trạm khoa cũ.
- **Tắt `strict` để "code nhanh hơn".** Mất chính lớp bắt null trên field an toàn người bệnh.
- **Hardcode chuỗi tiếng Việt rải rác thay vì qua AntD `ConfigProvider vi_VN` + chỗ tập trung i18n.** Khó kiểm phiếu pháp lý sau này.

---

## 7. Lộ trình luyện tập NGAY trong repo

> Repo chưa có `frontend/`. Các bài tập tạo đúng layout *(planned)* canon section 9 để khung sẵn sàng cho FE-2/FE-3.

🥉 **Cơ bản — scaffold SPA.** Tạo `frontend/` bằng Vite (`npm create vite@latest frontend -- --template react-ts`), bật `tsconfig.json strict`, thêm alias `@/`. Dựng `src/app/main.tsx` *(planned)* render một trang "HMS — OPD-BHYT" với AntD `ConfigProvider locale={viVN}`. Tiêu chí: `npm run build` ra static assets, `npm run dev` chạy HMR.

🥈 **Trung cấp — typed router + 5 route persona rỗng.** Cài TanStack Router + Query + Zustand. Dựng route tree `/reception /encounter/$encounterId /pharmacy /cashier /insurance`, mỗi route lazy-load một component placeholder. Thêm zod `validateSearch` cho `/reception?cccd=...`. Tạo `authStore` (Zustand) seed roles giả lập. Tiêu chí: chuyển route giữ search-params, mỗi persona là chunk riêng (xem `dist/` sau build).

🥇 **Nâng cao — RBAC gate + BFF auth mock.** Thêm `beforeLoad` trên mỗi route group chặn theo role; `useAuth` gọi `GET /auth/me` (mock bằng MSW trả `{roles, branchId, fullName}`), redirect 401 → `/auth/login`. Dựng nav menu sinh từ roles. Bonus: dựng `hard-online gate` HOC chặn route dispense/cashier khi `navigator.onLine === false` và hiển thị banner. Tiêu chí: đăng nhập role `le_tan` không vào được `/pharmacy`; offline chặn `/cashier`.

---

## 8. Skill/agent ECC nên dùng khi luyện

- **`ecc:react-build`** — fix lỗi build Vite/JSX/TSX, hydration, missing types khi scaffold.
- **`ecc:react-review`** — review hook correctness, render performance, server/client boundary, a11y; chạy kèm typescript-reviewer trên file TSX.
- **`ecc:react-test`** — TDD với React Testing Library (behavior-first, accessibility-first), phát hiện Vitest/Jest, kiểm coverage.
- **`ecc:vite-patterns`** / **`ecc:react-patterns`** — pattern dựng project Vite & cấu trúc React production.
- **`ecc:frontend-a11y`** / **`ecc:accessibility`** — WCAG 2.2 AA cho UI lâm sàng (chuẩn bị cho FE-2/FE-3 và phiếu pháp lý).
- **`ecc:security-review`** — soi token storage, CSP, XSS surface trước khi đụng auth flow (BFF cookie).

---

## 9. Tài nguyên học thêm (2024–2026)

- Vite — Guide & Why Vite: <https://vite.dev/guide/>
- React 19 — Release & new features: <https://react.dev/blog/2024/12/05/react-19>
- TanStack Router — Overview: <https://tanstack.com/router/latest/docs/framework/react/overview>
- TanStack Query v5 — Overview: <https://tanstack.com/query/latest/docs/framework/react/overview>
- Zustand — Docs: <https://zustand.docs.pmnd.rs/>
- TypeScript — tsconfig `strict`: <https://www.typescriptlang.org/tsconfig/#strict>
- OWASP Cheat Sheet Series (token storage, HTML5): <https://cheatsheetseries.owasp.org/>
- orval (OpenAPI → TS client/hooks): <https://orval.dev/>
- Ant Design v6 — i18n (ConfigProvider): <https://ant.design/docs/react/i18n>

---

## 10. Checklist "đã hiểu"

- [ ] Giải thích được vì sao HMS chọn Vite SPA tĩnh thay vì Next.js (ADR-018) và hệ quả "không SSR".
- [ ] Nêu được token nằm ở đâu và vì sao SPA không bao giờ thấy token (ADR-013, Kong BFF, HttpOnly cookie).
- [ ] Phân biệt rõ ba loại state: server (TanStack Query) / URL (router search-params) / global ephemeral (Zustand), và biết loại nào KHÔNG để vào Zustand.
- [ ] Biết branch_id không bao giờ gửi từ client (ADR-005) — FE chỉ hiển thị.
- [ ] Dựng được route group per-persona với `beforeLoad` RBAC gate và hiểu nó chỉ là UX gating, không phải control (authz ở Go).
- [ ] Cấu hình được code-splitting lazy theo persona và kiểm chứng chunk riêng trong `dist/`.
- [ ] Giải thích được "read-only cached reference + hard-online gate" và vì sao MVP KHÔNG có PWA write-outbox (ADR-018).
- [ ] Hiểu CDSS modal chỉ là UX, hard-stop enforce backend (ADR-008).
- [ ] Biết client API sinh từ OpenAPI (orval) chống FE↔BE drift, không sửa tay `src/api/`.
- [ ] Biết composition root FE là `src/app/main.tsx` *(planned)*, tương tự `cmd/hms-api/main.go` ở backend.
