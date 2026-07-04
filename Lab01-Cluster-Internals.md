# Lab 01 — Cơ chế hoạt động nội bộ & Vận hành vòng đời cụm

> Hệ thống xuyên suốt: **smartapp** — `web` (podinfo) + `postgres` (CloudNativePG) + `redis`, chạy trên cụm đã dựng sẵn (kubeadm + Cilium + Longhorn).
> Lab này: nhìn Kubernetes từ *bên trong* — soi container ở cấp node, phân tích scheduler, chèn một admission webhook, và diễn tập khôi phục etcd.

**Thời lượng:** ~60 phút · **Yêu cầu:** quyền `kubectl` cluster-admin; SSH + sudo vào ít nhất 1 control-plane node và 1 worker node; `crictl`, `etcdctl`, `openssl` có sẵn trên node.

---

## 0. Chuẩn bị — triển khai smartapp (nếu chưa có)

```bash
kubectl create namespace smartapp 2>/dev/null || true

# web = podinfo (image chính thức, có /metrics, /healthz)
kubectl -n smartapp create deployment web --image=stefanprodan/podinfo:6.7.0 --replicas=2
kubectl -n smartapp expose deployment web --port=9898 --target-port=9898

# redis (image chính thức)
kubectl -n smartapp create deployment redis --image=redis:7-alpine
kubectl -n smartapp expose deployment redis --port=6379

kubectl -n smartapp get pods -o wide
```
**Kết quả mong đợi:** các pod `web-*` và `redis-*` ở trạng thái `Running`, cột `NODE` cho biết chúng nằm trên node nào (ghi nhớ để dùng ở BT1).

> *Postgres/CloudNativePG sẽ được dựng đầy đủ ở Bài 08; bài này chỉ cần web + redis.*

---

## 1. BT1 — Soi container ở cấp node bằng `crictl`

Mục tiêu: thấy cùng một container qua ba tầng (kubectl → crictl → ctr).

Tìm node đang chạy một pod `web`:
```bash
NODE=$(kubectl -n smartapp get pod -l app=web -o jsonpath='{.items[0].spec.nodeName}')
POD=$(kubectl -n smartapp get pod -l app=web -o jsonpath='{.items[0].metadata.name}')
echo "pod $POD đang trên node $NODE"
```

SSH vào `$NODE`, rồi:
```bash
# Container thật theo CRI
sudo crictl ps --name podinfo
# Soi chi tiết (chú ý: PID, image digest OCI, mounts)
CID=$(sudo crictl ps --name podinfo -q | head -1)
sudo crictl inspect $CID | grep -E '"pid"|"image"|"runtimeType"' | head
# Tầng containerd
sudo ctr -n k8s.io containers ls | grep $(echo $CID | cut -c1-12) || true
```
**Kết quả mong đợi:** `crictl ps` liệt kê container `web`; `crictl inspect` cho thấy PID thật trên node và image digest (content-addressable). Đây là góc nhìn bạn vẫn có *kể cả khi API server gặp sự cố*.

**Câu hỏi suy ngẫm:** vì sao trên node không còn lệnh `docker ps`? *(Gợi ý: dockershim đã bị gỡ; kubelet nói chuyện với containerd qua CRI.)*

### (Nâng cao)
- Dừng tạm kubelet (`sudo systemctl stop kubelet`) và xác nhận container `web` **vẫn chạy** (crictl vẫn thấy) — chứng minh runtime độc lập với kubelet. Nhớ `systemctl start kubelet` lại.

---

## 2. BT2 — Phân tích quyết định của Scheduler

Mục tiêu: thấy pha **Filter** loại node, đọc lý do qua Events.

Tạo một pod cố tình không xếp được:
```bash
kubectl -n smartapp run pending-demo --image=stefanprodan/podinfo:6.7.0 \
  --overrides='{"apiVersion":"v1","spec":{"nodeSelector":{"disktype":"nvme-khong-ton-tai"}}}'
kubectl -n smartapp get pod pending-demo -o wide
kubectl -n smartapp describe pod pending-demo | sed -n '/Events/,$p'
```
**Kết quả mong đợi:** pod ở `Pending`; Events có dòng kiểu `FailedScheduling ... 0/N nodes are available: N node(s) didn't match Pod's node affinity/selector` — chính là pha Filter loại hết node.

Sửa cho chạy được (gán nhãn cho một node):
```bash
SOME_NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
kubectl label node $SOME_NODE disktype=nvme-khong-ton-tai --overwrite
kubectl -n smartapp get pod pending-demo -o wide   # giờ sẽ Running
```
Dọn dẹp: `kubectl -n smartapp delete pod pending-demo; kubectl label node $SOME_NODE disktype-`

**Câu hỏi suy ngẫm:** nếu nhiều node cùng qua Filter, cái gì quyết định node cuối cùng? *(Gợi ý: pha Score.)*

### (Nâng cao)
- Tạo 4 replica của `web` với `podAntiAffinity` theo `kubernetes.io/hostname` và quan sát scheduler trải chúng ra các node khác nhau (báo trước nội dung Bài 07).

---

## 3. BT3 — Đăng ký một Validating Admission Webhook

Mục tiêu: chặn mọi pod trong `smartapp` thiếu label `team`. Đây là cốt lõi của policy-as-code (đào sâu ở Bài 04).

> Webhook là một HTTPS service. Để gọn, ta dùng một webhook server có sẵn dạng "deny nếu thiếu label". Nếu môi trường có Internet, cách nhanh nhất là dùng **Kyverno** ở chế độ enforce; ở đây trình bày bản thủ công để hiểu cơ chế.

