# Lab 02 — Challenge: "Operator bàn giao đang lỗi"

> **Thời gian ước lượng: ~90 phút** (timebox phần code ≤ 60')
> **Đầu vào:** cluster với namespace `smartapp` và `web` (podinfo) đang chạy. *(Bài độc lập — không cần kết quả của Lab 01.)*
> **Đầu ra:** tầng web được quản bởi một `WebApp` CR tự vận hành (self-heal).
>
> ⚠️ Làm **bài hướng dẫn `Lab02-Operator-Pattern.md` trước** (phần học).

---

## Bối cảnh

Một đồng nghiệp bắt đầu wrap tầng `web` của smartapp vào một Operator để hết phải chăm tay, nhưng nghỉ giữa chừng và để lại một repo **chạy được nhưng có lỗi**. Operator build và start bình thường, nhưng nó **không tự vận hành đúng**: tạo `WebApp` mà Deployment không như mong đợi, xóa Deployment con thì không tự dựng lại, xóa `WebApp` thì rác còn sót.

Nhiệm vụ: tìm và gỡ các lỗi trong `Reconcile()` để Operator vận hành đúng vòng đời của `web`.

## Thông tin đang có

- Repo `webapp-operator/` đã `kubebuilder` scaffold sẵn:
  - `api/v1/webapp_types.go` — `WebAppSpec` (`Image`, `Replicas`, `Host`) + `WebAppStatus` (`ReadyReplicas`, `Phase`) đã điền sẵn.
  - `controllers/webapp_controller.go` — hàm `Reconcile()` **đã viết nhưng có lỗi cài sẵn**. Đây là nơi cần sửa.
- Go ≥ 1.21, `kubebuilder` ≥ 4.x, `make`, Docker, quyền `kubectl`. Kiến thức Go cơ bản.

> 💡 Đây là bài **debug-and-fix**, không phải viết từ đầu. Đừng `kubebuilder init` lại — chỉ sửa logic trong `Reconcile()` (và `SetupWithManager` nếu cần).

---

## Mục tiêu (trạng thái cuối được chấm)

1. **Cài & chạy Operator.** `make install` đưa CRD `WebApp` vào cluster; chạy controller (`make run` ngoài cụm, hoặc `make deploy`).
2. **Tạo đúng từ spec.** Apply một `WebApp` tên `smartapp-web` (image `stefanprodan/podinfo:6.7.0`, `replicas: 3`) → Operator tạo một Deployment đúng image và đúng số replicas.
3. **Self-heal.** Xóa Deployment con bằng tay → Operator dựng lại gần như tức thì.
4. **Hội tụ theo spec.** `patch` `spec.replicas` (vd 5) → Deployment đổi theo.
5. **Status đúng.** `WebApp.status.phase` chuyển `Ready` khi đủ replica sẵn sàng.
6. **Cascade delete.** Xóa `WebApp` → Deployment + pod con tự bị dọn (không viết code xóa nào).

## Tự chấm

```bash
./graders/grade02.sh smartapp
```

Operator phải đang chạy khi chấm. Lặp lại tới khi **mọi mục REQUIRED đều PASS**.

## Ràng buộc & gợi ý mức cao (không phải lời giải)

- `Reconcile()` phải **idempotent** — chạy lại nhiều lần không lỗi, không tạo trùng. Nghĩ tới `CreateOrUpdate` thay vì `Create` trần.
- Self-heal và cascade delete đều dựa trên **ownerReference** + việc controller `Owns(&Deployment{})` để watch con.
- `status.phase` không tự có — bạn phải cập nhật nó từ `Deployment.Status`.

---

### Tiếp theo

Loạt bài tiếp tục trên cùng hệ thống smartapp (nhưng mỗi bài độc lập). Ở Lab 03, smartapp bắt đầu lỗi rải rác mà bạn **không nhìn thấy gì** — bạn sẽ dựng observability (metrics/logs/traces) cho `web` (podinfo) và truy ra nguyên nhân.
