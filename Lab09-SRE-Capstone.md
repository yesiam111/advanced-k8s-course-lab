# Lab 09 — SRE, Xử lý sự cố & Capstone

> Bài tổng hợp toàn khóa trên hệ thống **smartapp**: chẩn đoán & chữa một cụm bị cố ý cài lỗi. Sửa bằng cách nào cũng được — như xử lý sự cố thật, chỉ kết quả cuối được chấm (bằng autograder).

**Thời lượng:** ~120 phút · **Hình thức:** cá nhân hoặc nhóm 2–3 · **Yêu cầu:** một cụm smartapp (lab/sandbox) đã được giảng viên "gài lỗi".

> 🧑‍🏫 **Chuẩn bị (giảng viên):** chạy autograder để gài lỗi trước buổi (mục 0). **Mỗi học viên một tổ hợp lỗi khác nhau** — không chỉ khác số mà khác cả LOẠI lỗi. Cho phép dùng `Ops-Checklist.md`. **Không phát danh mục lỗi cho học viên.**

---

## 0. Gài lỗi (giảng viên chạy trước)

> 🤖 Dùng autograder kèm theo. Nó chọn ngẫu nhiên một tổ hợp lỗi theo `student-id` (gồm lỗi đa lớp + mồi nhử), cài canary "good-posture" để phát hiện kiểu sửa ẩu, và ghi key để chấm. Chi tiết: `capstone-grader/README.md` và `capstone-grader/faults.sh`.

```bash
cd capstone-grader
./inject-faults.sh sv001 smartapp        # gài 7 lỗi ngẫu nhiên cho sv001
./inject-faults.sh sv002 smartapp
# (gỡ sạch khi cần: ./inject-faults.sh sv001 smartapp --clean)
```

> ⚠️ **Quan trọng:** keyfile trong `capstone-grader/keys/` là đáp án — **không** đưa học viên. Học viên chỉ nhận: một cụm hỏng + hợp đồng nộp bài + `Ops-Checklist.md`.

---

## Chẩn đoán & khắc phục (120')

Cụm smartapp của bạn đang hỏng theo **nhiều cách cùng lúc**. Số lượng, loại, và giá trị lỗi **khác nhau giữa các học viên**. Một số lỗi **đa lớp** (một nguyên nhân gốc gây nhiều triệu chứng) và một số là **mồi nhử** — triệu chứng trỏ sai chỗ. Không có danh sách lỗi. Việc của bạn là *điều tra*, không phải đoán.

### Luật chơi (bắt buộc)
1. **Phương pháp có hệ thống:** triệu chứng → giả thuyết → thu hẹp bằng *dữ liệu* → xác nhận nguyên nhân gốc → vá → kiểm chứng lại. Đoán mò không được tính.
2. **Thay đổi một thứ mỗi lần** và **ghi lại** ngay (tránh lặp ngõ cụt, biết đường quay lui).
3. **Sửa tối thiểu, có chủ đích.** KHÔNG "đốt sạch" (`delete ... --all`, gỡ mọi NetworkPolicy/PDB/policy, tắt mọi lan can bảo mật). Cụm có sẵn vài **kiểm soát ĐÚNG** phải được giữ nguyên — phá chúng bị **trừ điểm**.
4. **Sửa đúng tầng nguồn.** Nếu cụm do GitOps quản (ArgoCD self-heal), sửa tay sẽ bị hoàn tác — phải tìm và sửa đúng nguồn sự thật.

### Khởi động — bản đồ tình trạng
```bash
kubectl -n smartapp get pods,svc,endpoints,pvc -o wide
kubectl -n smartapp get events --sort-by=.lastTimestamp | tail -40
kubectl -n smartapp get netpol,pdb,resourcequota
kubectl get clusterpolicy,validatingwebhookconfiguration 2>/dev/null | grep -i capstone
kubectl get nodes
# Quan sát sâu hơn bằng chính công cụ đã học:
#  - Hubble (Bài 5) cho luồng/verdict mạng
#  - Grafana/Loki/Tempo (Bài 3) cho metric/log/trace
#  - kubectl debug + netshoot (Bài 9) cho DNS/kết nối
#  - argocd app get / list (Bài 6) nếu cụm do GitOps quản
```

