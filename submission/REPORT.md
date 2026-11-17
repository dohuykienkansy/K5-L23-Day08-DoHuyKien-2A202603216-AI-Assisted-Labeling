# Báo cáo Day 08 — 2A202603216 - Đỗ Huy Kiên

> Một file duy nhất cho cả bài (đã gộp recipe, QC và ghi chú cá nhân). Mọi con số kèm **mẫu số** và nguồn (file nào). Viết bằng lời của bạn.

## 1. Model và recipe (Part 2–3)

| Lớp | Model + cỡ | Ngưỡng | Vì sao (dẫn `run_id` trong `model_log.csv`) |
|---|---|---|---|
| person / bicycle / motorbike / car / bus / truck | YOLO26m | car=0.5; person, motorbike, bicycle, bus, truck dùng mặc định 0.3 | Dẫn `r01`, `r02`. Quét ngưỡng trên 4 ảnh gold (`threshold_sweep.md`): car đạt recall 90% ở conf 0.5 (precision 69%); các lớp person (max recall 74%), motorbike (max recall 68%), bicycle (max recall 60%) không đạt mục tiêu recall 90% ở bất kỳ ngưỡng nào <= 0.5 nên giữ mặc định 0.3; bus (1 vật) và truck (0 vật) mẫu quá ít (< 5) nên giữ mặc định 0.3. |
| three_wheeler | YOLOE-26s, prompt: 'tuk-tuk', 'cyclo', 'cargo tricycle' | 0.15 | Dẫn `r03`. Quét ngưỡng trên gold đạt recall 100% (7/7 vật), precision 78% (7/9 box) ở ngưỡng conf 0.15; bắt tốt các xe tuk-tuk đặc thù đường phố Đông Nam Á mà YOLO26 bỏ sót hoặc gọi nhầm car. |

- Lệnh `merge_prelabels.py` đã dùng (dán nguyên): `python scripts/merge_prelabels.py work/prelabel/yoloe_all/pre.coco.json work/prelabel/y26_all/pre.coco.json --thr "car=0.5,three_wheeler=0.15" --conf 0.3 --iou 0.6 --split "batch,gold" --out work/prelabel/merged_review/pre.coco.json`
- **Phiên bản SAM bạn chọn** (SAM 1 / 2.1 / 3) và lý do: SAM 2.1 (`sam2.1_b.pt`), dẫn `r04`. Chất lượng mask bám viền cực kỳ chính xác theo đường nét chi tiết của người và phương tiện; cân bằng tốt giữa độ nét biên và tốc độ xử lý trên CPU/RAM so với bản base/large.
- Lỗi nào **không sửa được bằng đổi ngưỡng** (ví dụ tuk-tuk bị gọi `car`)? Sửa thế nào? Lỗi phân loại sai bản chất (sematic error): xe ba bánh (tuk-tuk/xe ba gác) bị YOLO26 gán nhầm thành `car` hoặc `truck` với độ tin cậy cao do tập COCO chuẩn không có lớp three_wheeler. Đổi ngưỡng conf không giải quyết được vì box sai vẫn có điểm conf cao. Cách khắc phục: Dùng mô hình open-vocabulary YOLOE-26s với text prompt riêng cho `three_wheeler`, sau đó hợp nhất bằng `merge_prelabels.py` (loại bỏ box car/truck của YOLO26 bị trùng vị trí với three_wheeler của YOLOE).
- Box **điểm cao mà sai** hoặc **điểm thấp mà đúng** (ảnh nào):
  - Điểm cao mà sai: Chiếc tuk-tuk trong `img_002` bị YOLO26m gán nhãn `car` với điểm conf cao (~0.65).
  - Điểm thấp mà đúng: Chiếc tuk-tuk nhỏ ở hậu cảnh xa trong `img_011` có điểm conf chỉ ~0.18 từ YOLOE nhưng vẫn đúng là `three_wheeler`.
- Giới hạn: chọn ngưỡng trên chính gold (4 ảnh) nên số đo máy trên gold là lạc quan:
  1. **In-sample bias:** Ngưỡng được tối ưu hóa và đánh giá trực tiếp trên chính 4 ảnh gold, nên các chỉ số đo lường (Recall/Precision) trên tập gold mang tính lạc quan hơn so với thực tế khi áp dụng cho lô batch 24 ảnh.
  2. **Mẫu nhỏ (Small Sample Size):** 4 ảnh gold có số lượng mẫu rất hạn chế, đặc biệt các lớp hiếm như `bus` (1 vật), `truck` (0 vật) hay `three_wheeler` (7 vật) không thể đại diện cho toàn bộ góc chụp, ánh sáng và mật độ xe ngoài thực tế. Vì vậy các lớp thiếu mẫu bắt buộc phải giữ ngưỡng mặc định 0.3 để tránh overfit.

