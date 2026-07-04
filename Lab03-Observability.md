# Lab 03 — Quan sát hệ thống (Observability)

> Hệ thống xuyên suốt: **smartapp** (web = podinfo). Lab này dựng full stack quan sát rồi dùng nó để điều tra một sự cố mô phỏng.
> podinfo expose sẵn `/metrics` (Prometheus), ghi log có cấu trúc, hỗ trợ tracing và có endpoint tự gây lỗi/độ trễ (`/status/{code}`, `/delay/{sec}`) — rất tiện cho bài này.

**Thời lượng:** ~60 phút · **Yêu cầu:** `kubectl` cluster-admin, `helm` ≥ 3.12, cụm có Internet (pull Helm charts) hoặc registry nội bộ. Có sẵn namespace `smartapp` với deployment `web` (podinfo) từ Bài 01.

---

## 0. Chuẩn bị namespace giám sát

```bash
kubectl create namespace monitoring 2>/dev/null || true
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

---

## 1. BT1 — Metrics: Prometheus + Grafana

Cài kube-prometheus-stack (gồm Prometheus Operator, Prometheus, Alertmanager, Grafana):
```bash
helm install kps prometheus-community/kube-prometheus-stack \
  -n monitoring \
  --set grafana.adminPassword='smartapp123'
kubectl -n monitoring get pods
```
**Kết quả mong đợi:** các pod `kps-*` (prometheus, grafana, operator, kube-state-metrics, node-exporter) `Running`.

Cho Prometheus scrape web (podinfo) bằng một **ServiceMonitor** (Operator pattern — Bài 02):
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: web
  namespace: smartapp
  labels:
    release: kps          # để Prometheus của kps nhận
spec:
  selector:
    matchLabels: { app: web }
  targetLabels: [app]        # gắn nhãn 'app' của Service vào metric — các truy vấn {app="web"} cần dòng này
  endpoints:
  - port: http
    path: /metrics
EOF
```
> Lưu ý: service `web` cần có port tên `http`. Nếu chưa, sửa: `kubectl -n smartapp patch svc web --type=json -p '[{"op":"add","path":"/spec/ports/0/name","value":"http"}]'`.

Sinh tải để có số liệu:
```bash
kubectl -n smartapp run load --rm -it --image=busybox --restart=Never -- \
  sh -c 'for i in $(seq 1 20000); do wget -q -O- http://web.smartapp:9898/ >/dev/null; done'
```

Mở Grafana và truy vấn PromQL:
```bash
kubectl -n monitoring port-forward svc/kps-grafana 3000:80 --address 0.0.0.0
# Trình duyệt: http://localhost:3000  (admin / smartapp123)
```
Trong Grafana → Explore (data source Prometheus), chạy:
- Request rate trong 5 phút gần nhất
```promql
rate(http_requests_total{app="web"}[5m])
```
- P99 của thời gian phục vụ http request
```promql
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{app="web"}[5m])) by (le))
```
**Kết quả mong đợi:** thấy đồ thị tốc độ request và p99 latency của podinfo.

**Checkpoint:** dựng một dashboard nhỏ 4 panel golden signals (traffic, error, latency, saturation) cho web.

---

## 2. BT2 — Logs: Loki + Grafana Alloy

> ⚠️ Chart `loki-stack` và plugin `fluent-bit-plugin-loki` (đẩy vào endpoint cũ `/api/prom/push`) đã **deprecated** và không tương thích Loki mới (lỗi `connect: operation not permitted` / không có log). Dùng **Grafana Alloy** làm agent thu log, đẩy vào endpoint chuẩn `/loki/api/v1/push`.

```bash
# Loki: chỉ backend, KHÔNG kèm agent cũ
helm install loki grafana/loki-stack -n monitoring \
  --set fluent-bit.enabled=false \
  --set promtail.enabled=false \
  --set grafana.enabled=false

# Grafana Alloy: agent thu log (DaemonSet 1 pod/node)
# ⚠️ Cú pháp River: MỖI thuộc tính một dòng (không gộp 2 thuộc tính trên cùng 1 dòng).
cat > alloy-values.yaml <<'EOF'
alloy:
  configMap:
    content: |
      discovery.kubernetes "pods" {
        role = "pod"
      }
      discovery.relabel "pods" {
        targets = discovery.kubernetes.pods.targets
        rule {
          source_labels = ["__meta_kubernetes_namespace"]
          target_label  = "namespace"
        }
        rule {
          source_labels = ["__meta_kubernetes_pod_label_app"]
          target_label  = "app"
        }
        rule {
          source_labels = ["__meta_kubernetes_pod_name"]
          target_label  = "pod"
        }
        rule {
          source_labels = ["__meta_kubernetes_pod_container_name"]
          target_label  = "container"
        }
      }
      loki.source.kubernetes "pods" {
        targets    = discovery.relabel.pods.output
        forward_to = [loki.write.default.receiver]
      }
      loki.write "default" {
        endpoint {
          url = "http://loki:3100/loki/api/v1/push"
        }
      }
EOF
helm install alloy grafana/alloy -n monitoring -f alloy-values.yaml
kubectl -n monitoring get pods -l app.kubernetes.io/name=alloy -o wide   # DaemonSet: 1 pod/node
```
> ⚠️ **podinfo chỉ ghi log từng request ở level `debug`** (mặc định `info` = chỉ log lúc khởi động). Bật debug để có log request cho web:
> ```bash
> kubectl -n smartapp set env deploy/web PODINFO_LEVEL=debug
> ```
> Ngoài ra trong Grafana Explore: chọn đúng data source **Loki** (không phải Prometheus) và đặt time range đủ rộng (vd Last 1 hour).

