## WORKSHEET: PHÂN TÍCH CHI PHÍ HỆ THỐNG AI AGENT (VINCOM RETAIL)

### 1. Ước lượng Quy mô Vận hành (Traffic Estimation)
Để tính toán chi phí, chúng ta giả định quy mô cho giai đoạn MVP (Minimum Viable Product):

* **Số lượng User:** ~100 người (Trưởng bộ phận, DA, Marketing Manager, Quản lý TTTM).
* **Số lượng Request/Ngày:** ~200 requests (Trung bình mỗi user thực hiện 2 câu hỏi/ngày).
* **Peak Traffic:** 40-50 requests/giờ (Thường tập trung vào 8:00 - 9:00 sáng khi các phòng ban check report đầu ngày).
* **Khối lượng dữ liệu quét (Scanning):** ~500GB - 1TB dữ liệu mỗi tháng qua Athena/Trino.

### 2. Ước lượng Token LLM (LLM API Consumption)
Mỗi request của người dùng không chỉ là 1 dòng chat, mà bao gồm cả context (Metadata/Schema).

* **Input per Request:** ~2,500 tokens (Bao gồm: User Prompt + Table Schema + Few-shot examples + Instructions).
* **Output per Request:** ~500 tokens (SQL Query + Giải thích logic + Cấu trúc CSV).
* **Self-healing (Debug):** ~5,000 tokens/lần (Chỉ xảy ra khi có lỗi schema, cần đọc log và mapping lại). Giả định 5% số request cần debug.
* **Tổng Token/Tháng:** ~18M - 20M Tokens (Hỗn hợp Input/Output).

---

### 3. Liệt kê các Lớp Chi phí (Cost Layers)

| Lớp chi phí | Thành phần chi tiết | Loại chi phí |
| :--- | :--- | :--- |
| **Token API** | GPT-5 Mini (Logic) & Claude/GPT-5 (Complex Debugging). | Biến đổi (theo traffic) |
| **Compute** | AWS Lambda (chạy Agent), Spark (Mapping data từ JSONB sang Iceberg). | Biến đổi (theo volume) |
| **Storage** | S3 (Lưu Iceberg), RDS (Lưu metadata/mapping), Log storage. | Cố định + Tăng dần |
| **Human Review** | Thời gian DE/DA kiểm soát báo cáo tài chính/nhạy cảm. | Nhân sự (Ẩn) |
| **Logging & Ops** | CloudWatch/Datadog để giám sát hoạt động của Agent. | Cố định |
| **Maintenance** | Cập nhật mapping thủ công khi AI không tự xử lý được. | Nhân sự |

---

### 4. Tính toán Chi phí MVP (Sơ bộ hàng tháng)

Dựa trên công nghệ **Iceberg + JSONB + LLM Hybrid**:

1.  **AI Models (Token):** ~$150 (Tận dụng các model giá rẻ cho task đơn giản).
2.  **Data Infrastructure (S3 + Athena + RDS):** ~$400 (Phí quét dữ liệu Athena chiếm tỷ trọng chính).
3.  **Compute & Backend (Lambda/ECS):** ~$100.
4.  **Logging & Monitoring:** ~$50 (Quan trọng để track lỗi Agent).
5.  **Human-in-the-loop (Ước tính):** ~$1,000 (Khoảng 20 giờ làm việc của DE để giám sát hệ thống).

**=> Tổng cộng MVP:** **~$1,700 / tháng.**

---

### 5. Khả năng Mở rộng (Scalability Analysis)

Khi số lượng user tăng **5x hoặc 10x**:

* **Phần tăng mạnh nhất:** **Data Query Cost (Athena/Trino)**. Khi nhiều user query đồng thời vào các tập dữ liệu lớn trên S3, chi phí scan data sẽ tăng tuyến tính.
* **Phần tăng ổn định:** Token (LLM). Nhờ vào bộ nhớ đệm (Prompt Caching) và việc tối ưu prompt, chi phí token thường không tăng nhanh bằng chi phí hạ tầng dữ liệu.
* **Điểm nghẽn:** **Human Review**. Nếu không tự động hóa được lớp kiểm soát dữ liệu nhạy cảm, chi phí nhân sự sẽ trở thành rào cản lớn nhất khi quy mô tăng.

---

### 6. Trả lời Câu hỏi Cốt lõi

* **Cost driver lớn nhất của hệ thống là gì?**
    * **Data Scanning (Athena/Trino):** Với đặc thù dữ liệu bán lẻ Vincom cực lớn, việc Agent viết query không tối ưu (ví dụ: `SELECT *`) có thể làm bùng nổ chi phí scan dữ liệu trên S3 nhanh chóng.

* **Hidden cost nào dễ bị quên nhất?**
    * **Data Egress & Logging:** Phí đẩy dữ liệu ra ngoài (CSV export) và phí lưu trữ Logs (Agent hoạt động càng nhiều, Log càng khổng lồ để phục vụ debug).
    * **Cost of Inaccuracy:** Chi phí cơ hội và thời gian sửa lỗi khi Agent tạo ra báo cáo sai dẫn đến quyết định kinh doanh sai lầm.

* **Đội có chỗ nào đang ước lượng quá lạc quan không?**
    * **Khả năng Self-healing:** Chúng ta đang kỳ vọng Agent tự sửa được phần lớn lỗi Schema. Thực tế, các thay đổi API phức tạp (như thay đổi hoàn toàn logic tính thuế hoặc khuyến mãi) vẫn cần con người can thiệp sâu.
    * **Human-in-the-loop:** Thời gian để một chuyên gia dữ liệu "Verify" lại kết quả của AI thường bị đánh giá thấp. Thực tế, việc đọc và hiểu logic của AI có thể tốn gần bằng thời gian tự viết query.

---
