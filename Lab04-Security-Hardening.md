# Lab 04 — Bảo mật hệ thống Kubernetes (Security Hardening)

> Hệ thống xuyên suốt: **smartapp** (web = podinfo). Lab này áp phòng thủ nhiều lớp lên smartapp và dựng một tenant namespace an toàn.

**Thời lượng:** ~60 phút · **Yêu cầu:** `kubectl` cluster-admin, `helm`, cụm có Internet; có namespace `smartapp` + deployment `web` từ Bài 01.

---

## 1. BT1 — Hardening pod `web` (securityContext)

Trước tiên xem podinfo hiện chạy thế nào:
```bash
kubectl -n smartapp get pod -l app=web -o jsonpath='{.items[0].spec.containers[0].securityContext}{"\n"}'
```
Áp securityContext "Golden template":
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

> ⚠️ Nếu pod `CrashLoopBackOff` vì `readOnlyRootFilesystem`: ứng dụng cần ghi file tạm. Khắc phục đúng cách là mount `emptyDir` vào thư mục cần ghi (vd `/tmp`, `/data`) thay vì tắt read-only:
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

> 💡 secret KHÔNG nằm trong Git/manifest — nguồn thật ở Vault, ESO đồng bộ. ESO chính là một Operator (Bài 02).

### 4b. Ký ảnh & xác minh tại admission (Cosign + Kyverno `verifyImages`)

Ý tưởng: **chỉ cho chạy image ĐÃ KÝ** bởi identity tin cậy. Dùng **key-based** (khóa cục bộ) — hoạt động **offline/air-gap**, KHÔNG cần Fulcio/Rekor. **Private key** ở lại host ký (CI/giảng viên); chỉ **public key** vào cluster để xác minh. (Nếu cụm có Internet tới Sigstore, xem biến thể **keyless** ở cuối mục.)

> 💡 Vì sao tách vai trò? Người *ký* (giữ private key) khác người *dùng* (cluster chỉ có public key). Cluster verify được nhưng không tự ký → workload bị chiếm cũng không qua được policy.

**Cài cosign + crane nếu chưa có:**
```bash
curl -fsSL -o cosign https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64
sudo install -m 0755 cosign /usr/local/bin/cosign && rm -f cosign

# Fetch the latest version tag name
VERSION=$(curl -s "https://api.github.com/repos/google/go-containerregistry/releases/latest" | jq -r '.tag_name')

# Download the unified tarball archive (Notice the prefix 'go-containerregistry_')
curl -sL "https://github.com/google/go-containerregistry/releases/download/${VERSION}/go-containerregistry_Linux_x86_64.tar.gz" | sudo tar -zxvf - -C /usr/local/bin/ crane
```

**0) Ký 1 lần (giảng viên / CI thực hiện).** Tạo keypair rồi copy podinfo vào registry cụm PULL được và ký (chữ ký đẩy vào chính registry). `$REGISTRY` = registry cụm pull được & host này push được:
```bash
export REGISTRY=docker.io/jeffvn
export COSIGN_PASSWORD=""                       # key không mật khẩu cho lab
cosign generate-key-pair                        # -> cosign.key (PRIVATE, giữ lại), cosign.pub (PUBLIC, vào cluster)
crane copy ghcr.io/stefanprodan/podinfo:6.14.0 $REGISTRY/podinfo:6.14.0-signed

# ⚠ Ký bằng cosign 2.x — BẮT BUỘC để có định dạng chữ ký "legacy" (tag sha256-<digest>.sig)
#   mà Kyverno đọc được. cosign 3.x với --signing-config ép "new bundle format" → Kyverno
#   báo 'no signatures found'; mà 3.x lại không cho legacy + offline cùng lúc. Nên dùng 2.x để ký.
#   (Cài cosign2 song song, không đụng cosign 3.x hiện có.)
COSIGN2_VERSION=v2.4.3
curl -fsSL -o cosign2 https://github.com/sigstore/cosign/releases/download/${COSIGN2_VERSION}/cosign-linux-amd64
sudo install -m 0755 cosign2 /usr/local/bin/cosign2 && rm -f cosign2

# Ký, KHÔNG upload transparency log (offline), định dạng legacy:
cosign2 sign --key cosign.key --tlog-upload=false --yes $REGISTRY/podinfo:6.14.0-signed
crane ls $REGISTRY/podinfo | grep sig            # kiểm tra có tag sha256-....sig (định dạng Kyverno đọc)

# Kiểm chứng offline (không gọi Rekor):
cosign2 verify --key cosign.pub --insecure-ignore-tlog=true $REGISTRY/podinfo:6.14.0-signed
```
> 🛠️ For trainer: Helper `./graders/prep04-keybased.sh` làm trọn bước này (tạo keypair, copy+ký image) và export `REGISTRY`, `SIGNED_IMAGE`, `COSIGN_PUBKEY_FILE` vào `~/.bashrc` để các lệnh 4b ở terminal mới tự có. *(Chỉ là demo của bài hướng dẫn — challenge KHÔNG chấm ký ảnh; `setup04.sh`/`grade04.sh` không liên quan tới ký ảnh.)*

