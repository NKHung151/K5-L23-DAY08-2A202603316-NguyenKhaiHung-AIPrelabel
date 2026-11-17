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
- Lệnh `merge_prelabels.py` đã dùng (dán nguyên): <<điền ở Part 4>>
- **Phiên bản SAM bạn chọn** (SAM 1 / 2.1 / 3) và lý do: SAM 2.1 bản tiny (`sam2.1_t.pt`), chạy CPU khoảng 3.12 giây/ảnh (`r05`); chọn bản tiny vì máy không có GPU. <<bổ sung nhận xét về chất lượng mask của bạn>>
- Lỗi nào **không sửa được bằng đổi ngưỡng** (ví dụ tuk-tuk bị gọi `car`)? Sửa thế nào? YOLO26 gọi xe ba bánh là `truck` (img_002, img_011), `motorbike` (img_007), `bicycle` (đuôi xích lô) và sót xích lô ở img_032 (`r02`). Tăng ngưỡng lên 0.5 (`r03`) chỉ làm các box nhầm biến mất thành sót, không đổi được lớp. Hướng sửa: dùng YOLOE gõ chữ `three_wheeler` (`r04`), nhưng YOLOE cũng sót nhiều (img_007, img_011, img_032) và dương tính giả (chồng đồ, truck ở img_002), nên khi review phải kiểm cả hai hướng.
- Box **điểm cao mà sai** hoặc **điểm thấp mà đúng** (ảnh nào): <<>>
- Giới hạn: chọn ngưỡng trên chính gold (4 ảnh) nên số đo máy trên gold là lạc quan: ngưỡng `car` và `three_wheeler` chọn và chấm trên cùng 4 ảnh gold (mẫu 8 và 7 vật), nên là ước lượng thô. Các lớp `bus`, `truck` quá ít vật để chọn ngưỡng. Thêm nữa, YOLO26n (nano) bỏ sót nhiều người và xe đạp ngay cả ở ngưỡng thấp. <<bổ sung nhận xét của bạn>>

## 2. AI error profile (từ `qc/review_stats.md`)

| Trạng thái | Số | Tỉ lệ | Mẫu số |
|---|---|---|---|
| accepted | <<>> | <<>> | <<box máy>> |
| edited | <<>> | <<>> | <<>> |
| deleted | <<>> | <<>> | <<>> |
| added | <<>> | <<>> | <<đối tượng sau duyệt>> |

- Lớp nào máy tệ nhất? Vì sao (dẫn ảnh)? <<>>
- So với gold: precision / recall của **máy** <<>> (mẫu số: <<đối tượng gold>>)

## 3. Thiên kiến tự động hóa và ảnh bẫy

- Bắt bẫy: <<.../12>>; loại nào lọt: <<>>; vì sao (bạn đã bỏ bước nào của checklist)? <<>>
- Accepted cao mà bắt bẫy thấp (hoặc ngược lại) nói gì về cách bạn duyệt? <<>>

## 4. Máy có đáng dùng không?

- Thời gian (từ `time_log.csv`): gán tay <<... phút/ảnh>> vs duyệt <<... phút/ảnh>>. Thiên lệch (bạn gán gold trước nên đã quen cảnh): <<>>
- Kết luận: <<có / không / chỉ cho lớp nào>>, dựa trên <<số liệu>>
- **Ba quy tắc giảm thiên kiến của bạn** (mỗi quy tắc kiểm tra được bằng số hoặc việc làm):
  1. <<>>
  2. <<>>
  3. <<>>

## 5. Mask SAM cho 3 vật (Part 5)

| Ảnh | SAM (phiên bản) | Số click | Mask gồm người lái / hàng hoá? | Lỗi quan sát | File trong `masks/` |
|---|---|---|---|---|---|
| <<img_0xx>> | <<>> | <<>> | <<>> | <<>> | <<>> |

- Đối chiếu guideline mục 4.2: quy tắc nào giúp, quy tắc nào thiếu? <<>>
- *(Tuỳ chọn)* Interactor SAM trong CVAT: làm được / không (lỗi gì, đã thử gì); khác gì offline: <<>>

## 6. Calibration (Part 6)

- Số quyết định: <<>> (`calibration_decisions.csv`); quan trọng nhất (dẫn `CAL-xx`): <<>>
- Quy tắc mới đã vào `guideline_v1.1_calibrated.md`: <<>>
- Nguyên nhân gốc: <<guideline mơ hồ / chưa nắm quy tắc / máy>>

## 7. QC và quyết định (Part 6)

