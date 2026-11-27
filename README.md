# Nt334.R11.Antt-Docker-file-carving
# Docker `json-file` Log Carver (Baseline Implementation)

Công cụ khôi phục dữ liệu pháp y số (Digital Forensics) cho phép trích xuất và tái lắp ghép các file log của Docker container (`json-file` logging driver) từ ảnh đĩa thô mà **không cần phụ thuộc vào metadata của hệ thống tập tin** (inode, superblock).

Dự án được xây dựng dựa trên kiến trúc và thuật toán trong bài báo nghiên cứu:
> **A novel file carving algorithm for docker container logs recorded by json-file logging driver**  
> *Song Ge, Ming Xu, Tong Qiao, Ning Zheng — Forensic Science International: Digital Investigation (2021)*

---

## Cấu trúc thư mục

Dự án được tổ chức theo kiến trúc module hóa gồm 1 script tạo dữ liệu giả lập, 1 file điều phối chính và 4 module xử lý lõi:

```text
.
├── generate_sample.py   # Tạo ảnh đĩa thô giả lập (raw disk image) và dữ liệu đối chứng
├── main.py              # Điểm vào chương trình (CLI), điều phối luồng chạy và đo thời gian
├── scanner.py           # Quét sector, lọc nhiễu bằng Discriminator và phân loại khối (List_1..3)
├── indexer.py           # Tách từ/câu/thông điệp (xử lý mã ANSI) và dựng cây Tiền tố/Hậu tố, m-VP tree
├── reassembler.py       # Ghép nối mảnh vỡ 4 pha, tính điểm SOC (18 thành phần) và bỏ phiếu xung đột
└── evaluator.py         # Xuất file log khôi phục và tính toán chỉ số Precision / Recall
```

---

## Luồng xử lý (Pipeline)

```text
[sample_disk.img] ──▶ main.py
                        ├── 1. scanner.py     ──▶ Phân loại List_1, List_2, List_3
                        ├── 2. indexer.py     ──▶ Lập chỉ mục Prefix/Suffix Trees & m-VP Tree
                        ├── 3. reassembler.py ──▶ Ghép nối 4 pha & Chấm điểm độ mượt (SOC)
                        └── 4. evaluator.py   ──▶ Xuất recovered.log & Đối chiếu Ground Truth
```

---

## Môi trường & Cài đặt

* **Hệ điều hành:** Ubuntu 26.04 (hoặc các bản phân phối Linux tương đương)
* **Ngôn ngữ:** Python 3.12+ (chỉ sử dụng thư viện chuẩn, không yêu cầu cài đặt thư viện ngoài hoặc quyền `root`)

---

## Hướng dẫn sử dụng

### Bước 1: Tạo ảnh đĩa giả lập (Sandbox Dataset)
Chạy script sinh dữ liệu để tạo file ảnh đĩa nhị phân chứa các kịch bản phân mảnh thực tế (xáo trộn thứ tự vật lý, cắt ngang nội dung log, chèn dữ liệu rác và các khối phi cấu trúc Case a/b/c):

```bash
python3 generate_sample.py --size-mb 20
```

Lệnh trên sẽ sinh ra 4 file phục vụ kiểm thử:
* `sample_disk.img`: Ảnh đĩa thô giả lập (đầu vào duy nhất cho chương trình carving).
* `ground_truth.log`: Toàn bộ các dòng log hợp lệ đã nhúng vào ảnh đĩa.
* `ground_truth_case_c.log`: Các dòng log thuộc khối phi cấu trúc cô lập (Case c — giới hạn lý thuyết của thuật toán gốc).
* `disk_map.json`: Bản đồ tọa độ (`disk_offset`, `cut_location`) phục vụ gỡ lỗi và kiểm chứng từng pha.

### Bước 2: Chạy chương trình Carving
Khởi chạy pipeline khôi phục log và đánh giá độ chính xác:

```bash
python3 main.py \
  --image sample_disk.img \
  --step-size 512 \
  --output ./recovered_logs \
  --ground-truth ground_truth.log
```

**Các tham số dòng lệnh:**
* `--image`: Đường dẫn tới file ảnh đĩa thô (`.img`).
* `--step-size`: Kích thước bước quét theo byte (mặc định `512` B; hỗ trợ `1024`, `2048`, `4096`).
* `--output`: Thư mục lưu file log sau khi khôi phục.
* `--ground-truth`: Đường dẫn file đáp án chuẩn để tính tỷ lệ chính xác (tùy chọn).

---

## Tiến độ đồ án (Roadmap)

* [x] **Giai đoạn 1:** Xây dựng bộ sinh dữ liệu giả lập đa kịch bản phân mảnh (`generate_sample.py`).
* [x] **Giai đoạn 2:** Hoàn thiện kiến trúc 5 module tái hiện đầy đủ thuật toán gốc của bài báo (Baseline).
* [ ] **Giai đoạn 3:** Đo đạc hiệu năng Baseline và tối ưu tốc độ Pha 4 / SOC (phân vùng theo `attrs`, bộ nhớ đệm điểm SOC, lọc ứng viên nhanh và xử lý đa luồng).

---

## Kết quả thực nghiệm (Cập nhật sau khi chạy kiểm thử)

| Chế độ chạy | Kích thước ảnh | Bước quét | Precision (PR) | Recall (RR) | Thời gian thực thi |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **Baseline (Bản gốc)** | 20 MB | 512 B | *Đang cập nhật* | *Đang cập nhật* | *Đang cập nhật* |
| **Optimized (Cải tiến)** | 20 MB | 512 B | *Đang cập nhật* | *Đang cập nhật* | *Đang cập nhật* |
