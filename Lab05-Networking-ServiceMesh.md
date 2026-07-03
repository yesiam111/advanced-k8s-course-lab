# Lab 05 — Mạng nâng cao & Service Mesh (sidecarless-first)

> Hệ thống xuyên suốt: **smartapp** (web = podinfo, redis). Cụm đã chạy **Cilium** làm CNI (từ Intro).
> Triết lý bài này theo đúng thị trường 2026: **mesh không-sidecar (sidecarless) là hướng đi chính.**
> Ta khai thác mesh **đã có sẵn trong stack** (Cilium), rồi học **mesh doanh nghiệp de-facto** (Istio **ambient**), và chỉ xem Linkerd như bản "đơn giản nhất" để đối chiếu.

**Thời lượng:** ~150 phút · **Yêu cầu:** `kubectl` cluster-admin, `helm`, `cilium` CLI, `istioctl`; namespace `smartapp` với `web` + `redis` từ Bài 01.

> 🧑‍🏫 **TIMEBOX (giảng viên):** BT1–BT3 (Cilium: quan sát → L7 → mTLS sidecarless) là cốt lõi & nhanh vì Cilium đã có sẵn. **BT4 (Istio ambient)** là trọng tâm "kỹ năng tuyển dụng" — cap ~50'. BT5 (Linkerd) chỉ là đối chiếu, làm khi còn giờ.
>
> 🧭 **Vì sao bố cục này:** thị trường đang dịch chuyển khỏi sidecar. Cilium (eBPF) là nền mạng de-facto cho cụm nghiêm túc và là kỹ năng *khan hiếm, lương cao*; Istio là mesh doanh nghiệp phổ biến nhất và **ambient** đã GA (bỏ sidecar). Học viên rời lớp với đúng hai thứ tổ chức đang cần.

---

## 1. BT1 — Hubble: quan sát luồng mạng

Bật Hubble (nếu cụm chưa bật):
```bash
cilium hubble enable --ui
cilium status | grep -i hubble
```
Sinh lưu lượng trong smartapp:
```bash
# HTTP tới web (podinfo có sẵn curl)
kubectl -n smartapp exec deploy/web -- sh -c 'for i in $(seq 1 20); do curl -s http://web.smartapp:9898/ >/dev/null; done' || true
# TCP tới redis bằng đúng giao thức RESP
kubectl -n smartapp exec deploy/redis -- sh -c 'for i in $(seq 1 20); do redis-cli -h redis.smartapp ping >/dev/null; done' || true
```
Quan sát:
```bash
cilium hubble ui          # service map
hubble observe --namespace smartapp --last 50
```
**Kết quả mong đợi:** service map smartapp (web, redis) + danh sách luồng kèm `FORWARDED`/`DROPPED`, port, và **identity** nguồn–đích.

**Câu hỏi suy ngẫm:** vì sao quan sát theo *identity* (nhãn pod) tốt hơn theo IP? *(IP pod đổi liên tục; identity ổn định.)*

---

## 2. BT2 — CiliumNetworkPolicy ở tầng L7

Chỉ cho phép GET tới web (chặn method khác ngay ở L7 — điều NetworkPolicy chuẩn L3/L4 không làm được):
```bash
kubectl apply -f - <<'EOF'
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata: { name: web-l7, namespace: smartapp }
spec:
  endpointSelector:
    matchLabels: { app: web }
  ingress:
  - toPorts:
    - ports: [{ port: "9898", protocol: TCP }]
      rules:
        http:
        - method: "GET"
EOF
```
Kiểm thử & xác nhận drop ở L7 qua Hubble:
```bash
kubectl -n smartapp exec deploy/web -- curl -s -o /dev/null -w "GET -> %{http_code}\n" http://web.smartapp:9898/
kubectl -n smartapp exec deploy/web -- curl -s -o /dev/null -w "POST -> %{http_code}\n" -X POST http://web.smartapp:9898/ || echo "POST bị chặn"
hubble observe --namespace smartapp --protocol http --verdict DROPPED --last 20
```
**Kết quả mong đợi:** GET `FORWARDED`, POST `DROPPED` ở tầng HTTP.
Dọn dẹp: `kubectl -n smartapp delete ciliumnetworkpolicy web-l7`

---

## 3. BT3 — Service mesh BẠN ĐÃ CÓ: Cilium sidecarless (mã hóa + danh tính)

Cilium tự nó là một service mesh **không sidecar**: mã hóa traffic và xác thực danh tính do datapath eBPF/Envoy-per-node lo, **không** chèn container vào pod (không tốn RAM/độ trễ sidecar).

