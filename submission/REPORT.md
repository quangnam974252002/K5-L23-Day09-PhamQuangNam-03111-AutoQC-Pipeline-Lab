# Báo cáo Day 09 — Phạm Quang Nam, 2A202603111

## 1. Kết luận

Pipeline kết luận **HOLD** (mã thoát 10). Trong **224 ảnh** ở `raw/`, intake loại **21 ảnh lỗi**, giữ **203 ảnh** cho QC và xuất dataset sạch nháp tại `submission/export/clean_v1/`. Không phải Rework vì không còn lỗi bắt buộc (`n_errors = 0`), nhưng **44/203 ảnh (21.67%)** vẫn có cảnh báo ngữ nghĩa chưa duyệt, vượt ngưỡng **10%**.

## 2. Lỗi và phân việc

Tổng hợp từ `02_error_list_v1` (203 ảnh):

| Mã lỗi | Số lượng | Xử lý | Nguyên nhân / hành động |
|---|---:|---|---|
| `BAD_SIZE` | 3 ảnh | Máy (intake) | Cạnh ngắn < 250px; tự loại do không đạt độ phân giải. |
| `CORRUPT_IMAGE` | 2 ảnh | Máy (intake) | Ảnh hỏng/cắt cụt, không mở được; tự loại. |
| `DUP_IMAGE_EXACT` | 8 ảnh | Máy (intake) | Trùng byte (MD5) với ảnh đã nhận; tự loại. |
| `DUP_IMAGE_NEAR` | 3 ảnh | Máy (intake) | Trùng gần (pHash ≤ 6) do resize/đổi sáng; tự loại. |
| `LEAK_TEST` | 5 ảnh | Máy (intake) | Trùng ảnh trong `test/` hoặc `gold_val/`; tự loại để tránh rò rỉ. |
| `BAD_FORMAT` | 3 dòng | Máy (autofix) | Thiếu cột hoặc có NaN, không parse được; `drop_line`. |
| `BAD_CLASS_ID` | 6 dòng | Máy (autofix) | ID ngoài 0..3 (93, 4, -1); `drop_line`. |
| `BOX_ZERO` | 4 dòng | Máy (autofix) | Width hoặc height bằng 0; `drop_line`. |
| `BOX_OUT_OF_IMAGE` | 8 dòng | Máy (autofix) | Tọa độ vượt [0, 1] quá dung sai; `clip_box` về [0, 1]. |
| `DUP_BOX` | 7 dòng | Máy (autofix) | Box cùng lớp trùng box trước, IoU > 0.9; `drop_line`. |
| `BAD_CLASS_NAME` | 2 lỗi | Máy (autofix) | `data.yaml` sai chữ hoa (`Person`, `Bus`); `rename_classes`. |
| `MISSING_LABEL_FILE` | 6 ảnh | Người (review) | Thiếu file nhãn; đánh dấu `needs_label` để người gán. |
| `BOX_TINY` | 6 dòng | Người (review) | Cạnh ngắn < 12px; kiểm tra vật thể xa hay viền nhiễu. |
| `BOX_HUGE` | 3 dòng | Người (review) | Box chiếm > 95% ảnh; xác nhận đối tượng có hợp lệ không. |
| `CLASS_RARE` | 4 cảnh báo | Người (review) | Cảnh báo mất cân bằng dữ liệu ở cấp dataset. |
| `EMPTY_LABEL` | 12 ảnh | Người (review) | File nhãn rỗng; xác định ảnh nền (background) hay thiếu nhãn. |
| `POSSIBLE_WRONG_CLASS` | 18 box | Người (review) | Model nghi gán sai lớp (trọng số 3); cần sửa nhãn. |
| `POSSIBLE_MISSING` | 68 box | Người (review) | Model nghi bỏ sót vật thể (trọng số 2); cần thêm box. |
| `POSSIBLE_FALSE_POS` | 29 dòng | Người (review) | Có box nhưng model không phát hiện vật thể (trọng số 2); cần kiểm tra. |
| `POSSIBLE_LOOSE_BOX` | 31 box | Người (review) | Đúng lớp nhưng IoU < 0.5 (trọng số 1.5); cần chỉnh box. |

**Hàng đợi `05_review_queue.csv`:**
- Có **130 ảnh**, xếp theo tổng điểm lỗi giảm dần. Sáu ảnh `MISSING_LABEL_FILE` có trọng số cao nhất (`NEEDS_LABEL`: 10), được ưu tiên đầu hàng đợi; tiếp theo là `POSSIBLE_WRONG_CLASS` (3), `POSSIBLE_MISSING` (2) và `POSSIBLE_FALSE_POS` (2).
- Với ngân sách **80 ảnh**, reviewer xử lý 6 ảnh thiếu nhãn và 74 ảnh có lỗi ngữ nghĩa nghiêm trọng nhất. Cảnh báo checker giảm **146 xuống 74**, xử lý triệt để lỗi trên **86 ảnh**. Còn **44 ảnh** chưa duyệt (21.67%), nên pipeline vẫn HOLD.

## 3. Calibration (2 ngưỡng)

