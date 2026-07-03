# Lab 07 — Tự động mở rộng quy mô & Hiệu suất

> Hệ thống xuyên suốt: **smartapp** (web = podinfo, redis). Lab này trải nghiệm scaling, chi phí và lập lịch nâng cao.
> **Phụ thuộc:** Prometheus (Bài 03) đang chạy để xem metric khi scaling.

**Thời lượng:** ~120 phút · **Yêu cầu:** `kubectl` cluster-admin, `helm`, `k6` (hoặc chạy k6 trong pod); SSH + sudo vào 1 node cho BT1.

> 🧑‍🏫 **Nhịp độ (giảng viên):** cài KEDA/OpenCost nhanh → bài dễ **xong sớm**. Dồn cho BT4 (lập lịch) và (Nâng cao). Cluster Autoscaler chỉ hoạt động trên cụm cloud/có node group — nếu cụm cố định, demo bằng cách quan sát pod `Pending`.

---

## 1. BT1 — cgroups & OOMKilled

Xem requests/limits ánh xạ xuống cgroup v2 trên node:
```bash
# Đặt limit nhỏ cho web để dễ quan sát
kubectl -n smartapp set resources deploy/web --requests=cpu=100m,memory=64Mi --limits=cpu=200m,memory=128Mi
kubectl -n smartapp rollout status deploy/web

NODE=$(kubectl -n smartapp get pod -l app=web -o jsonpath='{.items[0].spec.nodeName}')
POD=$(kubectl -n smartapp get pod -l app=web -o jsonpath='{.items[0].metadata.name}')
# SSH vào $NODE:
sudo crictl inspect $(sudo crictl ps --name web -q | head -1) | grep -iE 'memory|cpu' | head
# hoặc đọc trực tiếp cgroup v2 (đường dẫn tuỳ runtime):
sudo find /sys/fs/cgroup -name 'memory.max' -path '*kubepods*' 2>/dev/null | head -3
```
**Kết quả mong đợi:** thấy `memory.max` ≈ 128Mi tương ứng limit đã đặt.

Gây OOMKilled có chủ đích:
```bash
# podinfo có thể tiêu RAM; hoặc dùng container stress
# (kubectl run đã bỏ cờ --limits/--requests từ v1.24 → dùng --overrides)
kubectl -n smartapp run oom --image=polinux/stress --restart=Never \
  --overrides='{"apiVersion":"v1","spec":{"containers":[{"name":"oom","image":"polinux/stress",
    "resources":{"limits":{"memory":"64Mi"}},
    "args":["stress","--vm","1","--vm-bytes","200M","--vm-hang","0"]}]}}'
kubectl -n smartapp get pod oom -o wide
kubectl -n smartapp describe pod oom | grep -iE 'OOMKilled|Reason|Exit Code'
```
**Kết quả mong đợi:** pod `oom` bị `OOMKilled`, exit code `137`.

**Câu hỏi suy ngẫm:** workload đặt ở QoS nào sẽ bị OOM giết đầu tiên khi node thiếu RAM? *(Gợi ý: BestEffort — không có requests/limits.)*

Dọn dẹp: `kubectl -n smartapp delete pod oom`

---

## 2. BT2 — KEDA: scale theo hàng đợi Redis (+ scale-to-zero)

```bash
helm repo add kedacore https://kedacore.github.io/charts && helm repo update
helm install keda kedacore/keda -n keda --create-namespace
```
Tạo một worker giả lập (đọc/đếm job) và ScaledObject:
```bash
kubectl -n smartapp create deployment worker --image=stefanprodan/podinfo:6.7.0
kubectl apply -f - <<'EOF'
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata: { name: worker, namespace: smartapp }
spec:
  scaleTargetRef: { name: worker }
  minReplicaCount: 0
  maxReplicaCount: 20
  pollingInterval: 5
  cooldownPeriod: 30
  triggers:
  - type: redis
    metadata:
      address: redis.smartapp:6379
      listName: jobs
      listLength: "5"
EOF
kubectl -n smartapp get deploy worker   # 0 replica khi rảnh (scale-to-zero)
```
Bơm job vào Redis và xem worker bung:
```bash
kubectl -n smartapp exec deploy/redis -- sh -c 'for i in $(seq 1 100); do redis-cli RPUSH jobs "job-$i"; done'
kubectl -n smartapp get deploy worker -w     # replica tăng theo listLength (Ctrl-C để thoát)
```
**Kết quả mong đợi:** worker scale 0 → N khi `LLEN jobs` lớn.

> ⚠️ **Lưu ý:** `worker` ở đây là **podinfo giả lập** — nó KHÔNG thực sự `LPOP` khỏi list, nên hàng đợi sẽ không tự cạn. Để minh họa **chiều scale-down**, ta chủ động làm cạn queue (mô phỏng worker đã xử lý xong):
```bash
# Cách 1: xóa nhanh toàn bộ hàng đợi
kubectl -n smartapp exec deploy/redis -- redis-cli DEL jobs
# Cách 2 (sát thực tế hơn): vòng lặp LPOP mô phỏng tiêu thụ dần
kubectl -n smartapp exec deploy/redis -- sh -c 'while [ "$(redis-cli LLEN jobs)" -gt 0 ]; do redis-cli LPOP jobs >/dev/null; done'

kubectl -n smartapp get deploy worker -w     # sau cooldownPeriod (30s) → về 0 replica
```
**Kết quả mong đợi:** khi `LLEN jobs` về 0, sau `cooldownPeriod` KEDA đưa worker về **0 replica** — scale-to-zero. *(Trong production, chính worker thật sẽ LPOP và làm cạn queue thay cho bước thủ công này.)*

