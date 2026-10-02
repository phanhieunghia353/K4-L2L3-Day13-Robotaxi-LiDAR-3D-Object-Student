# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: Cá nhân - Phan Hiệu Nghĩa
- Thành viên: xem `TEAMMATES.md` (Phan Hiệu Nghĩa - MSSV: 2A202602332; đảm nhận luân phiên các vai trò vận hành lệnh, kiểm cấu hình/JSON, quan sát hình học và ghi log).
- Trạng thái: `executed-by-group` (tự chạy trực tiếp trên máy cá nhân qua Docker Desktop CPU).
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Phan Hiệu Nghĩa; 2026-10-01 14:33 UTC+7; Windows 11 x86_64 (amd64), Intel/AMD CPU 4 cores / 4GB RAM container limit.
- Image tag và image ID; phiên bản repo:
  - Image Tag: `day13-pointpillars:lc-20261001-amd64`
  - Image ID: `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`
  - Repo revision: `0831856d921609312d42c7582c366e5a311bb7b1` (HEAD local: `f5f1de0`)
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp:
  - File: `data/demo.pcd` (mẫu KITTI frame 000008, 17,238 points)
  - PCD SHA256: `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`
  - Nơi chạy: Docker local trên máy cá nhân theo gói Student bundle được cấp phép (CC BY-NC-SA 3.0).
- Checkpoint: PointPillars KITTI có sẵn trong image `/opt/PointPillars/pretrained/epoch_160.pth` (SHA256: `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`).
- Phạm vi: front-window (`--from KITTI`, ROI: x in [0, 70.4], y in [-40, 40], z in [-3, 1]); score threshold: `0.3`.
- Giả định kênh thứ tư/intensity và nguồn z_ground: Kênh reflectance gốc bị loại bỏ trong file PCD KITTI Student, thay bằng RGB=0 và adapter kênh hằng số; `z_ground = 0.075 m` (ước lượng tự động từ cụm điểm mặt đường cục bộ quanh gốc tọa độ).

## Ba lượt inference thật

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 m | `run-A/boxes-demo-delta-0-voxel-0.16.json`<br>`run-A/side-demo-delta-0-voxel-0.16.png`<br>`run-A/summary.csv` | Khi `delta = 0`, không bù cao độ sensor (1.73m). Point cloud bị đưa vào mạng ở dải cao độ lệch hoàn toàn so với anchor được học của KITTI. Model chỉ nhận diện được đúng 1 hộp xe (`vehicles`: 1) với score thấp (0.322), bỏ sót hầu hết các đối tượng trong cảnh. |
| B | 1.73 | 0.16 | 13 | 1.034 m | `run-B/boxes-demo-delta-1.73-voxel-0.16.json`<br>`run-B/side-demo-delta-1.73-voxel-0.16.png`<br>`run-B/summary.csv` | Baseline chuẩn: Bù đúng cao độ sensor KITTI (`delta = 1.73m`). Điểm đám mây khớp với anchor 3D của mạng. Model phát hiện 13 hộp gồm 10 `vehicles`, 2 `pedestrian`, 1 `two-wheels`. Đáy các hộp bám sát mặt đường hợp lý ($mean\_z = 1.034 m$). |
| C | 1.73 | 0.32 | 6 | 1.091 m | `run-C/boxes-demo-delta-1.73-voxel-0.32.json`<br>`run-C/side-demo-delta-1.73-voxel-0.32.png`<br>`run-C/summary.csv` | Giữ `delta = 1.73m` nhưng tăng kích thước pillar lên 0.32m (gấp đôi cạnh, diện tích voxel gấp 4). Lưới voxel thô làm mất chi tiết hình học: toàn bộ 10 xe hơi biến mất, model phân loại nhầm toàn bộ 6 cụm điểm thành `pedestrian` (0 xe, 0 xe máy). |

