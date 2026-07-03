# Lab 06 — Challenge: "Canary kẹt cứng"

> **Thời gian ước lượng: ~100 phút**
> **Đầu vào:** cluster với `smartapp`; Argo Rollouts + Prometheus pre-baked. *(Bài độc lập.)*
> **Đầu ra:** canary chạy thông và promote tới production qua cổng phân tích metrics.
>
> ⚠️ Làm **bài hướng dẫn `Lab06-GitOps-ProgressiveDelivery.md` trước** (phần học).

---

## Bối cảnh

Đây **không** phải bài "tự dựng pipeline từ đầu" như phần hướng dẫn. Một `Rollout` cho `web` với chiến lược **canary + cổng analysis** đã được dựng sẵn, nhưng bản phát hành mới **kẹt mãi không lên** — canary không bao giờ promote, cũng không tự rollback dứt khoát. Ai đó đã cấu hình **cổng phân tích sai**.

Nhiệm vụ: chẩn đoán vì sao cổng analysis không cho qua, sửa nó, và đưa canary lên 100%.

## Thông tin đang có

- Một `Rollout/web` và một `AnalysisTemplate/success-rate` đã tồn tại trong `smartapp`.
- Bản phát hành mới đang dừng ở bước `analysis`. Xem:
  - `kubectl argo rollouts get rollout web -n smartapp` (hoặc `kubectl -n smartapp get rollout web -o yaml`)
  - `kubectl -n smartapp get analysisrun` và `describe` cái mới nhất → thông điệp lỗi của provider.
- Prometheus đã pre-baked trong cluster (cần biết đúng địa chỉ Service của nó).

> 🔍 Tự điều tra: AnalysisRun đang `Error` hay `Failed`? provider trả gì? truy vấn tới *đâu*?

---

## Mục tiêu (trạng thái cuối được chấm)

1. **Rollout Healthy.** `Rollout/web` ở trạng thái `Healthy`, không `Degraded`/`Paused`/`abort`.
2. **Canary promote 100%.** Bản mới đã lên toàn bộ (không còn kẹt giữa bước).
3. **Cổng analysis hoạt động.** Có ít nhất một `AnalysisRun` **Successful** (truy vấn Prometheus chạy được).
4. **Đã sửa đúng chỗ.** `AnalysisTemplate/success-rate` không còn trỏ địa chỉ Prometheus sai.

> 💡 Để analysis **Successful** cần có số liệu thật: đảm bảo `web` được Prometheus scrape (ServiceMonitor) và có **tải** để `http_requests_total` có dữ liệu.

## `answers.env`

```env
SUCCESS_THRESHOLD="..."    # ngưỡng success-rate cổng analysis dùng (vd 0.95)
```

## Tự chấm

```bash
./graders/grade06.sh smartapp
```

Lặp lại tới khi **mọi mục REQUIRED đều PASS**.

## Ràng buộc & gợi ý mức cao (không phải lời giải)

- Cách sai (không học được gì): xóa bước `analysis` khỏi Rollout để nó "tự lên". Đó là gỡ cổng an toàn.
- Cách đúng: đọc lỗi AnalysisRun → nhận ra provider không tới được Prometheus → sửa `address` (và đảm bảo có metric).
- **Mở rộng (BONUS):** đưa toàn bộ vào GitOps — một ArgoCD `Application` (Synced+Healthy) quản smartapp từ Git, bật `selfHeal`.

---

### Tiếp theo

Loạt bài tiếp tục trên smartapp (mỗi bài độc lập). Ở Lab 07, smartapp gục dưới tải và đốt tài nguyên — bạn sẽ right-size theo đo đạc, scale-to-zero theo hàng đợi, và lập lịch để phân bố đều.