### Kỷ luật chẩn đoán (gợi ý *cách nghĩ*, không phải đáp án)
- **Phân biệt triệu chứng với nguyên nhân.** "web không gọi được redis" có thể là redis chết, NetworkPolicy, DNS, hay Service selector — *bốn* nguyên nhân rất khác nhau. Đừng dừng ở triệu chứng đầu tiên.
- **Một thay đổi, nhiều hệ quả.** Nếu nhiều thứ hỏng cùng lúc, hỏi: có *một* nguyên nhân gốc nào giải thích tất cả không?
- **Pod `Ready` ≠ service dùng được.** Kiểm `endpoints`, không chỉ `get pods`.
- **Pod tạo được ≠ admission khỏe.** Có lỗi chỉ chặn pod *mới* (webhook/policy/quota) — `get pods` trông yên bình.
- **Có lỗi không hiện trong `get pods`** (latent): rò rỉ qua drain/rollout/backup. Đây là lúc `Ops-Checklist.md` cứu bạn — audit chủ động, đừng chỉ chữa cháy.

### Tiêu chí hoàn thành
- [ ] Mọi pod smartapp `Running`/`Ready`; không `Pending`/`CrashLoop`/`OOMKilled`/`ImagePull*`/`ConfigError`.
- [ ] `web` có endpoints; gọi được `redis` (và `postgres` nếu có) **bằng tên** (DNS hoạt động).
- [ ] Tạo được pod mới hợp lệ (admission/quota/webhook không chặn sai).
- [ ] PVC `Bound`; nếu có CNPG: có `currentPrimary`.
- [ ] Nếu có Prometheus Operator: ServiceMonitor của `web` scrape được (đúng tên port) — dashboard không "mù".
- [ ] **Giữ nguyên** các kiểm soát đúng (canary) — không đốt sạch.

### Chấm tự động (autograder)
Trỏ `kubectl` tới cụm học viên rồi chấm — soi cụm thật, mọi hạng mục kiểm chứng bằng script:
```bash
cd capstone-grader
./grade.sh sv001 smartapp
```
Grader chấm /100, **đạt khi ≥ 80 ĐIỂM**. Nó:
- chấm sức khỏe cụm + kết nối/DNS + từng lỗi đã gài (đối chiếu seed biến thể, chống chép);
- **trừ điểm** nếu canary good-posture bị xóa (chống "đốt sạch").

---

## Mở rộng (cho nhóm xong sớm — vẫn là việc thật, không phải phần thưởng)

- **Tự động hóa một mục checklist:** biến một lỗi vừa gặp thành Kyverno policy / CronJob cảnh báo để nó không tái diễn âm thầm.
- **Chaos:** cài LitmusChaos, chạy "pod-delete" trên web và chứng minh SLO không vỡ.
- **Đề thêm:** xin giảng viên gài thêm biến thể (`inject-faults.sh <id> smartapp 10`) với nhiều lỗi hơn.

---

## Tổng kết khóa học

Hành trình smartapp qua 9 bài: **internals → operator → observability → security → mesh → GitOps → scaling → data → SRE**.

Thông điệp cuối: vận hành Kubernetes **an toàn, bảo mật, tối ưu** là một kỷ luật liên tục — đo bằng SLO, giảm công sức bằng tự động hóa, và biến mỗi sự cố thành một cải tiến (checklist). Không có "xong"; chỉ có vận hành ngày càng tốt hơn.

### Checklist hoàn thành khóa
- [ ] Khôi phục cụm về khỏe mạnh **bằng chẩn đoán có dữ liệu**, giữ canary, đạt ≥80 điểm grader.
- [ ] Áp dụng phương pháp có hệ thống và `Ops-Checklist.md` xuyên suốt.
