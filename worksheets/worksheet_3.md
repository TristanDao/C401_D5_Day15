## WORKSHEET 3: CHIẾN LƯỢC VẬN HÀNH & GIẢM THIỂU RỦI RO

### 1. Phân tích Tình huống & Tác động

| Tình huống | Tác động tới User | Phản ứng ngắn hạn (Hot-fix) | Giải pháp dài hạn (Strategic) |
| :--- | :--- | :--- | :--- |
| **Traffic tăng đột biến** (Ví dụ: Sáng thứ 2 đầu tháng) | UI bị treo, lỗi 504 Gateway Timeout, thời gian chờ xuất CSV cực lâu. | Bật **Rate Limiting** (giới hạn mỗi user 2 request/phút). Ưu tiên các phòng ban trọng yếu. | Áp dụng **Asynchronous Queue** (Người dùng nhấn nút, hệ thống nhận và báo "đang xử lý", trả kết quả qua email/zalo sau). |
| **Provider Timeout** (OpenAI/Gemini gặp sự cố) | Agent không phản hồi hoặc trả về thông báo lỗi hệ thống chung chung. | Chuyển ngay lập tức sang **Backup Model** (ví dụ: từ GPT-5 sang Gemini 1.5 Flash hoặc một model local). | Triển khai **Circuit Breaker Pattern**: Tự động ngắt kết nối với Provider lỗi để tránh treo toàn bộ hệ thống. |
| **Response chậm** (Do query Athena/Trino quá nặng) | User thấy "Thinking..." quá lâu, dễ dẫn đến việc tắt trình duyệt hoặc gửi request trùng lặp. | Hiển thị **Progress Bar** chi tiết (ví dụ: "Đang quét 500GB dữ liệu... 40%"). | **Query Optimization**: Tối ưu hóa phân vùng (Partitioning) trên Iceberg và sử dụng **Result Caching** cho các câu hỏi phổ biến. |

---

### 2. Phân loại Request (Processing Strategy)

Để tối ưu tài nguyên, chúng ta cần tách bạch luồng xử lý:

* **Real-time (Đồng bộ):** Các câu hỏi tra cứu metadata đơn giản ("Bảng doanh thu có những cột nào?", "Hệ thống đang hoạt động tốt không?"). Yêu cầu phản hồi < 5 giây.
* **Asynchronous (Bất đồng bộ):** Các yêu cầu "Lấy dữ liệu CSV 6 tháng qua", "Tạo dashboard so sánh 10 Mega Mall".
    * *Flow:* User gửi request -> Nhận Job ID -> Worker xử lý ngầm -> Notify qua Teams/Zalo khi hoàn thành.

---

### 3. Metric cần Monitoring (Bảng điều khiển sức khỏe hệ thống)

Để "phản ứng" nhanh, đội vận hành cần quan sát các chỉ số sau:

1.  **Latency per Layer:** Thời gian phản hồi của LLM vs. Thời gian query của Athena.
2.  **Token Burn Rate:** Tốc độ tiêu thụ token (để cảnh báo nếu vượt ngân sách hoặc bị tấn công prompt injection liên tục).
3.  **Success Rate (2xx/4xx/5xx):** Tỷ lệ lỗi API. Đặc biệt chú ý lỗi 429 (Too many requests).
4.  **Queue Depth:** Số lượng request đang chờ xử lý trong hàng đợi.
5.  **Hallucination Rate:** (Đo bằng mẫu test định kỳ) Tỷ lệ Agent viết sai SQL khiến query không chạy được.

---

### 4. Fallback Proposal (Kế hoạch dự phòng)

Khi Agent chính gặp sự cố không thể xử lý, hệ thống sẽ tự động hạ cấp (Degrade) theo thứ tự sau:

1.  **Level 1: Model Switching.**
    * Nếu GPT-5 (Reasoning) lỗi -> Chuyển sang Gemini/Claude.
    * Nếu toàn bộ LLM API lỗi -> Chuyển sang **Small Local LLM** (như Llama 3 hoặc Mistral chạy trên server nội bộ) chỉ để trả lời các câu hỏi basic.
2.  **Level 2: Rule-based Templates.**
    * Nếu AI không thể sinh SQL, hệ thống cung cấp các **mẫu báo cáo cố định** (Pre-defined SQL). User chỉ được chọn Filter chứ không được chat tự do.
3.  **Level 3: Human Escalation.**
    * Nếu hệ thống lỗi quá 15 phút, tự động gửi cảnh báo qua Telegram cho đội DE/On-call.
    * Hiển thị thông báo cho User: *"Hệ thống đang bảo trì, vui lòng gửi yêu cầu ad-hoc qua ticket [Link] để DA hỗ trợ thủ công."*

---

### 5. Câu hỏi Phản biện (Self-Reflect)

* **Request nào có thể "hy sinh" khi tải cao?**
    * Các yêu cầu về "Giải thích dữ liệu" hoặc "Phân tích xu hướng" (nặng về reasoning) có thể tạm ngắt để ưu tiên yêu cầu "Xuất file CSV" (phục vụ vận hành trực tiếp).
* **Circuit Breaker nên đặt ở đâu?**
    * Nên đặt ở lớp kết nối LLM và lớp kết nối Data Source (Athena). Nếu Athena đang quá tải, Agent nên từ chối nhận query mới thay vì tiếp tục gửi thêm lệnh.
* **Sử dụng Human Review như thế nào cho hiệu quả?**
    * Nên áp dụng **Sampling Review** (Kiểm tra ngẫu nhiên 5% số request hàng ngày) thay vì kiểm tra mọi request để tránh làm thắt nút cổ chai quy trình.