Thêm data source Loki vào Grafana (URL `http://loki:3100`), rồi Explore với LogQL:
```logql
{namespace="smartapp", app="web"}
{namespace="smartapp", app="web"} |= "GET"
sum by (pod) (count_over_time({app="web"} |= "error" [5m]))
```
**Kết quả mong đợi:** thấy log truy cập của podinfo theo nhãn; câu cuối biến log thành metric đếm lỗi.

**Checkpoint:** tạo một cảnh báo Grafana khi `count_over_time(... |= "error" [5m])` vượt ngưỡng.

---

## 3. BT3 — Tracing: Tempo (qua Alloy làm OTLP gateway)

```bash
helm install tempo grafana/tempo -n monitoring
```
Cho **cùng Alloy** ở BT2 nhận OTLP và chuyển tiếp về Tempo — bổ sung vào `alloy-values.yaml`: thêm 2 khối vào `configMap.content` và mở cổng OTLP trên Service, rồi `helm upgrade`.
```river
# thêm vào cuối configMap.content (mỗi thuộc tính một dòng)
otelcol.receiver.otlp "in" {
  grpc {
    endpoint = "0.0.0.0:4317"
  }
  http {
    endpoint = "0.0.0.0:4318"
  }
  output {
    traces = [otelcol.exporter.otlp.tempo.input]
    logs   = [otelcol.exporter.loki.default.input]
  }
}
otelcol.exporter.otlp "tempo" {
  client {
    endpoint = "tempo.monitoring:4317"
    tls {
      insecure = true
    }
  }
}
# podinfo gửi cả LOG qua OTLP -> nhận và forward vào Loki (nếu thiếu sẽ lỗi Unimplemented LogsService)
otelcol.exporter.loki "default" {
  forward_to = [loki.write.default.receiver]
}
```
```yaml
# thêm vào cùng cấp `alloy:` trong alloy-values.yaml
alloy:
  extraPorts:
    - { name: otlp-grpc, port: 4317, targetPort: 4317, protocol: TCP }
    - { name: otlp-http, port: 4318, targetPort: 4318, protocol: TCP }
```
```bash
helm upgrade alloy grafana/alloy -n monitoring -f alloy-values.yaml
```

### Distributed trace qua 3 service: `web → backend → backend2`

`reset.sh` đã dựng sẵn chuỗi: `web` gọi `backend`, `backend` gọi `backend2` (leaf) qua `PODINFO_BACKEND_URL`. podinfo tự truyền `traceparent` nên cả chuỗi nằm chung một trace. Bật OTLP trên **cả 3** service (endpoint PHẢI có scheme `http://`):

```bash
for svc in web backend backend2; do
  kubectl -n smartapp set env deploy/$svc \
    PODINFO_OTEL_SERVICE_NAME=$svc \
    OTEL_EXPORTER_OTLP_ENDPOINT=http://alloy.monitoring:4317 \
    OTEL_EXPORTER_OTLP_INSECURE=true \
    PODINFO_LEVEL=debug
done
kubectl -n smartapp rollout status deploy/web deploy/backend deploy/backend2

# Gọi /echo trên web -> lan truyền xuống backend -> backend2 (tạo trace 3 span)
kubectl -n smartapp exec deploy/web -- curl -s http://localhost:9898/echo >/dev/null
```
Thêm data source Tempo (`http://tempo:3200`) vào Grafana → Explore → **Tempo** → Search service=`web`.
**Kết quả mong đợi:** waterfall gồm span của cả 3 service (`web` → `backend` → `backend2`) dưới cùng một `trace_id`; thấy rõ chặng nào tốn thời gian.

> *Nếu podinfo không bật được OTLP, dùng app demo có sẵn trace như OpenTelemetry demo; trọng tâm là đọc waterfall.*