### 3a. Mã hóa minh bạch toàn cụm (WireGuard)
```bash
# Bật transparent encryption (nếu Intro chưa bật)
cilium config set enable-wireguard true        # hoặc helm upgrade ... --set encryption.enabled=true,encryption.type=wireguard
kubectl -n kube-system rollout status ds/cilium
cilium encrypt status
```
Sinh traffic web↔redis rồi xác nhận đã mã hóa:
```bash
kubectl -n smartapp exec deploy/web -- sh -c 'for i in $(seq 1 20); do curl -s http://web.smartapp:9898/ >/dev/null; done'
hubble observe --namespace smartapp --last 20 -o jsonpb | grep -i -m1 encrypted || cilium encrypt status
```
**Kết quả mong đợi:** `cilium encrypt status` cho thấy WireGuard bật; traffic giữa node được mã hóa **mà pod web/redis vẫn 1 container** (sidecarless).

### 3b. (Nâng cao trong-bài) Mutual authentication — mTLS theo danh tính
Cilium mutual auth dùng SPIFFE/SPIRE cấp danh tính rồi yêu cầu xác thực ở policy:
```bash
# Yêu cầu tính năng mutual-auth (SPIRE) đã bật: helm ... --set authentication.mutual.spire.enabled=true
kubectl apply -f - <<'EOF'
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata: { name: web-mtls, namespace: smartapp }
spec:
  endpointSelector: { matchLabels: { app: web } }
  ingress:
  - fromEndpoints: [{ matchLabels: { app: redis } }]
    authentication: { mode: "required" }   # bắt buộc mTLS theo danh tính
EOF
hubble observe --namespace smartapp --last 20   # luồng tới web hiện 'auth: SPIRE/required'
```
**Kết quả mong đợi:** chỉ traffic có danh tính hợp lệ tới được web; Hubble cho thấy xác thực — vẫn **không sidecar**.

**Câu hỏi suy ngẫm:** so với sidecar mesh, mTLS sidecarless tiết kiệm gì? *(Một proxy/node thay vì một proxy/pod → ít RAM, ít độ trễ, ít thứ phải vận hành.)*

> Dọn dẹp tùy chọn: `kubectl -n smartapp delete ciliumnetworkpolicy web-mtls`

---

## 4. BT4 — Mesh doanh nghiệp de-facto: Istio **ambient** (sidecarless) + canary (TIMEBOX ~50')

Istio là mesh được dùng nhiều nhất ở doanh nghiệp; **ambient mode** (GA) bỏ sidecar: một `ztunnel` DaemonSet lo mTLS L4, và **waypoint** proxy (tùy chọn) lo L7. Đây là kỹ năng "đi làm" nên BT này là trọng tâm.

> ⚠️ **Tương thích với Cilium:** Istio ambient redirect traffic ở tầng dưới; chạy chung Cilium cần Cilium **không độc quyền CNI** (`--set cni.exclusive=false`) để Istio-CNI nối chuỗi (chaining) được. Nếu lớp gặp xung đột datapath, chạy BT4 trên **một namespace demo riêng** hoặc để giảng viên demo — trọng tâm là *hiểu mô hình ambient*, không phải vật lộn tích hợp.

### 4a. Cài ambient & bật mTLS tự động (không sidecar)
```bash
istioctl install --set profile=ambient -y
kubectl -n istio-system get ds ztunnel        # datapath mTLS chạy mỗi node
# Đưa smartapp vào ambient — KHÔNG restart pod, KHÔNG sidecar
kubectl label namespace smartapp istio.io/dataplane-mode=ambient --overwrite
kubectl -n smartapp get pods                   # vẫn 1/1 container (sidecarless!)
```
Bắt buộc mTLS chặt và xác minh:
```bash
kubectl apply -n smartapp -f - <<'EOF'
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata: { name: default }
spec: { mtls: { mode: STRICT } }
EOF
istioctl ztunnel-config workloads -n smartapp  # thấy web/redis được ztunnel quản, mTLS
```
**Kết quả mong đợi:** web↔redis mã hóa mTLS qua ztunnel; pod **không** thêm container.

### 4b. L7 + canary 90/10 qua waypoint + Gateway API
```bash
# Waypoint cho namespace để xử lý L7 (canary, retry, fault…)
istioctl waypoint apply -n smartapp --enroll-namespace
kubectl -n smartapp get gtw waypoint

# Phiên bản mới + HTTPRoute chia 90/10
kubectl -n smartapp create deployment web-v2 --image=stefanprodan/podinfo:6.7.1
kubectl -n smartapp expose deployment web-v2 --port=9898 --target-port=9898
kubectl apply -f - <<'EOF'
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata: { name: web-canary, namespace: smartapp }
spec:
  parentRefs: [{ group: "", kind: Service, name: web, port: 9898 }]
  rules:
  - backendRefs:
    - { name: web,    port: 9898, weight: 90 }
    - { name: web-v2, port: 9898, weight: 10 }
EOF
```
Sinh tải & quan sát phân bổ:
```bash
kubectl -n smartapp exec deploy/web -- sh -c 'for i in $(seq 1 100); do curl -s http://web.smartapp:9898/version; echo; done' | sort | uniq -c
```
**Kết quả mong đợi:** ~90% v1, ~10% v2 — canary qua mesh ambient. Đây là cơ chế Bài 06 sẽ **tự động hóa** bằng Argo Rollouts.

