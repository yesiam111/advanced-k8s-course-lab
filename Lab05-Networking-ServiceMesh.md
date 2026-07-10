# Lab 05 — Mạng nâng cao & Service Mesh (sidecarless-first)

> Hệ thống xuyên suốt: **smartapp** (web = podinfo, redis). Cụm đã chạy **Cilium** làm CNI (từ Intro).
> Triết lý bài này theo đúng thị trường 2026: **mesh không-sidecar (sidecarless) là hướng đi chính.**
> Ta khai thác mesh **đã có sẵn trong stack** (Cilium), rồi học **mesh doanh nghiệp de-facto** (Istio **ambient**), và chỉ xem Linkerd như bản "đơn giản nhất" để đối chiếu.

**Thời lượng:** ~60 phút · **Yêu cầu:** `kubectl` cluster-admin, `helm`, `cilium` CLI, `istioctl`; namespace `smartapp` với `web` + `redis` từ Bài 01.

---

## 1. BT1 — Hubble: quan sát luồng mạng

Bật Hubble (nếu cụm chưa bật):
```bash
cilium hubble enable --ui
cilium status | grep -i hubble
```
Cài **Hubble CLI** (binary riêng, không đi kèm `cilium` CLI) — bắt buộc để chạy `hubble observe`.
> ⚠️ CLI phải **khớp version với Hubble server** (cilium-agent). CLI mới hơn server sẽ lỗi `invalid fieldmask` khi chạy `--last`.
> Cách chắc chắn nhất: **copy binary từ chính pod cilium-agent đang chạy** — luôn đúng version, không cần internet (github thường bị chặn trong lab).
```bash
# Lấy binary hubble version-matched thẳng từ pod cilium-agent
CILIUM_POD=$(kubectl -n kube-system get pod -l k8s-app=cilium -o jsonpath='{.items[0].metadata.name}')
HUBBLE_BIN=$(kubectl -n kube-system exec "$CILIUM_POD" -c cilium-agent -- which hubble)
kubectl -n kube-system cp "$CILIUM_POD":"$HUBBLE_BIN" /tmp/hubble -c cilium-agent
sudo install -m 0755 /tmp/hubble /usr/local/bin/hubble
hash -r
hubble --version   # phải khớp cilium-agent (vd v1.17.4)
# Mở port-forward tới Hubble Relay (nền) rồi kiểm tra
pkill -f "hubble port-forward"; cilium hubble port-forward & sleep 2
hubble status
```

Quan sát:
```bash
# Hubble UI: 'cilium hubble ui' chỉ bind localhost. Port-forward service để nghe mọi interface:
kubectl port-forward -n kube-system svc/hubble-ui --address 0.0.0.0 12000:80 &
# → mở http://<node-ip>:12000  (service map)
hubble observe --namespace smartapp --last 50
```
Sinh lưu lượng trong smartapp:
```bash
# từ redis gọi HTTP tới web
kubectl -n smartapp exec deploy/redis -- sh -c 'for i in $(seq 1 20); do wget -O /dev/null -o /dev/null http://web.smartapp:9898/ ; done' || true
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
# Bật transparent encryption
cilium config set enable-wireguard true        # hoặc helm upgrade ... --set encryption.enabled=true,encryption.type=wireguard
kubectl -n kube-system rollout status ds/cilium 
sleep 5
cilium encrypt status
```
Sinh traffic web↔redis rồi xác nhận đã mã hóa:
```bash
kubectl -n smartapp exec deploy/web -- sh -c 'for i in $(seq 1 20); do curl -s http://web.smartapp:9898/ >/dev/null; done'
# LƯU Ý: WireGuard là mã hóa node↔node, TRONG SUỐT — Hubble flow KHÔNG có field 'encrypted'
# cho WireGuard (grep sẽ rỗng, đó là bình thường, không phải lỗi). Kiểm tra bằng:
cilium encrypt status
```
**Kết quả mong đợi:** `cilium encrypt status` báo `Encryption: Wireguard` với số peer khớp số node; traffic giữa node được mã hóa **mà pod web/redis vẫn 1 container** (sidecarless).

