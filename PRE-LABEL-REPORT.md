# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: VinAI20K (làm cá nhân)
- Thành viên: xem `TEAMMATES.md` (Thái Đức Cường — 2A202602065, đảm nhận mọi vai ở cả ba lượt).
- Trạng thái: `executed-by-group` — tự chạy thật trên máy cá nhân bằng runner `student-bundle.py` (làm một mình).
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Thái Đức Cường; 2026-10-02, 06:16:18–06:17:17 UTC (13:16–13:17 giờ VN); Windows 11 Home x86_64, Docker Desktop Linux engine, runtime `linux/amd64` (native, không emulation); Python 3.14.7 trên host.
- Image tag và image ID; phiên bản repo: `day13-pointpillars:lc-20261001-amd64`, image ID `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`; gói `student-prelabel-amd64.zip` từ release `student-prelabel-v1` (SHA256 `f58ca337…37aa9`, khớp `SHA256SUMS.txt`); code trong image `repo_revision 0831856d921609312d42c7582c366e5a311bb7b1` (working_tree_dirty=true theo `smoke.json`); repo Student cá nhân tại commit `e226b93`.
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp: `input/demo.pcd` của gói Student = KITTI 000008 (MMDetection3D demo, CC BY-NC-SA 3.0), 17 238 điểm, `frame_id=demo`; chạy trên laptop cá nhân theo giấy phép gói Student; PCD SHA256 `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60` (khớp `provenance.json` và `smoke.json`). LC không cấp fingerprint riêng.
- Checkpoint: PointPillars KITTI có sẵn trong image `/opt/PointPillars/pretrained/epoch_160.pth`, SHA256 `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`. LC không cấp ID/hash riêng.
- Phạm vi: front-window (không `--full-scene`); score threshold: 0.3.
- Giả định kênh thứ tư/intensity và nguồn z_ground: reflectance KITTI gốc đã bị bỏ trong PCD, adapter dùng kênh hằng số (RGB=0 chỉ là placeholder, không phải intensity); `z_ground = 0.075 m` do script ước lượng từ chính PCD (cùng giá trị ở cả 3 lượt), không phải mặt đường đo chính xác. PCD đã được dịch z +1.73 m so với KITTI gốc, x/y giữ nguyên.

## Ba lượt inference thật

A/B/C là ba lượt trên cùng PCD. Runner chạy đủ ba lượt từ một lệnh. Lấy **Số hộp** từ `n_boxes`, **mean_z** từ `mean_z` trong `run-A/B/C/summary.csv`; không tự tính lại hoặc đoán. `mean_z` không phải điểm chất lượng. Mở `side-*.png`, đối chiếu `boxes-*.json` để ghi quan sát. Số hộp không phải đáp án cần khớp nhóm khác.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | `run-A/boxes-demo-delta-0-voxel-0.16.json`, `run-A/side-demo-delta-0-voxel-0.16.png`, `run-A/summary.csv` | Chỉ 1 hộp `vehicles` tại x=13.15, y=−0.45, z=0.33 (L/W/H 3.62/1.52/1.46, yaw 2.67, score 0.32 — sát ngưỡng 0.3). Trên Side, đáy hộp ≈ −0.40 m, chìm dưới đường z=0; cụm xe gần x≈3–10 m và các cụm xa x≈30–55 m không có hộp. |
| B | 1.73 | 0.16 | 13 | 1.034 | `run-B/boxes-demo-delta-1.73-voxel-0.16.json`, `run-B/side-demo-delta-1.73-voxel-0.16.png`, `run-B/summary.csv` | 10 `vehicles` + 1 `two-wheels` + 2 `pedestrian`; 5 hộp score ≥ 0.81 (x≈3.7–33.7 m). Trên Side, phần lớn hộp xe có đáy gần z≈0 và bao cụm điểm. Hộp xe x=9.38, y=4.23 (z=1.43, đáy ≈0.66 m) có vẻ lơ lửng và chồng XY với hộp `two-wheels` x=10.32, y=5.25 (score 0.38) → nghi trùng/xung đột class. Các hộp xa (x≈40.98, 55.58) có đáy ≈0.4 m, điểm thưa. |
| C | 1.73 | 0.32 | 6 | 1.091 | `run-C/boxes-demo-delta-1.73-voxel-0.32.json`, `run-C/side-demo-delta-1.73-voxel-0.32.png`, `run-C/summary.csv` | Toàn bộ 6 hộp là `pedestrian` (score 0.30–0.81), không còn hộp `vehicles` nào. Nhiều hộp pedestrian nằm ở vùng B có xe: x=9.11, y=0.40 (gần xe B x=8.09, y=1.21); x=13.15, y=4.20 và x=10.46, y=4.93 (gần xe/two-wheels B x≈9.4–10.3, y≈4.2–5.3). Kích thước ~0.6–1.1 × 0.55–0.76 × 1.7 m không khớp các cụm xe. |

