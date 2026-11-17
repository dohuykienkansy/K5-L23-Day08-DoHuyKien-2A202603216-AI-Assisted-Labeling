# Số đo review (det)

- Ảnh tính: 24 (batch,gold); box máy: 432; đối tượng sau duyệt: 556; ngưỡng accepted: IoU >= 0.9
- Đã **loại 27 đối tượng nằm quanh các bẫy** khỏi bảng này: lỗi nhân tạo không phản ánh chất lượng máy.

## 1. Bốn trạng thái (AI error profile)
| Trạng thái | Số | Tỉ lệ | Mẫu số |
|---|---|---|---|
| accepted (giữ nguyên) | 396 | 396/432 = 91.7% | 432 box máy |
| edited (sửa) | 15 | 15/432 = 3.5% | 432 box máy |
| deleted (xoá) | 21 | 21/432 = 4.9% | 432 box máy |
| added (vẽ thêm) | 144 | 144/556 = 25.9% | 556 đối tượng sau duyệt |

Trong số deleted + added, **5 trường hợp là đổi lớp** (máy gán sai lớp, box đúng chỗ): đó không phải máy bỏ sót. Số vật máy **sót thật** ≈ added − đổi lớp = 139.

### Theo lớp
| Lớp | accepted | edited | deleted | added |
|---|---|---|---|---|
| person | 247 | 0 | 1 | 39 |
| bicycle | 6 | 0 | 3 | 7 |
| motorbike | 81 | 7 | 1 | 68 |
| car | 45 | 0 | 5 | 20 |
| bus | 3 | 0 | 0 | 1 |
| truck | 6 | 0 | 1 | 2 |
| three_wheeler | 8 | 8 | 10 | 7 |

## 2. So với gold (nhãn gán tay từ đầu)
Ảnh gold: 4; đối tượng gold: 98 (mẫu số của recall)
| | TP | FP | FN | Precision | Recall |
|---|---|---|---|---|---|
| Máy (pre-label) | 51 | 15 | 47 | 77.3% (KTC 95%: 66–86%) | 52.0% (KTC 95%: 42–62%) |

*Lạc quan: ngưỡng được chọn trên chính các ảnh gold này (in-sample), và chỉ có 4 ảnh. Đọc khoảng tin cậy, đừng đọc con số đơn.*


(Ảnh gold không nằm trong nhãn sau duyệt: không tính residual / new error. Đó là chủ ý: duyệt lại ảnh bạn vừa tự gán tay sẽ đo trí nhớ, không đo thiên kiến.)

## 3. Ảnh bẫy
- Bắt được 3/12 = 25.0% (KTC 95%: 9–53%) (ngưỡng đạt: 80%). Chỉ 12 bẫy nên khoảng tin cậy rất rộng: đọc cùng danh sách bẫy lọt, đừng kết luận chỉ từ %.
  - wrong_class: 1/3
  - loose: 0/3
  - extra: 1/3
  - missing: 1/3
  - lọt: img_027.jpg (loose), img_017.jpg (wrong_class), img_033.jpg (extra), img_005.jpg (wrong_class), img_018.jpg (loose), img_035.jpg (extra), img_006.jpg (missing), img_025.jpg (missing), img_034.jpg (loose)

## Gợi ý đọc số
- `deleted` cao → ngưỡng thấp hoặc ánh xạ nhãn sai; `added` cao → máy bỏ sót (xem lớp nào);
- `accepted` thấp (< 50%) → sửa mỗi box lâu hơn vẽ mới: đổi model/ngưỡng hoặc vẽ tay;
- bắt bẫy < 80% → duyệt lại lô, siết checklist (quét ảnh TRƯỚC khi nhìn nhãn máy).