## 2. AI error profile (từ `qc/review_stats.md`)

| Trạng thái | Số | Tỉ lệ | Mẫu số |
|---|---|---|---|
| accepted | 396 | 396/432 = 91.7% | 432 box máy |
| edited | 15 | 15/432 = 3.5% | 432 box máy |
| deleted | 21 | 21/432 = 4.9% | 432 box máy |
| added | 144 | 144/556 = 25.9% | 556 đối tượng sau duyệt |

- Lớp nào máy tệ nhất? Vì sao (dẫn ảnh)?
  - `three_wheeler`: Máy sinh 26 box nhưng có tới 10 box bị xoá (deleted), 8 box phải chỉnh sửa (edited) và chỉ 8 box được chấp nhận (accepted = 8/26 = 30.8%). Nguyên nhân do YOLOE-26s nhận dạng theo prompt mở thường tạo box lỏng hoặc nhận nhầm xe tải nhỏ/xe bán tải (`img_002`, `img_023`).
  - `motorbike`: Máy bỏ sót nhiều nhất với 68 box phải vẽ thêm (`added = 68`), chủ yếu là các xe máy nhỏ, bị che khuất một phần ở hậu cảnh trong các ảnh mật độ dày (`img_005`, `img_024`).
- So với gold: precision / recall của **máy**: Precision = 77.3% (51/66, KTC 95%: 66–86%), Recall = 52.0% (51/98, KTC 95%: 42–62%, mẫu số: 98 đối tượng gold).

## 3. Thiên kiến tự động hóa và ảnh bẫy

- Bắt bẫy: 3/12 = 25.0% (KTC 95%: 9–53%); loại nào lọt: lọt 9 bẫy gồm:
  - `loose`: lọt 3/3 (`img_027`, `img_018`, `img_034`).
  - `extra`: lọt 2/3 (`img_033`, `img_035`).
  - `wrong_class`: lọt 2/3 (`img_017`, `img_005`).
  - `missing`: lọt 2/3 (`img_006`, `img_025`).
  - Vì sao (bạn đã bỏ bước nào của checklist)? Người duyệt bị chi phối bởi **Automation Bias** (thiên kiến tự động hóa). Cụ thể đã bỏ qua **Bước 0: Quét mắt trước khi nhìn nhãn máy** (không ẩn nhãn máy để tự nhẩm đếm độc lập số vật, dẫn đến phụ thuộc vào gợi ý của máy và bỏ sót bẫy missing), và bỏ qua **Bước 4: Chỉnh lỏng** (không zoom lớn kiểm tra độ khít sát biên $\le 2\text{ px}$ nên lọt toàn bộ 3 bẫy loose).
- Accepted cao mà bắt bẫy thấp (hoặc ngược lại) nói gì về cách bạn duyệt? Tỷ lệ `accepted` đạt tới 91.7% nhưng bắt bẫy chỉ đạt 25.0% thể hiện rõ hiện tượng **"phê duyệt tự động / lướt qua mặc định" (rubber-stamping)**. Người duyệt có xu hướng tin tưởng quá mức vào AI, chỉ can thiệp vào các lỗi cực kỳ lộ liễu mà không kiểm tra tỉ mỉ từng cạnh box hay quét tìm các vùng trống bị gán nhầm.

## 4. Máy có đáng dùng không?

- Thời gian (từ `time_log.csv`): gán tay 3.75 phút/ảnh (15 phút / 4 ảnh) vs duyệt 2.08 phút/ảnh (50 phút / 24 ảnh). Thiên lệch: Gán tay được thực hiện ở 4 ảnh gold đầu tiên khi chưa quen guideline và thao tác, còn duyệt batch thực hiện sau khi đã quen bối cảnh. Ngoài ra, tâm lý ỷ lại vào pre-label khiến tốc độ duyệt tăng lên nhưng làm giảm độ kỹ lưỡng.
- Kết luận: Có đáng dùng cho các lớp đối tượng phổ biến tiền cảnh (`person`, `car`, `motorbike` gần) vì giúp tiết kiệm ~44.5% thời gian (từ 3.75 xuống 2.08 phút/ảnh). Tuy nhiên, KHÔNG ĐƯỢC ỷ lại hoàn toàn ở các lớp đặc thù đường phố (`three_wheeler`) hoặc các cụm đông đúc hậu cảnh, mà cần có quy trình kiểm soát nghiêm ngặt.
- **Ba quy tắc giảm thiên kiến của bạn** (mỗi quy tắc kiểm tra được bằng số hoặc việc làm):
  1. **Ẩn nhãn máy trong 15 giây đầu mỗi ảnh (Bước 0 bắt buộc):** Luôn tắt layer annotation để mắt quét tự nhiên và nhẩm tính số lượng đối tượng trước khi bật nhãn AI lên đối chiếu.
  2. **Quy tắc phóng to 200% kiểm tra biên:** Với mỗi ảnh, chọn ngẫu nhiên ít nhất 3 box để zoom 200%, kiểm tra 4 cạnh có ôm sát vật trong phạm vi $\le 2\text{ px}$ không (chống lỗi loose).
  3. **Lọc kiểm tra riêng các box có độ tin cậy thấp và lớp rủi ro cao:** Dùng bộ lọc trên thanh công cụ lọc riêng các box có `conf < 0.35` và toàn bộ nhãn `three_wheeler` để rà soát trước khi lưu ảnh.