### 3b. Mutual authentication — mTLS theo danh tính
Cilium mutual auth dùng SPIFFE/SPIRE cấp danh tính rồi yêu cầu xác thực ở policy.

**Bước 1 — Bật SPIRE**
```bash
# repo cilium được thêm khi provisioning (chạy sudo). Nếu user hiện tại chưa có, thêm lại:
helm repo add cilium https://helm.cilium.io 2>/dev/null; helm repo update >/dev/null
helm upgrade cilium cilium/cilium -n kube-system --reuse-values \
  --set authentication.enabled=true \
  --set authentication.mutual.spire.enabled=true \
  --set authentication.mutual.spire.install.enabled=true
kubectl -n kube-system rollout restart ds/cilium deploy/cilium-operator
kubectl -n cilium-spire rollout status statefulset/spire-server --timeout=180s
kubectl -n cilium-spire get pods           # spire-server + spire-agent phải Running
```

**Bước 2 — Áp policy yêu cầu xác thực:**
```bash
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
```

**Bước 3 — Sinh traffic rồi CHỨNG MINH mutual auth chạy (không chỉ nhìn Hubble):**
```bash
# QUAN TRỌNG: 'bpf auth list' là PER-NODE. Phải exec đúng cilium-agent trên node chạy 'web',
# KHÔNG dùng 'ds/cilium' (vì chọn pod bất kỳ → dễ ra "No entries found" giả).
WEB_NODE=$(kubectl -n smartapp get pod -l app=web -o jsonpath='{.items[0].spec.nodeName}')
CILIUM_WEB=$(kubectl -n kube-system get pod -l k8s-app=cilium \
  -o jsonpath="{.items[?(@.spec.nodeName=='$WEB_NODE')].metadata.name}")
# Sinh traffic NGAY TRƯỚC khi kiểm (entry bị GC mỗi 5m)
kubectl -n smartapp exec deploy/redis -- sh -c 'for i in $(seq 1 10); do wget -qO- http://web.smartapp:9898/ >/dev/null; done'
# BẰNG CHỨNG: auth cache có cặp identity đã bắt tay xong (TYPE=spire, có expiry)
kubectl -n kube-system exec "$CILIUM_WEB" -c cilium-agent -- cilium-dbg bpf auth list
# Flow chuyển từ vài DROP ban đầu -> FORWARDED (auth=SPIRE)
hubble observe --namespace smartapp --last 20 -o json \
  | jq -r '.flow | "\(.source.identity) -> \(.destination.identity)  auth=\(.auth_type)  verdict=\(.verdict)"' 2>/dev/null | grep -v null
```
**Kết quả mong đợi:** `bpf auth list` có ít nhất 1 entry `spire` với expiry (vd `SRC→DST ... spire ... EXPIRATION`); flow redis→web chuyển sang `FORWARDED`.
> ⚠️ Nếu RỖNG: (1) đúng node của `web` chưa? (2) đã sinh traffic ngay trước khi kiểm chưa? (3) `kubectl -n cilium-spire get pods` đã Running chưa? Cả ba OK mà vẫn toàn `DROPPED` mới là SPIRE lỗi (fail-closed).

**Câu hỏi suy ngẫm:** so với sidecar mesh, mTLS sidecarless tiết kiệm gì? *(Một proxy/node thay vì một proxy/pod → ít RAM, ít độ trễ, ít thứ phải vận hành.)*

> Dọn dẹp: `kubectl -n smartapp delete ciliumnetworkpolicy web-mtls`

---

## 4. BT4 — Mesh doanh nghiệp de-facto: Istio **ambient** (sidecarless) + canary (TIMEBOX ~50')

Istio là mesh được dùng nhiều nhất ở doanh nghiệp; **ambient mode** (GA) bỏ sidecar: một `ztunnel` DaemonSet lo mTLS L4, và **waypoint** proxy (tùy chọn) lo L7.

