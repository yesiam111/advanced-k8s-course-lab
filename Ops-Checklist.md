# Sổ tay vận hành Kubernetes — Day-2 Ops Checklist (bản mẫu)

---

## 1. Mindset: vì sao admin cần checklist

- **Giảm tải nhận thức.** Khi xử lý sự cố lúc 3 giờ sáng, trí nhớ là thứ kém tin cậy nhất. Checklist gánh phần "nhớ" để bạn dồn sức cho phần "suy nghĩ".
- **Chống bỏ sót dưới áp lực.** Phần lớn sự cố nghiêm trọng không đến từ điều khó, mà từ một bước cơ bản bị quên (quên gia hạn chứng chỉ, quên kiểm tra snapshot etcd có restore được không).
- **Chuyển giao kinh nghiệm.** Checklist biến kiến thức nằm trong đầu một người thành tài sản chung của cả đội — người mới đọc checklist là tiếp nhận được hàng năm kinh nghiệm vận hành.

Nguyên tắc thiết kế: **mỗi mục trong checklist nên truy được về một nhóm sự cố thực tế.** Nếu không giải thích được "mục này phòng chống điều gì", hãy bỏ nó đi.

---

## 2. Checklist Day-2 Ops

Định dạng mỗi mục: `[ ] Việc cần làm` — **Vì sao** (gắn với nhóm sự cố / bài học) — *Công cụ / lệnh*.

### A. Sức khỏe cụm — hằng ngày
- `[ ]` Tất cả node `Ready`, không node nào disk/memory-pressure — **Vì sao:** node NotReady âm thầm làm pod dồn lên node còn lại → quá tải dây chuyền (Bài 7) — *`kubectl get nodes`, k9s*
- `[ ]` Không có pod trong trạng thái `CrashLoopBackOff` / `ImagePullBackOff` / `Pending` quá ngưỡng — **Vì sao:** dấu hiệu sớm của config sai, hết tài nguyên, hoặc registry lỗi (Bài 9) — *`kubectl get pods -A`, Popeye*
- `[ ]` Control-plane (apiserver/etcd/scheduler/controller-manager) khỏe, etcd còn quorum — **Vì sao:** mất quorum etcd = mất khả năng ghi toàn cụm (Bài 1) — *`kubectl get cs`, etcdctl endpoint health*
- `[ ]` Không có cảnh báo nghiêm trọng đang mở trong Alertmanager — **Vì sao:** alert bị bỏ qua tích tụ thành sự cố lớn (Bài 3) — *Grafana / Alertmanager*

### B. Định kỳ — hằng tuần / tháng
- `[ ]` Rà soát PV/PVC mồ côi và dung lượng còn trống của SAN/NAS/storage appliance — **Vì sao:** hết storage làm pod stateful không mount được (Bài 8) — *`kubectl get pv`, Admin UI*
- `[ ]` Quét cấu hình lệch best-practice (request/limit thiếu, probe thiếu, chạy root) — **Vì sao:** nợ kỹ thuật cấu hình là nguồn sự cố âm ỉ (Bài 4, 7) — *Polaris, Popeye*
- `[ ]` Rà soát RBAC dư quyền và ServiceAccount không dùng — **Vì sao:** least-privilege trôi dạt theo thời gian (Bài 4) — *`kubectl-who-can`, rbac-tool*
- `[ ]` Kiểm tra API/CRD sắp deprecated trước version kế tiếp — **Vì sao:** manifest dùng API cũ sẽ vỡ khi upgrade (Bài 1, 6) — *kubent, pluto*

### C. Trước & sau nâng cấp cụm
- `[ ]` Đọc release notes, xác định version skew cho phép — **Vì sao:** nhảy version sai gây control-plane không tương thích kubelet (Bài 1) — *docs*
- `[ ]` Quét deprecated API và sửa manifest **trước** khi nâng — **Vì sao:** tránh workload vỡ ngay sau upgrade (Bài 1) — *kubent, pluto*
- `[ ]` Có snapshot etcd mới **và đã thử restore** — **Vì sao:** snapshot chưa từng restore = không có backup (Bài 1) — *etcdctl snapshot*
- `[ ]` Nâng cấp theo thứ tự control-plane → worker, drain từng node — **Vì sao:** drain sai làm gián đoạn dịch vụ đột ngột (Bài 1, 5) — *kubeadm, `kubectl drain`*
- `[ ]` Sau nâng cấp: chạy conformance + smoke test — **Vì sao:** xác nhận cụm vẫn đúng chuẩn trước khi trả về production — *Sonobuoy*

### D. Backup & khôi phục thảm họa
- `[ ]` Lịch snapshot etcd chạy đúng, có giám sát — **Vì sao:** mất etcd không backup = dựng lại cụm từ đầu (Bài 1)
- `[ ]` Backup namespace quan trọng theo lịch, **diễn tập restore định kỳ** — **Vì sao:** chỉ backup mà không diễn tập là rủi ro ẩn (Bài 8)
- `[ ]` Chính sách snapshot volume cho DB stateful + thử PITR — **Vì sao:** mất dữ liệu DB là sự cố không thể đảo ngược (Bài 8)

### E. Bảo mật & tuân thủ
- `[ ]` Pod Security Admission đang thực thi (Restricted/Baseline) ở namespace cần — **Vì sao:** chặn pod đặc quyền ngay từ admission (Bài 4)
- `[ ]` Quét CIS benchmark, xử lý phát hiện mới — **Vì sao:** cấu hình cụm trôi khỏi chuẩn theo thời gian (Bài 4) — *kube-bench*
- `[ ]` Quét lỗ hổng image đang chạy, chặn CVE nghiêm trọng — **Vì sao:** image cũ tích lũy CVE (Bài 4) — *Trivy / Trivy-Operator*
- `[ ]` Secrets không nằm trần trong manifest/Git; chứng chỉ chưa hết hạn — **Vì sao:** rò rỉ secret và cert hết hạn là sự cố phổ biến (Bài 1, 4) — *ESO/Vault, kubeadm certs check*
- `[ ]` Audit logging bật và có cảnh báo cho API call nhạy cảm — **Vì sao:** không có audit thì không điều tra được sự cố bảo mật (Bài 4)

### F. Observability & SLO
- `[ ]` Metrics/logs/traces đang thu thập đủ, không đứt mạch — **Vì sao:** mất quan sát = mù khi sự cố xảy ra (Bài 3)
- `[ ]` Còn đủ error budget; cảnh báo dựa trên SLO chứ không chỉ ngưỡng thô — **Vì sao:** alert nhiễu làm đội bỏ qua alert thật (Bài 3, 9)
- `[ ]` Dashboard tương quan trace↔log↔metric dùng được khi cần — **Vì sao:** rút ngắn thời gian từ triệu chứng đến nguyên nhân (Bài 3, 9)

### G. Capacity & chi phí
- `[ ]` requests/limits sát thực tế, không over/under-provision — **Vì sao:** under → OOMKilled; over → đốt tiền (Bài 7) — *VPA recommender, OpenCost*
- `[ ]` Cluster Autoscaler còn dư đầu để scale; PriorityClass đúng cho workload trọng yếu — **Vì sao:** hết node lúc cao điểm gây pod Pending (Bài 7)
- `[ ]` Rà soát chi phí theo namespace, phát hiện lãng phí — **Vì sao:** chi phí trôi dạt nếu không ai theo dõi (Bài 7) — *OpenCost/Kubecost*

### H. Sẵn sàng sự cố
- `[ ]` Runbook cho các sự cố phổ biến luôn cập nhật và truy cập được — **Vì sao:** runbook cũ tệ hơn không có runbook (Bài 9)
- `[ ]` Công cụ debug sẵn sàng (kubectl debug, netshoot, crictl) và đội biết dùng — **Vì sao:** học cách dùng lúc đang cháy là quá muộn (Bài 9)
- `[ ]` Đã diễn tập chaos / break-fix gần đây — **Vì sao:** quy trình chưa diễn tập sẽ thất bại khi cần thật (Bài 9) — *LitmusChaos*

---

## 3. Công cụ giúp tự động hóa checklist

| Công cụ | Dùng cho mục | Vai trò |
|---------|--------------|---------|
| **k9s** | A, daily | Bảng điều khiển terminal cho thao tác hằng ngày |
| **Popeye** | A, B | Quét "vệ sinh" cụm: tài nguyên lỗi, cấu hình thiếu |
| **Polaris** | B, E | Chấm điểm cấu hình theo best-practice |
| **kube-bench** | E | Kiểm tra CIS benchmark |
| **Trivy / Trivy-Operator** | E | Quét lỗ hổng image & cấu hình |
| **kubent / pluto** | B, C | Phát hiện API/CRD deprecated trước upgrade |
| **Sonobuoy** | C | Kiểm tra conformance sau upgrade |
| **Velero** | D | Backup/restore namespace, diễn tập DR |
| **OpenCost/Kubecost** | G | Phân tích chi phí & right-sizing |
| **kubectl-who-can / rbac-tool** | B, E | Soát quyền RBAC |

Quy tắc: dùng công cụ để **kiểm tra**, dùng con người để **quyết định**. Công cụ trả về phát hiện; checklist quyết định phát hiện nào quan trọng và phải làm gì.

---

## 4. Cách tùy biến & tiến hóa checklist (phần quan trọng nhất)

Checklist mẫu này là **điểm xuất phát**, không phải đích đến. Cách giữ nó sống:

1. **Mỗi postmortem sinh ra một mục.** Sau mỗi sự cố, hỏi: "mục checklist nào đáng lẽ đã ngăn được điều này?" — nếu chưa có, thêm vào. Đây là cơ chế học hỏi chính.
2. **Mỗi mục phải giải thích được *vì sao*.** Nếu không ai nhớ một mục phòng chống điều gì, đó là ứng viên để loại bỏ — checklist phình to vô nghĩa sẽ bị bỏ qua.
3. **Version trong Git, có người sở hữu.** Checklist là code: review qua PR, ghi lại lý do mỗi thay đổi, gán người chịu trách nhiệm rà soát định kỳ (vd hằng quý).
4. **Phân biệt nhịp độ.** Tách rõ mục *hằng ngày / hằng tuần / theo sự kiện (upgrade)* — nhồi mọi thứ vào một danh sách dài làm không ai chạy nổi.
5. **Mã hóa dần thành tự động.** Mức trưởng thành cao nhất: một mục thủ công lặp lại nên trở thành kiểm soát tự động —
   - kiểm tra cấu hình → **Kyverno/Gatekeeper policy** (chặn ngay tại admission),
   - kiểm tra trước deploy → **CI gate** (Polaris/Trivy/kubent trong pipeline),
   - kiểm tra định kỳ → **CronJob + alert** (kube-bench/Popeye chạy tự động).
   Khi một mục đã được tự động hóa và tin cậy, gỡ nó khỏi checklist thủ công — checklist chỉ nên chứa thứ *con người vẫn phải tự làm*.

> Tóm lại: checklist bắt đầu là trí nhớ ngoài, trưởng thành thành chính sách tự động. Mục tiêu cuối không phải checklist dài hơn, mà là checklist **ngắn dần** vì ngày càng nhiều thứ đã được hệ thống tự lo.
