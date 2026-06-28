# [K8S-1] Kubernetes architecture & components (onshore VN)

> Module K8S-1 · Kiến trúc & thành phần Kubernetes cho HMS chạy onshore Việt Nam, chuẩn bảo vệ PHI · Độ khó: 🥉→🥇 · Prereqs: BE-1 (Go production-grade)

Module này dạy bạn đọc-hiểu-vận hành cụm Kubernetes (K8s) làm nền chạy `hms-api` (Go modular monolith), Kong và các workload nền. Repo HIỆN CHƯA CÓ CODE — mọi path là layout MỤC TIÊU *(planned)* ở canon section 9 (`deploy/`, `infra/`). Triển khai cụ thể HMS + Kong KIC + CNPG nằm ở module K8S-2; ở đây ta nắm vững **thành phần và bất biến kiến trúc** trước.

---

## 1. Vì sao kỹ năng này quan trọng trong HMS

HMS là hệ thống production số hóa một bệnh viện Việt Nam: mọi thao tác giấy trở thành sự kiện số có ký số + vết audit bất biến. Hậu quả là **uptime và tính toàn vẹn dữ liệu là vấn đề an toàn người bệnh**, không chỉ vấn đề IT. K8s là lớp điều phối quyết định:

- **PHI residency & isolation**: PHI phải onshore VN (NĐ 53/2022 + NĐ 13/2023). K8s namespace + NetworkPolicy default-deny là ranh giới cô lập network giữa workload PHI và phần còn lại. Đặt sai = leak hoặc vi phạm pháp lý.
- **Degraded-mode không tự nhiên xảy ra**: ADR-006 yêu cầu tiếp đón/cashier không bao giờ chặn người bệnh khi cổng BHYT lỗi. Nhưng nếu `hms-api` pod tự crash-loop vì thiếu probe/resource limit thì degraded-mode trong code cũng vô nghĩa — K8s phải giữ pod sống và không kill nhầm pod đang xử lý charge/claim.
- **Signed-EMR durability**: ADR-004/ADR-015 — write path EMR ký số cần synchronous durability. Cách K8s rolling-update (`maxUnavailable=0`) và PodDisruptionBudget (PDB) quyết định một pod đang commit chữ ký có bị evict giữa chừng không.
- **Operating model nhỏ**: ADR-002 chốt đội IT bệnh viện kiêm ops ở MVP với *MVP component budget* cứng. K8s manifests phải đơn giản, đọc được, GitOps-versioned — không service mesh, không canary ở MVP (ADR-019).

