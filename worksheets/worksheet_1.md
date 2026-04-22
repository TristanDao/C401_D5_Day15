## WORKSHEET 1: CHIẾN LƯỢC TRIỂN KHAI ENTERPRISE (VINCOM RETAIL)

### 1. Bối cảnh Tổ chức & Khách hàng
* **Tổ chức:** Vincom Retail (Vingroup) – Đơn vị vận hành TTTM lớn nhất Việt Nam.
* **Người dùng cuối (End-users):** Cấp lãnh đạo (C-level), Trưởng bộ phận Marketing, Quản lý vận hành tại các cơ sở (Mega Mall, Plaza), và đội ngũ Data Analyst (DA).
* **Mục tiêu:** Xóa bỏ "nút thắt cổ chai" tại đội DA/DE, giúp các phòng ban tự truy xuất báo cáo (Self-service) qua ngôn ngữ tự nhiên.

### 2. Danh mục Dữ liệu Hệ thống tác động
Hệ thống không chỉ chạm vào các bảng doanh thu mà còn bao phủ toàn bộ hệ sinh thái bán lẻ:
* **Dữ liệu Giao dịch (Transaction):** Doanh thu POS, hóa đơn, thời gian mua sắm.
* **Dữ liệu Khách hàng (Customer):** Thông tin thành viên, hành vi di chuyển trong TTTM, lịch sử phản hồi.
* **Dữ liệu Vận hành (Operations):** Chi phí điện nước, nhân sự, mặt bằng trống, hợp đồng thuê gian hàng.
* **Dữ liệu Nền tảng (External API):** Chỉ số Ads (Facebook, Google), lượt check-in trên mạng xã hội.

### 3. Đánh giá Mức độ Nhạy cảm
* **Cực kỳ nhạy cảm (High):** Thông tin định danh khách hàng (PII) như SĐT, Email; Doanh thu chi tiết từng gian hàng (bí mật kinh doanh); Các điều khoản hợp đồng thuê.
* **Trung bình (Medium):** Traffic khách ra vào TTTM, hiệu quả các chiến dịch marketing chung.
* **Thấp (Low):** Danh mục phân loại sản phẩm, vị trí các TTTM.

### 4. 3 Ràng buộc Enterprise lớn nhất (Top 3 Constraints)
1.  **Chủ quyền Dữ liệu & Bảo mật (Data Sovereignty):** Dữ liệu nhạy cảm không được phép "rời khỏi" biên giới hạ tầng của tập đoàn hoặc bị gửi trực tiếp lên các public LLM mà không qua lớp làm sạch (Masking).
2.  **Tính chính xác và Kiểm định (Auditability):** Hệ thống phải có **Audit Trail**. Mọi câu trả lời của Agent phải có khả năng truy hồi (Ai đã hỏi? Agent lấy dữ liệu từ đâu? Query SQL gốc là gì?). Sai sót trong báo cáo tài chính có thể dẫn đến sai lầm chiến lược hàng tỷ đồng.
3.  **Tích hợp hệ thống cũ (Legacy Integration):** Phải kết nối được với các hệ thống ERP, POS hiện có và hệ thống quản trị định danh (Azure AD/Okta) của tập đoàn để phân quyền.

### 5. Mô hình Triển khai: Hybrid Cloud (Đám mây lai)

### 6. Lý do lựa chọn mô hình Hybrid
* **Bảo mật lớp lõi (On-prem/Private Cloud):** Lưu trữ Data Lake (Iceberg) và dữ liệu thô (JSONB) tại Private Cloud hoặc VPC riêng của tập đoàn để đảm bảo an toàn PII và tuân thủ pháp luật về dữ liệu.
* **Hiệu năng và Tính linh hoạt (Public Cloud):** Tận dụng Public Cloud (AWS/Azure) để sử dụng các dịch vụ Managed Services như LLM API (Gemini/GPT qua VPN bảo mật) và các dịch vụ Compute (Athena/Lambda) để xử lý lượng traffic biến động lớn mà không cần đầu tư phần cứng quá mức tại chỗ.

---
