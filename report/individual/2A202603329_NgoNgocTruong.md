# Báo cáo Day 14 — Ngô Ngọc Trường — 2A202603329

- Mã học viên dùng với `--student`: `329`
- `seed_tag` trong `frames_A.txt`: `82ac3100fd`

## 1. Dự đoán trước sweep (bước 5b)

Tập A gồm 3 frame `s02f13`, `s03f11`, `s05f24`. Cấu hình gốc của model trên tập A, đo với `ref_A.json` đã sửa tay trên CVAT:

| Frame | Dự đoán | Xóa | Thêm | Sửa | Chi phí |
|---|---:|---:|---:|---:|---:|
| s02f13 | 29 | 13 | 4 | 0 | 7.9 |
| s03f11 | 25 | 10 | 1 | 2 | 5.0 |
| s05f24 | 34 | 15 | 4 | 0 | 8.5 |
| **Tổng** | **88** | **38** | **9** | **2** | **21.4** |

- Tham số tôi nghĩ giảm công sửa nhiều nhất: **`score_thr` tăng lên khoảng 0.4**, kết hợp **`min_points` khoảng 5–10**.
- Vì sao:
  - Box thừa nhiều hơn hẳn box sót: 38 box phải xóa, 9 box phải thêm. Công xóa (38 × 0.3 = 11.4) lớn hơn công thêm (9 × 1.0 = 9.0).
  - Xem ảnh BEV và camera của tập A, box thừa chủ yếu là Car đặt lên xe máy, xe đạp đỗ trên vỉa hè (s02f13, s05f24), lên bụi cây, hàng rào, tường quây công trường (s03f11), và 3 box ngay tâm LiDAR, tức là chính xe ghi dữ liệu. Phần lớn các box này có score thấp (khoảng 0.2–0.4), trong khi xe thật ở gần có score trên 0.8.
  - Tôi không đẩy ngưỡng lên quá cao, vì thêm 1 box sót tốn 1.0, gấp hơn 3 lần xóa 1 box thừa (0.3). Tôi đoán ngưỡng từ 0.5 trở lên sẽ bỏ cả xe thật ở xa hoặc bị che (score thấp), làm số box phải thêm tăng và tổng chi phí tăng lại.
  - `min_points` nhỏ (5–10) bỏ được vài box gần như không có điểm LiDAR (ví dụ box Car chỉ có 1 điểm) mà ít khi mất xe thật. Đặt `min_points` lớn sẽ mất xe ở xa, vốn thưa điểm.
  - Tôi đoán tune sẽ **không** giải quyết được 9 box sót (người đi bộ, người đi xe máy, xe MPV bị che) và lỗi lớp (model KITTI không có Truck/Bus, nên xe van chở hàng và xe tải nhỏ ở s03f11 vẫn ra Car). Vì vậy chi phí sau tune vẫn còn khá lớn.

## 2. Tập A: sửa tay và tune

Khi sửa tay tập A trên CVAT, tôi xóa box thừa (xe hai bánh đỗ không người lái, bụi cây, hàng rào, box tại tâm LiDAR), thêm vật bị sót (Cyclist, Pedestrian, 1 Car MPV), đổi lớp Car → Truck cho xe van chở hàng và xe tải nhỏ ở s03f11, đổi Car → Cyclist cho người lái xe máy điện ở s05f24, và bật `occluded` cho các vật bị xe khác hoặc hàng rào che một phần, theo `docs/GUIDELINE.md`.

| Cấu hình | Xóa | Thêm | Sửa | Chi phí |
|---|---:|---:|---:|---:|
| Gốc | 38 | 9 | 2 | 21.4 |
| `best_config.json` | _(điền sau bước 6)_ | | | |

- Tham số đã đổi trong `best_config.json`: _(điền sau bước 6)_
- Kết quả leave-one-out (dán các dòng `scripts/pp_sweep.py` in ra): _(điền sau bước 6)_
- Cấu hình có ổn định trên từng frame không? Vì sao tôi vẫn chọn (hoặc không chọn) nó: _(điền sau bước 6)_

## 3. Trước/sau (bước 8)

_(điền sau bước 8: chèn 1 ảnh từ `d14_329/compare/`)_

<!-- ![so sánh](../../d14_329/compare/<frame>.jpg) -->

- `scripts/pp_compare.py` báo giảm (hoặc tăng) ...% công sửa trên tập A.
- Dự đoán ở mục 1 đúng hay sai? Giải thích bằng số box xóa/thêm/sửa: _(điền sau bước 8)_

## 4. Tập B

Số đo từ `scripts/pp_prelabel.py --frames d14_329/frames_B.txt --config d14_329/best_config.json --gt d14_329/ref_B.json`:

| Xóa | Thêm | Sửa | Chi phí |
|---:|---:|---:|---:|
| _(điền sau bước 9)_ | | | |

- Loại box tốn công sửa nhất ở tập B: _(điền sau bước 9)_

## 5. Giới hạn

- **Chỉ tune trên 3 frame.** Cấu hình chọn trên tập A dễ overfit với đặc điểm riêng của 3 frame này (nhiều xe đỗ, hàng rào cây). Tập B có thể có kiểu cảnh khác.
- **Model huấn luyện trên KITTI.** Model chỉ có 3 lớp Car/Pedestrian/Cyclist, không biết Truck/Bus và chưa thấy loại xe máy, xe đạp điện phổ biến ở dữ liệu VF. Lỗi lớp và box sót không sửa được bằng tune ngưỡng.
- **Chi phí là công thức, không phải thời gian thật.** `0.3·xóa + 1.0·thêm + 0.5·sửa` là trọng số quy ước. Thời gian sửa thật còn phụ thuộc độ khó từng box (vật bị che, thưa điểm) và thao tác trên CVAT. Không nên quy đổi trực tiếp sang giờ công.
- **Nhãn tham chiếu do một người sửa.** `ref_A.json` là nhãn của chính tôi, có thể còn sai kích thước, sót vật thưa điểm hoặc khó nhìn (ví dụ người đứng sau hàng rào ở s02f13).
- **Ranh giới 50 m nhạy.** Hai xe buýt đỗ ở s05f24 có tâm sát mép 50 m. Chỉ cần chỉnh hộp làm tâm lệch vài chục cm là chúng ra khỏi vùng chấm, khiến chi phí cấu hình gốc đổi từ 23.4 xuống 21.4 (UNCERTAIN_SCOPE).
