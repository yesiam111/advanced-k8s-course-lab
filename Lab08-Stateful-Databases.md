# Lab 08 — Ứng dụng Stateful & Cơ sở dữ liệu

> Hệ thống xuyên suốt: **smartapp**. Lab này dựng và bảo vệ **postgres** của smartapp bằng CloudNativePG, snapshot và Velero.
> **Giả định:** cụm có **Longhorn** với StorageClass mặc định `longhorn` (hỗ trợ CSI snapshot — xem `provision/`).

**Thời lượng:** ~60 phút · **Yêu cầu:** `kubectl` cluster-admin, `helm`, StorageClass có hỗ trợ snapshot; (BT4) một S3 bucket hoặc MinIO nội bộ cho Velero.

---

## 1. BT2 — PostgreSQL bằng CloudNativePG (+ failover)

> ⚠️ **Phiên bản:** URL dưới ghim CNPG 1.24 (cũ). Trước buổi, kiểm tra release ổn định mới nhất tại https://github.com/cloudnative-pg/cloudnative-pg/releases và thay `release-1.24`/`cnpg-1.24.0.yaml` cho khớp (API `postgresql.cnpg.io/v1` ổn định giữa các bản).
```bash
kubectl apply --server-side -f \
  https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.30/releases/cnpg-1.30.0.yaml
kubectl -n cnpg-system rollout status deploy/cnpg-controller-manager
```
Dựng cluster postgres của smartapp:
```bash
kubectl apply -f - <<'EOF'
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata: { name: smartapp-db, namespace: smartapp }
spec:
  instances: 3
  storage:
    size: 2Gi
    storageClass: longhorn     # đổi theo SC của cụm
EOF
kubectl -n smartapp get cluster smartapp-db
kubectl -n smartapp get pods -l cnpg.io/cluster=smartapp-db -o wide
```
**Kết quả mong đợi:** 3 pod (1 primary + 2 replica); service `smartapp-db-rw` (read-write) và `smartapp-db-ro`.

Ghi dữ liệu rồi **gây failover**:
```bash
kubectl -n smartapp exec -it smartapp-db-1 -- psql -c "CREATE TABLE t(id int); INSERT INTO t VALUES (1);"
# Xóa primary hiện tại
PRIMARY=$(kubectl -n smartapp get cluster smartapp-db -o jsonpath='{.status.currentPrimary}')
kubectl -n smartapp delete pod $PRIMARY
kubectl -n smartapp get cluster smartapp-db -w   # currentPrimary đổi sang instance khác
```
**Kết quả mong đợi:** CNPG tự promote một replica thành primary; dữ liệu bảng `t` còn nguyên. Không cần can thiệp tay.

**Câu hỏi suy ngẫm:** StatefulSet thô có tự làm failover này không? *(Gợi ý: không — đó là 'bộ não' do Operator cung cấp.)*

---

## 3. BT3 — Point-in-Time Recovery (PITR)

> Cần cấu hình backup/WAL archiving (barmanObjectStore tới S3/MinIO). Rút gọn các bước chính:

```bash
# 1) Bật backup liên tục trên cluster (thêm spec.backup.barmanObjectStore trỏ S3)
# 2) Ghi dữ liệu và GHI LẠI thời điểm — exec vào PRIMARY hiện tại (sau failover BT2,
#    smartapp-db-1 có thể chỉ là replica read-only)
PRIMARY=$(kubectl -n smartapp get cluster smartapp-db -o jsonpath='{.status.currentPrimary}')
kubectl -n smartapp exec -it $PRIMARY -- psql -c "INSERT INTO t VALUES (2),(3);"
date -u +"%Y-%m-%dT%H:%M:%SZ"     # T0 — ghi nhớ mốc này
# 3) "Lỡ tay" xóa dữ liệu
kubectl -n smartapp exec -it $PRIMARY -- psql -c "DELETE FROM t;"
```
Khôi phục về thời điểm T0 bằng một Cluster mới `recovery`:
```bash
kubectl apply -f - <<'EOF'
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata: { name: smartapp-db-restored, namespace: smartapp }
spec:
  instances: 1
  storage: { size: 2Gi, storageClass: longhorn }
  bootstrap:
    recovery:
      source: smartapp-db
      recoveryTarget: { targetTime: "<T0>" }   # thay bằng mốc đã ghi
  externalClusters:
  - name: smartapp-db
    barmanObjectStore: { destinationPath: s3://backups/smartapp, ... }
EOF
kubectl -n smartapp exec -it smartapp-db-restored-1 -- psql -c "SELECT * FROM t;"
```
**Kết quả mong đợi:** cluster khôi phục có lại các dòng `1,2,3` (trạng thái tại T0, trước khi DELETE). PITR mạnh hơn ảnh chụp định kỳ vì khôi phục đúng thời điểm.