## 5. Mask SAM cho 3 vật (Part 5)

| Ảnh | SAM (phiên bản) | Số click | Mask gồm người lái / hàng hoá? | Lỗi quan sát | File trong `masks/` |
|---|---|---|---|---|---|
| `img_013.jpg` | SAM 2.1 base (`sam2.1_b.pt`) | 3 click (2 pos, 1 neg) | Điểm âm đã khoét bỏ được phần lưng và áo người lái, nhưng chân và tay vẫn dính mask xe | Điểm neg đỏ khoét tốt thân trên người lái nhưng vùng chân/tay màu tiệp với xe bị dính; viền bánh xe và mui bám rất mịn | `tuktuk_img013_sam2.jpg` |
| `img_013.jpg` | MobileSAM (`mobile_sam.pt`) | 3 click (2 pos, 1 neg) | Khoét được lưng áo nhưng mặt và chân người lái vẫn bị phủ mask; viền xe ít mượt hơn | Rìa mask răng cưa rõ rệt quanh bánh xe và thanh sắt; tách người lái kém mịn hơn SAM 2.1 dù tốc độ inference nhanh hơn | `tuktuk_img013_mobilesam.jpg` |
| `img_004.jpg` | SAM 2.1 base (`sam2.1_b.pt`) | 3 click (2 pos, 1 neg) | Mask bao trọn xe tuk-tuk (mui, khung, bánh); điểm âm khoét tách sạch người đứng sau xe | Mask ôm sát thân xe và kính chắn gió; có dính một phần bóng đổ nhỏ dưới bánh trước | `tuktuk_img004_sam2.jpg` |
| `img_017.jpg` | SAM 2.1 base (`sam2.1_b.pt`) | 3 click (2 pos, 1 neg) | Mask bao toàn bộ thân xe, mui, ghế sau cam, bánh xe; không có người ngồi trên xe | Điểm neg đặt sát mép mui vô tình khoét mất một mảng chữ vẽ graffiti trên mui xe do SAM hiểu nhầm là vùng loại trừ | `tuktuk_img017_sam2.jpg` |

- Đối chiếu guideline mục 4.2: quy tắc nào giúp, quy tắc nào thiếu?
  - **Quy tắc giúp ích:**
    1. *Quy tắc lỗ thủng (khoảng trống nhìn xuyên khung xe):* Lỗ nhỏ $< 5\%$ diện tích không cần khoét, lỗ lớn loại bỏ. Giúp người gán không phải tốn hàng chục click âm vào từng nan sắt nhỏ sau lưng ghế tuk-tuk.
    2. *Quy tắc tách người và xe:* Xác định rõ người lái là `person` riêng, xe là `three_wheeler` riêng, định hướng việc phải dùng điểm neg để khoét người lái.
    3. *Kính xe thuộc xe, bóng đổ không thuộc xe:* Xác định rõ kính chắn gió trước của tuk-tuk là thuộc mask của xe.
  - **Quy tắc còn thiếu trong Guideline v1.0:**
    1. *Chi tiết người lái ngồi lọt thỏm trong cabin:* Khi chân/tay người lái khuất một phần sau vô lăng và màu quần áo tiệp màu sơn xe, guideline chưa nói rõ mask của xe có được phép đè qua chân/tay hay bắt buộc tách triệt để pixel-level.
    2. *Chi tiết phụ kiện gắn cố định:* Biển hiệu "TAXI" trên nóc, gương chiếu hậu và hoa văn/graffiti trên mui chưa được nêu rõ ràng có tính là bộ phận của xe hay không.
