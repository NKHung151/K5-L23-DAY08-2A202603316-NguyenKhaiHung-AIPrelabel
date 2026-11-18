# Báo cáo Day 08 — 2A202603316 - Nguyễn Khải Hưng

> Một file duy nhất cho cả bài (đã gộp recipe, QC và ghi chú cá nhân). Mọi con số kèm **mẫu số** và nguồn (file nào). Viết bằng lời của bạn.

## 1. Model và recipe (Part 2–3)

| Lớp | Model + cỡ | Ngưỡng | Vì sao (dẫn `run_id` trong `model_log.csv`) |
|---|---|---|---|
| person / bicycle / motorbike / car / bus / truck | YOLO26n (bản nano, chạy CPU; không dùng bản m) | `car` = 0.45; `person`, `bicycle`, `motorbike`, `bus`, `truck` = mặc định 0.3 (nguồn: `qc/threshold_sweep.md`) | `car` = 0.45 vì đạt recall 100% trên gold (8 vật); các lớp còn lại không ngưỡng nào ≤ 0.5 đạt recall 90% nên giữ mặc định. YOLO26 chạy nhanh trên CPU (0.31 giây/ảnh, `r02`) nhưng nhầm xe ba bánh sang truck / motorbike / bicycle, và tăng ngưỡng lên 0.5 chỉ biến lỗi nhầm thành sót (`r03`), nên không dùng cho `three_wheeler`. |
| three_wheeler | YOLOE-26s (`yoloe-26s-seg`), prompt: `three_wheeler` | 0.15 (recall 100%, precision 78%, 7 vật gold) | YOLOE gõ chữ chỉ trả về `three_wheeler`, không nhầm sang lớp COCO như YOLO26; ngưỡng 0.15 đạt recall 100% trên gold (7 vật), precision 78%. Hạn chế: sót xe ba bánh (img_007, img_011, img_032) và dương tính giả ở img_002 (`r04`), nên phải review kỹ (xem mục 3–4). |

Kết quả `threshold_sweep` trên 4 ảnh gold (mục tiêu recall 90%, ngưỡng thử ≤ 0.5):

| Lớp | Số vật gold | Kết quả | Ngưỡng dùng |
|---|---|---|---|
| person | 23 | recall tối đa 61%, không đạt 90% | mặc định 0.3 |
| bicycle | 5 | recall tối đa 20%, không đạt 90% | mặc định 0.3 |
| motorbike | 10 | recall tối đa 80%, không đạt 90% | mặc định 0.3 |
| car | 8 | recall 100%, precision 67% | **0.45** |
| bus | 1 | < 5 vật: mẫu quá ít, không chọn ngưỡng riêng | mặc định 0.3 |
| truck | 0 | không có vật trong gold, không chọn ngưỡng riêng | mặc định 0.3 |
| three_wheeler | 7 | recall 100%, precision 78% | **0.15** |