- Mẫu: <<n>> ảnh (<<m>> ảnh `can_review` + <<k>> ngẫu nhiên, seed 42); người kiểm: bạn (**hạn chế**: bạn vừa là người duyệt).
- Lỗi theo **đối tượng**: <<.../... = ...%>> · theo **ảnh**: <<.../... = ...%>> (ngưỡng Release ≤ 2%)
- Top lỗi (trong: bỏ sót, thừa, sai lớp, sai vị trí/kích thước, sai thuộc tính, sai mask) + một ảnh ví dụ mỗi loại: <<>>
- **Giới hạn mẫu nhỏ** (cận trên 95% trong `qc_report_auto.md`): <<>>
- Script kiểm được / không kiểm được (ví dụ ôm sát, cùng bỏ sót với máy, theo checklist thật): <<>>

**Quyết định: <<Release / Hold / Rework>>**, ngày <<YYYY-MM-DD>>. Lý do (đi theo Hold → Rework → Release): <<>>
Nếu Rework / Hold, việc cần làm tiếp: <<>>

## 8. Ghi chú cá nhân

- **Ca khó khi gán gold (tick `can_review`, nguyên liệu cho calibration Part 6):**
  - `img_020`: một vật thể lớn không phân biệt được là `bus` hay `truck`. Đã chọn `bus` và tick `can_review`. Lý do chọn / dấu hiệu nhìn thấy: <<điền: ví dụ buồng lái + thùng riêng, không thấy cửa hành khách, số bánh...>>. Guideline v1.0 chưa nêu dấu hiệu phân biệt `bus` / `truck` khi nhìn không rõ.
- **Dùng AI để làm bài** (công cụ nào, việc gì, kiểm tra thế nào; "không dùng" nếu không): Claude Code (Claude Sonnet 5.5) trong VS Code. Việc AI làm: đọc repo và giải thích các Part; cài thư viện thiếu và copy trọng số vào `models/`; tạo bản `cvat/labels_cvat_raw.json` để CVAT chấp nhận; chạy `prepare_workdirs.py`, `validate_labels.py`; copy file export CVAT thành `submission/labels/gold_manual.json`; ghi các dòng `model_log.csv`, `time_log.csv` và các số liệu `threshold_sweep` vào `REPORT.md`. AI **không** gán nhãn và không gợi ý nhãn cho 4 ảnh gold. Các nhận xét về lỗi của YOLO26 / YOLOE / SAM trong `model_log.csv` là do tôi nhìn ảnh `viz/` rồi nói lại, AI chỉ ghi vào file. Kiểm tra: đối chiếu số liệu với `work/runs.csv`, `threshold_sweep.md` và kết quả `validate_labels.py`.
- Gold có bị nhiễm không (bạn có nhìn pre-label hay hỏi AI trước khi gán xong)? <<không / có, như sau: ...>>
- Máy / GPU / CVAT dùng; vướng mắc cài đặt: Windows 11, không có GPU (chạy CPU), Python 3.11 trong `.venv`; CVAT chạy bằng Docker local, kèm 3 function Nuclio (YOLO26n, YOLOE, SAM2 tiny). Vì CPU nên dùng `yolo26n` (nano), `yoloe-26s-seg` và `sam2.1_t.pt` thay cho bản `m` / `sam2.1_b` trong GUIDE. Vướng mắc: `pip.exe` bị Windows Application Control chặn (dùng `python -m pip`); `check_env.py` ban đầu FAIL vì thiếu `transformers`, và trọng số nằm ngoài `models/` của repo nên phải copy vào.
- Dùng cache `data/cache/` thay vì tự chạy? không, đã tự chạy `prelabel.py` trên 8 ảnh pilot và trên cả 36 ảnh (`work/runs.csv`: r02–r07).
- Lỗi / bug script đã gặp (nếu có): (1) `cvat/labels_generated.json` có `"values": []` cho `conf` / `can_review`, bị bộ soạn thảo Raw của CVAT từ chối ("attribute values must be a non-empty array"); đã tạo bản `cvat/labels_cvat_raw.json` (đổi thành `[""]` và `["false"]`), giữ nguyên 8 tên nhãn, không sửa file gốc. (2) `prepare_workdirs.py` lỗi `UnicodeEncodeError` (cp1252) khi in tiếng Việt trên Windows; chạy lại với `PYTHONUTF8=1`, không sửa script. (3) `check_submission.py` kiểm tra tiêu đề README bằng regex `\d{5,}` nên không nhận MSHV có chữ cái (`2A202603316`); chưa sửa script.
- Một điều bạn làm khác lần sau: <<>>