**1) Bắt buộc tại admission bằng Kyverno** (Audit trước để quan sát, rồi Enforce). Dán nội dung `cosign.pub` vào `publicKeys`:
```bash
kubectl apply -f - <<'EOF'
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata: { name: verify-image-signatures }
spec:
  validationFailureAction: Enforce   # sau khi log Audit sạch: đổi 'Enforce' VÀ đặt mutateDigest: true
  webhookTimeoutSeconds: 30
  background: false
  rules:
  - name: require-signed
    match: { any: [{ resources: { kinds: [Pod], namespaces: [smartapp] } }] }
    verifyImages:
    - imageReferences: ["*"]
      required: true
      mutateDigest: true         # ⚠ với Audit PHẢI là false; khi chuyển Enforce mới đặt true (ghim tag->digest)
      attestors:
      - entries:
        - keys:
            publicKeys: |-        # nội dung cosign.pub
              -----BEGIN PUBLIC KEY-----
              MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEBuP8+j/jcXCaM5Y6DkEnzHBaVtbf
              vZEzLfvJWhpiDKHm6YRJ6d0U9xRilfxy8q+rOopwjQocIrTl4vJLir+7yQ==
              -----END PUBLIC KEY-----
            rekor: { ignoreTlog: true }   # offline: bỏ qua transparency log
            ctlog: { ignoreSCT: true }
EOF
```

**2) Thử nghiệm** (pod cũng phải qua `require-non-root` ở BT2 → thêm securityContext):
```bash
# Image ĐÃ KÝ + non-root -> ĐƯỢC (khi Enforce)
kubectl -n smartapp run signed --image=$REGISTRY/podinfo:6.14.0-signed --restart=Never \
  --overrides='{"spec":{"securityContext":{"runAsNonRoot":true,"runAsUser":65532,"seccompProfile":{"type":"RuntimeDefault"}}}}'
# Image CHƯA KÝ -> BỊ TỪ CHỐI bởi verify-image-signatures
kubectl -n smartapp run unsigned --image=busybox --restart=Never --overrides='{"spec":{"securityContext":{"runAsNonRoot":true,"runAsUser":65532,"seccompProfile":{"type":"RuntimeDefault"}}}}'  --command -- sleep 30
```

**Kết quả mong đợi:** khi `Enforce`, pod ảnh chưa ký bị admission từ chối kèm message; pod ảnh đã ký (và non-root) lên bình thường, image được ghim sang `@sha256:...`.

> ⚠️ **Dọn dẹp:** `kubectl delete clusterpolicy verify-image-signatures` trước Bài 05 (nếu không sẽ chặn image chưa ký của bài sau).

**Biến thể keyless (CHỈ khi cụm có Internet tới Sigstore).** Không quản lý khóa; xác minh chữ ký OIDC publisher đã ký sẵn (`ghcr.io/stefanprodan/podinfo`). Thay `attestors` bằng:
```yaml
      attestors:
      - entries:
        - keyless:
            subject: "https://github.com/stefanprodan/*"
            issuer: "https://token.actions.githubusercontent.com"
            rekor: { url: "https://rekor.sigstore.dev" }
```
> ⚠️ Keyless cần webhook Kyverno gọi được **registry + Fulcio/Rekor** (`tuf-repo-cdn.sigstore.dev`). Nếu thấy lỗi `failed to get roots from fulcio` / `tls: unrecognized name` → cụm bị chặn egress → dùng key-based ở trên.

> 🩺 **Troubleshooting — Kyverno báo `tls: unrecognized name` khi kéo chữ ký (dù node vẫn pull image được):**
> Thường KHÔNG phải chặn egress mà là **DNS**. Node phân giải `ghcr.io`/`registry-1.docker.io` đúng, nhưng POD lại phân giải sai do `search ... lab.com` + `ndots:5` trong `/etc/resolv.conf`: tên có ít hơn 5 dấu chấm bị thử qua search-domain trước (vd `ghcr.io.lab.com`), và nếu DNS homelab trả **wildcard `*.lab.com`** thì pod nối tới IP sai → server đó từ chối SNI (`unrecognized name`).
> Chẩn đoán nhanh: `kubectl -n kyverno run t --rm -it --image=curlimages/curl --restart=Never -- curl -sv https://ghcr.io/v2/` — xem dòng `Trying <IP>` có khác `getent hosts ghcr.io` trên node không.
> - **Khắc phục nhanh (chỉ Kyverno):** hạ `ndots` để tên có dấu chấm được thử nguyên trạng trước search-domain:
>   ```bash
>   kubectl -n kyverno patch deploy kyverno-admission-controller --type merge \
>     -p '{"spec":{"template":{"spec":{"dnsConfig":{"options":[{"name":"ndots","value":"1"}]}}}}}'
>   ```
> - **Gốc rễ (cả cụm):** bỏ wildcard `*.lab.com` trên DNS homelab để `<tên>.lab.com` không tồn tại trả về **NXDOMAIN** (giữ A record cho host thật). Khi `dig +short doesnotexist.lab.com` rỗng là xong.

### 4c. Tenant namespace an toàn
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
**Thử các pod BỊ CHẶN (mỗi lệnh minh hoạ một lan can khác nhau):**
```bash
# a) PSA 'restricted' chặn pod không hardened (busybox mặc định: root, không seccomp, không drop caps)
kubectl -n tenant-a run bad --image=busybox --restart=Never --command -- sleep 3600
#   -> Error ... violates PodSecurity "restricted:latest": allowPrivilegeEscalation != false,
#      unrestricted capabilities, runAsNonRoot != true, seccompProfile ...

# b) PSA 'restricted' chặn pod privileged
kubectl -n tenant-a run priv --image=busybox --restart=Never \
  --overrides='{"spec":{"containers":[{"name":"priv","image":"busybox","securityContext":{"privileged":true}}]}}' \
  --command -- sleep 3600
#   -> Error ... privileged != true (hoặc bị chặn ngay ở restricted)

# c) ResourceQuota chặn pod xin vượt trần (requests.cpu 5 > hard 4)
kubectl -n tenant-a run toobig --image=busybox --restart=Never \
  --overrides='{"spec":{"securityContext":{"runAsNonRoot":true,"runAsUser":65532,"seccompProfile":{"type":"RuntimeDefault"}},"containers":[{"name":"toobig","image":"busybox","command":["sh","-c","sleep 3600"],"resources":{"requests":{"cpu":"5"}},"securityContext":{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]}}}]}}'
#   -> Error ... exceeded quota: quota, requested: requests.cpu=5, limited: requests.cpu=4
```

**Pod TUÂN THỦ thì được (đối chứng):**
```bash
kubectl -n tenant-a run good --image=busybox --restart=Never \
  --overrides='{"spec":{"securityContext":{"runAsNonRoot":true,"runAsUser":65532,"seccompProfile":{"type":"RuntimeDefault"}},"containers":[{"name":"good","image":"busybox","command":["sh","-c","sleep 3600"],"securityContext":{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]}}}]}}'
kubectl -n tenant-a get pod good        # Running; LimitRange tự gán requests/limits mặc định
kubectl -n tenant-a delete pod good bad priv toobig --ignore-not-found
```

**Kết quả mong đợi:** namespace `tenant-a` có trần tài nguyên, mặc định container, default-deny mạng, và PSA restricted. Pod `bad`/`priv` bị **PSA restricted** chặn, `toobig` bị **ResourceQuota** chặn, còn `good` (đã hardened) chạy được.

**Câu hỏi suy ngẫm:** vì sao "chỉ tạo namespace" chưa đủ để cô lập một tenant? *(Gợi ý: cần thêm lan can tài nguyên, mạng, và workload.)*

---

## 5. (Nâng cao — nếu còn thời gian)

- **Custom seccomp:** sinh profile từ `strace`/audit của podinfo, áp `localhostProfile` thay cho RuntimeDefault.
- **Cosign — biến thể keyless:** nếu cụm có Internet tới Sigstore, thay `keys` (mục 4b) bằng `keyless` (subject/issuer OIDC) để khỏi quản lý khóa. Xem cuối mục 4b.
- **Cosign — attestation:** thêm `verifyImages.attestations` (vd SBOM/provenance) chứ không chỉ chữ ký.
- **Audit logging:** bật audit policy ghi các API call nhạy cảm (exec, secret read) và cảnh báo.
- **HNC:** cài Hierarchical Namespace Controller, tạo namespace con của `tenant-a` kế thừa quota/policy.

---

## 6. Cleanup

> ⚠️ **Bắt buộc trước bài sau:** các ClusterPolicy đang enforce trên `smartapp` (require-non-root, disallow-privileged, và `verify-image-signatures` nếu bạn làm mục 4b) sẽ **chặn hầu hết pod demo** của Bài 05–08 (web-v2, errgen, oom/stress, filler, k6, busybox…). Pod `web` đã hardened thì giữ nguyên.

```bash
kubectl delete clusterpolicy require-non-root disallow-privileged verify-image-signatures --ignore-not-found
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