- Chuỗi ngưỡng sẽ dùng cho `merge_prelabels.py`: `--thr "car=0.45,three_wheeler=0.15"`
- Lệnh `merge_prelabels.py` đã dùng (dán nguyên): `python scripts/merge_prelabels.py work/prelabel/yoloe_all/pre.coco.json work/prelabel/y26_all/pre.coco.json --thr "car=0.45,three_wheeler=0.15" --conf 0.3 --iou 0.6 --split batch,gold --out work/prelabel/merged_review/pre.coco.json`
- **Phiên bản SAM bạn chọn** (SAM 1 / 2.1 / 3) và lý do: SAM 2.1 bản tiny (`sam2.1_t.pt`), chạy CPU khoảng 3.12 giây/ảnh (`r05`); chọn bản tiny vì máy không có GPU. Mask tốt khi click có đủ điểm dương và âm, nhưng lẫn vật khác khi hai vật chồng nhau (xem mục 5).
- Lỗi nào **không sửa được bằng đổi ngưỡng** (ví dụ tuk-tuk bị gọi `car`)? Sửa thế nào? YOLO26 gọi xe ba bánh là `truck` (img_002, img_011), `motorbike` (img_007), `bicycle` (đuôi xích lô) và sót xích lô ở img_032 (`r02`). Tăng ngưỡng lên 0.5 (`r03`) chỉ làm các box nhầm biến mất thành sót, không đổi được lớp. Hướng sửa: dùng YOLOE gõ chữ `three_wheeler` (`r04`), nhưng YOLOE cũng sót nhiều (img_007, img_011, img_032) và dương tính giả (chồng đồ, truck ở img_002), nên khi review phải kiểm cả hai hướng.
- Box **điểm cao mà sai** hoặc **điểm thấp mà đúng** (ảnh nào): điểm cao mà sai (điểm lấy từ `work/prelabel/merged_review/pre.coco.json`): `img_003` có 8 box `three_wheeler` (3 box điểm 0.61–0.9, 5 box điểm 0.16–0.25) nhưng thực tế chỉ có 1 xe ba bánh thật, hai xe tải ở ảnh này bị gán thành `three_wheeler`; `img_008` có 4 box `three_wheeler` (điểm 0.21, 0.25, 0.86, 0.9) nhưng thực tế chỉ có 3 xe; `img_013` có 4 box `three_wheeler` (điểm 0.18, 0.63, 0.67, 0.75), thực tế 1 xe bị gán 2 box ba bánh. Điểm thấp mà sai: `img_009` không có xe ba bánh nào nhưng 2 box (điểm 0.27, 0.29) gán cho một cái lều và một xe máy chở hàng; `img_014` 3 box (điểm 0.19, 0.33, 0.63) gán cho một cái thùng và hai thùng đồ hàng. Lỗi không chỉ ở `three_wheeler`: nhiều ảnh phía sau cũng có box `person` sai. Điểm thấp mà đúng: chưa thống kê riêng, không nêu ca cụ thể trong 24 ảnh batch.
- Giới hạn: chọn ngưỡng trên chính gold (4 ảnh) nên số đo máy trên gold là lạc quan: ngưỡng `car` và `three_wheeler` chọn và chấm trên cùng 4 ảnh gold (mẫu 8 và 7 vật), nên là ước lượng thô. Các lớp `bus`, `truck` quá ít vật để chọn ngưỡng. Thêm nữa, YOLO26n (nano) bỏ sót nhiều người và xe đạp ngay cả ở ngưỡng thấp. Khi áp ngưỡng `three_wheeler = 0.15` lên 24 ảnh batch, 20/52 box `three_wheeler` của file gộp (batch + gold) có điểm chỉ 0.15–0.3, và khi review thấy rất nhiều box nhầm: ngưỡng chọn từ 4 ảnh gold (7 vật) không khái quát sang lô thật, đúng như cảnh báo của `threshold_sweep`. Ngoài ra khi gộp, YOLOE đứng trước YOLO26 nên box `three_wheeler` có thể đã đè lên `truck` / `car` đúng (suy luận từ cách `merge_prelabels.py` giữ box ưu tiên cao hơn, chưa kiểm chứng từng box). 

## 2. AI error profile (từ `qc/review_stats.md`)

| Trạng thái | Số | Tỉ lệ | Mẫu số |
|---|---|---|---|
| accepted | 265 | 81.3% | 326 box máy (đã loại 29 đối tượng quanh bẫy) |
| edited | 24 | 7.4% | 326 box máy |
| deleted | 34 | 10.4% | 326 box máy |
| added | 92 | 24.0% | 384 đối tượng sau duyệt (≈ 88 là máy sót thật, 4 là đổi lớp) |

Bảng trên **không tính 29 đối tượng nằm quanh các ảnh bẫy** (`review_stats.md`): bẫy là lỗi nhân tạo do script cài sẵn (box thừa, sai lớp, thiếu, lỏng), không phản ánh chất lượng thật của máy, nên nếu tính vào sẽ làm sai tỉ lệ accepted / edited / deleted / added của máy. Các bẫy được đo riêng ở mục 3.

Theo lớp (accepted / edited / deleted / added): person 157/6/4/34 · motorbike 67/9/3/31 · car 24/2/2/16 · three_wheeler 12/7/23/5 · bicycle 1/0/2/3 · bus 2/0/0/1 · truck 2/0/0/2 (nguồn: `qc/review_stats.md`).