---

## 4. BT4 — Velero: backup & restore xuyên namespace

> **Object store OFF-CLUSTER (bắt buộc trước khi cài Velero).** Backup nằm trong chính cụm mà
> nó bảo vệ thì không phải backup (Bài 01). Ta dựng **MinIO 1 instance standalone** do **containerd
> quản lý trực tiếp qua nerdctl** — KHÔNG phải workload K8s — ở containerd namespace riêng `dr-store`
> (KHÔNG phải `k8s.io` của kubelet), nên `kubectl` không thấy và xoá cụm vẫn còn store.
>
> ```bash
> # TRÊN 1 node cụm (có containerd). KHÔNG cần sudo cả lệnh — script tự sudo phần chạm containerd.
> ./provision/minio-dr-host.sh up      # in ra endpoint http://<NODE_IP>:9000 + bucket 'velero'
> ./provision/minio-dr-host.sh status  # lấy lại thông tin nối Velero bất kỳ lúc nào
> # Kiểm chứng nó KHÔNG thuộc K8s:
> #   sudo ctr -n k8s.io   c ls   # pod của kubelet   |   sudo ctr -n dr-store c ls   # MinIO
> ```
> Thay `s3Url` bên dưới bằng `http://<NODE_IP>:9000` mà script in ra (KHÔNG dùng `minio.velero:9000`
> nội cụm nữa). ⚠ Off-cluster nhưng vẫn on-host: production đặt MinIO trên máy/khu vực tách rời.

```bash
# Cài Velero (S3 = MinIO OFF-CLUSTER ở trên; đổi s3Url thành http://<NODE_IP>:9000)
# ⚠ Snapshotter 'aws' KHÔNG snapshot được PV Longhorn — backup dữ liệu volume bằng
#   file-system backup (node-agent/Kopia). Muốn dùng CSI snapshot thật: thêm
#   --features=EnableCSI + VolumeSnapshotClass (longhorn-snapshot-vsc).
# (Kiểm tra bản plugin mới nhất trước buổi: github.com/vmware-tanzu/velero-plugin-for-aws/releases)
velero install \
  --provider aws --plugins velero/velero-plugin-for-aws:v1.14.2 \
  --bucket velero --secret-file ./minio-creds \
  --backup-location-config region=minio,s3ForcePathStyle=true,s3Url=http://<NODE_IP>:9000,checksumAlgorithm= \
  # ⚠ checksumAlgorithm= (rỗng) BẮT BUỘC với MinIO + plugin aws ≥1.9, nếu không BSL 'Unavailable'.
  --use-volume-snapshots=false \
  --use-node-agent --default-volumes-to-fs-backup

# Backup toàn bộ namespace smartapp (manifest + dữ liệu volume)
velero backup create smartapp-bk --include-namespaces smartapp --wait
velero backup describe smartapp-bk
```
Khôi phục sang namespace mới (mô phỏng DR):
```bash
velero restore create --from-backup smartapp-bk \
  --namespace-mappings smartapp:smartapp-dr
kubectl -n smartapp-dr get pods,pvc
kubectl -n smartapp-dr exec -it smartapp-db-1 -- psql -c "SELECT count(*) FROM t;" 2>/dev/null || true
```
**Kết quả mong đợi:** namespace `smartapp-dr` được dựng lại từ backup với dữ liệu volume; xác minh bảng `t` còn dữ liệu.

> 💡 Nguyên tắc vàng (Bài 01): chỉ tin backup **đã từng restore thành công**. Đưa diễn tập restore vào lịch định kỳ (Ops-Checklist mục D).

---

## 5. (Nâng cao — nếu còn thời gian)

- **VolumeSnapshot:** tạo `VolumeSnapshot` cho một PVC của postgres, rồi tạo PVC mới từ snapshot và mount thử.
- **Redis Operator:** dựng Redis (cache/queue smartapp) bằng một Redis Operator với sentinel/failover.
- **Topology-aware:** đặt StorageClass `volumeBindingMode: WaitForFirstConsumer`; tạo PVC và xem PV chỉ được cấp khi pod được lập lịch (đúng zone).
- **Velero schedule:** `velero schedule create nightly --schedule="0 2 * * *" --include-namespaces smartapp`.

---

## 6. Tổng kết & nối sang Bài 09

- Bạn đã vận hành postgres của smartapp bằng Operator (failover tự động), thực hiện PITR, và backup/restore xuyên namespace với Velero.
- Dữ liệu stateful giờ được bảo vệ ở mức production: persistent + snapshot + backup off-site + diễn tập restore.
- **Bài 09:** SRE, xử lý sự cố & Capstone — ghép toàn bộ khóa (gồm khôi phục dữ liệu hôm nay) thành kỹ năng vận hành và một bài tổng hợp.