**Câu hỏi suy ngẫm:** ambient khác sidecar mesh ở đâu về chi phí/vận hành? *(L4 dùng ztunnel/node; chỉ trả phí L7 ở waypoint khi cần — rẻ hơn nhiều so với Envoy/pod.)*

---

## 5. BT5 — Đối chiếu: Linkerd, "mesh đơn giản nhất" (tùy chọn)

Để học viên cảm nhận đánh đổi, cài Linkerd — đơn giản & nhẹ, nhưng **dùng sidecar** và hệ sinh thái nhỏ hơn, và bản *stable* nay cần Buoyant Enterprise (miễn phí ≤50 nhân sự).
```bash
curl -sL https://run.linkerd.io/install-edge | sh    # edge (OSS); stable = Buoyant Enterprise
export PATH=$PATH:$HOME/.linkerd2/bin
linkerd install --crds | kubectl apply -f -
linkerd install | kubectl apply -f -
linkerd viz install | kubectl apply -f -     # extension viz (cần cho 'viz edges' bên dưới)
kubectl get deploy -n smartapp -o yaml | linkerd inject - | kubectl apply -f -
kubectl -n smartapp get pods            # giờ 2/2 container — ĐÂY là khác biệt: có sidecar
linkerd -n smartapp viz edges deploy    # cột SECURED = mTLS
```
**Điểm dạy:** cùng kết quả (mTLS, canary) nhưng Linkerd thêm một container vào *mỗi* pod. So với Cilium/Istio-ambient (sidecarless), đây là sự đánh đổi đơn-giản-nhưng-tốn-tài-nguyên.
> Gỡ sau khi so sánh: `kubectl get deploy -n smartapp -o yaml | linkerd uninject - | kubectl apply -f -`

---

## 6. (Mở rộng — làm tiếp khi xong phần chính)

- **Fault injection & circuit breaker** qua waypoint Istio (`VirtualService fault`, `DestinationRule outlierDetection`) — gây 500/độ trễ, xem retry & cô lập.
- **Gateway API ingress** (thay Ingress): `Gateway` + `HTTPRoute` route `/web` từ ngoài vào — Cilium cũng triển khai Gateway API nên có thể làm **không cần Istio**.
- **Hubble forensics:** `hubble observe` lọc theo verdict/identity điều tra một luồng bị chặn.
- **So sánh tài nguyên:** đo RAM/độ trễ smartapp ở 3 chế độ (Cilium sidecarless / Istio ambient / Linkerd sidecar) → bảng đánh đổi thực nghiệm.

---

## 7. Cleanup

> ⚠️ **Bắt buộc trước Bài 06/07:** HTTPRoute canary 90/10 còn sót sẽ **làm lệch metrics** của Argo Rollouts (Bài 06) và load test (Bài 07) — tệ nhất là 10% traffic đi vào backend đã chết (503).

```bash
kubectl -n smartapp delete httproute web-canary --ignore-not-found
kubectl -n smartapp delete deploy web-v2 --ignore-not-found
kubectl -n smartapp delete svc web-v2 --ignore-not-found
istioctl waypoint delete -n smartapp --all 2>/dev/null || true
# Ambient label có thể GIỮ (mTLS tiếp tục hoạt động, không ảnh hưởng bài sau).
# Nếu đã cài Linkerd (BT5) — gỡ để tiết kiệm RAM:
linkerd viz uninstall | kubectl delete -f - 2>/dev/null || true
linkerd uninstall | kubectl delete -f - 2>/dev/null || true
```

---

## 8. Tổng kết & nối sang Bài 06

- Bạn quan sát mạng bằng Hubble, áp policy **L7** bằng Cilium, bật **mTLS sidecarless** hai cách (Cilium & Istio ambient), và chạy **canary** qua Gateway API.
- Thông điệp thị trường: **sidecarless là mặc định mới** — Cilium (đã có trong stack, kỹ năng khan hiếm) + Istio ambient (de-facto doanh nghiệp). Linkerd là lựa chọn đơn giản, đánh đổi bằng sidecar.
- **Bài 06:** GitOps & Progressive Delivery — tự động hóa chính canary này bằng Argo Rollouts với cổng phân tích metrics.

### Checklist hoàn thành
- [ ] Hubble hiển thị service map + luồng smartapp.
- [ ] CiliumNetworkPolicy L7 cho GET, chặn POST (xác nhận qua Hubble).
- [ ] Cilium sidecarless: WireGuard bật (`cilium encrypt status`); (nâng cao) mutual-auth mTLS.
- [ ] Istio ambient: ns ambient-labeled, ztunnel quản web/redis, mTLS STRICT — **pod vẫn 1 container**.
- [ ] Canary 90/10 v1/v2 qua HTTPRoute hoạt động.
- [ ] (Tùy chọn) Linkerd inject để thấy khác biệt sidecar.
