# Thông tin học viên

- Họ và tên: Đỗ Huy Kiên
- MSHV: 2A202603216
- Lớp (L2 / L3 / khác): L2
- Tài khoản GitHub: dohuykienkansy
- Repo bài nộp (URL): https://github.com/dohuykienkansy/K5-L23-Day08-DoHuyKien-2A202603216-AI-Assisted-Labeling.git
- Commit chốt bài (hash): 9bd8c19
- Máy: Windows 11, NVIDIA GeForce RTX 4050 Laptop GPU (CUDA)
- CVAT (Docker): v2.81.0

## Kết quả tóm tắt (điền cuối bài; số lấy từ `submission/qc/*`, kèm mẫu số)

- Máy: accepted / edited / deleted: 396 / 15 / 21 (mẫu số: 432 box máy); added: 144 (mẫu số: 556 đối tượng)
- So với gold: precision / recall của **máy**: 77.3% / 52.0% (mẫu số: 98 đối tượng gold)
- Bắt bẫy: 3/12 = 25.0%
- Thời gian: gán tay 3.75 phút/ảnh; duyệt có pre-label 2.08 phút/ảnh
- Lỗi QC: 50.0% theo ảnh (4/8), 1.8% theo đối tượng (4/224)
- **Quyết định:** Rework — lý do một dòng: Tỷ lệ ảnh có lỗi 50.0% > 2% và tỷ lệ bắt bẫy đạt 25.0% < 80% do ảnh hưởng của thiên kiến tự động hóa khi duyệt batch.