> ⚠️ **Tương thích với Cilium:** Istio ambient redirect traffic ở tầng dưới; chạy chung Cilium cần Cilium **không độc quyền CNI** để Istio-CNI chaining được.
>
> 🔴 **BẮT BUỘC — KHÔNG chạy ambient trên `smartapp`.** ztunnel bọc traffic bằng HBONE; nếu `smartapp` vừa có ambient vừa có L7 CiliumNetworkPolicy (BT2/Challenge) thì L7 sẽ **drop** luồng → hỏng bài chấm. Vì vậy BT4 chạy trên **namespace demo riêng `smartapp-mesh`** (seed bên dưới). Trọng tâm là *hiểu mô hình ambient*, không phải tích hợp chung ns với bài chấm.

### 4.0. Chuẩn bị: cài `istioctl` + mở CNI chaining cho Cilium
```bash
# --- Cài istioctl (nếu chưa có) ---
curl -L https://istio.io/downloadIstio | sh -         
sudo install -m 0755 istio-*/bin/istioctl /usr/local/bin/istioctl
istioctl version --remote=false                        # xác nhận CLI chạy

# --- Cho phép Istio-CNI chaining trên cụm Cilium ---
# Cilium mặc định cni.exclusive=true (xóa CNI khác) + socketLB bỏ qua redirect của ztunnel.
helm upgrade cilium cilium/cilium -n kube-system --reuse-values \
  --set cni.exclusive=false \
  --set socketLB.hostNamespaceOnly=true
kubectl -n kube-system rollout restart ds/cilium
kubectl -n kube-system rollout status ds/cilium --timeout=180s

# --- Seed namespace demo RIÊNG cho ambient (KHÔNG đụng 'smartapp' đang chấm) ---
kubectl create namespace smartapp-mesh
kubectl -n smartapp-mesh create deployment web   --image=stefanprodan/podinfo:6.13.0
kubectl -n smartapp-mesh create deployment redis --image=redis:7-alpine
kubectl -n smartapp-mesh expose deployment web   --port=9898 --target-port=9898
kubectl -n smartapp-mesh expose deployment redis --port=6379
kubectl -n smartapp-mesh rollout status deploy/web --timeout=90s
```
> Nếu mạng lab chặn `istio.io`/github, tải `istioctl` sẵn vào image/host trước buổi học rồi bỏ qua bước curl.

### 4a. Cài ambient & bật mTLS tự động (không sidecar)
```bash
istioctl install --set profile=ambient -y
kubectl -n istio-system get ds ztunnel        # datapath mTLS chạy mỗi node
# Đưa smartapp-mesh vào ambient — KHÔNG restart pod, KHÔNG sidecar. (KHÔNG dán nhãn lên 'smartapp'!)
kubectl label namespace smartapp-mesh istio.io/dataplane-mode=ambient --overwrite
kubectl -n smartapp-mesh get pods              # vẫn 1/1 container (sidecarless!)
```
Bắt buộc mTLS chặt và xác minh:
```bash
kubectl apply -n smartapp-mesh -f - <<'EOF'
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata: { name: default }
spec: { mtls: { mode: STRICT } }
EOF
# LƯU Ý: '-n' của ztunnel-config trỏ tới namespace của ZTUNNEL (istio-system), KHÔNG phải app.
# Bỏ '-n', lọc bằng grep để xem workload smartapp-mesh:
istioctl ztunnel-config workloads | grep smartapp-mesh   # thấy web/redis được ztunnel quản, mTLS
```
**Kết quả mong đợi:** web↔redis mã hóa mTLS qua ztunnel; pod **không** thêm container.

### 4b. L7 + canary 90/10 qua waypoint + Gateway API
```bash
# Waypoint cho namespace để xử lý L7 (canary, retry, fault…)
istioctl waypoint apply -n smartapp-mesh --enroll-namespace
kubectl -n smartapp-mesh get gtw waypoint

# 'web' của smartapp-mesh đã là 6.13.0 (seed) để thấy rõ khác biệt version khi split (qua /version)
# Phiên bản mới (candidate) + HTTPRoute chia 90/10
kubectl -n smartapp-mesh create deployment web-v2 --image=stefanprodan/podinfo:6.14.0
kubectl -n smartapp-mesh expose deployment web-v2 --port=9898 --target-port=9898
kubectl apply -f - <<'EOF'
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata: { name: web-canary, namespace: smartapp-mesh }
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
kubectl -n smartapp-mesh exec deploy/web -- sh -c 'for i in $(seq 1 100); do curl -s http://web.smartapp-mesh:9898/version; echo; done' | grep "version" | sort | uniq -c
```
**Kết quả mong đợi:** ~90% v1, ~10% v2 — canary qua mesh ambient. Đây là cơ chế Bài 06 sẽ **tự động hóa** bằng Argo Rollouts.

**Câu hỏi suy ngẫm:** ambient khác sidecar mesh ở đâu về chi phí/vận hành? *(L4 dùng ztunnel/node; chỉ trả phí L7 ở waypoint khi cần — rẻ hơn nhiều so với Envoy/pod.)*

---

## 5. (Mở rộng)

- **Fault injection & circuit breaker** qua waypoint Istio (`VirtualService fault`, `DestinationRule outlierDetection`) — gây 500/độ trễ, xem retry & cô lập.
- **Gateway API ingress** (thay Ingress): `Gateway` + `HTTPRoute` route `/web` từ ngoài vào — Cilium cũng triển khai Gateway API nên có thể làm **không cần Istio**.
- **Hubble forensics:** `hubble observe` lọc theo verdict/identity điều tra một luồng bị chặn.
- **So sánh tài nguyên:** đo RAM/độ trễ smartapp ở 3 chế độ (Cilium sidecarless / Istio ambient / Linkerd sidecar) → bảng đánh đổi thực nghiệm.

---

## 6. Cleanup

> ⚠️ **Bắt buộc trước Bài 06/07:** HTTPRoute canary 90/10 còn sót sẽ **làm lệch metrics** của Argo Rollouts (Bài 06) và load test (Bài 07) — tệ nhất là 10% traffic đi vào backend đã chết (503).

```bash
# Canary demo (BT4) nằm trong ns riêng smartapp-mesh — xoá cả ns là sạch:
istioctl waypoint delete -n smartapp-mesh --all 2>/dev/null || true
kubectl delete namespace smartapp-mesh --ignore-not-found

# 🔴 Gỡ Istio ambient ĐÚNG CÁCH (đừng chỉ xoá ns istio-system!).
#    istio-cni chạy chained trong conflist Cilium; xoá thẳng ns sẽ để lại entry lỗi
#    -> MỌI pod mới kẹt sandbox ("istio-cni ... Unauthorized"). Uninstall để DS tự dọn:
istioctl uninstall --purge -y 2>/dev/null || true
kubectl -n kube-system wait --for=delete ds/istio-cni-node --timeout=90s 2>/dev/null || true
kubectl delete namespace istio-system --ignore-not-found
# Trả Cilium về độc quyền CNI như baseline:
helm upgrade cilium cilium/cilium -n kube-system --reuse-values --set cni.exclusive=true 2>/dev/null || true
kubectl -n kube-system rollout restart ds/cilium
```
> Nếu chỉ làm mTLS bằng **Cilium mutual-auth** (BT3) và bỏ qua BT4, không cần đoạn gỡ Istio ở trên.

---

## 8. Tổng kết & nối sang Bài 06

- Bạn quan sát mạng bằng Hubble, áp policy **L7** bằng Cilium, bật **mTLS sidecarless** hai cách (Cilium & Istio ambient), và chạy **canary** qua Gateway API.
- Thông điệp thị trường: **sidecarless là mặc định mới** — Cilium (đã có trong stack, kỹ năng khan hiếm) + Istio ambient (de-facto doanh nghiệp). Linkerd là lựa chọn đơn giản, đánh đổi bằng sidecar.
- **Bài 06:** GitOps & Progressive Delivery — tự động hóa chính canary này bằng Argo Rollouts với cổng phân tích metrics.
