# Lab 06 — GitOps, Progressive Delivery & Đa cụm

> Hệ thống xuyên suốt: **smartapp** (web = podinfo). Lab này xây pipeline GitOps hoàn chỉnh: Git → ArgoCD (staging) → Argo Rollouts canary (production) có cổng phân tích metrics.
> **Phụ thuộc:** Prometheus (Bài 03) phải đang chạy để cổng phân tích truy vấn.

**Thời lượng:** ~150 phút (trải buổi 6 + đầu buổi 7) · **Yêu cầu:** `kubectl` cluster-admin, `helm`, một Git repo (GitHub/GitLab nội bộ), `kubectl-argo-rollouts` plugin.

> 🧑‍🏫 **Nhịp độ (giảng viên):** BT4 (auto-rollback) là điểm nhấn — chừa đủ thời gian. Nếu thiếu Git ngoài, dùng Gitea nội bộ. (Nâng cao) multi-cluster để lấp nếu lớp nhanh.

---

## 1. BT1 — Tổ chức cấu hình với Kustomize

Tạo repo `smartapp-gitops` với cấu trúc:
```
base/
  deployment.yaml      # web = podinfo
  service.yaml
  kustomization.yaml
overlays/
  staging/kustomization.yaml      # replicas 2, tag 6.14.0
  production/kustomization.yaml   # replicas 5, tag 6.14.0
```
`base/kustomization.yaml`:
```yaml
resources: [deployment.yaml, service.yaml]
```
`overlays/production/kustomization.yaml`:
```yaml
resources: [../../base]
namespace: smartapp
replicas:
- { name: web, count: 5 }
images:
- { name: web, newName: stefanprodan/podinfo, newTag: "6.14.0" }
```
Kiểm tra render: `kubectl kustomize overlays/production`
**Kết quả mong đợi:** manifest production có 5 replica; staging có 2. Commit & push lên Git.

---

## 2. BT2 — ArgoCD đồng bộ lên staging

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
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
    repoURL: https://git.example/smartapp-gitops
    path: overlays/staging
    targetRevision: main
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
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata: { name: success-rate, namespace: smartapp }
spec:
  metrics:
  - name: success-rate
    interval: 1m
    successCondition: "result >= 0.95"
    failureLimit: 1
    provider:
      prometheus:
        # ⚠️ Địa chỉ này TÙY CỤM (tên Service của Prometheus). Xác minh trên cụm của bạn:
        #   kubectl -n monitoring get svc | grep -i prometheus
        address: http://kps-kube-prometheus-stack-prometheus.monitoring:9090
        query: |
          sum(rate(http_requests_total{app="web",status!~"5.."}[2m]))
            / sum(rate(http_requests_total{app="web"}[2m]))
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
# (a) Đơn giản nhất: reset về baseline (nhớ cài lại Prometheus trước Bài 07 nếu cần)
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

### Checklist hoàn thành
- [ ] smartapp có base + overlay staging/production (Kustomize) trong Git.
- [ ] ArgoCD đồng bộ staging; self-heal hoàn tác drift.
- [ ] Argo Rollouts canary promote khi success-rate ≥ 95%.
- [ ] Bản lỗi bị auto-rollback nhờ cổng phân tích Prometheus.
