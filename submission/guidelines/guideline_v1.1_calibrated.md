# Label contract — Phố châu Á — BỔ SUNG v1.1 — 2A202603216 - Đỗ Huy Kiên

> **Chỉ viết phần thay đổi so với v1.0** (không copy cả file). Chốt các câu hỏi mở ở mục 6 của v1.0
> và thêm quy tắc mới rút ra từ lỗi của máy, bẫy lọt, và các ca phân vân. Mỗi ca khó **có ảnh ví dụ** (`img_0xx`) và khớp một dòng trong `qc/calibration_decisions.csv`.

## 1. Câu hỏi mở đã chốt (từ mục 6 của v1.0)

| # | Câu hỏi | Quyết định | Ảnh ví dụ | `case_id` |
|---|---|---|---|---|
| 1 | Xe đạp điện / xe máy điện nhỏ: `bicycle` hay `motorbike`? | Xe hai bánh chạy điện có bàn đạp cơ học trợ lực tính là `bicycle`. Xe hai bánh chạy điện không có bàn đạp (chỉ có sàn để chân/gác chân) tính là `motorbike`. | `img_011.jpg`, `img_023.jpg` | `CAL-02` |
| 2 | Khối xe máy đỗ dày đặc gần camera (tiền cảnh): từng xe hay `ignore_region`? | Tuyệt đối không dùng `ignore_region` ở tiền cảnh. Nếu nhận diện được $\ge 20\%$ thân xe hoặc đầu/biển số thì bắt buộc vẽ từng box riêng. `ignore_region` chỉ áp dụng cho cụm xe ở xa có chiều cao $< 20\text{ px}$. | `img_018.jpg`, `img_021.jpg` | `CAL-03` |
| 5 | Kính xe nhìn xuyên thấu: có gán `person` cho người ngồi trong ô tô/xe bus không? | Nếu người ngồi trong xe nhìn thấy rõ qua kính trong suốt $\ge 20\%$ cơ thể thì gán 1 box `person` độc lập. Nếu kính phản chiếu bóng chói lóa hoặc quá mờ không phân biệt rõ đường nét thì không gán. | `img_007.jpg`, `img_012.jpg` | `CAL-05` |

## 2. Ca khó mới (từ lỗi của máy, bẫy lọt, box `can_review`)

| Ảnh | Tình huống | Máy làm gì (model nào) | Quyết định của bạn | Quy tắc mới (vào mục nào của v1.0) |
|---|---|---|---|---|
| `img_013.jpg`, `img_002.jpg` | Xe tuk-tuk chở khách có mui và khung rỗng ruột | YOLO26m nhận nhầm thành `car`; YOLOE-26s đôi khi sinh 2 box chồng lặp lên cùng 1 xe | Luôn phân loại là `three_wheeler`; xóa box `car`/`truck` chồng đè (IoU $\ge 0.6$). Người lái/khách ngồi bên trong nếu nhìn thấy $\ge 20\%$ thì vẽ box `person` riêng | **Mục 2 & 4.1.3:** Phân loại chuẩn hóa `three_wheeler` cho mọi biến thể tuk-tuk đường phố; khử lỗi lan truyền chồng đè nhãn (`CAL-01`) |
| `img_018.jpg`, `img_027.jpg` | Bounding box bị vẽ rộng lỏng (bẫy loose) hoặc viền máy sinh lệch ngoài đối tượng | Máy sinh box rộng quá mức hoặc người duyệt bỏ sót bẫy loose do không zoom kiểm tra | Quy định dung sai biên khắt khe: 4 cạnh phải chạm điểm ngoài cùng nhìn thấy của vật, sai số lệch $\le 2\text{ px}$. Bắt buộc zoom 200% kiểm tra tối thiểu 3 box mỗi ảnh | **Mục 4.1.4:** Quy tắc kiểm tra biên khít sát (tightness) bắt buộc khi duyệt có AI hỗ trợ (`CAL-04`) |
| `img_004.jpg`, `img_017.jpg` | Tách mask SAM cho xe tuk-tuk rỗng ruột và có phụ kiện gắn cố định | SAM tự động có thể phủ mask lên cả người lái hoặc ăn lẹm vào họa tiết trang trí trên mui xe | Thùng xe, khung sắt, mui xe, bánh xe, kính chắn gió và biển hiệu TAXI gắn cố định thuộc mask xe. Phải đặt điểm âm (`--neg`) khoét sạch người lái ra khỏi mask | **Mục 4.2.5:** Quy định phân tách mask pixel-level giữa buồng lái xe ba bánh và người ngồi lái (`CAL-01`) |

## 3. Lịch sử thay đổi

| Phiên bản | Thay đổi | Lý do (ca nào, `case_id`) |
|---|---|---|
| v1.0 | Bản giảng viên ban đầu | Nền tảng hợp đồng gán nhãn ban đầu |
| v1.1 | Chốt phân loại xe điện theo bàn đạp cơ học; siết chặt ranh giới `ignore_region` tiền cảnh vs hậu cảnh; chuẩn hóa lớp `three_wheeler` và khử box chồng đè; bổ sung tiêu chuẩn dung sai biên $\le 2\text{ px}$ kèm checklist zoom 200%; hướng dẫn tách mask SAM cho xe có phụ kiện | Rút ra từ phân tích lỗi máy YOLO26/YOLOE/SAM, bẫy lọt trong đợt review batch, và các câu hỏi mở còn tồn đọng (`CAL-01`, `CAL-02`, `CAL-03`, `CAL-04`, `CAL-05`) |
