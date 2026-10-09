# Báo cáo bài nộp — Day 23 Sensor Fusion Lab

> Điền file này rồi commit. Cách nộp: [hướng dẫn nộp](../SUBMISSION.md).

## Thông tin học viên

- Họ tên: Trương Thị Lan Anh
- MSSV: 2A202602451
- Email: anhtruong23072004@gmail.com
- Link repo (fork): https://github.com/SxAinsworth/K4-Track4-Day23-TruongThiLanAnh-2A202602451-Sensor-Fusion
- Commit hash nộp (`git rev-parse HEAD`): 4a6456331b19740faa7b6da48efd1c11637d6aa3 (code E–H, artifacts và báo cáo; commit ngay sau chỉ ghi thêm dòng hash này)

## Tóm tắt kết quả

- `fusion_mode` (bắt buộc `compare`), `frames`, `segment`, `seed`: `compare`, `[0, 198]` (199 frame), `training_segment-1005081002024129653_5313_150_5333_150_with_camera_labels.tfrecord`, `seed = 0`
- `detection.precision`, `detection.recall`, `detection.tp/fp/fn`: precision 0.9701, recall 0.7004, tp/fp/fn = 519 / 16 / 222
- `tracking.lidar.rmse`, `matches`, `sum_sq_err`, `ghost_track_frames`, `missed_gt_frames`, `mean_confirmed_tracks`: 0.1503 m, 502, 11.3437 m², 0, 239, 2.523
- `tracking.fused.rmse`, `matches`, `sum_sq_err`, `ghost_track_frames`, `missed_gt_frames`, `mean_confirmed_tracks`: 0.1359 m, 502, 9.2668 m², 0, 239, 2.523
- Giải thích khác biệt hai mode, đọc RMSE cùng số ghép và ghost/miss: xem bảng và phân tích dưới đây.

| Mode | RMSE (m) | matches | ghosts | misses | precision_track | coverage = matches/det_tp |
|---|---|---|---|---|---|---|
| lidar | 0.1503 | 502 | 0 | 239 | 502/502 = 1.000 | 502/519 = 0.967 |
| fused | 0.1359 | 502 | 0 | 239 | 502/502 = 1.000 | 502/519 = 0.967 |

`rmse_fused − rmse_lidar = −0.0145 m` (fused tốt hơn khoảng 1.4 cm, trong giới hạn ≤ 0.05 m).

- **Hai mode ghép đúng cùng tập track–GT.** `matches`, `ghost_track_frames`, `missed_gt_frames` và `mean_confirmed_tracks` giống hệt nhau. Điều này khớp với thiết kế: camera chỉ update EKF, không cộng/trừ score, không tạo hay xoá track, nên tập confirmed track ở mỗi frame không đổi. Vì số cặp như nhau, so sánh RMSE giữa hai mode là so trên cùng một mẫu, không bị lệch do mode này ghép được ít hay nhiều cặp hơn.
- **Khác biệt chỉ nằm ở độ chính xác vị trí.** `sum_sq_err` giảm từ 11.34 xuống 9.27 m² (−18%). Trong 195 frame có cặp ghép, fused có tổng sai số nhỏ hơn LiDAR ở 137 frame. Ví dụ frame 50 trong `grade_run.log`: cả hai mode `confirmed = 3`, `matches = 3`, `sum_sq_err` 0.1230 (lidar) so với 0.1143 (fused).
- **Không có ghost:** `precision_track = 1.0`, mọi confirmed track đều ghép được một xe thật trong cổng 2 m.
- **239 misses chủ yếu do detector, không phải do tracker:** tổng `det_fn` trên log là 222, tức detector đã bỏ sót 222 xe-frame. Ngoài ra có 8 miss ở frame 0–3 (`confirmed = 0` cho tới frame 4) vì track cần 4 hit LiDAR mới vượt `confirmed_threshold` (1/6 → 5/6 > 0.8). Phần còn lại là các track mới hoặc đang thiếu detection. Vì `coverage` tính trên `det_tp`, xe detector bỏ sót không làm giảm `coverage` (0.967).
- **Giới hạn:** đo camera là tâm hộp 2D ground-truth của camera FRONT cộng nhiễu theo seed, không phải output của một camera detector. Mức cải thiện RMSE của fused vì vậy chỉ cho thấy việc kết hợp đo 2D vào EKF là đúng; nó không chứng minh một hệ thống camera perception thật cũng cải thiện như vậy. Lần chạy chấm điểm chỉ dùng một segment và một seed; kết quả trên 3 segment còn lại ở mục ngay dưới.

Chạy từ root repo:

```bash
fusion-run-lab --config student/config/paths.yaml --fusion compare --seed 0
```

