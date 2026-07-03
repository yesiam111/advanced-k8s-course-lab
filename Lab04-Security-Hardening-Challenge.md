# Lab 04 — Challenge: "Bảo mật làm sập ứng dụng"

> **Thời gian ước lượng: ~90 phút**
> **Đầu vào:** cluster với namespace `smartapp` (`web` = podinfo). Kyverno pre-baked. *(Bài độc lập.)*
> **Đầu ra:** `web` vừa **chạy** vừa **tuân thủ** chuẩn bảo mật — mà **không** làm yếu hệ thống.
>
> ⚠️ Làm **bài hướng dẫn `Lab04-Security-Hardening.md` trước** (phần học).

---

## Bối cảnh

Đây **không** phải bài "tự dựng hardening từ đầu" như phần hướng dẫn. Tổ chức vừa bật một **chính sách bảo mật ở chế độ Enforce**, và ngay sau đó `web` **ngừng lên pod mới** — không ai biết vì sao. Bộ phận vận hành đang bị cám dỗ "gỡ chính sách cho xong". Nhiệm vụ của bạn ngược lại: **làm ứng dụng tuân thủ**, giữ nguyên lan can bảo mật.

Đây là tình huống kinh điển: bảo mật và tính sẵn sàng *xung đột*, và kỹ năng thật là sửa cho cả hai cùng đúng — không hi sinh cái này vì cái kia.

## Thông tin đang có

- `web` đang ở cấu hình **không tuân thủ** (chạy root, không `securityContext`, không seccomp).
- Một `ClusterPolicy` của tổ chức đang **Enforce** trong cluster. Nó là **kiểm soát đúng** — không được gỡ.
- Triệu chứng: `kubectl -n smartapp get deploy web` cho thấy `web` **không có pod sẵn sàng**; xem `kubectl -n smartapp describe rs` / sự kiện để thấy admission đang từ chối pod.

> 🔍 Tự điều tra: vì sao pod `web` bị từ chối? Chính sách nào? Nó đòi gì?

---

## Mục tiêu (trạng thái cuối được chấm)

1. **web chạy lại VÀ tuân thủ.** `web` có pod `Available`, đồng thời: `runAsNonRoot`, `readOnlyRootFilesystem`, `allowPrivilegeEscalation=false`, drop **ALL** capabilities, `seccompProfile: RuntimeDefault`.
2. **Không gỡ bảo mật.** Chính sách tổ chức (`org-require-seccomp`) vẫn **tồn tại và Enforce** sau khi bạn sửa.
3. **Chính sách còn hiệu lực.** Một pod thiếu seccomp tạo mới vẫn **bị từ chối** (chứng minh bạn không vô hiệu hóa nó).

> 💡 Nếu `readOnlyRootFilesystem` làm app crashloop: **mount `emptyDir`** vào thư mục cần ghi (vd `/tmp`), đừng tắt read-only.

## `answers.env`

```env
BLOCKER_POLICY="..."       # tên ClusterPolicy đang chặn web (tự tìm)
```

## Tự chấm

```bash
./graders/grade04.sh smartapp
```

Lặp lại tới khi **mọi mục REQUIRED đều PASS**.

## Ràng buộc & gợi ý mức cao (không phải lời giải)

- Cách sai (bị đánh trượt R2/R3): `kubectl delete clusterpolicy ...` để web lên. Đó là gỡ lan can, không phải sửa.
- Cách đúng: đọc message admission → biết policy đòi gì → bổ sung đúng cấu hình vào `web`.
- **Mở rộng (BONUS):** dựng thêm một `tenant-a` cô lập (Quota + LimitRange + default-deny + PSA restricted) như chuẩn production.

---

### Tiếp theo

Loạt bài tiếp tục trên smartapp (mỗi bài độc lập). Ở Lab 05, một luồng mạng đang bị chặn bí ẩn và traffic service-to-service vẫn ở dạng clear text — bạn sẽ dùng Hubble, thêm policy L7 và bật mTLS.