| Tham số | Ban đầu | Chọn | Precision / Recall (trước → sau) | Lý do |
|---|---:|---:|---|---|
| `duplicates.phash_dist` | 2 | 6 | DUP_IMAGE_NEAR: 1.00 / 0.50 → 1.00 / 0.67 (calib) | Tăng recall để bắt ảnh trùng sau resize nhẹ (`img_0223`/`img_0077`, dist=6) và rò rỉ test (`img_0221`), vẫn giữ Precision 1.00. Trên calib (39 ảnh), dist=4 và 6 đều đạt R=0.67; chọn 6 vì cặp ảnh trên raw có pHash=6, ngưỡng 4 sẽ bỏ sót. |
| `second_opinion.conf` | 0.25 | 0.50 | MISSING: 0.50 / 1.00 → 0.57 / 1.00; WRONG_CLASS: 0.56 / 0.83 → 0.56 / 0.83 | Precision tăng 0.50 lên 0.57, giảm cảnh báo giả mà vẫn giữ Recall 1.00 cho MISSING. Không chọn conf > 0.6 vì Recall giảm (0.88 ở 0.7; 0.75 ở 0.8). |

**Calibration Trap:**
- Sweep trên `data/calib/` cho thấy `phash_dist=16` có Recall cao nhất (0.83 so với 0.67), Precision vẫn 1.00.
- Tuy nhiên, calib chỉ có 39 ảnh và không có kho test tham chiếu; không xuất hiện va chạm băm trong không gian 64-bit. Khi áp dụng dist=16 lên `raw` (224 ảnh) và kho test (140 ảnh), số ảnh bị gắn cờ trùng/rò rỉ tăng từ **16 lên 26**:
  - `img_0013` bị gắn `LEAK_TEST` nhầm với `test/test_087.jpg`, dù độ lệch pixel trung bình là 114.56 và nội dung khác nhau.
  - `img_0011` bị gắn trùng với `img_0008` (độ lệch pixel 66.21).
- Vì `remove_image` tự loại ảnh, dùng ngưỡng 16 sẽ xóa nhầm ít nhất **9 ảnh sạch**. Do đó giữ `phash_dist=6` để bảo toàn Precision 1.00 trên dữ liệu thật.

## 4. Cửa chặn và điểm mù

**(a) Cửa chặn hoạt động đúng:**
- Chạy thử với ngân sách duyệt 0: `python -m pipeline.run --budget 0 --reports work/rep_nb --export-dir work/exp_nb`. Cả **6 ảnh** thiếu nhãn (`MISSING_LABEL_FILE`) đều chưa được gán, nên vòng 3 còn `n_errors = 6`.
- Gate kết luận **REWORK**, mã thoát **20**; pipeline dừng và không xuất `work/exp_nb/clean_v1`, ngăn phát hành dataset lỗi.

**(b) Ba điểm mù có bằng chứng:**
1. **Box lỏng nhẹ/lệch viền:** Kiểm tra cú pháp không đánh giá độ ôm sát. Checker chỉ gắn `POSSIBLE_LOOSE_BOX` khi IoU < `iou_loose` (0.50); box có IoU 0.55–0.70 vẫn được xem là tốt.
2. **Người gán và model cùng nhầm:** Với vật thể che khuất nặng hoặc góc chụp khó (ví dụ xe tải nhỏ bị gán thành `car`), nếu model thứ hai (`yolo26l`) cũng dự đoán sai, checker đồng thuận ngầm và bỏ lọt lỗi ngữ nghĩa.
3. **Ảnh trùng bị biến đổi mạnh:** Để tránh xóa nhầm, `phash_dist` cố định ở 6. Ảnh crop > 50%, xoay hoặc chỉnh màu mạnh có thể có pHash > 6 nên không được phát hiện.

## 5. Train: raw vs clean (tùy chọn)

Chưa chạy; bỏ qua train để tập trung kiểm định pipeline lõi bằng `make check-core`.

## 6. Quyết định và ghi chú AI

**(a) Quyết định cuối:**
- Hiện tại: **HOLD**.
- Để chuyển sang **RELEASE**, tăng `budget_images` từ **80 lên ít nhất 110 ảnh** (duyệt thêm khoảng **24/44 ảnh** còn cảnh báo), đưa tỷ lệ ảnh nghi ngờ xuống dưới 10% (**≤ 20/203 ≈ 9.85%**). Sau khi reviewer gán/sửa nhãn, chạy lại pipeline để phát hành Release.

**(b) Sử dụng AI:**
- **Prompt:** Yêu cầu triển khai 4 module theo TDD (`label_format.py`, `gate.py`, `autofix.py`, `second_opinion.py`) theo docstring và chuẩn slide Day 09; phải pass toàn bộ unit test, không ảnh hưởng `data/raw/`.
- **Kiểm tra code:** Chạy 79 test (`pytest -q`), đối chiếu sweep calibration trên `data/calib/`, và quét lỗi tĩnh bằng `scripts/check_submission.py --no-train`.
- **Lỗi AI và cách sửa:** Trong `label_format.py`, AI đặt `continue` ngay sau lỗi `BOX_OUT_OF_IMAGE`, khiến bỏ qua lỗi sau đó trên cùng dòng như `DUP_BOX`. Test `test_line_with_two_errors_is_dropped_once` phát hiện lỗi; tôi bỏ thoát sớm để ghi nhận đủ cả hai lỗi.