`rmse = sqrt(sum_sq_err/matches)` trên vị trí 3D của confirmed tracks ghép
một-một với GT xe trong cửa sổ BEV, gate XY **2.0 m**; `null` nếu không có cặp.
Camera dùng tâm hộp 2D ground-truth FRONT có nhiễu seeded, **không** dùng camera
detector. Kết quả này không đo hiệu quả một perception system độc lập với GT.

`grade_run.log` là JSONL, mỗi `(mode,frame)` đúng một record với các trường:
`mode`, `frame`, `det_tp`, `det_fp`, `det_fn`, `valid_gt`, `confirmed`, `matches`,
`sum_sq_err`, `ghosts`, `misses`. Đảm bảo `matches+ghosts==confirmed` và
`matches+misses==valid_gt`; tổng/trung bình record phải khớp `metrics.json`.
File per-mode `metrics_lidar.json`, `metrics_fused.json`, `grade_run_lidar.log`,
`grade_run_fused.log` được giữ để đối chiếu.

### Kiểm tra thêm trên 4 segment của khóa học

Ngoài lần chạy chấm điểm, mình chạy cùng lệnh (`--fusion compare --seed 0`, frame 0–198) trên 3 segment còn lại để xem kết quả có giữ được không. Vì `fusion-run-lab` luôn ghi đè `student/artifacts/`, sau các lần chạy phụ mình khôi phục artifacts về bản đã commit của segment mặc định (chạy lại segment mặc định cho ra artifacts giống hệt từng byte). `metrics*.json` của 3 segment phụ lưu trong [`bonus/segments/`](bonus/segments/).

| Segment | Det. P / R | LiDAR RMSE (m) | Fused RMSE (m) | Fused − LiDAR (m) | matches | ghosts | misses | mean confirmed |
|---|---|---|---|---|---|---|---|---|
| **1005081002024129653** (chấm điểm) | 0.970 / 0.700 | 0.1503 | 0.1359 | −0.0145 | 502 | 0 | 239 | 2.52 |
| 10072231702153043603 | 0.645 / 0.740 | 0.2193 | 0.2105 | −0.0088 | 55 | 14 | 41 | 0.35 |
| 10094743350625019937 | 0.959 / 0.402 | 0.3462 | 0.3712 | +0.0250 | 1000 | 0 | 1534 | 5.03 |
| 10963653239323173269 | 0.995 / 0.576 | 0.2153 | 0.2207 | +0.0054 | 746 | 3 | 680 | 3.76 |

Segment 10072231702153043603 chỉ có 198 frame (`frames = [0, 197]`).

- **Ngưỡng:** cả 4 segment có LiDAR RMSE ≤ 0.45 m, và fused không kém LiDAR quá 0.05 m (lệch lớn nhất +0.025 m).
- **Fused không phải lúc nào cũng tốt hơn:** fused tốt hơn ở 2 segment, kém hơn chút ở 2 segment. Ở cả 4 segment, `matches`, `ghosts`, `misses` và `mean confirmed` của hai mode giống hệt nhau, đúng với thiết kế track-then-fuse: camera chỉ update EKF, không đổi vòng đời track. Khác biệt chỉ nằm ở `sum_sq_err`.
- **10094743350625019937:** cảnh đông xe (trung bình 5 track confirmed), detector bỏ sót nhiều (recall 0.40, 1534 misses), RMSE cao nhất. Fused kém hơn 2.5 cm (`sum_sq_err` 119.85 → 137.75 m²). Một khả năng là khi nhiều xe gần nhau trong ảnh FRONT, đo camera 2D bị gán nhầm sang track bên cạnh và kéo track lệch theo hướng ngang.
- **10072231702153043603:** cảnh rất ít xe (0.35 track confirmed mỗi frame, chỉ 55 cặp ghép). Precision detector thấp (0.645) và có 14 frame ghost, nên RMSE ở segment này dựa trên ít mẫu và kém tin cậy hơn.

## Giải thích ngắn (Parts E–H — tự viết)

1. Khác biệt đo lidar 3D và camera 2D trong EKF (`z`, `R`)?

   LiDAR đo trực tiếp vị trí: `z = (x, y, z)` 3×1 (mét), `R = diag(0.1², 0.1², 0.1²)`, `h(x) = R_vs·p + t` là hàm tuyến tính nên `H` là ma trận 3×6 hằng (khối quay ở 3 cột vị trí, 3 cột vận tốc bằng 0). Camera đo toạ độ ảnh: `z = (u, v)` 2×1 (pixel), `R = diag(5², 5²)` px² ([camera_fusion.py `build_camera_measurement`](workspace/camera_fusion.py)). `h(x)` là phép chiếu pinhole `u = c_i − f_i·y_s/x_s`, `v = c_j − f_j·z_s/x_s`, phi tuyến vì chia cho độ sâu `x_s`, nên EKF dùng Jacobian `H` 2×6 do platform tính tại trạng thái hiện tại. Camera không đo được độ sâu: một đo 2D chỉ sửa track theo hướng vuông góc với tia nhìn. Cùng một hàm [`ekf_update`](workspace/kalman.py) dùng cho cả hai vì nó lấy `H`, `h(x)`, `R` từ `meas.sensor` và `meas`.

2. Vì sao cần gating Mahalanobis trước khi gán?

   Greedy luôn chọn cặp rẻ nhất còn lại, nên nếu không có cổng nó sẽ ghép cả những cặp vô lý (một detection ở xa, hoặc false positive) vào track và kéo track lệch đi. Khoảng cách Mahalanobis `d² = γᵀ S⁻¹ γ` ([association.py `mahalanobis_distance`](workspace/association.py)) chuẩn hoá innovation theo `S = H P Hᵀ + R`. Track mới có `P` lớn thì cổng tự nới rộng, track ổn định thì cổng hẹp lại. Nó cũng so được LiDAR (mét) và camera (pixel) cùng một thang. Vì `d²` của innovation Gauss có phân phối χ² với số bậc tự do bằng số chiều đo, ngưỡng `chi2.ppf(0.995, dim_meas)` ([`chi2_gate`](workspace/association.py), khoảng 12.84 cho 3D, 10.60 cho 2D) giữ lại 99.5% cặp đúng. Với camera, cặp ngoài FOV hoặc sau camera bị đặt `inf` **trước** khi tính Mahalanobis (`meas.sensor.in_fov(track.x)` trong `association_cost_matrix`), vì phép chiếu không xác định khi độ sâu ≤ 0.

3. Pipeline là track-then-fuse hay fuse-then-track? Chỉ ra trên log `fusion-run-lab`.

   Track-then-fuse. Chỉ có một danh sách track. Mỗi frame, `run_lab.py` gọi `KF.predict(track)` một lần cho mọi track, rồi `associate_and_update(..., lidar_sensor)`, và ở mode fused gọi thêm `associate_and_update(..., camera_sensor)` trên cùng các track đó. Không có bước gộp điểm LiDAR với pixel trước detection. Trên log: record `fused` và `lidar` của cùng frame có `det_tp/det_fp/det_fn`, `confirmed`, `matches`, `ghosts`, `misses` giống hệt nhau, chỉ `sum_sq_err` khác (frame 50: `confirmed = 3`, `matches = 3` ở cả hai; `sum_sq_err` 0.1230 so với 0.1143). Nghĩa là detection và vòng đời track đến từ cùng một nhánh LiDAR, còn camera chỉ tinh chỉnh trạng thái của những track đã có.

4. Nếu camera lệch calibration, triệu chứng gì trên innovation/residual?

   Extrinsic sai làm `h(x)` chiếu track sai chỗ một cách có hệ thống, nên innovation camera `γ = z − h(x)` không còn trung bình bằng 0 mà có độ lệch cố định (bias). Lệch về góc yaw làm dịch `u` theo một hướng ở mọi track, lệch về pitch làm dịch `v`. `d²` tăng trung bình, nhiều cặp camera bị cổng χ² loại, nên số update camera giảm. Với các cặp vẫn lọt cổng, EKF bị kéo về phía sai và lệch khỏi vị trí LiDAR. Sau đó innovation LiDAR cũng có bias ngược lại, hai sensor "giằng co", và RMSE fused có thể xấu hơn LiDAR. Cách nhận ra là log trung bình innovation theo từng sensor qua nhiều frame: nếu trung bình ≠ 0 và cùng dấu cho mọi track, đó là calibration, không phải nhiễu.

5. Vì sao `associate_and_update(..., sensor)` cần sensor tường minh ở frame rỗng?
   Giải thích vì sao lidar quyết định score/init/delete còn camera chỉ EKF update.

   Ở frame không có detection, `meas_list` rỗng nên không thể suy ra lượt này là của sensor nào từ `meas.sensor`. Nhưng lượt LiDAR rỗng vẫn có ý nghĩa: mọi track trong FOV LiDAR đều bị **miss**, phải trừ score và có thể bị xoá. Vì vậy [`associate_and_update`](workspace/association.py) luôn gọi `manager.manage_tracks(unassigned_tracks, unassigned_meas, sensor)` ở cuối, kể cả khi danh sách rỗng, và `manage_tracks` dựa vào `sensor.name` để chạy hay bỏ qua lifecycle (`manager.py`, dòng 111). Nếu nhầm, một lượt camera rỗng sẽ bị hiểu là "LiDAR miss" và xoá oan track. LiDAR quyết định tồn tại vì nó phủ 360° và đo vị trí 3D đầy đủ, nên việc nó không thấy xe là bằng chứng mạnh rằng xe không còn ở đó. Camera trong lab chỉ có FOV phía trước, không đo độ sâu, và đo được mô phỏng từ nhãn, nên một hit hay miss camera không đủ tin cậy để tạo hay xoá track. Camera chỉ được dùng để giảm sai số vị trí.

