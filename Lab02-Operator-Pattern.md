# Lab 02 — Xây Operator với Kubebuilder

> Hệ thống xuyên suốt: **smartapp**. Trong lab này ta xây một Operator quản lý thành phần **web** (podinfo) của smartapp qua một CRD `WebApp`.
> Mục tiêu: hoàn thiện hàm `Reconcile` để Operator tự tạo/duy trì Deployment + Service từ một đối tượng `WebApp`.

**Thời lượng:** ~60 phút · **Yêu cầu:** Go ≥ 1.21, `kubebuilder` ≥ 4.x, `make`, Docker, quyền `kubectl` vào cụm. Kiến thức Go cơ bản.

> 🧑‍🏫 **Ghi chú timebox (giảng viên):** **Phát sẵn repo đã scaffold** (mục 0 đã chạy trước) để học viên vào thẳng BT2 (viết Reconcile). Không bắt học viên chạy `kubebuilder init` tại lớp trừ khi còn nhiều thời gian. Có lời giải mẫu ở cuối — nếu một học viên kẹt quá 25', đưa lời giải để không trễ cả lớp.

---

## 0. Repo scaffold sẵn

Các lệnh dưới đây **đã được chạy sẵn**; kết quả đóng gói thành `webapp-operator/` phát cho học viên. (In ra đây để học viên hiểu repo từ đâu mà có.)

```bash
mkdir webapp-operator && cd webapp-operator
go mod init smartapp.io/webapp-operator
kubebuilder init --domain smartapp.io --repo smartapp.io/webapp-operator
kubebuilder create api --group web --version v1 --kind WebApp --resource --controller
```

Sau bước trên, `api/v1/webapp_types.go` và `controllers/webapp_controller.go` đã được điền **phần khung** (xem dưới). `WebAppSpec/Status` và phần thân `Reconcile` để trống có chủ đích cho học viên hoàn thiện.

`api/v1/webapp_types.go` (đã điền sẵn spec/status):
```go
type WebAppSpec struct {
    // +kubebuilder:validation:Required
    Image string `json:"image"`
    // +kubebuilder:validation:Minimum=1
    Replicas int32 `json:"replicas"`
    Host     string `json:"host,omitempty"`
}
type WebAppStatus struct {
    ReadyReplicas int32  `json:"readyReplicas"`
    Phase         string `json:"phase"`
}
// +kubebuilder:subresource:status
// +kubebuilder:printcolumn:name="Replicas",type=integer,JSONPath=`.spec.replicas`
// +kubebuilder:printcolumn:name="Phase",type=string,JSONPath=`.status.phase`
```

---

## 1. BT1 — Cài CRD và chạy manager (khung rỗng)

```bash
cd webapp-operator
make manifests          # sinh CRD từ struct
make install            # đưa CRD WebApp vào cụm
kubectl api-resources | grep webapps    # xác nhận 'kind' mới đã tồn tại
make run                # chạy manager NGOÀI cụm (dev), giữ terminal này
```

Trong terminal khác, tạo một đối tượng WebApp:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: web.smartapp.io/v1
kind: WebApp
metadata:
  name: smartapp-web
  namespace: smartapp
spec:
  image: stefanprodan/podinfo:6.7.0
  replicas: 3
  host: smartapp.local
EOF

kubectl -n smartapp get webapp smartapp-web
kubectl -n smartapp get deploy    # CHƯA có gì — Reconcile còn rỗng
```
**Kết quả mong đợi:** `kubectl get webapp` hiển thị object (kèm cột Replicas/Phase), nhưng **chưa có Deployment** — vì `Reconcile` chưa làm gì. Đây là điểm xuất phát của BT2.

---

## 2. BT2 — Viết `Reconcile` (trọng tâm, timebox ≤ 90')

Mở `controllers/webapp_controller.go`. Điền thân hàm `Reconcile` để: lấy WebApp → dựng Deployment podinfo mong muốn → đặt ownerReference → tạo/cập nhật idempotent → cập nhật status.

```go
func (r *WebAppReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := log.FromContext(ctx)

    // 1) Lấy CR; nếu đã bị xóa thì bỏ qua
    var app webv1.WebApp
    if err := r.Get(ctx, req.NamespacedName, &app); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 2) Dựng Deployment mong muốn từ spec
    dep := &appsv1.Deployment{
        ObjectMeta: metav1.ObjectMeta{Name: app.Name, Namespace: app.Namespace},
    }
    _, err := ctrl.CreateOrUpdate(ctx, r.Client, dep, func() error {
        dep.Spec.Replicas = &app.Spec.Replicas
        labels := map[string]string{"app": app.Name}
        dep.Spec.Selector = &metav1.LabelSelector{MatchLabels: labels}
        dep.Spec.Template.ObjectMeta.Labels = labels
        dep.Spec.Template.Spec.Containers = []corev1.Container{{
            Name:  "web",
            Image: app.Spec.Image,
            Ports: []corev1.ContainerPort{{ContainerPort: 9898}},
        }}
        // ownerReference: xóa WebApp -> Deployment tự bị dọn
        return ctrl.SetControllerReference(&app, dep, r.Scheme)
    })
    if err != nil {
        return ctrl.Result{}, err
    }

    // 3) Cập nhật status
    app.Status.ReadyReplicas = dep.Status.ReadyReplicas
    if dep.Status.ReadyReplicas == app.Spec.Replicas {
        app.Status.Phase = "Ready"
    } else {
        app.Status.Phase = "Progressing"
    }
    if err := r.Status().Update(ctx, &app); err != nil {
        return ctrl.Result{}, err
    }
    log.Info("reconciled", "webapp", app.Name, "phase", app.Status.Phase)
    return ctrl.Result{}, nil
}
```

Đừng quên RBAC marker (ngay trên hàm) để controller được phép thao tác Deployment:
```go
// +kubebuilder:rbac:groups=web.smartapp.io,resources=webapps,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=web.smartapp.io,resources=webapps/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=apps,resources=deployments,verbs=get;list;watch;create;update;patch;delete
```

Và khai báo "owns" trong `SetupWithManager` để watch cả Deployment con:
```go
func (r *WebAppReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&webv1.WebApp{}).
        Owns(&appsv1.Deployment{}).
        Complete(r)
}
```

Chạy lại:
```bash
make manifests && make install    # cập nhật RBAC/CRD
make run
kubectl -n smartapp get deploy,pods -l app=smartapp-web
kubectl -n smartapp get webapp smartapp-web   # Phase chuyển Ready
```
**Kết quả mong đợi:** một Deployment `smartapp-web` 3 replica podinfo được tạo; `WebApp.status.phase` chuyển `Ready`.

---

## 3. BT3 — Self-heal & cập nhật spec

Chứng minh tính "tự lành" của reconciliation loop:
```bash
# Xóa Deployment con thủ công
kubectl -n smartapp delete deploy smartapp-web
# Quan sát: Operator tạo lại gần như tức thì (vì Owns + watch)
kubectl -n smartapp get deploy -w
```
**Kết quả mong đợi:** Deployment xuất hiện lại mà không cần can thiệp — Operator phản ứng với sự kiện xóa object con.

Đổi spec và xem Operator điều hòa:
```bash
kubectl -n smartapp patch webapp smartapp-web --type=merge -p '{"spec":{"replicas":5}}'
kubectl -n smartapp get deploy smartapp-web   # READY tiến tới 5/5
```

**Câu hỏi suy ngẫm:** vì sao xóa Deployment con lại kích hoạt Reconcile? *(Gợi ý: `.Owns(&Deployment{})` khiến controller watch các con thuộc sở hữu của WebApp.)*

---

## 4. BT4 — Cascade delete (ownerReference)

```bash
kubectl -n smartapp delete webapp smartapp-web
kubectl -n smartapp get deploy,pods -l app=smartapp-web   # tự biến mất
```
**Kết quả mong đợi:** xóa CR `WebApp` → Garbage Collector tự dọn Deployment + Pod con nhờ ownerReference. Bạn **không** viết code xóa nào.

---

## 5. (Nâng cao — lab nhanh thì làm thêm)

- **Tạo thêm Service:** mở rộng Reconcile dựng một `Service` cho web (port 9898) cũng với ownerReference; xác minh cascade delete dọn cả Service.
- **Finalizer:** thêm finalizer `webapp.smartapp.io/cleanup`, log "dọn tài nguyên ngoài cụm" trước khi cho xóa CR; quan sát object kẹt ở `Terminating` cho tới khi gỡ finalizer.
- **Status điều kiện:** dùng `meta.SetStatusCondition` để ghi condition `Available=True/False` thay vì chuỗi Phase đơn giản (chuẩn production).
- **Deploy thật:** `make docker-build docker-push IMG=...` rồi `make deploy` để chạy Operator **trong cụm** thay vì `make run`.

---

## 6. Tổng kết & nối sang Bài 03

- Bạn đã xây một Operator hoạt động: CRD `WebApp` + controller điều hòa Deployment podinfo của smartapp.
- Bạn đã thấy ba điều cốt lõi vận hành: Reconcile idempotent, self-heal (watch object con), và cascade delete (ownerReference).
- **Bài 03:** chuyển từ "điều khiển" sang "nhìn thấy" — dựng hệ quan sát (metrics/logs/traces) cho chính smartapp, gồm cả podinfo `/metrics` mà ta vừa triển khai.

### Checklist hoàn thành
- [ ] CRD `WebApp` cài được; `kubectl get webapp` hiển thị cột Replicas/Phase.
- [ ] Reconcile tạo Deployment podinfo đúng image/replicas từ spec.
- [ ] Xóa Deployment con → Operator tạo lại (self-heal).
- [ ] Đổi `replicas` trong spec → Deployment điều chỉnh theo.
- [ ] Xóa `WebApp` → Deployment/Service con tự bị dọn (cascade).