- A/B — chỉ đổi delta: A có 1 hộp; B có 13 hộp. Ảnh `side-demo-delta-0-voxel-0.16.png` và `side-demo-delta-1.73-voxel-0.16.png` khác ở toàn dải x≈2–58 m: A chỉ có một hộp ở x≈11–15 m với đáy chìm dưới z=0, còn B có hộp ở hầu hết cụm điểm dạng xe (x≈2–27 m) và cả các cụm xa x≈31–58 m. Hộp duy nhất của A (x=13.15, y=−0.45) không trùng hộp nào của B (gần nhất là x=14.77, y=−1.08, khác yaw ~3 rad, z 0.33 vs 0.90) → đây là chạy lại model trên input khác, không chỉ dịch hộp cũ 1.73 m. Với delta=0, đám mây điểm sau khi trừ z_ground nằm cao hơn ~1.73 m so với cao độ sensor mà checkpoint KITTI kỳ vọng, nên model gần như không nhận ra xe. Điều em còn chưa chắc là: 1.73 m chỉ là giả định cao sensor của checkpoint; em chưa có đo đạc mặt đường cục bộ hay camera để khẳng định các hộp B có z đúng tuyệt đối, chỉ thấy đáy hộp B bám gần z≈0 hợp lý hơn A.
- B/C — chỉ đổi pillar: B có 13 hộp; C có 6 hộp. Ảnh `side-demo-delta-1.73-voxel-0.16.png` và `side-demo-delta-1.73-voxel-0.32.png` khác ở chỗ các hộp rộng kiểu xe (dài ~3.1–4.2 m) của B biến mất hoàn toàn trong C, thay bằng các hộp hẹp cao kiểu người ở x≈9–34 m. Số lượng/lớp/vị trí thay đổi như sau: 10 vehicles + 1 two-wheels + 2 pedestrian → 0 vehicles + 0 two-wheels + 6 pedestrian; vị trí C không trùng tâm B (lệch ≥ 0.9 m), một số nằm trên cụm điểm mà B gán là xe. Có đủ bằng chứng để kết luận tốt hơn không? Không. Checkpoint được train với pillar 0.16 m; đổi 0.32 m làm thay đổi biểu diễn/kích thước lưới feature mà không train lại, nên kết quả C nhiều khả năng là lỗi lệch biểu diễn đầu vào chứ không phải cải thiện. Cũng không thể kết luận B đúng chỉ vì nhiều hộp/score cao hơn — chưa có nhãn hay camera để đối chiếu.
- Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào? Cả ba lượt chỉ chạy front-window của checkpoint nên vật ngoài ROI (phía sau, hai bên xa) không được tính là model bỏ sót. Side là hình chiếu x-z toàn scene, chồng các vật khác y lên nhau (ví dụ các cụm ở x≈8–11 m gồm cả y≈1.2 và y≈4.2–5.3), nên không đọc được yaw và không tách được hộp chồng nhau; yaw (vd B có cặp ~2.8 rad và ~−0.3 rad, ngược chiều ~π) cần Top view/camera mới kiểm được. Đường z=0 chỉ là tham chiếu của plot; vùng xa x>40 m điểm mặt đất cao ~0.3–0.5 m nên đáy hộp ≈0.4 m ở đó chưa chắc là lơ lửng.
- JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp? Không file nào được import CVAT (đây là KITTI demo, khác frame Robotaxi). Về nội dung: `boxes-demo-delta-0-voxel-0.16.json` (A) sai giả định cao sensor; `boxes-demo-delta-1.73-voxel-0.32.json` (C) dùng pillar không khớp checkpoint; `boxes-demo-delta-1.73-voxel-0.16.json` (B) hợp lý nhất về pipeline nhưng vẫn cần kiểm Top/Front + camera cho: cặp hộp chồng xe x=9.38/two-wheels x=10.32 (trùng/sai class), 2 pedestrian score 0.32–0.34, các hộp xa x>40 m điểm thưa, và yaw từng xe.

## Ca QC có kiểm soát — không import CVAT