6. Nêu điều kiện xác nhận, giữ confirmed sau miss, và điều kiện xóa track.

   Theo [track_management.py](workspace/track_management.py), `window = 6`:
   - **Khởi tạo:** track mới có `score = 1/6`, `state = "initialized"`, vận tốc bằng 0 với độ bất định lớn (σ = 50, 50, 5 m/s).
   - **Mỗi lượt LiDAR:** hit cộng `1/6` (tối đa 1); miss trong FOV trừ `1/6`.
   - **Xác nhận:** khi `score > 0.8`, track chuyển sang `confirmed`. Track chưa đạt ngưỡng mà có hit thì là `tentative`. Bắt đầu từ 1/6, cần 4 hit liên tiếp (5/6 ≈ 0.833) mới được xác nhận, khớp với log: `confirmed = 0` ở frame 0–3 và bằng 2 từ frame 4.
   - **Giữ confirmed sau miss:** trạng thái `confirmed` không bao giờ bị hạ. Một miss từ `score = 1` chỉ còn 5/6, vẫn ≥ 0.6, nên track vẫn sống.
   - **Xoá** (`should_delete_track`, các điều kiện là OR):
     - `P[0,0] > 9` hoặc `P[1,1] > 9` (σ vị trí > 3 m), bất kể score.
     - Track confirmed có `score < 0.6`, tức khoảng 3 miss liên tiếp từ score 1.
     - Track chưa confirmed có `score ≤ 0`, ví dụ track mới bị miss ngay frame sau.

## Bonus (không bắt buộc)

Liệt kê phần bonus đã làm, file bằng chứng trong `student/bonus/` và kết quả chính
(xem [RUBRIC.md](../RUBRIC.md) mục 2). Không làm thì ghi "Không".

- Không

## Khai báo sử dụng AI (bắt buộc)

Ghi rõ, kể cả khi không dùng ("Không dùng AI"). Xem [RULES.md](../RULES.md) mục 2.

- Công cụ đã dùng (ChatGPT, Copilot, Claude, …): Claude (Claude Code trong VS Code)
- Dùng cho phần nào (hàm, câu hỏi, debug): viết code Part E–H (`kalman.py`, `camera_fusion.py`, `association.py`, `track_management.py`) theo các dòng `# vi: TODO`; debug lỗi `float()` trên ma trận 1×1 với numpy mới trong `mahalanobis_distance`; cài môi trường; tải dữ liệu; chạy `fusion-run-lab`; phân tích log (đếm misses theo `det_fn`, so `sum_sq_err` từng frame); soạn bản nháp tóm tắt kết quả và 6 câu giải thích; chạy `compare` trên 3 segment còn lại và soạn bảng so sánh 4 segment.
- Cách bạn đã kiểm tra lại (pytest, chạy Waymo, đối chiếu công thức): `pytest student/tests -q` được 128 passed, không failed/xfailed. Chạy `fusion-run-lab --fusion compare --seed 0` trên frame 0–198. Đối chiếu số trong báo cáo với `metrics.json` và `grade_run.log` (đủ 398 record, các invariant đúng). Đối chiếu công thức EKF và Mahalanobis với tài liệu `docs/HUONG_DAN_KY_THUAT.md`. Đọc lại từng hàm E–H để tự giải thích được khi vấn đáp.

## Checklist nộp

- [x] **Part E–H** trong `workspace/` đã implement; `pytest student/tests -q` không còn `failed`/`xfailed`
- [x] Part A–D: không bắt buộc sửa (hoặc ghi chú nếu bạn đã sửa)
- [x] Lần chạy chấm điểm: `--fusion compare --seed 0`, `frame_start: 0`, `frame_end: 198`
- [x] Đã commit `student/artifacts/metrics*.json` và `student/artifacts/grade_run*.log` (không sửa tay)
- [x] Đã điền đủ file này, gồm khai báo AI
- [x] Không commit dữ liệu Waymo, weights, `paths.yaml`, API key
- [x] `python tools/check_submission.py` báo `KẾT QUẢ: SẴN SÀNG NỘP`
- [ ] Đã push và nộp link repo + commit hash trên LMS ([hướng dẫn nộp](../SUBMISSION.md))
