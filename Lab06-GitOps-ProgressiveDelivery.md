# Lab 06 — GitOps, Progressive Delivery & Đa cụm

> Hệ thống xuyên suốt: **smartapp** (web = podinfo). Lab này xây pipeline GitOps hoàn chỉnh: Git → ArgoCD (staging) → Argo Rollouts canary (production) có cổng phân tích metrics.
> **Phụ thuộc:** Prometheus (Bài 03) phải đang chạy để cổng phân tích truy vấn.

**Thời lượng:** ~60 phút · **Yêu cầu:** `kubectl` cluster-admin, `helm`, một Git repo (GitHub/GitLab nội bộ), `kubectl-argo-rollouts` plugin.

---

## 1. BT1 — Tổ chức cấu hình với Kustomize

Repo `smartapp-gitops` (https://github.com/yesiam111/smartapp-gitops) (web = podinfo), có 2 phiên bản đóng tag:

- tag `v1` — podinfo **6.13.0** (phiên bản khởi đầu, cũng là nhánh `main`)
- tag `v2` — podinfo **6.14.0** (phiên bản nâng cấp ở BT sau)

Cấu trúc:
```
base/
  deployment.yaml      # web = podinfo
  service.yaml
  kustomization.yaml
overlays/
  staging/kustomization.yaml      # replicas 2
  production/kustomization.yaml   # replicas 5
argocd/
  smartapp-staging.yaml           # Application mẫu (đổi repoURL)
```
`overlays/production/kustomization.yaml`:
```yaml
resources: [../../base]
namespace: smartapp
replicas:
- { name: web, count: 5 }
images:
- { name: web, newName: stefanprodan/podinfo, newTag: "6.13.0" }
```
> **Lưu ý:** `images.name: web` khớp theo *tên container* `web` trong base — nếu đổi tên container thì phải đổi cả ở đây.

**Các bước:**

1. **Fork** repo gốc của trainer về tài khoản GitHub/GitLab của bạn (fork công khai để ArgoCD đọc được, khỏi cần token). Không cần `git clone` hay CLI.
2. Trong fork, mở `argocd/smartapp-staging.yaml` bằng **Web IDE** và đổi `repoURL` thành fork của bạn.
3. (Tuỳ chọn) Kiểm tra render trên máy trainer: `kubectl kustomize overlays/production` → kỳ vọng 5 replica; `overlays/staging` → 2 replica.

**Kết quả mong đợi:** bạn có fork riêng ở podinfo 6.13.0; mọi thay đổi ở BT sau đều commit qua Web IDE ngay trên trình duyệt.

> **Trainer** dựng repo gốc 1 lần bằng `./scaffold-smartapp-gitops.sh` rồi `git push -u origin main --tags` lên remote công khai.

---

## 2. BT2 — ArgoCD đồng bộ lên staging

```bash
kubectl create namespace argocd
kubectl apply --server-side -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server
```
Tạo Application cho staging:
```bash
kubectl apply -f - <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: { name: smartapp-staging, namespace: argocd }
spec:
  project: default
  source:
    repoURL: https://github.com/<YOUR-USER>/smartapp-gitops.git   # ĐỔI thành fork của bạn
    path: overlays/staging
    targetRevision: main        # hoặc pin tag: v1 (cũ) -> v2 (mới)
  destination:
    server: https://kubernetes.default.svc
    namespace: smartapp-staging
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [CreateNamespace=true]
EOF
kubectl -n argocd get applications
```
**Kết quả mong đợi:** Application `Synced` + `Healthy`; namespace `smartapp-staging` có web đang chạy.

Thử **self-heal** (gây drift):
```bash
kubectl -n smartapp-staging scale deploy web --replicas=1
# Quan sát ArgoCD tự đưa về 2 (đúng Git) trong vài giây
kubectl -n smartapp-staging get deploy web -w
```
**Câu hỏi suy ngẫm:** vì sao thay đổi tay bị "hoàn tác"? *(Gợi ý: Git là nguồn sự thật; selfHeal đưa cụm về khớp Git.)*

---

## 3. BT3 — Argo Rollouts canary lên production

```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
# plugin CLI
curl -sLO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x kubectl-argo-rollouts-linux-amd64 && sudo mv kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts
```
Chuyển web (production) sang `Rollout` với phân tích Prometheus.
> ⚠️ Trước tiên **xóa Deployment `web` cũ** (từ Bài 01) — nếu không, Deployment và Rollout cùng quản pod `app=web` sau cùng một Service:
```bash
kubectl -n smartapp delete deploy web --ignore-not-found
```
```yaml
kubectl apply -f - <<'EOF'
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata: { name: web, namespace: smartapp, labels: { release: kps } }   # release=kps để Prometheus (Bài 03) chọn
spec:
  selector: { matchLabels: { app: web } }
  endpoints: [{ targetPort: 9898, path: /metrics, interval: 15s }]        # scrape podinfo /metrics
---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata: { name: success-rate, namespace: smartapp }
spec:
  metrics:
  - name: success-rate
    interval: 30s
    count: 3                     # BẮT BUỘC cho analysis trong bước canary — thiếu 'count' ⇒ "runs indefinitely" (InvalidSpec)
    successCondition: "len(result) > 0 && result[0] >= 0.95"   # provider Prometheus trả []float64 ⇒ dùng result[0]
    failureLimit: 1
    provider:
      prometheus:
        # ⚠️ Địa chỉ này TÙY CỤM (tên Service của Prometheus). Xác minh trên cụm của bạn:
        #   kubectl -n monitoring get svc | grep -i prometheus
        address: http://kps-kube-prometheus-stack-prometheus.monitoring:9090
        # Lọc theo nhãn 'service' (operator TỰ gắn từ tên Service 'web') — không cần targetLabels.
        query: |
          sum(rate(http_requests_total{service="web",status!~"5.."}[1m]))
            / sum(rate(http_requests_total{service="web"}[1m]))
---
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata: { name: web, namespace: smartapp }
spec:
  replicas: 4
  selector: { matchLabels: { app: web } }
  template:
    metadata: { labels: { app: web } }
    spec:
      containers:
      - name: web
        image: stefanprodan/podinfo:6.13.0    # bản hiện tại; canary sẽ promote lên 6.14.0
        ports: [{ containerPort: 9898 }]
  strategy:
    canary:
      steps:
      - setWeight: 25
      - pause: { duration: 1m }
      - analysis: { templates: [{ templateName: success-rate }] }
      - setWeight: 50
      - pause: { duration: 1m }
      - setWeight: 100
EOF
```
Theo dõi:
```bash
kubectl argo rollouts get rollout web -n smartapp --watch
```
Trigger một bản mới (tốt) và promote:
```bash
kubectl argo rollouts set image web web=stefanprodan/podinfo:6.14.0 -n smartapp
kubectl argo rollouts get rollout web -n smartapp --watch
```
**Kết quả mong đợi:** rollout đi qua các bước canary; cổng analysis PASS (success-rate ≥ 95%); promote tới 100%.

---

## 4. BT4 — Auto-rollback khi metrics xấu

Đẩy một bản "lỗi" và để hệ thống tự quyết:
```bash
# Dùng tag lỗi hoặc bơm lỗi 500 song song để success-rate tụt
kubectl argo rollouts set image web web=stefanprodan/podinfo:6.14.0 -n smartapp
# Trong khi canary chạy, bơm lỗi để vi phạm ngưỡng:
kubectl -n smartapp run errgen --rm -it --image=busybox --restart=Never -- \
  sh -c 'for i in $(seq 1 800); do wget -q -O- http://web.smartapp:9898/status/500 >/dev/null 2>&1; done'
kubectl argo rollouts get rollout web -n smartapp --watch
```
**Kết quả mong đợi:** cổng analysis FAIL (success-rate < 95%) → Argo Rollouts **tự động Degraded/Abort** và quay về bản ổn định, KHÔNG cần con người. Đây là "tự lái" cho triển khai.

**Câu hỏi suy ngẫm:** điều gì sẽ xảy ra nếu không có cổng phân tích? *(Gợi ý: bản lỗi sẽ lên 100% và gây sự cố production.)*

---

## 5. (Nâng cao — nếu còn thời gian)

- **app-of-apps:** một Application "cha" trỏ tới thư mục chứa Application con (monitoring, kyverno, smartapp) → bootstrap cả cụm từ một repo.
- **Sync waves:** annotation `argocd.argoproj.io/sync-wave` để postgres (wave 0) lên trước web (wave 1).
- **Multi-cluster:** `argocd cluster add <context>` rồi dùng **ApplicationSet** generator để deploy smartapp ra nhiều cụm.
- **Blue-green:** đổi strategy Rollout sang blueGreen với `previewService`/`activeService`.

---

## 6. Cleanup

> ⚠️ **web hiện là `Rollout`, không còn `Deployment`** — các lệnh `deploy/web` của Bài 07 sẽ lỗi. Trước Bài 07, chọn một trong hai:

```bash
# (a) Đơn giản nhất: reset về baseline (cài lại Prometheus trước Bài 07 nếu cần)
./graders/reset.sh smartapp

# (b) Giữ cụm, chỉ chuyển web về Deployment:
kubectl -n smartapp delete rollout web --ignore-not-found
kubectl -n smartapp create deployment web --image=stefanprodan/podinfo:6.14.0 --replicas=2
kubectl -n smartapp delete pod errgen --ignore-not-found
```

---

## 7. Tổng kết & nối sang Bài 07

- Bạn đã xây pipeline GitOps đầy đủ: Git → ArgoCD (staging, self-heal) → Argo Rollouts canary (production) với cổng metrics tự rollback.
- Đây là đỉnh tổng hợp: reconcile (Bài 02) + Prometheus (Bài 03) + canary (Bài 05) hợp lại.
- **Bài 07:** tự động mở rộng quy mô & hiệu suất — VPA, KEDA, Cluster Autoscaler, custom metrics và lập lịch nâng cao.
