# Lab 01 — Challenge: "Kế thừa một cụm đang lỗi"

> **Thời gian ước lượng: ~45 phút**
> **Đầu vào:** cluster được cấp sẵn (kubeadm + Cilium + Longhorn) với smartapp đang chạy một phần.
> **Đầu ra:** cluster ở trạng thái healty + một "cổng" admission đang hoạt động + một snapshot etcd đã kiểm chứng + `baseline/answers.env` — dùng làm đầu vào cho Lab 02.
>
> ⚠️ Làm **bài hướng dẫn `Lab01-Cluster-Internals.md` trước** (phần học).

---

## Bối cảnh

Bạn vừa tiếp quản một cluster Kubernetes từ một đồng nghiệp đã nghỉ. Không có tài liệu bàn giao, chỉ có vài triệu chứng:

1. Một pod dịch vụ (`reporting`) **mãi không chạy** — kẹt ở trạng thái `Pending` và không ai biết vì sao.
2. **Không ai tạo được pod mới trong `smartapp`** — mọi `kubectl run`/`apply` pod đều bị một lỗi admission chặn lại. Đồng nghiệp cũ từng "thử làm bảo mật" nhưng để lại thứ gì đó cấu hình sai. Nghịch lý: cụm vừa **bị khóa** (không tạo được pod) lại vừa **không có kiểm soát đúng** (chưa có chính sách nào bắt buộc chuẩn của đội).
3. Có một file snapshot etcd được tạo, nhưng **chưa từng được thử khôi phục** — coi như chưa có backup.

Nhiệm vụ: chẩn đoán và đưa cụm về trạng thái healthy, thay "quả bom hẹn giờ" admission bằng một cổng kiểm soát **đúng và an toàn**, và **tự chứng minh** bạn khôi phục được etcd.

## Thông tin đang có

- Cụm với quyền `kubectl` cluster-admin; SSH + sudo vào ít nhất 1 control-plane node và 1 worker node.
- `crictl`, `etcdctl`, `openssl` có sẵn trên node.
- Namespace `smartapp` với `web` (podinfo) + `redis` đang chạy, và một pod `reporting` đang `Pending`.

> ⚠️ **An toàn:** thao tác trực tiếp với etcd rất nguy hiểm — **chỉ làm trên cụm lab**.

---

## Mục tiêu (trạng thái cuối được chấm)

1. **Hết Pending.** Chẩn đoán vì sao `reporting` không chọn node được (dựa vào đọc quyết định của scheduler qua Events) và đưa nó về `Running`. Không còn pod nào `Pending` trong `smartapp`.
2. **Gỡ quả bom admission.** Có một `ValidatingWebhookConfiguration` "di sản" đang chặn mọi pod mới trong `smartapp`: nó đặt `failurePolicy: Fail` nhưng trỏ tới một **service backend đã chết**, nên mọi lần gọi webhook đều lỗi → mọi CREATE pod bị từ chối. Tìm ra nó (`kubectl get validatingwebhookconfigurations`, soi `failurePolicy` + `clientConfig.service`) và **gỡ bỏ**. *Không được* tắt bừa mọi kiểm soát — chỉ gỡ đúng cái hỏng.
3. **Dựng cổng admission ĐÚNG.** Sau khi gỡ landmine, đăng ký một cơ chế admission (validating webhook **hoặc** Kyverno — tùy chọn) sao cho:
   - pod tạo trong `smartapp` **thiếu** label `team` → **bị từ chối**;
   - pod **có** label `team` → được chấp nhận.
   - ⚠️ Cổng của bạn **không được** lặp lại sai lầm cũ: loại trừ namespace hệ thống (`kube-system`), và nếu dùng webhook riêng thì backend phải sống (nếu không grader sẽ báo landmine).
4. **Diễn tập khôi phục etcd.** Trước khi diễn tập, gắn annotation `smartapp.io/lab01-canary=<chuỗi-bạn-chọn>` lên `deploy/web`. Snapshot → mô phỏng mất etcd → restore → xác minh cụm sống lại và annotation **còn nguyên** (bằng chứng "snapshot chưa restore ≠ backup").
5. **Khôi phục nội tại bằng tay** và điền `baseline/answers.env`:
   - `WEB_NODE` — tên node đang chạy một pod `web`.
   - `WEB_IMAGE_DIGEST` — digest `sha256:...` của image `web`, **lấy ở cấp node bằng `crictl`** (không lấy từ Dockerfile hay đoán).
   - `QUORUM_MIN` — số node etcd **tối thiểu** để giữ quorum trên cụm này.
   - `CANARY_TOKEN` — đúng chuỗi bạn đã gắn ở mục 4.

## `baseline/answers.env`

```env
WEB_NODE="..."
WEB_IMAGE_DIGEST="sha256:..."
QUORUM_MIN=...
CANARY_TOKEN="..."
```

## Tự chấm

```bash
./graders/grade01.sh smartapp
```

Lặp lại tới khi **mọi mục REQUIRED đều PASS**.

## Ràng buộc & gợi ý mức cao (không phải lời giải)

- **Triệu chứng "không tạo được pod" chỉ về admission, không phải scheduler.** Đọc thông báo lỗi khi `kubectl run` thất bại — nó nêu tên webhook. Rồi soi `kubectl get validatingwebhookconfiguration <tên> -o yaml`: `failurePolicy`, `clientConfig.service`, `namespaceSelector`.
- **Vì sao landmine nguy hiểm:** `failurePolicy: Fail` + backend chết = "fail-closed" → mọi request khớp rule bị chặn. Đây đúng là lỗi "tự khóa cứng cụm" mà bài học cảnh báo — nay bạn gặp nó thật.
- **Gỡ đúng, đừng phá diện rộng.** Chỉ xóa webhook hỏng; giữ nguyên phần còn lại của cụm.
- "Đọc quyết định scheduler" nghĩa là dùng **dữ liệu của cụm** (Events, `describe`) để biết pha nào loại node — không sửa mò.
- Annotation canary phải còn nguyên **sau** khi restore: đó là bằng chứng restore thật sự hoạt động.
- `crictl` cho bạn digest thật ở node — đây là góc nhìn bạn vẫn có kể cả khi API server gặp sự cố.

---

### Input cho Lab 02

Giữ cluster healhty + Admission Controller đang chạy. Ở Lab 02, bạn sẽ wrap tầng web trong một **Operator** (`WebApp` CR) tự vận hành — bắt đầu từ một `Reconcile()` đã **có sẵn lỗi** mà bạn phải gỡ.