**Câu hỏi suy ngẫm:** HPA chuẩn có làm được scale-to-zero không? *(Gợi ý: không; KEDA cần thiết cho việc này.)*

---

## 3. BT3 — Load test (k6) + HPA + chi phí

Tạo HPA cho web (theo CPU cho đơn giản; custom metric ở Nâng cao):
```bash
kubectl -n smartapp autoscale deploy web --cpu-percent=50 --min=2 --max=10
kubectl -n smartapp get hpa web
```
Chạy k6 tạo tải:
```bash
cat > load.js <<'EOF'
import http from 'k6/http';
export const options = { stages: [
  { duration: '1m', target: 50 },
  { duration: '2m', target: 200 },
  { duration: '1m', target: 0 },
]};
export default function () { http.get('http://web.smartapp:9898/'); }
EOF
kubectl -n smartapp run k6 --rm -it --image=grafana/k6 --restart=Never -- run - < load.js
```
Trong khi chạy, quan sát:
```bash
watch kubectl -n smartapp get hpa,pods
# (nếu cụm có Cluster Autoscaler) xem node tăng khi pod Pending:
kubectl get nodes
```
**Kết quả mong đợi:** HPA tăng replica web khi CPU vượt 50%; nếu thiếu chỗ, pod `Pending` → CA thêm node (cụm cloud).

Chi phí với OpenCost:
```bash
helm repo add opencost https://opencost.github.io/opencost-helm-chart && helm repo update
helm install opencost opencost/opencost -n opencost --create-namespace
kubectl -n opencost port-forward svc/opencost 9003:9003
# UI cho thấy chi phí theo namespace/workload
```
**Kết quả mong đợi:** thấy chi phí phân bổ theo namespace smartapp; thảo luận right-sizing.

---

## 4. BT4 — Lập lịch: topology spread + preemption

### 4a. Phân bố đều theo zone/node
```bash
kubectl -n smartapp patch deploy web --type=merge -p '
spec:
  template:
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector: { matchLabels: { app: web } }
'
kubectl -n smartapp scale deploy web --replicas=6
kubectl -n smartapp get pods -l app=web -o wide   # trải đều trên các node
```
**Kết quả mong đợi:** 6 replica phân bố đều giữa các node (lệch tối đa 1).

### 4b. PriorityClass & preemption
```bash
kubectl apply -f - <<'EOF'
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata: { name: low-priority }
value: 100
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata: { name: high-priority }
value: 1000000
EOF

# Lấp cụm bằng pod ưu tiên thấp
kubectl -n smartapp create deploy filler --image=stefanprodan/podinfo:6.7.0 --replicas=20
kubectl -n smartapp patch deploy filler --type=merge -p '{"spec":{"template":{"spec":{"priorityClassName":"low-priority"}}}}'
kubectl -n smartapp set resources deploy/filler --requests=cpu=500m

# Tạo pod ưu tiên cao đòi nhiều tài nguyên -> đẩy filler
kubectl -n smartapp run vip --image=stefanprodan/podinfo:6.7.0 \
  --overrides='{"apiVersion":"v1","spec":{"priorityClassName":"high-priority","containers":[{"name":"vip","image":"stefanprodan/podinfo:6.7.0","resources":{"requests":{"cpu":"2"}}}]}}'
kubectl -n smartapp get events | grep -i preempt
```
**Kết quả mong đợi:** khi cụm chật, pod `vip` (ưu tiên cao) khiến scheduler **preempt** (evict) pod `filler` (ưu tiên thấp) để có chỗ.

Dọn dẹp: `kubectl -n smartapp delete deploy filler; kubectl -n smartapp delete pod vip`

---

## 5. (Nâng cao — nếu còn thời gian)

- **Custom metric HPA:** cài Prometheus Adapter; scale web theo `http_requests_per_second` thay vì CPU.
- **VPA recommender:** cài VPA ở chế độ Off; đọc khuyến nghị request và so với giá trị hiện tại (right-sizing).
- **Descheduler:** cài descheduler, tạo mất cân bằng (cordon→uncordon) rồi xem nó evict pod để cân bằng lại.
- **KEDA cron scaler:** scale web theo lịch giờ cao điểm.

---

## 6. Cleanup

```bash
kubectl -n smartapp delete hpa web --ignore-not-found          # tránh HPA can thiệp Bài 08/09
kubectl -n smartapp delete scaledobject worker --ignore-not-found
kubectl -n smartapp delete deploy worker --ignore-not-found
kubectl delete priorityclass low-priority high-priority --ignore-not-found
# Giữ lại: requests/limits + topologySpreadConstraints trên web (cấu hình tốt, nên giữ).
```

---

## 7. Tổng kết & nối sang Bài 08

- Bạn đã chạm cả ba chiều scaling (HPA/VPA/CA) + KEDA, đo bằng load test, thấy chi phí, và kiểm soát vị trí pod.
- Cân bằng: đáp ứng nhanh, tiết kiệm chi phí, phân bố tin cậy.
- **Bài 08:** ứng dụng stateful & cơ sở dữ liệu — vận hành postgres của smartapp bằng Operator, snapshot và backup/restore (workload stateful khó scale ngang, nơi VPA/right-sizing đặc biệt quan trọng).

### Checklist hoàn thành
- [ ] Đọc được cgroup limit của web trên node; gây OOMKilled (exit 137) và hiểu QoS.
- [ ] KEDA scale worker 0→N→0 theo hàng đợi Redis.
- [ ] k6 tạo tải; HPA tăng/giảm replica web; xem chi phí OpenCost.
- [ ] topology spread phân bố web đều; quan sát preemption với PriorityClass.
