# Lab 03 — Challenge: "smartapp lỗi mà bạn đang mù"

> **Thời gian ước lượng: ~90 phút**
> **Đầu vào:** cluster với namespace `smartapp` (`web` = podinfo, `redis`) đang chạy. *(Bài độc lập.)*
> **Đầu ra:** observability stack (metrics/logs/traces) + một SLO alert đang kích hoạt + nguyên nhân gốc đã chỉ ra.
>
> ⚠️ Làm **bài hướng dẫn `Lab03-Observability.md` trước** (phần học).

---

## Bối cảnh

Người dùng báo smartapp lỗi `5xx` rải rác, nhưng cluster **không có monitoring** — bạn đang mù. Có một thứ gì đó trong cluster đang liên tục tạo lỗi, nhưng không ai biết là cái gì hay từ đâu.

Nhiệm vụ: dựng đủ khả năng quan sát để **nhìn thấy** sự cố, đặt một cảnh báo theo SLO, rồi đi từ metric → log để **chỉ đúng tên** nguyên nhân.

## Thông tin đang có

- `smartapp/web` là podinfo: expose `/metrics` (Prometheus), `/healthz`, và endpoint tự gây lỗi `/status/{code}`.
- Helm charts cho stack quan sát đã được mirror sẵn trên lab image (không cần Internet ngoài).
- Có một nguồn lỗi đang chạy sẵn trong cluster (do setup thả vào) — bạn phải tự tìm ra nó.

---

## Mục tiêu (trạng thái cuối được chấm)

1. **Metrics.** Dựng Prometheus (+ Grafana); cho Prometheus scrape `web` qua một `ServiceMonitor`.
2. **Logs.** Dựng Loki (+ Grafana Alloy) để thu log của `web`.
3. **SLO alert.** Tạo một `PrometheusRule` cảnh báo khi tỉ lệ `5xx` của `web` vượt ngưỡng (vd > 1%) — alert này **phải kích hoạt** dưới sự cố đang diễn ra.
4. **Điều tra & chỉ tên.** Đi metric → log để xác định **đường dẫn (path)** đang sinh lỗi và **pod thủ phạm** đang gọi nó. Điền `answers.env`.

## `answers.env`

```env
ROOT_CAUSE_PATH="/..."     # path trên web đang trả lỗi 5xx
CULPRIT_POD="..."          # tên pod đang gọi path lỗi (tiền tố là đủ)
```

## Tự chấm

```bash
./graders/grade03.sh smartapp
```

Lặp lại tới khi **mọi mục REQUIRED đều PASS**.

## Ràng buộc & gợi ý mức cao (không phải lời giải)

- ServiceMonitor cần khớp đúng nhãn của service `web` và port tên `http` (podinfo metric: `http_requests_total`).
- Alert phải dựa trên **tỉ lệ lỗi** (5xx / tổng), không phải ngưỡng thô như CPU.
- "Chỉ tên" nghĩa là dùng **dữ liệu** (PromQL ra path lỗi, LogQL ra pod gọi nó) — không đoán.

---

### Tiếp theo

Loạt bài tiếp tục trên smartapp (mỗi bài độc lập). Ở Lab 04, một báo cáo pentest cho thấy smartapp đang chạy quá nhiều đặc quyền và có secret lộ — bạn sẽ siết bảo mật nhiều lớp.