### 3a. Cách nhanh (khuyến nghị nếu có Kyverno): policy tương đương
> ⚠️ Kyverno ≥ 1.13: `spec.validationFailureAction` đã deprecated → chuyển thành `validate.failureAction` trong mỗi rule (xem ghi chú Lab04 BT2).

Install Kyverno
```bash
kubectl create -f https://github.com/kyverno/kyverno/releases/download/v1.16.2/install.yaml
```
 
```bash
kubectl create -f - <<'EOF'
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-team-label
spec:
  validationFailureAction: Enforce
  rules:
  - name: check-team
    match:
      any:
      - resources:
          kinds: [Pod]
          namespaces: [smartapp]
    validate:
      message: "Pod phải có label 'team'."
      pattern:
        metadata:
          labels:
            team: "?*"
EOF
```
Kiểm thử:
```bash
# Thiếu label -> bị từ chối
kubectl -n smartapp run no-label --image=stefanprodan/podinfo:6.7.0
# Có label -> được chấp nhận
kubectl -n smartapp run with-label --image=stefanprodan/podinfo:6.7.0 --labels=team=platform
```
**Kết quả mong đợi:** lệnh đầu bị từ chối với message "Pod phải có label 'team'"; lệnh sau tạo thành công.

### 3b. Hiểu cấu hình webhook thật
Xem một `ValidatingWebhookConfiguration` do Kyverno tạo và chú ý các trường quan trọng:
```bash
kubectl get validatingwebhookconfigurations
kubectl get validatingwebhookconfigurations kyverno-resource-validating-webhook-cfg -o yaml \
  | grep -E 'failurePolicy|namespaceSelector|matchPolicy|caBundle' | head
```
**Điểm cần quan sát:**
- `failurePolicy: Fail` (gác cổng chặt) vs `Ignore` — đánh đổi an toàn/sẵn sàng.
- `namespaceSelector` loại trừ namespace hệ thống — **bắt buộc** để không "khóa cứng" cụm.

> ⚠️ **Bài học vận hành:** một webhook `failurePolicy=Fail` mà service chết sẽ khiến *không tạo được pod nào*, kể cả pod hệ thống. Luôn loại trừ `kube-system` qua `namespaceSelector`.

Dọn dẹp: `kubectl delete clusterpolicy require-team-label; kubectl -n smartapp delete pod with-label --ignore-not-found`

### (Nâng cao)
- Triển khai một webhook server thủ công (Go/Python) trả `allowed:false` khi thiếu label, tự ký TLS bằng `openssl`, nhồi `caBundle`, và đăng ký `ValidatingWebhookConfiguration`. So sánh độ phức tạp với Kyverno.

---

## 4. BT4 — Diễn tập khôi phục etcd (snapshot → mất quorum → restore)

> ⚠️ **CHỈ trên cụm sandbox.** Thao tác này cố ý làm hỏng etcd.

Trên control-plane node (cụm kubeadm, etcd dạng static pod):
> 💡 etcd ≥ 3.5: `snapshot status/restore` chuyển sang `etcdutl` (etcdctl vẫn chạy nhưng in cảnh báo deprecated). Nếu node có `etcdutl`, dùng nó cho status/restore.
```bash
# 4.1 Snapshot
sudo mkdir -p /var/lib/etcd-backup
SNAP=/var/lib/etcd-backup/snap-$(date +%s).db
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save $SNAP
sudo ETCDCTL_API=3 etcdutl snapshot status $SNAP -w table
```
**Kết quả mong đợi:** file snapshot được tạo; `snapshot status` in ra số key, revision — bằng chứng snapshot hợp lệ.

```bash
# 4.2 Tạo một "dấu vết" để xác minh sau restore
kubectl -n smartapp annotate deployment web canary-marker="truoc-restore"

# 4.3 Gây sự cố (single-node etcd): dừng etcd để mô phỏng mất etcd
sudo mv /etc/kubernetes/manifests/etcd.yaml /tmp/etcd.yaml
# -> apiserver sẽ mất kết nối etcd; kubectl bắt đầu treo/timeout
```

```bash
# 4.4 Restore từ snapshot vào data-dir mới
sudo ETCDCTL_API=3 etcdutl snapshot restore $SNAP \
  --data-dir /var/lib/etcd-restored         
# Trỏ static pod etcd vào data-dir mới
sudo sed 's#/var/lib/etcd#/var/lib/etcd-restored#g' /tmp/etcd.yaml | sudo tee /etc/kubernetes/manifests/etcd.yaml >/dev/null
```

```bash
# 4.5 Xác minh
sleep 20
kubectl -n smartapp get deployment web -o jsonpath='{.metadata.annotations.canary-marker}{"\n"}'
kubectl get nodes
```
**Kết quả mong đợi:** cụm hồi phục; annotation `canary-marker=truoc-restore` còn nguyên → chứng minh dữ liệu được khôi phục từ snapshot.

**Câu hỏi suy ngẫm:** nếu bạn *chưa từng* chạy `snapshot restore` trước đây, bạn có chắc nó hoạt động lúc 3 giờ sáng không? *(Đây là lý do "snapshot chưa từng restore = không có backup".)*

---

## 5. Tổng kết & nối sang Bài 02

- Bạn đã thấy K8s từ bên trong: runtime/CRI ở node, lõi etcd/apiserver/scheduler/controller, và admission như cổng kiểm soát.
- Bạn đã thực hành ba kỹ năng vận hành sống còn: đọc quyết định scheduler, chèn policy qua admission, và khôi phục etcd.
- **Bài 02:** thay vì chỉ *hiểu* reconciliation loop, ta sẽ *tự viết* một Operator quản lý vòng đời một thành phần của smartapp.
