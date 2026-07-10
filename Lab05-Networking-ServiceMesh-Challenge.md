# Lab 05 — Challenge: "Luồng bị chặn & traffic phơi trần"

> **Thời gian ước lượng: ~110 phút**
> **Đầu vào:** cluster có Cilium (từ Intro) + namespace `smartapp` (`web`, `redis`). *(Bài độc lập.)*
> **Đầu ra:** luồng đã thông + một policy L7 + mTLS giữa các service + canary 90/10.
>
> ⚠️ Làm **bài hướng dẫn `Lab05-Networking-ServiceMesh.md` trước** (phần học).

---

## Bối cảnh

`web` đột nhiên **không truy cập được** từ bên trong cluster, và toàn bộ traffic service-to-service vẫn đang **clear text**. Có một NetworkPolicy ai đó để lại đang âm thầm **drop** một luồng — nhưng log ứng dụng không nói gì.

Nhiệm vụ: dùng **Hubble** để thấy luồng bị drop và sửa nó; siết truy cập tới `web` ở tầng **L7**; bật **mTLS** bằng service mesh; rồi chạy **canary** 90/10 cho một phiên bản mới.

## Thông tin đang có

- Cilium đã là CNI (từ Intro) — KHÔNG cài lại; chỉ bật tính năng và thêm mesh.
- Có một `CiliumNetworkPolicy` lỗi đang chặn truy cập tới `web` (do setup thả vào).
- `web` = podinfo (có `/version` để phân biệt phiên bản khi canary).

---

## Mục tiêu (trạng thái cuối được chấm)

1. **Thông luồng.** Tìm policy đang drop (qua Hubble), sửa để `web` truy cập được trở lại trong cluster.
2. **Policy L7.** Một `CiliumNetworkPolicy` ở tầng HTTP cho `web` (vd chỉ cho `GET`).
3. **mTLS service-to-service.** Bật mã hóa mTLS cho `smartapp` — **khuyến nghị cho bài chấm này: Cilium mutual-auth (SPIRE) hoặc WireGuard** (cùng datapath với L7 policy).
   > ⚠️ **KHÔNG bật Istio ambient trên chính `smartapp`.** ztunnel bọc traffic bằng HBONE nên L7 CiliumNetworkPolicy (Mục 2) redirect sang Envoy sẽ không parse được HTTP → **drop** → Mục 1 (`web` truy cập được) tạch dù policy đúng. Muốn thử ambient thì làm ở **namespace demo riêng**, không phải ns đang chấm. Sidecar (Linkerd/Istio classic) chấp nhận được nhưng KHÔNG phải hướng của bài (sidecarless-first).
4. **Canary.** Triển khai `web-v2` và chia traffic 90/10 (ưu tiên Gateway API `HTTPRoute`; chấp nhận Istio VirtualService).

## `answers.env`

```env
CULPRIT_POLICY="..."       # tên policy đang chặn luồng mà bạn tìm ra
```

## Tự chấm

```bash
./graders/grade05.sh smartapp
```

Lặp lại tới khi **mọi mục REQUIRED đều PASS**.

## Ràng buộc & gợi ý mức cao (không phải lời giải)

- Quan sát theo **identity** (nhãn pod) trong Hubble, không theo IP.
- L7 policy là thứ NetworkPolicy chuẩn (L3/L4) không làm được — phải dùng `CiliumNetworkPolicy` với `rules.http`.
- mTLS "trong suốt" với app: mesh lo TLS, không sửa code podinfo.
- **Đặt yêu cầu `authentication.mode: required` trên đường `web→redis`** (ingress của `redis`), ĐỪNG đặt trên ingress của `web` — nếu không, probe `GET /healthz` của grader có thể vướng handshake mutual-auth và trượt Mục 1.
- Chỉ chọn **một** datapath cho toàn bộ luồng: trộn Cilium L7 với ztunnel/ambient trên cùng ns sẽ xung đột (xem cảnh báo Mục 3).

---

### Tiếp theo

Loạt bài tiếp tục trên smartapp (mỗi bài độc lập). Ở Lab 06, bạn sẽ tự động hóa chính canary này bằng GitOps + Argo Rollouts có cổng phân tích metrics.