- Lớp nào máy tệ nhất? Vì sao (dẫn ảnh)? `three_wheeler`: trong 42 box máy của lớp này (12 + 7 + 23) có 23 box bị xoá (23/42 = 54.8%), nhiều nhất trong các lớp; các ca đã gặp: `img_009` (cái lều và xe máy chở hàng bị gán ba bánh), `img_014` (thùng hàng), `img_003` (8 box nhưng chỉ 1 xe thật), `img_013`, `img_008` (xem mục 1). Về số vật máy bỏ sót, `person` có 34 box, `motorbike` 31 box và `car` 16 box phải vẽ thêm. Lý do: ngưỡng 0.15 thấp (20/52 box `three_wheeler` điểm 0.15–0.3) và YOLOE chỉ có một prompt `three_wheeler` nên gán lớp này cho cả vật giống xe (lều, thùng, đồ hàng).
- So với gold: precision / recall của **máy** 81% (KTC 95%: 67–90%) / 63% (KTC 95%: 50–75%) (mẫu số: 54 đối tượng gold; TP 34, FP 8, FN 20). Con số lạc quan vì ngưỡng chọn trên chính 4 ảnh gold.

## 3. Thiên kiến tự động hóa và ảnh bẫy

- Bắt bẫy: 11/12 = 91.7% (KTC 95%: 65–99%, vì chỉ có 12 bẫy nên khoảng rất rộng); theo loại: `loose` 3/3, `wrong_class` 3/3, `extra` 3/3, `missing` 2/3. Loại nào lọt: 1 bẫy `missing` ở `img_018` (ảnh dày xe máy). Vì sao (bạn đã bỏ bước nào của checklist)? `img_018` là ảnh dày xe máy (29 box pre-label), lỗi thiếu khó thấy khi nhiều vật; lượt quét ảnh trước chưa đủ với ảnh dày (nhận định, chưa kiểm chứng riêng).
- Accepted cao mà bắt bẫy thấp (hoặc ngược lại) nói gì về cách bạn duyệt? Accepted 81.3% và bắt bẫy 91.7%, nhưng recall của máy so với gold chỉ 63% và phải vẽ thêm khoảng 88 vật sót: loại lỗi khó bắt nhất là lỗi **thiếu** (`missing`), không phải lỗi sai lớp. Tôi bắt tốt lỗi sai lớp và box thừa nhưng không bắt hết lỗi thiếu; accepted cao một mình không chứng minh máy tốt.

## 4. Máy có đáng dùng không?

- Thời gian (từ `time_log.csv`): gán tay 19 phút / 4 ảnh = 4.75 phút/ảnh vs duyệt 41 phút / 24 ảnh ≈ 1.7 phút/ảnh (thời gian duyệt do tôi tự báo, có thể không bấm giờ chính xác). Thiên lệch (bạn gán gold trước nên đã quen cảnh): gold 4 ảnh vẽ tay từ đầu (4.75 phút/ảnh) còn batch 24 ảnh chỉ sửa nhãn máy (1.7 phút/ảnh), mỗi ảnh có số vật khác nhau nên không so được công bằng; tôi gán gold trước nên đã quen cảnh phố và quy tắc, làm duyệt nhanh hơn.
- Kết luận: Có, nhưng chỉ cho `person` (accepted 157/167 box máy = 94.0%) và `motorbike` (67/79 = 84.8%); không dùng nguyên trạng cho `three_wheeler` (accepted 12/42 = 28.6%, xoá 23/42 = 54.8%), dựa trên `qc/review_stats.md` (recall của máy so với gold 63%, 34/54).
- **Ba quy tắc giảm thiên kiến của bạn** (mỗi quy tắc kiểm tra được bằng số hoặc việc làm):
  1. Quét ảnh trước khi nhìn nhãn máy; với ảnh có > 20 box máy thêm một lượt chỉ tìm vật bị thiếu. Kiểm tra bằng số: bắt bẫy `missing` (lần này 2/3) phải đạt 3/3 ở lô sau.
  2. Lớp có accepted < 60% (`three_wheeler` 28.6%) thì kiểm từng box, không chấp nhận hàng loạt, và lọc theo lớp để rà lại sau khi sửa. Kiểm tra bằng số: QC không còn box `three_wheeler` thừa (QC mục 7: 0 box thừa trong mẫu).
  3. Bấm Save và export sớm, rồi đếm lại số box ở vài ảnh đã sửa trong file export trước khi chuyển bước (từng có lần export thiếu chỉnh sửa vì chưa lưu kịp). Kiểm tra bằng việc làm: số box file export khớp số box trong CVAT.

## 5. Mask SAM cho 3 vật (Part 5)

