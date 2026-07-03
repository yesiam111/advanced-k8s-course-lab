# Lab 07 — Challenge: "smartapp gục dưới tải & đốt tiền"

> **Thời gian ước lượng: ~90 phút**
> **Đầu vào:** cluster với `smartapp` (`web`, `redis`) + một `worker`. KEDA/metrics-server pre-baked. *(Bài độc lập.)*
> **Đầu ra:** smartapp đã right-size + tự động scale + lập lịch đúng.
>
> ⚠️ Làm **bài hướng dẫn `Lab07-Scaling-Performance.md` trước** (phần học).

---

## Bối cảnh

Dưới tải, `web` bị **OOMKilled** liên tục; một `worker` chạy 24/7 dù hàng đợi rỗng (đốt tài nguyên); và pod dồn cục bộ trên một node. Phải right-size theo **bằng chứng** và để hệ thống tự co giãn.

## Thông tin đang có

- `web` đang đặt **memory limit quá thấp** → OOMKilled.
- `worker` (podinfo) chưa có cơ chế scale; `redis` dùng làm nguồn hàng đợi.
- KEDA + metrics-server đã cài sẵn (nếu thiếu, tự cài).

---

## Mục tiêu (trạng thái cuối được chấm)

1. **Hết OOM.** Đặt memory limit hợp lý (dựa trên đo đạc thực tế, không đoán) sao cho `web` **không còn OOMKilled**.
2. **Scale-to-zero theo hàng đợi.** Một KEDA `ScaledObject` cho `worker` với `minReplicaCount: 0`, scale theo độ sâu hàng đợi Redis.
3. **HPA cho web.** Một HorizontalPodAutoscaler nhắm `web`.
4. **Trải đều.** `web` có `topologySpreadConstraints` để phân bố trên nhiều node.

## `answers.env`

```env
OOM_MB=...                 # memory limit (MB) bạn đặt cho web sau khi đo
FIRST_OOM_QOS="..."        # QoS class nào bị OOM giết ĐẦU TIÊN khi node thiếu RAM
```

## Tự chấm

```bash
./graders/grade07.sh smartapp
```

Lặp lại tới khi **mọi mục REQUIRED đều PASS**.

## Ràng buộc & gợi ý mức cao (không phải lời giải)

- "Dựa trên đo đạc" — quan sát mức RAM thực tế (cgroup/OOM) rồi chọn limit, đừng đặt bừa số to.
- HPA chuẩn **không** scale-to-zero — đó là lý do cần KEDA cho `worker`.
- Trải đều dùng `topologySpreadConstraints` theo `kubernetes.io/hostname`.

---

### Tiếp theo

Loạt bài tiếp tục trên smartapp (mỗi bài độc lập). Ở Lab 08, cơ sở dữ liệu của smartapp chưa có HA hay backup — bạn sẽ vận hành postgres bằng Operator, PITR và Velero.