Nguồn: `qc-cases/manifest.json` — `source_prediction = boxes-demo-delta-1.73-voxel-0.16.json` (B, SHA256 `16f30b08…4cf61`), `height_offset_m = delta + z_ground = 1.73 + 0.075 = 1.805`, 13 hộp.

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0 / 13 | 0 m | Không — bản sao nguyên prediction B | Pipeline giữ phép chuyển nguồn; vẫn phải kiểm từng hộp như mọi pre-label (không phải nhãn đúng) | `case-correct.json`, `side-correct.png` giống hệt `side-demo-delta-1.73-voxel-0.16.png` |
| case-batch-z | 13 / 13 | −1.805 m (mọi hộp, cùng một lượng = delta + z_ground) | Không — class, x, y, L/W/H, yaw, score giữ nguyên; chỉ z đổi | **Dừng batch**: lỗi hệ thống quên phép ngược z; báo LC kiểm transform, tạo lại prediction, không sửa tay từng hộp | `case-batch-z.json`; `side-batch-z.png`: toàn bộ hộp chìm xuống vùng z≈−1.9…+0.4, các cụm điểm xe ở trên không có hộp nào bao |
| case-one-box-z | 1 / 13 (hộp 0: vehicles x=8.09, y=1.21) | −1.805 m (z 0.92 → −0.89) | Không — chỉ z của hộp 0 đổi; 12 hộp còn lại giữ nguyên | **Kiểm từng hộp**: lỗi cục bộ của một đối tượng; kiểm hộp đó trên Top/Side/Front + camera rồi sửa z, không kết luận lỗi pipeline | `case-one-box-z.json`; `side-one-box-z.png`: chỉ một hộp ở x≈6–10 m chìm xuống z≈−1.65…−0.15, các hộp khác bám cụm điểm như case-correct |

Các ca trên do helper `pipeline-qc-cases.py` tạo bằng biến đổi có chủ đích từ prediction thật của lượt B (TRAINING ONLY), không phải kết quả inference riêng và không phải nhãn đúng. Không import `case-*.json` vào CVAT.

## Nhận xét cá nhân

### Thái Đức Cường — 2A202602065

- **Vai trò đã làm:** làm một mình nên đảm nhận cả vận hành runner, kiểm `smoke.json`/JSON cấu hình, xem ảnh Side và ghi log cho cả ba lượt A/B/C và ba ca QC.
- **Một quan sát A/B/C có dẫn file:** trong `run-B/side-demo-delta-1.73-voxel-0.16.png` các hộp xe ở x≈2–27 m có đáy gần z≈0 và bao cụm điểm; còn `run-A/side-demo-delta-0-voxel-0.16.png` chỉ có 1 hộp (x=13.15, score 0.32) với đáy chìm ≈ −0.40 m. Ở `run-C` (pillar 0.32) mọi hộp đều thành `pedestrian`, kể cả trên cụm điểm mà B gán là xe (x≈9.1, y≈0.4) → đổi pillar mà không train lại làm lệch biểu diễn, không phải cải thiện.
- **Diễn giải phép z thuận/ngược:** thuận `z_model = z_source − z_ground − delta` đưa điểm về hệ cao độ mà checkpoint KITTI kỳ vọng (sensor ~1.73 m trên mặt đất); ngược `z_source = z_model + z_ground + delta` trả hộp về hệ PCD nguồn. Đổi delta trước inference làm model chạy trên input khác nên số hộp/vị trí/class đổi (A 1 hộp → B 13 hộp), khác hẳn việc cộng/trừ cùng một lượng cho mọi hộp sau inference.
- **Một quyết định lỗi batch và hành động:** với `case-batch-z` cả 13/13 hộp lệch đúng −1.805 m = delta + z_ground, x/y/yaw/class không đổi → đây là lỗi pipeline (quên phép ngược). Hành động: dừng sửa tay, báo LC kiểm transform và tạo lại prediction. Với `case-one-box-z` chỉ 1/13 hộp lệch → kiểm riêng hộp đó bằng nhiều view.
- **Điều chưa chắc:** chưa có camera/nhãn để khẳng định B đúng tuyệt đối; cặp hộp chồng nhau xe (x=9.38, y=4.23, đáy ≈0.66 m) và two-wheels (x=10.32, y=5.25) trong B có thể là trùng hoặc sai class; vùng xa x>40 m điểm thưa nên đáy hộp ≈0.4 m có thể do mặt đất dốc chứ chưa chắc lơ lửng.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:
