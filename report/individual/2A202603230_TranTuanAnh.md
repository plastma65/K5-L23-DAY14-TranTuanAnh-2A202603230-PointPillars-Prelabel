# Báo cáo Day 14 — Trần Tuấn Anh — 2A202603230

> Sao chép file này thành `report/individual/<MSSV>_<HoTen>.md` (không dấu), ví dụ `2A20260000_NguyenVanAn.md`.

- Mã học viên dùng với `--student`: `230`
- `seed_tag` trong `frames_A.txt`: `393590736f`

## 1. Dự đoán trước sweep (bước 5b)

Ghi **trước** khi chạy `scripts/pp_sweep.py`, rồi commit và push ngay (lịch sử commit là bằng chứng thời điểm viết). Không sửa phần này sau khi có kết quả.

- Tham số tôi nghĩ giảm công sửa nhiều nhất: score_threshold
- Vì sao (nhìn ảnh BEV tập A: box thừa nhiều hay box sót nhiều?): Mô hình đang sinh ra quá nhiều box thừa trong khi đã bắt trúng hầu hết các vật thể thật.

## 2. Tập A: sửa tay và tune

| Cấu hình | Xóa | Thêm | Sửa | Chi phí |
|---|---:|---:|---:|---:|
| Gốc | 79 | 4 | 2 | 28.7 |
| `best_config.json` | 18 | 10 | 0 | 15.4 |

- Tham số đã đổi trong `best_config.json`: Tăng score_thr từ 0.1 đến 0.5, giữ nguyên min_points ở mức 0.
- Kết quả leave-one-out (dán các dòng `scripts/pp_sweep.py` in ra): 
Leave-one-out (chọn trên các frame còn lại, chấm trên frame bỏ ra):
    s01f54: cấu hình chọn {'score_thr': '0.5', 'min_points': '0'} → 6.5 (tốt nhất có thể 6.5)
    s02f27: cấu hình chọn {'score_thr': '0.5', 'min_points': '0'} → 4.2 (tốt nhất có thể 3.8)
    s05f37: cấu hình chọn {'score_thr': '0.5', 'min_points': '0'} → 4.7 (tốt nhất có thể 4.7)
- Cấu hình có ổn định trên từng frame không? Vì sao tôi vẫn chọn (hoặc không chọn) nó: 
Cấu hình này khá ổn định trên từng frame: khi đánh giá chéo leave-one-out, cấu hình {'score_thr': '0.5', 'min_points': '0'} đều đạt mức chi phí tối ưu tuyệt đối trên 2/3 frame (s01f54: 6.5/6.5 và s05f37: 4.7/4.7), và chỉ lệch rất nhỏ trên frame s02f27 (4.2 so với mức tốt nhất 3.8). Tôi vẫn chọn cấu hình này vì nó giúp giảm gần một nửa tổng chi phí chỉnh sửa (từ 28.7 xuống 15.4 — giảm 46.3%), loại bỏ phần lớn box nhiễu (giảm số lần xóa từ 79 xuống 18) và không bị quá khớp (overfit) vào một frame cụ thể nào.

## 3. Trước/sau (bước 8)

![so sánh](../K5-L23-DAY14-TranTuanAnh-2A202603230-PointPillars-Prelabel/d14_230/compare/s05f37.jpg)

- `scripts/pp_compare.py` báo giảm 46% công sửa trên tập A (tổng chi phí giảm từ 28.7 xuống 15.4)
- Dự đoán ở mục 1 đúng hay sai? Giải thích bằng số box xóa/thêm/sửa:
    Đúng Ở mục 1 đã dự đoán việc tăng ngưỡng độ tự tin (score_thr) sẽ giúp giảm công sửa nhiều nhất nhờ loại bỏ hàng loạt box nhiễu. Thực tế khi tăng score_thr từ 0.1 lên 0.5:Xóa: Giảm mạnh từ 79 xuống còn 18 box (giảm 61 box thừa, tiết kiệm được 61 x 0.3 = 18.3 điểm chi phí).   Thêm: Tăng nhẹ từ 4 lên 10 box (tăng 6 box bị sót do nâng ngưỡng lọc, tốn thêm 6 x 1.0 = 6.0$ điểm chi phí).   Sửa: Giảm từ 2 xuống 0 box (tiết kiệm thêm 2 x 0.5 = 1.0 điểm chi phí).   Lượng công giảm được từ việc bớt xóa và sửa lớn hơn nhiều so với phần tăng lên do thêm, kéo tổng chi phí giảm từ 28.7 xuống 15.4 (giảm 46%).

## 4. Tập B

Số đo từ `scripts/pp_prelabel.py --frames d14_<ID>/frames_B.txt --config d14_<ID>/best_config.json --gt d14_<ID>/ref_B.json`:

| Xóa | Thêm | Sửa | Chi phí |
|---:|---:|---:|---:|
| 31 | 13 | 0 | 22.3 |

- Loại box tốn công sửa nhất ở tập B: Box bị bỏ sót (cần Thêm) — tuy số lượng ít hơn box thừa (13 so với 31), nhưng mỗi lần vẽ mới tốn 1.0 điểm chi phí (tổng 13.0 điểm, chiếm 58.3% tổng công sửa), cao hơn chi phí xóa box thừa (9.3 điểm). Nguyên nhân do cấu hình score_thr = 0.5 tối ưu trên tập A hơi chặt khi áp dụng sang tập B, làm lọc nhầm một số vật thể thật ở xa hoặc ít điểm LiDAR.

## 5. Giới hạn

- Tập dữ liệu tinh chỉnh (tune) quá nhỏ: Cấu hình best_config.json (score_thr = 0.5) chỉ được tìm trên 3 frame của tập A (s01f54, s02f27, s05f37). Do đó, ngưỡng 0.5 hơi quá chặt khi áp dụng sang 12 frame của tập B, khiến mô hình bỏ sót nhiều vật thể hơn (thêm 13 box, chiếm 58.3% tổng chi phí sửa trên tập B).

- Lệch miền dữ liệu và giới hạn lớp nhận diện của mô hình: Mô hình PointPillars được huấn luyện sẵn trên tập dữ liệu KITTI nên chỉ hỗ trợ 3 lớp cơ bản (Car, Cyclist, Pedestrian), thiếu các lớp phương tiện phổ biến trên đường thực tế (như Truck, Bus, xe máy) và dễ nhận nhầm các vật thể lạ hoặc cụm điểm thưa ở xa.

- Thước đo chi phí chỉ mang tính ước lượng lý thuyết: Công thức tính chi phí (xóa × 0.3 + sửa × 0.5 + thêm × 1.0) sử dụng trọng số cố định để thuận tiện cho việc tối ưu tự động, chưa phản ánh chính xác thời gian thao tác thực tế của người gán nhãn trên CVAT (vì thời gian vẽ mới hoặc chỉnh góc xoay của một box ở xa, ít điểm LiDAR thường tốn công hơn nhiều so với một box rõ nét ở gần).