| Ảnh | SAM (phiên bản) | Số click | Mask gồm người lái / hàng hoá? | Lỗi quan sát | File trong `masks/` |
|---|---|---|---|---|---|
| img_013 (tuk-tuk) | SAM 2.1 tiny (`sam2.1_t.pt`) | 5 (3 dương, 2 âm) | Có: gồm cả người lái và hành khách ngồi trong xe (mask 48.2% ảnh) | Lẫn người vào mask xe; vài chỗ thủng / tràn nhỏ ở mép trước và kính | `submission/masks/tuktuk_img013.jpg` |
| img_013 (người lái tuk-tuk) | SAM 2.1 tiny | 4 (2 dương, 2 âm) | Chỉ người lái (đầu, thân, chân) | Ôm khá sát, thêm 1 polygon nhỏ lẻ (2 polygon) | `submission/masks/driver_img013.jpg` |
| img_013 (xe máy bị tuk-tuk che một phần) | SAM 2.1 tiny | 4 (2 dương, 2 âm) | Không có người | Mask tràn sang phần đầu xe màu đỏ của tuk-tuk, dính hai vật vào một mask | `submission/masks/motorbike_occluded_img013.jpg` |

- Đối chiếu guideline mục 4.2: quy tắc nào giúp, quy tắc nào thiếu? Giúp: 4.2 quy tắc 4 (hai vật chạm nhau: mỗi vật một polygon, vùng chạm thuộc vật phía trước) hợp với mask người lái. Thiếu: mask xe có gồm người ngồi trong xe không (mask tuk-tuk gồm cả người lái và hành khách, trong khi mục 2 coi người là vật riêng), và quy tắc khi vật bị che dính vật che (xe máy tràn sang đầu xe tuk-tuk).
- *(Tuỳ chọn)* Interactor SAM trong CVAT: làm được / không (lỗi gì, đã thử gì); khác gì offline: chưa thử Interactor SAM trong CVAT (tuỳ chọn); chỉ dùng `sam2_points.py` offline.

## 6. Calibration (Part 6)

- Số quyết định: 3 (`calibration_decisions.csv`: CAL-01 `loi_may`, CAL-02 và CAL-03 `cau_hoi_mo`); quan trọng nhất: CAL-01 (lỗi lặp lớn nhất của máy: `three_wheeler` xoá 23/42).
- Quy tắc mới đã vào `guideline_v1.1_calibrated.md`: Dấu hiệu `bus` / `truck` (thùng hàng riêng), thêm "không gồm" cho `three_wheeler` (lều, thùng hàng, đồ hàng), và ngưỡng dùng `ignore_region` (từ 5 vật, sát nhau, không tách được, hoặc cao dưới 20 px).
- Nguyên nhân gốc: CAL-01: máy (ngưỡng thấp, một prompt); CAL-02 và CAL-03: guideline mơ hồ.

## 7. QC và quyết định (Part 6)

- Mẫu: 5 ảnh (0 ảnh `can_review` + 5 ngẫu nhiên, seed 42); người kiểm: Nguyễn Khải Hưng (tôi), bằng mắt trên ảnh `work/qc/review/` (**hạn chế**: người kiểm cũng là người review nhãn, và kiểm bằng mắt không đo được độ ôm sát từng box).
- Lỗi theo **đối tượng**: 0/75 = 0.0% · theo **ảnh**: 0/5 = 0.0% (ngưỡng Release ≤ 2%). Nhãn kiểm là nhãn cuối `submission/labels/batch_reviewed.json` (419 đối tượng, gồm 19 `ignore_region` ở 10 ảnh theo CAL-03).
- Top lỗi (trong: bỏ sót, thừa, sai lớp, sai vị trí/kích thước, sai thuộc tính, sai mask) + một ảnh ví dụ mỗi loại: không ghi nhận lỗi nào trong mẫu (kiểm tổng quát, không rà kỹ từng đối tượng); các vùng `ignore_region` ở `img_014` và `img_035` chưa kiểm được vì script không vẽ vùng này.
- **Giới hạn mẫu nhỏ** (cận trên 95% trong `qc_report_auto.md`): 0 ảnh lỗi trong 5 ảnh mẫu, cận trên 95% (Wilson) của tỉ lệ ảnh lỗi thật là **43%**; mẫu 5/24 ảnh không thể chứng minh ngưỡng 2%, nên kết quả Release dưới đây là yếu về mặt thống kê.
- Script kiểm được / không kiểm được (ví dụ ôm sát, cùng bỏ sót với máy, theo checklist thật): Script kiểm được tên file, lớp, box trong ảnh, số đo accepted/edited/deleted/added; không kiểm được box có ôm sát vật thật, lớp có đúng thực tế, hay vật mà cả tôi và máy cùng bỏ sót.