- *(Tuỳ chọn)* Interactor SAM trong CVAT: làm được / không (lỗi gì, đã thử gì); khác gì offline: Thực hiện offline qua script `scripts/sam2_points.py` vì CVAT local trên Docker không bật serverless Nuclio (thiếu container nuclio/GPU phục vụ tương tác trực tiếp). Làm offline bằng script có ưu điểm vượt trội: lưu lại được file ảnh có đánh dấu toạ độ điểm click xanh/đỏ (`submission/masks/*.jpg`), đo đạc được diện tích mask chính xác, và dễ dàng so sánh giữa các phiên bản model (SAM 2.1 vs MobileSAM).

## 6. Calibration (Part 6)

- Số quyết định: 5 quyết định (`calibration_decisions.csv`); quan trọng nhất (dẫn `CAL-xx`): `CAL-01` (chuẩn hóa phân loại `three_wheeler` cho xe tuk-tuk, giải quyết dứt điểm lỗi máy YOLO26 nhận nhầm thành `car` và khử box chồng đè của YOLOE) và `CAL-04` (siết chặt tiêu chuẩn dung sai biên $\le 2\text{ px}$ kèm checklist zoom 200% để khắc phục việc lọt bẫy loose do thiên kiến tự động hóa).
- Quy tắc mới đã vào `guideline_v1.1_calibrated.md`:
  1. Phân định xe đạp điện (`bicycle`) và xe máy điện (`motorbike`) theo tiêu chí bàn đạp cơ học trợ lực (`CAL-02`).
  2. Cấm tuyệt đối việc dùng `ignore_region` cho các cụm xe máy chen chúc ở tiền cảnh cự ly gần; chỉ áp dụng cho hậu cảnh xa $< 20\text{ px}$ (`CAL-03`).
  3. Khẳng định xe tuk-tuk và xe ba gác luôn thuộc lớp `three_wheeler`, ưu tiên nhãn YOLOE loại bỏ box car/truck của YOLO26 (`CAL-01`).
  4. Chuẩn hóa quy định dung sai biên $\le 2\text{ px}$ và quy trình zoom 200% bắt buộc khi duyệt có AI hỗ trợ (`CAL-04`).
  5. Quy định rõ việc gán `person` khi nhìn xuyên qua kính chắn gió ô tô/xe bus nếu thấy $\ge 20\%$ cơ thể (`CAL-05`).
- Nguyên nhân gốc: Kết hợp từ cả 3 yếu tố:
  1. *Guideline ban đầu mơ hồ:* Chưa có quy định cụ thể về phân loại xe điện, ranh giới cự ly của `ignore_region`, và việc nhìn xuyên qua kính chắn gió.
  2. *Lỗi từ mô hình AI:* YOLO26 thiếu nhãn xe ba bánh dẫn đến lỗi phân loại ngữ nghĩa, YOLOE sinh box lỏng và đôi khi tạo box chồng lặp.
  3. *Thiên kiến tự động hóa (Automation Bias):* Người duyệt có tâm lý tin tưởng vào nhãn gợi ý có sẵn của máy nên lướt qua nhanh, không zoom to để kiểm tra độ khít sát biên dẫn đến lọt các bẫy loose.

## 7. QC và quyết định (Part 6)

- Mẫu: 8 ảnh (3 ảnh `can_review` + 5 ngẫu nhiên, seed 42); người kiểm: bạn (**hạn chế**: bạn vừa là người duyệt nên có nguy cơ lặp lại điểm mù / thiên kiến cá nhân khi tự kiểm định lại lô của mình).
- Lỗi theo **đối tượng**: 4/224 = 1.8% (mẫu số: 224 đối tượng trong mẫu) · theo **ảnh**: 4/8 = 50.0% (mẫu số: 8 ảnh trong mẫu) (ngưỡng tiêu chuẩn Release ≤ 2%).
- Top lỗi (trong: bỏ sót, thừa, sai lớp, sai vị trí/kích thước, sai thuộc tính, sai mask) + một ảnh ví dụ mỗi loại:
  - `thua` (2 lỗi): Box ma ở khoảng không hậu cảnh hoặc lề đường (`img_033.jpg`, `img_035.jpg` - bẫy extra bị lọt).
  - `bo_sot` (1 lỗi): Bỏ sót người đi bộ nhỏ ở hậu cảnh (`img_006.jpg` - bẫy missing bị lọt).
  - `sai_vi_tri_kich_thuoc` (1 lỗi): Box xe máy vẽ rộng lỏng không ôm sát viền $\le 2\text{ px}$ (`img_018.jpg` - bẫy loose bị lọt).
- **Giới hạn mẫu nhỏ** (cận trên 95% trong `qc_report_auto.md`): Với 4 ảnh lỗi trên 8 ảnh mẫu, cận trên khoảng tin cậy Wilson 95% của tỷ lệ ảnh lỗi thật lên tới **78%**. Cỡ mẫu 8 ảnh là quá nhỏ để có thể khẳng định tỷ lệ lỗi toàn bộ lô batch đạt ngưỡng khắt khe $\le 2\%$.
- Script kiểm được / không kiểm được:
  - *Kiểm được:* Kiểm tra tính hợp lệ của cấu trúc COCO JSON, tính toán tự động số lượng và tỷ lệ lỗi từ phiếu kiểm định mắt `qc_review.csv`, tính khoảng tin cậy Wilson và đối chiếu ngưỡng cây quyết định.
  - *Không kiểm được:* Script không thể tự đánh giá độ khít sát biên ($\le 2\text{ px}$), không phát hiện được đối tượng mà cả AI lẫn người duyệt cùng bỏ sót (điểm mù chung), và không phân tích được ngữ cảnh che khuất phức tạp nếu không có con người soi từng pixel.

**Quyết định: REWORK**, ngày 2026-11-17. Lý do (đi theo Hold → Rework → Release):
1. *Xét Hold:* Guideline v1.1 đã làm rõ và chốt toàn bộ các câu hỏi mở (`--guideline-gap = 0`), bộ ảnh gốc rõ nét không có ảnh hỏng $\rightarrow$ Loại trừ Hold.
2. *Xét Rework:* Tỷ lệ ảnh có lỗi đạt $50.0\% > 2\%$ (vượt ngưỡng cho phép) và tỷ lệ bắt bẫy đạt $25.0\% < 80\%$ (dưới ngưỡng tối thiểu) $\rightarrow$ Lô nhãn chưa đạt chuẩn chất lượng phát hành $\rightarrow$ **Quyết định: REWORK**.
Nếu Rework / Hold, việc cần làm tiếp: Rà soát lại toàn bộ 24 ảnh batch trên CVAT theo checklist chống Automation Bias nghiêm ngặt: ẩn nhãn máy ở 15 giây đầu mỗi ảnh để tự đếm số lượng đối tượng trước (khắc phục bẫy missing), phóng to 200% để kéo khít 4 cạnh biên từng box (khắc phục bẫy loose), dùng filter lọc rà soát các box nghi vấn (khắc phục bẫy extra). Sau đó xuất lại nhãn và chạy lại QC để đạt tỷ lệ bắt bẫy $\ge 80\%$ và ảnh lỗi $\le 2\%$.

## 8. Ghi chú cá nhân

- **Dùng AI để làm bài** (công cụ nào, việc gì, kiểm tra thế nào; "không dùng" nếu không): Sử dụng trợ lý AI để tra cứu các quy tắc gán nhãn, đối chiếu công thức tính khoảng tin cậy Wilson và hỗ trợ định dạng báo cáo; toàn bộ thao tác gán nhãn trên CVAT, chạy các script thực nghiệm mô hình và chấm điểm phiếu QC đều do học viên trực tiếp thực hiện và kiểm chứng trên máy cá nhân.
- Gold có bị nhiễm không (bạn có nhìn pre-label hay hỏi AI trước khi gán xong)? Hoàn toàn KHÔNG. Tập 4 ảnh gold được gán tay độc lập 100% ở Part 1 trước khi chạy bất kỳ mô hình AI nào (tuyệt đối không bật pre-label hay tra cứu AI trước khi xuất file).
- Máy / GPU / CVAT dùng; vướng mắc cài đặt: Windows 11 Home 64-bit, GPU NVIDIA GeForce RTX 4050 Laptop GPU (CUDA 12.8), CVAT v2.81.0 chạy Docker local qua port 8080. Cài đặt thuận lợi, các container chạy ổn định.
- Dùng cache `data/cache/` thay vì tự chạy? Không dùng cache có sẵn; học viên tự chạy toàn bộ các lệnh inference YOLO26, YOLOE và SAM offline từ đầu trên máy.
- Lỗi / bug script đã gặp (nếu có): Thao tác tương tác SAM trên CVAT local gặp giới hạn do thiếu container GPU Nuclio serverless, đã giải quyết triệt để bằng cách dùng script offline `sam2_points.py` để tách mask và lưu vết toạ độ click.
- Một điều bạn làm khác lần sau: Tuyệt đối không để bị cuốn theo nhãn gợi ý của AI (chống Automation Bias); sẽ kiên quyết thực hiện Bước 0 (tắt nhãn máy để tự quét mắt độc lập 15s) và zoom to kiểm tra biên cho từng ảnh để bắt trọn vẹn 100% bẫy.
