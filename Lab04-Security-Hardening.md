# Lab 04 — Bảo mật hệ thống Kubernetes (Security Hardening)

> Hệ thống xuyên suốt: **smartapp** (web = podinfo). Lab này áp phòng thủ nhiều lớp lên smartapp và dựng một tenant namespace an toàn.

**Thời lượng:** ~150 phút · **Yêu cầu:** `kubectl` cluster-admin, `helm`, cụm có Internet; có namespace `smartapp` + deployment `web` từ Bài 01.

---

## 1. BT1 — Hardening pod `web` (securityContext)

Trước tiên xem podinfo hiện chạy thế nào:
```bash
kubectl -n smartapp get pod -l app=web -o jsonpath='{.items[0].spec.containers[0].securityContext}{"\n"}'
```
Áp securityContext "mẫu vàng":
```bash
kubectl -n smartapp patch deploy web -p '
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 65532
        seccompProfile: { type: RuntimeDefault }
      containers:
      - name: podinfo
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities: { drop: ["ALL"] }
'
kubectl -n smartapp rollout status deploy/web
```
**Kết quả mong đợi:** pod chạy lại thành công với non-root + read-only fs.

> ⚠️ Nếu pod `CrashLoopBackOff` vì `readOnlyRootFilesystem`: ứng dụng cần ghi tạm. Khắc phục đúng cách là mount `emptyDir` vào thư mục cần ghi (vd `/tmp`, `/data`) thay vì tắt read-only:
```bash
kubectl -n smartapp patch deploy web --type=json -p '[
 {"op":"add","path":"/spec/template/spec/volumes","value":[{"name":"tmp","emptyDir":{}}]},
 {"op":"add","path":"/spec/template/spec/containers/0/volumeMounts","value":[{"name":"tmp","mountPath":"/tmp"}]}
]'
```
**Câu hỏi suy ngẫm:** vì sao read-only root filesystem lại tăng bảo mật? *(Gợi ý: kẻ tấn công không ghi được binary/độc hại vào container.)*

---

## 2. BT2 — Thực thi chính sách bằng Kyverno

```bash
kubectl apply --server-side -f https://github.com/kyverno/kyverno/releases/download/v1.16.2/install.yaml
```
Áp hai policy (enforce):

```bash
kubectl apply -f - <<'EOF'
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata: { name: require-non-root }
spec:
  background: false
  rules:
  - name: check-non-root
    match: { any: [{ resources: { kinds: [Pod], namespaces: [smartapp] } }] }
    validate:
      failureAction: Enforce
      message: "Container phải chạy non-root."
      pattern: { spec: { securityContext: { runAsNonRoot: "true" } } }
---
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata: { name: disallow-privileged }
spec:
  rules:
  - name: no-privileged
    match: { any: [{ resources: { kinds: [Pod] } }] }
    validate:
      failureAction: Enforce
      message: "Không cho phép privileged container."
      pattern:
        spec:
          =(containers):
          - =(securityContext): { =(privileged): "false" }
EOF
```
Kiểm thử bị chặn:
```bash
# Privileged -> bị từ chối
kubectl -n smartapp run bad --image=stefanprodan/podinfo:6.14.0 --privileged
# Thiếu runAsNonRoot -> bị từ chối
kubectl -n smartapp run bad2 --image=stefanprodan/podinfo:6.14.0
```
**Kết quả mong đợi:** cả hai lệnh bị Kyverno từ chối kèm message. Pod `web` đã hardened ở BT1 vẫn chạy.

### (Nâng cao)
- So sánh: thay vì Kyverno, bật **Pod Security Admission** cho namespace: `kubectl label ns smartapp pod-security.kubernetes.io/enforce=restricted`. Quan sát pod chưa hardened bị chặn.

---

## 3. BT3 — Scan: kube-bench & Trivy

CIS benchmark với kube-bench:
```bash
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
kubectl wait --for=condition=complete job/kube-bench --timeout=120s
kubectl logs job/kube-bench | grep -E '\[FAIL\]|\[WARN\]' | head -20
```
**Kết quả mong đợi:** danh sách phát hiện FAIL/WARN theo CIS; chọn 1–2 mục và thảo luận cách khắc phục.

```bash
## install trivy nếu chưa có
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sudo sh -s -- -b /usr/local/bin
```

Quét image podinfo với Trivy:
```bash
# Bản CLI nhanh (nếu có trivy)
trivy image stefanprodan/podinfo:6.14.0 --severity HIGH,CRITICAL
# hoặc quét cấu hình + secret
trivy image --scanners vuln,secret,misconfig stefanprodan/podinfo:6.14.0
```
**Kết quả mong đợi:** báo cáo CVE theo mức; thảo luận: vá bằng cách nâng base image / đổi tag.

### (Nâng cao)
- Cài **Trivy-Operator** để quét liên tục image đang chạy trong cụm và sinh `VulnerabilityReport` CR.

---

## 4. BT4 — Secrets (Vault + ESO) & Tenant isolation

### 4a. Vault (dev) + External Secrets Operator
```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install vault hashicorp/vault -n vault --create-namespace \
  --set "server.dev.enabled=true"
helm repo add external-secrets https://charts.external-secrets.io
helm install eso external-secrets/external-secrets -n external-secrets --create-namespace
```
Tạo secret trong Vault và đồng bộ về K8s:
```bash
kubectl -n vault exec -it vault-0 -- vault kv put secret/smartapp/web api_key="s3cr3t-from-vault"

# (Cấu hình SecretStore trỏ tới Vault — xem README chart)
kubectl -n smartapp create secret generic vault-token \
  --from-literal=token=root

kubectl apply -f - <<'EOF'
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: smartapp
spec:
  provider:
    vault:
      server: "http://vault.vault.svc:8200"
      path: "secret"          # KV mount
      version: "v2"           # dev mode = KV v2
      auth:
        tokenSecretRef:
          name: vault-token
          key: token
EOF

# Tạo external secret -> sync thành k8s secret

kubectl apply -f - <<'EOF'
apiVersion: external-secrets.io/v1     # v1 đã GA; v1beta1 cũ vẫn chạy nhưng nên dùng v1
kind: ExternalSecret
metadata: { name: web-secret, namespace: smartapp }
spec:
  refreshInterval: 1m
  secretStoreRef: { name: vault-backend, kind: SecretStore }
  target: { name: web-secret }
  data:
  - secretKey: api_key
    remoteRef: { key: secret/smartapp/web, property: api_key }
EOF
kubectl -n smartapp get secret web-secret -o jsonpath='{.data.api_key}' | base64 -d; echo
```
**Kết quả mong đợi:** một `Secret` K8s `web-secret` được ESO tự tạo từ Vault; giá trị khớp `s3cr3t-from-vault`.

> 💡 Điểm dạy: secret KHÔNG nằm trong Git/manifest — nguồn thật ở Vault, ESO đồng bộ. ESO chính là một Operator (Bài 02).

### 4b. Tenant namespace an toàn
```bash
kubectl create namespace tenant-a
kubectl label ns tenant-a pod-security.kubernetes.io/enforce=restricted

kubectl apply -f - <<'EOF'
apiVersion: v1
kind: ResourceQuota
metadata: { name: quota, namespace: tenant-a }
spec:
  hard: { requests.cpu: "4", requests.memory: 8Gi, pods: "20" }
---
apiVersion: v1
kind: LimitRange
metadata: { name: defaults, namespace: tenant-a }
spec:
  limits:
  - type: Container
    default: { cpu: 500m, memory: 256Mi }
    defaultRequest: { cpu: 100m, memory: 128Mi }
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: default-deny, namespace: tenant-a }
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
EOF
kubectl -n tenant-a describe resourcequota quota
```
**Kết quả mong đợi:** namespace `tenant-a` có trần tài nguyên, mặc định container, default-deny mạng, và PSA restricted. Thử tạo pod vượt quota hoặc privileged để thấy bị chặn.

**Câu hỏi suy ngẫm:** vì sao "chỉ tạo namespace" chưa đủ để cô lập một tenant? *(Gợi ý: cần thêm lan can tài nguyên, mạng, và workload.)*

---

## 5. (Nâng cao — nếu còn thời gian)

- **Custom seccomp:** sinh profile từ `strace`/audit của podinfo, áp `localhostProfile` thay cho RuntimeDefault.
- **Cosign verify tại admission:** ký một image và viết Kyverno `verifyImages` chỉ cho chạy image có chữ ký.
- **Audit logging:** bật audit policy ghi các API call nhạy cảm (exec, secret read) và cảnh báo.
- **HNC:** cài Hierarchical Namespace Controller, tạo namespace con của `tenant-a` kế thừa quota/policy.

---

## 6. Cleanup

> ⚠️ **Bắt buộc trước bài sau:** hai ClusterPolicy đang enforce trên `smartapp` sẽ **chặn hầu hết pod demo** của Bài 05–08 (web-v2, errgen, oom/stress, filler, k6, busybox…). Pod `web` đã hardened thì giữ nguyên.

```bash
kubectl delete clusterpolicy require-non-root disallow-privileged --ignore-not-found
# Nếu đã thử PSA ở phần (Nâng cao) — gỡ nhãn enforce:
kubectl label ns smartapp pod-security.kubernetes.io/enforce- 2>/dev/null || true
# Giữ lại: web đã hardened, Vault/ESO, tenant-a (không ảnh hưởng bài sau)
```
*(Kỹ năng policy sẽ được kiểm tra lại trong challenge — setup04 tự gài policy riêng của nó.)*

---

## 7. Tổng kết & nối sang Bài 05

- Bạn đã áp phòng thủ nhiều lớp: hardening pod, enforce policy (Kyverno/PSA), quét (kube-bench/Trivy), secrets ngoài cụm (Vault/ESO), và tenant cô lập.
- Điểm cốt lõi: chuyển bảo mật từ "tự giác" sang "được thực thi tự động".
- **Bài 05:** mạng nâng cao & service mesh — default-deny NetworkPolicy bạn vừa tạo sẽ được nâng lên L7 với Cilium, cùng mTLS giữa các service.
