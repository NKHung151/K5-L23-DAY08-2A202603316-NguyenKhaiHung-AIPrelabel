# Label contract — Phố châu Á — BỔ SUNG v1.1 — 2A202603316 - Nguyễn Khải Hưng

> **Chỉ phần thay đổi so với v1.0** (`guideline/guideline_v1.0_label_contract.md`). Các câu hỏi mở của v1.0 không nêu dưới đây là **chưa chốt** (xem mục 4).
> Mỗi ca khó có ảnh ví dụ (`img_0xx`) và khớp một dòng trong `qc/calibration_decisions.csv`.

## 1. Câu hỏi mở đã chốt (từ mục 6 của v1.0)

| # | Câu hỏi | Quyết định | Ảnh ví dụ | `case_id` |
|---|---|---|---|---|
| 2 | Khối xe máy đỗ dày đặc gần camera: từng xe hay `ignore_region`? Ngưỡng cụ thể? | (a) Vật nhìn rõ và cao **từ 20 px** trở lên: **vẽ từng vật**, dù cụm dày hay gần camera. (b) Chỉ dùng `ignore_region` khi cụm có **từ 5 vật trở lên**, **sát nhau và không tách được ranh giới** từng vật (hoặc từng vật cao dưới 20 px). (c) Vật đơn lẻ cao dưới 20 px: không vẽ, không cần `ignore_region`. | `img_018`, `img_027` | CAL-03 |

## 2. Ca khó mới (từ lỗi của máy, bẫy lọt, box `can_review`)

| Ảnh | Tình huống | Máy làm gì (model nào) | Quyết định của tôi | Quy tắc mới (vào mục nào của v1.0) |
|---|---|---|---|---|
| `img_009`, `img_014` | Lều, thùng hàng, đồ hàng treo trên đòn gánh, xe máy chở hàng | YOLOE (ngưỡng 0.15) gán `three_wheeler` cho các vật này (điểm từ 0.19 đến 0.63); YOLO26 gọi xe ba bánh là `truck` / `motorbike` / `bicycle` | Chỉ gán `three_wheeler` khi thấy **khung xe có bánh**. Lều, thùng hàng, hàng chở trên xe máy **không gán** | Mục 2, cột "Không gồm" của `three_wheeler`: thêm "lều, thùng hàng, đồ hàng treo / chở, xe máy chở hàng" (CAL-01) |
| `img_020` | Vật lớn nhìn không rõ là `bus` hay `truck` (box tick `can_review`) | — (box do tôi gán tay ở gold) | Có **thùng hàng riêng** phía sau buồng lái → `truck`. Thân liền một khối chở khách → `bus`. Không phân biệt được: chọn lớp tin nhất và tick `can_review` | Mục 2, các dòng `bus` và `truck`: thêm dấu hiệu phân biệt (CAL-02) |

## 3. Lịch sử thay đổi

| Phiên bản | Thay đổi | Lý do (ca nào, `case_id`) |
|---|---|---|
| v1.0 | Bản giảng viên | — |
| v1.1 | Thêm dấu hiệu `bus` / `truck`; thêm "không gồm" cho `three_wheeler`; chốt ngưỡng dùng `ignore_region` | CAL-01 (lỗi máy ở `img_009`, `img_014`), CAL-02 (`img_020`), CAL-03 (`img_018`, `img_027`) |

## 4. Câu hỏi mở chưa chốt ở v1.1

Các câu hỏi 1, 3, 4, 5, 6, 7 của mục 6 v1.0 (xe đạp điện; hàng nhô ra; người ngồi trong xe ba bánh có mui; kính xe; box chồng nhau trong ảnh đêm; mức tin cậy tối thiểu) **chưa chốt** vì chưa gặp đủ ca trong 24 ảnh batch để quyết định có cơ sở. Riêng mask của tuk-tuk có gồm người lái / hành khách hay không (`img_013`, mục 4.2 v1.0): mask SAM gồm cả người, trong khi mục 2 v1.0 coi người là vật riêng; chưa có quyết định chính thức.
