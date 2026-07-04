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

## 2. BT2 — Logs: Loki + Fluent Bit

```bash
helm install loki grafana/loki-stack \
  -n monitoring \
  --set fluent-bit.enabled=true \
  --set promtail.enabled=false \
  --set grafana.enabled=false
kubectl -n monitoring get pods -l app=fluent-bit -o wide   # DaemonSet: 1 pod/node
```
Thêm data source Loki vào Grafana (URL `http://loki:3100`), rồi Explore với LogQL:
```logql
{namespace="smartapp", app="web"}
{namespace="smartapp", app="web"} |= "GET"
sum by (pod) (count_over_time({app="web"} |= "error" [5m]))
```
**Kết quả mong đợi:** thấy log truy cập của podinfo theo nhãn; câu cuối biến log thành metric đếm lỗi.

**Checkpoint:** tạo một cảnh báo Grafana khi `count_over_time(... |= "error" [5m])` vượt ngưỡng.

---

## 3. BT3 — Tracing: Tempo

```bash
helm install tempo grafana/tempo -n monitoring
```
Bật tracing trên podinfo (xuất OTLP về Tempo) và sinh một request có DB/cache call mô phỏng:
```bash
# podinfo bật tracing khi đặt --otel-service-name (env PODINFO_OTEL_SERVICE_NAME);
# endpoint OTLP dùng biến chuẩn OpenTelemetry (gRPC :4317)
kubectl -n smartapp set env deploy/web \
  PODINFO_OTEL_SERVICE_NAME=web \
  OTEL_EXPORTER_OTLP_ENDPOINT=tempo.monitoring:4317 \
  OTEL_EXPORTER_OTLP_INSECURE=true
# Gọi endpoint có độ trễ để tạo span dễ thấy
kubectl -n smartapp exec deploy/web -- curl -s localhost:9898/delay/1 >/dev/null
```
Thêm data source Tempo (`http://tempo:3100`) vào Grafana → Explore → tìm trace.
**Kết quả mong đợi:** xem được biểu đồ waterfall của một request, thấy span nào tốn thời gian nhất.

> *Nếu bản podinfo trong môi trường không bật được OTLP, dùng app demo có sẵn trace như `grafana/xk6-...` hoặc OpenTelemetry demo; trọng tâm là đọc waterfall.*

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
Thực hiện quy trình điều tra (mục tiêu của cả bài):
1. **Metric** — trong Prometheus/Alertmanager, thấy alert `WebHighErrorRate` kích hoạt; xác nhận tỉ lệ lỗi 5xx tăng.
2. **Trace** — từ điểm latency/lỗi trong Grafana, theo exemplar (hoặc tìm theo thời điểm) tới trace lỗi → xác định chặng gây lỗi.
3. **Log** — dùng LogQL `{app="web"} |= "500"` đúng pod/thời điểm để đọc thông điệp lỗi cụ thể.

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

- Bạn đã dựng full stack quan sát cho smartapp: Prometheus+Grafana (metrics), Loki+Fluent Bit (logs), Tempo (traces).
- Bạn đã đo bằng SLO/error budget và đi trọn quy trình tương quan metric → trace → log để tìm nguyên nhân.
- **Bài 04:** bảo mật hệ thống — hardening pod, policy-as-code, supply chain, secrets. (Audit log và quan sát ở bài này sẽ hỗ trợ phát hiện sự cố bảo mật.)

### Checklist hoàn thành
- [ ] Prometheus scrape được web qua ServiceMonitor; PromQL ra số liệu podinfo.
- [ ] Loki + Fluent Bit thu log; LogQL truy vấn được log của web.
- [ ] Tempo nhận trace; đọc được waterfall một request.
- [ ] SLO alert kích hoạt khi bơm lỗi; đi trọn metric → trace → log tìm nguyên nhân.