- **A/B: thay input trước model có khác dịch cùng một hằng số cho output không? Vì sao?**
  - *Hoàn toàn khác nhau.*
  - Nếu chỉ dịch cùng một hằng số cho output sau inference ($z_{out} = z_{pred} + c$), số lượng hộp, phân loại class và confidence score sẽ được giữ nguyên 100%, chỉ có vị trí các hộp bị tịnh tiến lên/xuống theo trục $z$.
  - Ngược lại, thay đổi `delta` ở input trước inference làm thay đổi giá trị tọa độ thực tế của điểm đưa vào Pillar Feature Net. Mạng PointPillars sử dụng các 3D anchor boxes định sẵn ở các cao độ cố định để dự đoán class và bounding box regression. Khi điểm bị lệch cao độ (lượt A: $z_{model} = z_{source} - z_{ground} - 0$), các điểm không nằm trong receptive field của anchor phù hợp, khiến mạng không kích hoạt feature map tương ứng, dẫn tới mất gần như toàn bộ hộp (từ 13 hộp rơi xuống còn 1 hộp duy nhất).

- **B/C: thấy gì khi đổi pillar? Có đủ bằng chứng để nói cấu hình nào tốt hơn không?**
  - Khi tăng voxel size từ 0.16m lên 0.32m, số hộp giảm từ 13 xuống 6, và xuất hiện sự sai lệch phân loại trầm trọng: toàn bộ xe hơi biến mất, 100% đối tượng được dự đoán thành `pedestrian`.
  - Checkpoint này được huấn luyện đặc thù cho độ phân giải voxel 0.16m của KITTI. Khi đưa voxel 0.32m, biểu diễn đặc trưng không còn khớp với trọng số mạng. Cấu hình B (0.16m) phù hợp với pretrained model hơn nhiều. Tuy nhiên, việc B có 13 hộp không có nghĩa là mọi hộp của B đều hoàn hảo; ta vẫn cần đối chiếu từng hộp với ảnh camera và kiểm tra 4 góc nhìn trong CVAT.

- **Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào?**
  - *Giới hạn ROI:* Lệnh chạy inference chỉ quét vùng phía trước (Front ROI, $x \in [0, 70.4]$). Các vật thể nằm phía sau xe hoặc ngoài biên quét không xuất hiện hộp thì không được coi là model bỏ sót (false negative).
  - *Góc Side (hình chiếu $x-z$):* Đây là hình chiếu nén toàn bộ trục $y$ lại, khiến các xe đỗ song song hoặc ở các làn khác nhau bị đè lên nhau. Hình chiếu Side không thể dùng độc lập để xác định chiều rộng hoặc góc quay yaw (xe quay đầu $0^\circ$ hay $180^\circ$ trên Side view đều có footprint chiếu giống nhau). Cần kết hợp Top view ($x-y$) và ảnh camera cùng frame.

- **JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp?**
  - Cả 3 file JSON đều **chưa đủ cơ sở** để import trực tiếp làm ground truth vào CVAT:
    - JSON A sai nặng do tiền xử lý cao độ.
    - JSON C sai toàn bộ phân loại vật thể do kích thước voxel.
    - Ngay cả JSON B (kết quả baseline tốt nhất) cũng chỉ là pretrained prediction mang tính gợi ý. Checkpoint KITTI hoàn toàn không nhận diện được 2 class trong schema bài học là `Animal` và `Obstacle`. Hơn nữa, các hộp trong B cần được kiểm tra: đáy bám mặt đường cục bộ (local ground), loại bỏ các hộp hallucinate (nhiễu), kiểm tra hướng đầu xe (tránh ngược yaw $180^\circ$) và bổ sung các đối tượng bị che khuất một phần mà model bỏ sót.

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| `case-correct` | 0 / 13 | 0 m | Không đổi | **Kiểm từng hộp** | Giữ nguyên prediction gốc của lượt B ($mean\_z = 1.034 m$, $min\_z = 0.698 m$, $max\_z = 1.426 m$). Đáy các hộp nằm sát mặt đường cục bộ quanh xe ($z_{ground} \approx 0.075 m$). Tiến hành kiểm tra chi tiết từng hộp trên CVAT. |
| `case-batch-z` | 13 / 13 | Lệch đồng loạt $-1.805 m$ (`delta + z_ground`) | Không đổi | **DỪNG BATCH, kiểm tra pipeline** | Toàn bộ 13/13 hộp (100%) đều bị tụt sâu xuống lòng đất ($mean\_z = -0.771 m$, $min\_z = -1.107 m$, $max\_z = -0.379 m$). Lượng lệch đúng bằng $delta + z_{ground} = 1.73 + 0.075 = 1.805 m$. Đây là lỗi hệ thống do pipeline quên phép chuyển đổi ngược $z_{source} = z_{model} + z_{ground} + delta$. Không sửa tay từng hộp vì sẽ làm hỏng dữ liệu và tốn công vô ích; phải báo phụ trách pipeline tạo lại dữ liệu chuẩn. |
| `case-one-box-z` | 1 / 13 | Duy nhất 1 hộp lệch $-1.805 m$ | Không đổi | **Kiểm từng hộp, không dừng batch** | 12/13 hộp vẫn bám mặt đường chuẩn ($z \in [0.698, 1.426]$), chỉ có duy nhất hộp đầu tiên (ID 0) bị chìm xuống $z = -0.884 m$ (lệch đúng 1.805m). Đây là lỗi cá biệt của một đối tượng cụ thể (nhiễu điểm hoặc model predict lỗi cục bộ), không phải lỗi pipeline toàn bộ frame. Xử lý bằng cách chỉnh sửa hoặc xóa hộp lỗi đó trong CVAT. |