**Quyết định: Release**, ngày 2026-11-18. Lý do (đi theo Hold → Rework → Release): không phải Hold (guideline không hỏng); không phải Rework (ảnh lỗi 0/5 = 0.0% ≤ 2%, không có lỗi hệ thống, `qc/qc_report_auto.md`). Đây là kết luận có điều kiện: mẫu chỉ 5/24 ảnh (cận trên 43%), phần QC kiểm tổng quát, máy sót nhiều (recall 63% so với gold, `added` 92/384 = 24.0%), và một số box `three_wheeler` điểm thấp (ví dụ `img_010`, `img_023`, `img_022`) chưa được xác minh từng box.
Nếu Rework / Hold, việc cần làm tiếp: không áp dụng (Release). Nên làm nếu có thời gian: kiểm từng box `three_wheeler` điểm thấp và các ảnh dày vật; mở rộng mẫu QC.

## 8. Ghi chú cá nhân

- **Ca khó khi gán gold (tick `can_review`, nguyên liệu cho calibration Part 6):**
  - `img_020` (box `bus`, id 40; đây là box **duy nhất** tick `can_review` trong `gold_manual.json`): một vật thể lớn không phân biệt được là `bus` hay `truck`. Đã chọn `bus` và tick `can_review`. Lý do chọn / dấu hiệu nhìn thấy: Không thấy thùng hàng, phần cabin nhìn có dạng phẳng, giống với xe bus. Guideline v1.0 chưa nêu dấu hiệu phân biệt `bus` / `truck` khi nhìn không rõ.
- **Dùng AI để làm bài** (công cụ nào, việc gì, kiểm tra thế nào; "không dùng" nếu không): Claude Code (Claude Sonnet 5.5) trong VS Code. Việc AI làm: đọc repo và giải thích các Part; cài thư viện thiếu và copy trọng số vào `models/`.
- Gold có bị nhiễm không (bạn có nhìn pre-label hay hỏi AI trước khi gán xong)? Không: gán tay và export 4 ảnh gold trước khi chạy model lên bất kỳ ảnh nào. Trong lúc gán có gửi AI một ảnh chụp màn hình để hỏi cách xử lý ca không chắc; AI chỉ dẫn quy tắc guideline, không gợi ý nhãn.
- Máy / GPU / CVAT dùng; vướng mắc cài đặt: Windows 11, không có GPU (chạy CPU), Python 3.11 trong `.venv`; CVAT chạy bằng Docker local, kèm 3 function Nuclio (YOLO26n, YOLOE, SAM2 tiny). Vì CPU nên dùng `yolo26n` (nano), `yoloe-26s-seg` và `sam2.1_t.pt` thay cho bản `m` / `sam2.1_b` trong GUIDE. Vướng mắc: `pip.exe` bị Windows Application Control chặn (dùng `python -m pip`); `check_env.py` ban đầu FAIL vì thiếu `transformers`, và trọng số nằm ngoài `models/` của repo nên phải copy vào.
- Dùng cache `data/cache/` thay vì tự chạy? không, đã tự chạy `prelabel.py` trên 8 ảnh pilot và trên cả 36 ảnh (`work/runs.csv`: r02–r07).
- Lỗi / bug script đã gặp (nếu có): (1) `cvat/labels_generated.json` có `"values": []` cho `conf` / `can_review`, bị bộ soạn thảo Raw của CVAT từ chối ("attribute values must be a non-empty array"); đã tạo bản `cvat/labels_cvat_raw.json` (đổi thành `[""]` và `["false"]`), giữ nguyên 8 tên nhãn, không sửa file gốc. (2) `prepare_workdirs.py` lỗi `UnicodeEncodeError` (cp1252) khi in tiếng Việt trên Windows; chạy lại với `PYTHONUTF8=1`, không sửa script. (3) `check_submission.py` kiểm tra tiêu đề README bằng regex `\d{5,}` nên không nhận MSHV có chữ cái (`2A202603316`); chưa sửa script.

