# Lab 08 — Challenge: "Diễn tập khôi phục thảm họa (DR)"

> **Thời gian ước lượng: ~100 phút**
> **Đầu vào:** cluster có Longhorn (StorageClass `longhorn`, hỗ trợ snapshot) + `smartapp`. CNPG/Velero pre-baked + S3/MinIO. *(Bài độc lập.)*
> **Đầu ra:** dữ liệu đã mất được **khôi phục** sang một namespace mới và **xác minh** đầy đủ.
>
> ⚠️ Làm **bài hướng dẫn `Lab08-Stateful-Databases.md` trước** (phần học).

---

## Bối cảnh

Đây **không** phải bài "tự dựng HA + backup từ đầu" như phần hướng dẫn. Mọi thứ *đã* được dựng: một cơ sở dữ liệu CNPG có dữ liệu, và một **bản backup off-cluster đã được tạo**. Rồi **thảm họa xảy ra** — database bị xóa, dữ liệu trong namespace gốc mất sạch.

Đây là phép thử thật của mọi chiến lược backup: **bạn có thực sự khôi phục được không?** (Nguyên tắc vàng Bài 01: backup chưa từng restore = không có backup.) Nhiệm vụ: thực hiện một cuộc **diễn tập DR** — khôi phục từ backup sang một namespace mới và chứng minh dữ liệu trở lại nguyên vẹn.

## Thông tin đang có

- Một Velero backup tên `smartapp-bk-pre` đã tồn tại (chụp `smartapp` *trước* khi mất dữ liệu).
- `smartapp-db` (CNPG) trong namespace gốc đã **bị xóa** — dữ liệu không còn ở đó.
- Velero + MinIO + StorageClass hỗ trợ snapshot đã pre-baked.

> 🔍 Tự kiểm: `kubectl get backups.velero.io -A` (hoặc `velero backup get`) để thấy backup; bảng `t` có 5 dòng trước thảm họa.

---

## Mục tiêu (trạng thái cuối được chấm)

1. **Có restore thành công.** Một Velero `Restore` ở trạng thái `Completed`.
2. **DB hồi sinh.** CNPG `smartapp-db` chạy trong namespace đích `smartapp-dr` (có `currentPrimary`).
3. **Dữ liệu đầy đủ.** Bảng `t` trong DB đã khôi phục có **đúng số dòng như trước thảm họa** (5).

## `answers.env`

```env
ROWS_BEFORE=...            # số dòng trong bảng t trước thảm họa (tự xác định từ backup)
ROWS_AFTER=...             # số dòng sau khi khôi phục (phải khớp ROWS_BEFORE)
```

## Tự chấm

```bash
./graders/grade08.sh smartapp
```

Lặp lại tới khi **mọi mục REQUIRED đều PASS**.

## Ràng buộc & gợi ý mức cao (không phải lời giải)

- Trọng tâm là **khôi phục có kiểm chứng**, không phải dựng lại bằng tay rồi gõ lại dữ liệu — đó không phải DR.
- Velero hỗ trợ `--namespace-mappings` để restore sang namespace khác (tránh đè lên môi trường gốc khi diễn tập).
- Sau restore, CNPG operator sẽ nhận lại `Cluster` CR + PVC; chờ cluster `Ready` rồi truy vấn `SELECT count(*)`.
- **Mở rộng (BONUS):** khôi phục cả namespace gốc, và lên lịch backup định kỳ (`velero schedule`) + ghi diễn tập restore vào Ops-Checklist mục D.

---

### Tiếp theo

Đây là challenge cuối trước Capstone (Bài 09): bạn sẽ kế thừa một cụm bị gài **nhiều lỗi đa lớp cùng lúc** (mỗi học viên một biến thể) và phải chẩn đoán & khắc phục dưới áp lực thời gian, kèm postmortem.