---

## 4. BT4 — SLO, error budget & tương quan (trọng tâm)

### 4a. Định nghĩa SLO và cảnh báo
Tạo một PrometheusRule SLO (lỗi 5xx < 1%):
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: smartapp-slo
  namespace: monitoring
  labels: { release: kps }
spec:
  groups:
  - name: smartapp-slo
    rules:
    - alert: WebHighErrorRate
      expr: |
        sum(rate(http_requests_total{app="web",status=~"5.."}[5m]))
          / sum(rate(http_requests_total{app="web"}[5m])) > 0.01
      for: 2m
      labels: { severity: page }
      annotations:
        summary: "web (smartapp) lỗi 5xx > 1%"
EOF
```

### 4b. Gây sự cố và điều tra theo 3 bước
```bash
# Bơm lỗi 500 vào web để vi phạm SLO
kubectl -n smartapp run errgen --rm -it --image=busybox --restart=Never -- \
  sh -c 'for i in $(seq 1 500); do wget -q -O- http://web.smartapp:9898/status/500 >/dev/null 2>&1; done'
```
**(Tùy chọn) Inject lỗi vào CHẶNG SÂU của chuỗi trace** — để waterfall chỉ đúng service thủ phạm là `backend2`, đổi URL backend của `backend` sang endpoint lỗi/chậm rồi gọi lại `/echo`:
```bash
# Lỗi 500 ở backend2:  .../status/500   |   Chậm 3s ở backend2:  .../delay/3
kubectl -n smartapp set env deploy/backend PODINFO_BACKEND_URL=http://backend2:9898/status/500
kubectl -n smartapp rollout status deploy/backend
for i in $(seq 1 50); do kubectl -n smartapp exec deploy/web -- curl -s http://localhost:9898/echo >/dev/null; done
# Khôi phục chuỗi khỏe mạnh:
# kubectl -n smartapp set env deploy/backend PODINFO_BACKEND_URL=http://backend2:9898/echo
```
Thực hiện quy trình điều tra (mục tiêu của cả bài):
1. **Metric** — trong Prometheus/Alertmanager, thấy alert `WebHighErrorRate` kích hoạt; xác nhận tỉ lệ lỗi 5xx tăng.
2. **Trace** — mở trace lỗi trong Tempo: waterfall `web → backend → backend2` cho thấy span `backend2` bị lỗi/chậm → chỉ đúng chặng gây sự cố (không chỉ "web lỗi").
3. **Log** — dùng LogQL `{app="backend2"} |= "500"` (hoặc `{app="web"}`) đúng pod/thời điểm để đọc thông điệp lỗi cụ thể; đối chiếu `trace_id` giữa log và trace.

**Kết quả mong đợi:** học viên đi trọn metric → trace → log và phát biểu được "nguyên nhân gốc" từ log, thay vì chỉ thấy "có lỗi".

**Câu hỏi suy ngẫm:** vì sao cảnh báo theo SLO/error budget tốt hơn cảnh báo "CPU > 80%"? *(Gợi ý: SLO gắn với trải nghiệm người dùng; CPU cao chưa chắc ảnh hưởng người dùng.)*

---

## 5. (Nâng cao — nếu còn thời gian)

- **Recording rules:** tạo recording rule precompute tỉ lệ lỗi để dashboard nhẹ và nhanh hơn.
- **Alertmanager routing:** định tuyến alert `severity=page` tới một webhook/Slack test; `severity=ticket` đi nơi khác.
- **Burn-rate multi-window:** viết alert error-budget burn rate 2 cửa sổ (5m & 1h) thay vì ngưỡng tĩnh.
- **Dashboard-as-code:** export dashboard JSON, lưu vào Git (chuẩn bị cho GitOps Bài 06).
- **Exemplars:** bật exemplars để nhảy thẳng từ điểm metric sang trace trong Grafana.

---

## 6. Tổng kết & nối sang Bài 04

- Bạn đã dựng full stack quan sát cho smartapp: Prometheus+Grafana (metrics), Loki+Alloy (logs), Tempo (traces) — Alloy làm agent hợp nhất cho cả log lẫn trace.
- Bạn đã đọc distributed trace qua chuỗi 3 service `web → backend → backend2`, và inject lỗi vào chặng sâu để trace chỉ đúng service thủ phạm.
- Bạn đã đo bằng SLO/error budget và đi trọn quy trình tương quan metric → trace → log để tìm nguyên nhân.
- **Bài 04:** bảo mật hệ thống — hardening pod, policy-as-code, supply chain, secrets. (Audit log và quan sát ở bài này sẽ hỗ trợ phát hiện sự cố bảo mật.)