*Ghi chú:* Helper `pipeline-qc-cases.py` tạo các biến đổi có chủ đích từ prediction lượt B nhằm mục đích huấn luyện nhận diện lỗi hệ thống vs lỗi đơn lẻ; không phải kết quả inference độc lập và không phải ground-truth nhãn đúng.

## Nhận xét cá nhân

- **Họ tên & MSSV:** Phan Hiệu Nghĩa — 2A202602332
- **Vai trò đã thực hiện:** Thực hành độc lập, trực tiếp vận hành container Docker CPU qua runner `student-bundle.py`, kiểm tra các file cấu hình và manifest JSON, phân tích đối chiếu kết quả hình học qua các góc nhìn và tổng hợp báo cáo.
- **Quan sát từ thực nghiệm A/B/C:**
  - Nhận thấy rõ sự phụ thuộc sống còn của PointPillars vào bước tiền xử lý tọa độ: ở Lượt A (`delta = 0`), mạng mất gần như toàn bộ phát hiện (chỉ còn 1 hộp xe, score 0.322) so với 13 hộp ở Lượt B (`delta = 1.73`).
  - Kích thước voxel ở Lượt C (0.32m) gây ra hiện tượng sụp đổ phân loại (classification collapse): mất toàn bộ 10 xe hơi, phân loại nhầm 100% thành người đi bộ (`pedestrian`).
- **Diễn giải phép biến đổi $z$ thuận/ngược:**
  - Phép thuận: $z_{model} = z_{source} - z_{ground} - delta$. Giúp đưa point cloud thực tế về không gian quy chuẩn của mô hình KITTI (nơi gốc $z=0$ nằm tại mặt đất dưới tâm cảm biến).
  - Phép nghịch: $z_{source} = z_{model} + z_{ground} + delta$. Sau khi model dự đoán các bounding box trong hệ $z_{model}$, phải cộng bù lại $z_{ground} + delta$ để trả các cuboid về đúng hệ tọa độ gốc của file PCD.
- **Quyết định khi gặp lỗi batch:** Nếu phát hiện toàn bộ các hộp trong frame bị lệch cùng một lượng $z$ (chìm xuống đất hoặc bay lên không) như trong `case-batch-z`, hành động dứt khoát là **Dừng sửa tay**, báo ngay cho LC/kỹ sư pipeline để kiểm tra lại code transform tọa độ. Nếu chỉ một vài hộp bị lệch cục bộ như `case-one-box-z`, pipeline vẫn đúng và ta tiến hành kiểm tra, chỉnh sửa từng đối tượng cụ thể qua 4 góc nhìn trong CVAT.
- **Điều chưa chắc chắn:** File demo PCD KITTI đã bị lược bỏ cường độ phản xạ (reflectance) thật và thay bằng hằng số RGB=0; do đó không thể dựa vào intensity để phân định ranh giới vật thể. Đồng thời, các đối tượng ở rìa phạm vi quét và vùng bị che khuất nhiều điểm cần được đối chiếu thận trọng với ảnh camera, không được suy đoán chủ quan.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:
