# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: Chưa được cung cấp.
- Thành viên: xem `TEAMMATES.md` (họ tên/MSSV, vai trò từng lượt).
- Trạng thái: `provided-results` — phân tích kết quả validation có sẵn, không tự chạy inference trong phiên này.
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Validation ghi nhận chạy ngày 01/10/2026 trên ThinkPad Linux amd64 và Mac Apple Silicon arm64; không nêu tên người chạy hoặc giờ cụ thể. Thành viên nhóm hiện tại chưa chạy.
- Image tag và image ID; phiên bản repo: `day13-pointpillars:lc-20261001-amd64`; image ID `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`; revision gói đã chạy `0831856d921609312d42c7582c366e5a311bb7b1` (manifest đánh dấu working tree lúc đóng gói là dirty).
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp: KITTI Student `student-prelabel-amd64/input/demo.pcd`, frame `demo`, 17.238 điểm; chỉ dùng trong gói Student cho học thuật phi thương mại, không phải dữ liệu Robotaxi. SHA256 `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`.
- Checkpoint: PointPillars KITTI `epoch_160.pth`; SHA256 `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`.
- Phạm vi: front-window; score threshold `0.3`; giữ cùng checkpoint và score cho A/B/C.
- Giả định kênh thứ tư/intensity và nguồn z_ground: Reflectance nguồn đã bị loại khỏi PCD; adapter dùng kênh hằng theo class, RGB=0 chỉ là placeholder, không phải intensity khôi phục. `z_ground` được ước lượng từ PCD; giá trị riêng từng lượt không có trong tài liệu kết quả hiện có. PCD được dịch z `+1.73 m` trước khi chạy.

## Ba lượt inference thật

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | Không có trong validation; cần summary.csv/JSON gốc để xác minh. | `run-A/boxes-demo-delta-0-voxel-0.16.json`, `side-demo-delta-0-voxel-0.16.png`, `summary.csv` (tên theo runner; file không có trong worktree hiện tại). | Validation ghi nhận tổng 1 hộp trên cả hai kiến trúc đã thử; không ghi class/vị trí hộp A. |
| B | 1.73 | 0.16 | 13 | Không có trong validation; cần summary.csv/JSON gốc để xác minh. | `run-B/boxes-demo-delta-1.73-voxel-0.16.json`, `side-demo-delta-1.73-voxel-0.16.png`, `summary.csv` (tên theo runner; file không có trong worktree hiện tại). | Validation ghi nhận 10 `vehicles`, 2 `pedestrian`, 1 `two-wheels` trên cả hai kiến trúc đã thử. Đây là baseline được helper dùng để tạo các ca QC. |
| C | 1.73 | 0.32 | 6 | Không có trong validation; cần summary.csv/JSON gốc để xác minh. | `run-C/boxes-demo-delta-1.73-voxel-0.32.json`, `side-demo-delta-1.73-voxel-0.32.png`, `summary.csv` (tên theo runner; file không có trong worktree hiện tại). | Validation ghi nhận 6 `pedestrian` trên cả hai kiến trúc đã thử; prediction hash có thể khác giữa kiến trúc. |

- A/B: Số hộp được ghi nhận đổi từ 1 lên 13 khi dịch input z trước inference. Đây là hai lần inference trên hai input khác nhau, không phải cộng/trừ một hằng số cho cùng output; model có thể đổi số hộp và vị trí. Không có JSON gốc nên chưa thể so mean_z, từng class hay độ lệch từng hộp.
- B/C: Khi pillar XY đổi từ 0.16 m lên 0.32 m, số hộp giảm từ 13 xuống 6 và phân bố class được ghi nhận chuyển từ 3 class ở B sang chỉ `pedestrian` ở C. Chưa đủ bằng chứng chọn cấu hình tốt hơn: không có reference/ground truth, số hộp nhiều hơn không đồng nghĩa chính xác hơn, và validation không công bố kết quả theo từng hộp.
- Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào? ROI là cửa sổ phía trước; vật ngoài ROI không được xem là miss trong lượt này. Side là hình chiếu x-z nên các vật ở y khác nhau có thể chồng lên nhau; không dùng Side một mình để kết luận miss hoặc yaw. Cần đối chiếu Top/Front và camera khi được cấp.
- JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp? Không import bất kỳ prediction KITTI hoặc `case-*.json` vào CVAT/ frame Robotaxi. Các output JSON/Side/CSV gốc không có trong worktree; cần lấy đúng bộ output từ kênh private, xác nhận frame/config/hash và rà hình học qua nhiều view. Prediction cũng không phải ground truth.

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0/13 | 0 m so với prediction B | Không; đây là bản sao prediction B. | Giữ làm đối chứng pipeline; không coi là nhãn đúng. | Helper tạo bản sao không sửa box; source là prediction B gồm 13 hộp theo validation. File ca không có trong worktree hiện tại. |
| case-batch-z | 13/13 | Mỗi hộp bị trừ `z_ground + 1.73 m` (giá trị số cụ thể cần manifest/output helper). | Không; helper chỉ đổi z. | Dừng batch, báo kiểm tra phép đổi hệ tọa độ/pipeline; không sửa tay từng hộp. | Helper áp cùng offset cho mọi box từ prediction B; cần `manifest.json` của ca để xác nhận offset số. |
| case-one-box-z | 1/13 | Một hộp bị trừ `z_ground + 1.73 m`; 12 hộp còn lại không đổi. | Không; helper chỉ đổi z của box đầu tiên. | Kiểm hộp bị lệch qua nhiều view; chưa quy kết lỗi cả pipeline. | Helper áp offset cho box đầu tiên từ prediction B; cần `manifest.json`/JSON ca để xác nhận offset số và box cụ thể. |

Ghi rõ helper tạo biến đổi có chủ đích từ prediction B, không phải kết quả inference riêng hoặc nhãn đúng. Validation xác nhận B có đủ hộp để tạo ba ca; các file ca/manifest không được lưu trong worktree hiện tại.

## Nhận xét cá nhân

**Lê Trung Toán (2A202602203):** Vai trò là phân tích kết quả validation có sẵn, không vận hành máy và không tự chạy inference. Quan sát: số hộp B/C được validation ghi nhận lần lượt là 13 và 6; A/B là 1 và 13 (tham chiếu `bundle/VALIDATION.md`, không có file output gốc trong worktree). Phép đổi thuận là `z_model = z_source - z_ground - delta`; đổi ngược là `z_source = z_model + z_ground + delta`, nên không cộng bù thêm lần nữa vào JSON đã ở hệ nguồn. Với `case-batch-z`, dừng batch và yêu cầu kiểm transform; với `case-one-box-z`, kiểm riêng hộp qua nhiều view. Chưa chắc mean_z, `z_ground` cụ thể, class/vị trí hộp A, sai khác theo từng box và chất lượng tương đối của B/C vì thiếu JSON/CSV/ảnh và ground truth. Chưa tự thực hiện lượt chạy; cần chạy bổ sung trên máy Linux Docker native được xác nhận trước khi ghi nhận là đã thực hành.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca: Chỉ có thể xác nhận quyền dùng PCD KITTI Student theo tài liệu/giấy phép trong repo; quyền và ca Robotaxi do LC xác nhận riêng.
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung: Bản này chỉ phân tích kết quả validation (`provided-results`); cần một lượt chạy thật được ghi nhận.
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT: Validation mô tả các lượt và QC chạy thành công, nhưng output gốc không có trong worktree để LC đối chiếu; không đưa prediction/case vào CVAT.
- Nhận xét từng thành viên và quyết định dừng pipeline: Chưa được LC ghi nhận.
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do: Chưa được LC quyết định; cần nộp output gốc và hoàn thành lượt thực hành thật trước khi xác nhận.