Hiểu sai K8s ở đây không phải "site chậm" — mà là "bác sĩ quay về giấy" (trigger #1 abandon system) hoặc "leak PHI đa chi nhánh".

---

## 2. Mô hình tư duy (first principles) — từ con số 0

Bắt đầu từ một câu hỏi: *làm sao chạy một process Go (`hms-api`) trên nhiều máy, sống sót khi máy chết, mà người vận hành chỉ khai báo trạng thái mong muốn?*

1. **Container** = một process + filesystem đóng gói, chạy giống nhau mọi nơi. `hms-api` build thành một image.
2. **Pod** = đơn vị nhỏ nhất K8s lập lịch: một (hoặc vài) container chia network/storage. HMS: 1 pod = 1 instance `hms-api`.
3. **Declarative & reconciliation**: bạn không ra lệnh "khởi động pod". Bạn khai báo *desired state* (YAML, lưu Git). K8s **control loop** liên tục so sánh desired vs actual và tự sửa. Đây là cùng triết lý GitOps của Argo CD (ADR-019).
4. **Controller hierarchy**: bạn không tạo Pod trực tiếp. Bạn tạo **Deployment** → tạo **ReplicaSet** → tạo **Pod**. Deployment cho phép rolling update + rollback.
5. **Service & DNS**: Pod IP thay đổi liên tục → **Service** cho một tên DNS ổn định + load-balance tới các pod healthy.
6. **Scheduler đặt pod ở đâu**: dựa resource request, taint/toleration, affinity, topology spread → phân bố qua ≥2 AZ.

Tư duy cốt lõi: **mọi thứ là một object có spec (mong muốn) và status (thực tế); control plane là vòng lặp kéo status → spec.** Khi debug, luôn hỏi "spec là gì, status là gì, controller nào reconcile, vì sao chưa khớp".

---

## 3. Khái niệm cốt lõi (tăng dần độ khó) 🥉→🥇

🥉 **Control plane vs data plane**
- Control plane: `kube-apiserver` (cổng vào duy nhất, mọi `kubectl`/controller nói chuyện qua đây), `etcd` (key-value store giữ toàn bộ state — **phải bật encryption-at-rest**, canon datastore), `kube-scheduler`, `kube-controller-manager`.
- Data plane (worker node): `kubelet` (chạy pod, báo cáo status), container runtime (containerd), `kube-proxy` (routing Service). Canon deploy: **≥3 worker, ≥2 AZ**.

🥈 **Workload & ổn định**
- **Namespace**: ranh giới logic + quota + RBAC + NetworkPolicy. HMS: `hms-dev`, `hms-staging`, `hms-prod` (namespace-per-env ngày 1; prod tách cluster riêng tại go-live — ADR-002/deploy).
- **Deployment**: stateless `hms-api`. Canon: **≥3 replica, rolling `maxUnavailable=0`**.
- **Probe**: `livez` (process treo → restart), `readyz` (chưa sẵn sàng → ngừng route traffic), `startupProbe` (cho cold start chậm, hoãn liveness). Phân biệt liveness vs readiness là chỗ sai phổ biến nhất (mục 6).
- **HPA** (HorizontalPodAutoscaler): scale theo CPU/metric. **PDB** (PodDisruptionBudget): `minAvailable=2` — chặn voluntary disruption (drain node, rolling) làm sập quá số pod cho phép.

🥇 **Bảo mật & phân bố**
- **securityContext** (canon, cứng): `runAsNonRoot`, `readOnlyRootFilesystem`, `drop ALL capabilities`, `seccompProfile: RuntimeDefault`. Áp dụng Pod Security Admission mức `restricted` cho namespace PHI.
- **NetworkPolicy default-deny**: K8s mặc định *cho phép mọi traffic* — phải khai báo policy `deny-all` rồi allow tường minh. PHI namespace isolated.
- **topologySpreadConstraints**: rải pod qua AZ/node để một AZ chết không sập toàn bộ replica.
- **ResourceQuota + LimitRange + request/limit**: chống một workload ăn hết tài nguyên node — bảo vệ pod đang xử lý charge/claim.
- **StatefulSet + PVC**: cho workload có state (CNPG Postgres — chi tiết K8S-2). Khác Deployment ở stable identity + ordered.

---

## 4. HMS dùng nó thế nào (bám code path — *(planned)*)

Layout mục tiêu (canon section 9): manifests sống trong `deploy/` + `infra/`, reconcile bởi Argo CD app-of-apps (ADR-019). Tất cả dưới đây *(planned)* — repo chưa có code.

| Thành phần | Path *(planned)* | Quyết định neo |
|---|---|---|
| Cluster/cloud IaC | `infra/` (OpenTofu) | ADR-002 (onshore VN, managed) |
| K8s base manifests | `deploy/kustomize/base/` | ADR-019 (Kustomize overlays) |
| Per-env overlay | `deploy/kustomize/overlays/{dev,staging,prod}/` | namespace-per-env |
| Kong KIC DB-less | `deploy/kong/` (Gateway API + KongPlugin CRD) | ADR-019, K8S-2 |
| Argo CD app-of-apps | `deploy/argocd/` | ADR-019 (rolling, manual promotion) |
| Helm cho operators | `deploy/helm/` (cert-manager, CNPG, ESO) | ADR-015/secrets |

Deployment `hms-api` *(planned)* — đọc image build từ BE-1 (`cmd/hms-api/main.go` là composition root duy nhất):

```yaml
# deploy/kustomize/base/hms-api/deployment.yaml  (planned)
apiVersion: apps/v1
kind: Deployment
metadata: { name: hms-api, namespace: hms-prod }
spec:
  replicas: 3                       # canon: ≥3 replica
  strategy:
    type: RollingUpdate
    rollingUpdate: { maxUnavailable: 0, maxSurge: 1 }   # không giảm capacity khi deploy
  template:
    spec:
      securityContext: { runAsNonRoot: true, seccompProfile: { type: RuntimeDefault } }
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone     # rải ≥2 AZ
          whenUnsatisfiable: DoNotSchedule
          labelSelector: { matchLabels: { app: hms-api } }
      containers:
        - name: hms-api
          image: registry.onshore.vn/hms/hms-api:<git-sha>   # version-pin, không :latest
          ports: [{ containerPort: 8080 }]
          securityContext:
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
            capabilities: { drop: ["ALL"] }
          resources:
            requests: { cpu: "250m", memory: "256Mi" }
            limits:   { cpu: "1",    memory: "512Mi" }
          startupProbe:  { httpGet: { path: /startupz, port: 8080 }, failureThreshold: 30, periodSeconds: 2 }
          livenessProbe: { httpGet: { path: /livez,    port: 8080 }, periodSeconds: 10 }
          readinessProbe:{ httpGet: { path: /readyz,   port: 8080 }, periodSeconds: 5 }
```

```yaml
# deploy/kustomize/base/hms-api/pdb.yaml  (planned)
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: hms-api, namespace: hms-prod }
spec:
  minAvailable: 2                   # canon: PDB minAvailable=2
  selector: { matchLabels: { app: hms-api } }
```

NetworkPolicy default-deny cho namespace PHI *(planned)* — chốt ranh giới ADR-003/ADR-005 ở tầng network (bổ trợ RLS ở tầng DB, không thay thế):

```yaml
# deploy/kustomize/base/netpol/default-deny.yaml  (planned)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: default-deny-all, namespace: hms-prod }
spec:
  podSelector: {}                   # mọi pod
  policyTypes: [Ingress, Egress]    # chặn cả vào lẫn ra; allow tường minh sau
```

```yaml
# allow hms-api → Postgres (planned): chỉ mở đúng cặp + cổng
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: allow-hmsapi-to-pg, namespace: hms-prod }
spec:
  podSelector: { matchLabels: { app: hms-api } }
  policyTypes: [Egress]
  egress:
    - to: [{ podSelector: { matchLabels: { app: postgres } } }]
      ports: [{ protocol: TCP, port: 5432 }]
```

**Bốn tầng defense-in-depth** (canon doc/01) và vị trí K8s trong đó: (1) Kong edge-auth, (2) Go object-level authz, (3) Postgres FORCE RLS, (4) **K8s/network isolation** — module này là tầng 4. Lưu ý: audit log compliance ship riêng tới **WORM sink** (ADR-009), KHÔNG chỉ ở Loki — sink này là một egress được allow tường minh, không nằm trong cụm app.

**KHÔNG ở MVP** (ADR-019/ADR-002, đánh dấu để bạn không scaffold sớm): service mesh (Linkerd) *(Phase 3)*, Argo Rollouts canary *(Phase 3)*, Tempo distributed tracing *(Phase 3)*, DB-per-branch *(Phase 4)*.

---

## 5. Best practices (mỗi mục kèm 1 nguồn đã research)

1. **Luôn set resource requests + limits; đặt liveness ≠ readiness rõ ràng.** Không có request → scheduler đặt mù; không có limit → một pod ăn hết node. Nguồn: Kubernetes — *Configure Liveness, Readiness and Startup Probes* (https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/).
2. **Áp Pod Security Admission mức `restricted` cho namespace PHI** thay vì tự viết policy rời rạc. Nguồn: Kubernetes — *Pod Security Standards* (https://kubernetes.io/docs/concepts/security/pod-security-standards/).
3. **Bắt đầu mọi namespace bằng NetworkPolicy default-deny rồi allow tường minh.** Mặc định K8s allow-all. Nguồn: Kubernetes — *Network Policies* (https://kubernetes.io/docs/concepts/services-networking/network-policies/).
4. **Định nghĩa PDB cho mọi workload nhiều replica** để node drain/upgrade không sập quá ngưỡng. Nguồn: Kubernetes — *Specifying a Disruption Budget* (https://kubernetes.io/docs/tasks/run-application/configure-pdb/).
5. **Dùng topologySpreadConstraints rải pod qua zone/node**, không phó mặc scheduler mặc định. Nguồn: Kubernetes — *Pod Topology Spread Constraints* (https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/).
6. **Pin image theo digest/sha, không `:latest`; quét image (Trivy) là gate.** Nguồn: Kubernetes — *Configuration Best Practices: Container Images* (https://kubernetes.io/docs/concepts/configuration/overview/).
7. **Bật encryption-at-rest cho etcd** (etcd giữ Secret + toàn bộ state). Nguồn: Kubernetes — *Encrypting Confidential Data at Rest* (https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/).
8. **Kiểm manifest tự động theo CIS baseline.** Nguồn: CIS *Kubernetes Benchmark* (https://www.cisecurity.org/benchmark/kubernetes).

---

## 6. Lỗi thường gặp & anti-patterns

- **Liveness = readiness (cùng endpoint).** Khi DB chậm, readiness fail (đúng: ngừng route) nhưng nếu cùng path thì liveness cũng fail → K8s restart pod → crash-loop toàn cụm khi BHYT/PG chậm. HMS: `livez` chỉ kiểm process; `readyz` kiểm dependency.
- **Liveness probe gọi xuống Postgres/BHYT.** Dependency ngoài chết → toàn bộ pod bị restart, phá degraded-mode (ADR-006). Liveness chỉ self-check.
- **Quên NetworkPolicy → namespace PHI mở toang.** RLS bảo vệ row trong DB, nhưng nếu network mở thì một pod bị compromise nói chuyện tự do với mọi service. Tầng 3 và tầng 4 phải cùng có.
- **`maxUnavailable>0` trên signed-EMR path.** Rolling update evict pod đang commit chữ ký giữa chừng → mất durability (ADR-004). Dùng `maxUnavailable=0` + PDB + graceful `terminationGracePeriodSeconds`.
- **Không có PDB** → cluster autoscaler/node drain giết cùng lúc nhiều pod → mất quorum xử lý charge/claim.
- **Chạy container as root / writable rootfs.** Vi phạm securityContext cứng của canon; tăng bề mặt tấn công PHI.
- **Front-load stateful systems** (Vault-đầy-đủ/Kafka/NATS/mesh) vào MVP — vi phạm MVP component budget (ADR-002): "Vault/Kafka vận hành kém còn nguy hiểm cho PHI hơn managed đơn giản".
- **Đặt PHI/secret thẳng trong manifest Git.** Dùng ESO/KMS (K8S-2), không hardcode (security checklist).

---

## 7. Lộ trình luyện tập NGAY trong repo 🥉→🥈→🥇

> Repo chưa có manifest — bạn sẽ tạo skeleton *(planned)* dưới `deploy/`. Cài `kind` (K8s local) + `kubectl` + `kustomize`.

🥉 **Cơ bản** — Dựng cụm local + chạy một pod giả `hms-api`:
```bash
kind create cluster --name hms-dev
kubectl create namespace hms-dev
kubectl run hms-api --image=nginx --port=8080 -n hms-dev   # placeholder cho hms-api
kubectl get pods -n hms-dev -o wide        # xem node/IP
kubectl describe pod -n hms-dev            # đọc Events, hiểu vòng đời pod
```
Mục tiêu: phân biệt được spec vs status, đọc Events khi pod Pending/CrashLoop.

🥈 **Trung cấp** — Viết Deployment thật theo canon dưới `deploy/kustomize/base/hms-api/` *(planned)*: 3 replica, `maxUnavailable=0`, full securityContext (runAsNonRoot/readOnlyRootFilesystem/drop ALL/seccompRuntimeDefault), 3 probe tách biệt, resource request/limit, kèm PDB `minAvailable=2`. Apply, rồi:
```bash
kubectl rollout status deploy/hms-api -n hms-dev
kubectl drain <node> --ignore-daemonsets   # quan sát PDB chặn drain quá ngưỡng
```

🥇 **Nâng cao** — Thêm `deploy/kustomize/base/netpol/default-deny.yaml` + allow `hms-api→postgres:5432` *(planned)*; thêm overlays `dev/staging/prod` (đổi namespace + replica + image tag); thêm topologySpreadConstraints qua zone. Chứng minh isolation: deploy một pod "rogue" và xác nhận nó **không** mở được kết nối tới pod PHI khi default-deny bật. Viết Argo CD `Application` skeleton trỏ vào overlay (chuẩn bị cho K8S-2).

---

## 8. Skill/agent ECC nên dùng khi luyện

- **`ecc:kubernetes-patterns`** — review manifest theo best practice (probe, PDB, securityContext, topology). Dùng ngay sau khi viết Deployment/PDB ở 🥈.
- **`ecc:docker-patterns`** — tối ưu Dockerfile `hms-api` (multi-stage, distroless/nonroot base) trước khi build image cho cluster.
- **`ecc:security-scan` / `ecc:security-review`** — soi securityContext, NetworkPolicy, secret exposure trên manifest PHI namespace (security checklist + ADR-003/005).
- **`ecc:deployment-patterns`** — kiểm rolling vs canary; xác nhận MVP giữ rolling (ADR-019), không scaffold canary sớm.
- **`ecc:harness-audit` / `ecc:repo-scan`** — audit `deploy/` skeleton so với layout canon section 9.
- **`ecc:plan`** — lập kế hoạch trước khi tạo overlay đa môi trường để tránh scope creep (cảnh báo ADR-002 budget).

---

## 9. Tài nguyên học thêm (2024–2026)

- Kubernetes Documentation — Concepts & Tasks (cập nhật liên tục): https://kubernetes.io/docs/concepts/
- Kubernetes — Pod Security Standards & Admission: https://kubernetes.io/docs/concepts/security/pod-security-standards/
- CIS Kubernetes Benchmark (baseline hardening): https://www.cisecurity.org/benchmark/kubernetes
- NSA/CISA — Kubernetes Hardening Guide: https://media.defense.gov/2022/Aug/29/2003066362/-1/-1/0/CTR_KUBERNETES_HARDENING_GUIDANCE_1.2_20220829.PDF
- Argo CD Docs (app-of-apps, GitOps) — chuẩn bị K8S-2/DSO-1: https://argo-cd.readthedocs.io/
- CloudNativePG (CNPG) Docs — Postgres trên K8s, dùng ở K8S-2: https://cloudnative-pg.io/documentation/
- Kong Ingress Controller (KIC) Docs — Gateway API + DB-less: https://docs.konghq.com/kubernetes-ingress-controller/

---

## 10. Checklist "đã hiểu"

- [ ] Phân biệt được control plane (apiserver/etcd/scheduler) vs data plane (kubelet/kube-proxy) và vì sao etcd cần encryption-at-rest
- [ ] Giải thích được reconciliation loop (spec vs status) và liên hệ với GitOps/Argo CD (ADR-019)
- [ ] Viết được Deployment `hms-api` theo canon: ≥3 replica, `maxUnavailable=0`, full securityContext, 3 probe tách biệt, request/limit
- [ ] Phân biệt liveness vs readiness vs startup và biết vì sao liveness KHÔNG được gọi dependency ngoài (degraded-mode, ADR-006)
- [ ] Định nghĩa được PDB `minAvailable=2` và demo nó chặn node-drain quá ngưỡng
- [ ] Hiểu NetworkPolicy default-deny là tầng 4 defense-in-depth, bổ trợ chứ không thay RLS (tầng 3, ADR-003/005)
- [ ] Biết topologySpreadConstraints rải pod qua ≥2 AZ và vì sao quan trọng cho HMS
- [ ] Nêu được những thành phần KHÔNG có ở MVP (mesh/canary/Tempo) và trigger earn-in (ADR-002/ADR-019)
- [ ] Map được layout `deploy/` + `infra/` *(planned)* theo canon section 9 và biết Argo CD reconcile từ đâu
- [ ] Biết audit log đi WORM sink riêng (ADR-009), không chỉ Loki, và đó là egress allow tường minh
