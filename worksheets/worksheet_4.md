## WORKSHEET 4: CHIẾN LƯỢC VẬN HÀNH & GIẢM THIỂU RỦI RO (RESILIENCE STRATEGY)

Để một hệ thống AI Agent đi từ "demo" sang "production" trong môi trường khắt khe như Vincom Retail, khả năng chịu tải và xử lý lỗi (Fault Tolerance) là yếu tố sống còn. Dưới đây là kế hoạch chi tiết.

---

### 1. Phân tích Tình huống & Tác động

| Tình huống | Tác động tới User | Phản ứng ngắn hạn | Giải pháp dài hạn |
| :--- | :--- | :--- | :--- |
| **Traffic tăng đột biến** (VD: Chốt số cuối tháng) | Hệ thống phản hồi chậm hoặc trả về lỗi "504 Gateway Timeout". User nhấn gửi liên tục làm tình hình tệ hơn. | Kích hoạt **Rate Limiting** (giới hạn request/user). Ưu tiên các phòng ban vận hành trực tiếp (Operations). | Triển khai **Message Queue (RabbitMQ/Kafka)**. Chuyển đổi sang kiến trúc **Asynchronous** cho các báo cáo nặng. |
| **Provider Timeout** (OpenAI/Gemini lỗi) | Agent hoàn toàn im lặng hoặc báo lỗi "Service Unavailable". Toàn bộ luồng Logic & Query Gen bị đứt. | Chuyển đổi thủ công sang **Backup Model** (ví dụ: đang dùng GPT-5 chuyển sang Claude 3.5 hoặc Gemini 1.5 Flash). | Xây dựng **Model Router/Load Balancer**. Tự động chuyển vùng hoặc nhà cung cấp khi nhận thấy tỷ lệ lỗi (Error Rate) vượt ngưỡng. |
| **Response chậm** (Do Athena/S3 scan quá lâu) | User thấy trạng thái "Thinking..." quá 30 giây, dẫn đến việc bỏ dở tác vụ hoặc cho rằng AI bị "ngáo". | Hiển thị **Loading State chi tiết** (VD: "Đang quét 2TB dữ liệu bán lẻ..."). Gửi thông báo sẽ trả kết quả qua Email/Zalo sau. | **Data Optimization**: Tối ưu lại Partitioning trên Iceberg. Áp dụng **Caching lớp 2** cho các kết quả query giống nhau trong vòng 1 giờ. |

---

### 2. Chiến lược Phân loại Xử lý (Processing Strategy)

Không phải mọi yêu cầu đều cần phản hồi ngay lập tức. Chúng ta sẽ chia làm hai luồng:

* **Real-time (Đồng bộ):**
    * *Loại request:* Tra cứu Metadata ("Cột này nghĩa là gì?"), các query nhỏ trên các bảng Dimension (Thông tin cửa hàng, danh mục sản phẩm).
    * *Cơ chế:* Phản hồi trực tiếp trong < 5 giây. Nếu quá 10 giây -> Chuyển sang Async.
* **Asynchronous (Bất đồng bộ):**
    * *Loại request:* Xuất file CSV doanh thu năm, tổng hợp dữ liệu từ nhiều API nền tảng, tạo Dashboard tổng thể.
    * *Cơ chế:* User gửi yêu cầu -> Hệ thống xác nhận "Đã nhận job" -> Agent xử lý ngầm -> Thông báo kết quả qua **Teams/Slack/Zalo Webhook**.

---

### 3. Hệ thống Giám sát (Monitoring Metrics)

Cần thiết lập dashboard giám sát thời gian thực với các chỉ số:

1.  **Time to First Token (TTFT):** Tốc độ phản hồi ban đầu của LLM.
2.  **Request Latency (End-to-End):** Tổng thời gian từ lúc User hỏi đến khi nhận được data/file.
3.  **Success Rate per Provider:** Theo dõi độ ổn định riêng của từng Model (GPT, Gemini, Claude).
4.  **Athena Data Scanned:** Cảnh báo nếu Agent viết SQL "tệ" gây quét quá nhiều dữ liệu (>100GB/query).
5.  **Queue Depth:** Số lượng request đang xếp hàng chờ xử lý.

---

### 4. Fallback Proposal (Kế hoạch Dự phòng đa tầng)

Khi hệ thống gặp sự cố, chúng ta áp dụng chiến thuật **"Graceful Degradation" (Hạ cấp mượt mà)**:

* **Tầng 1: Model Fallback.** Nếu LLM chính (high-reasoning) timeout, hệ thống tự retry 1 lần. Nếu vẫn lỗi, tự động chuyển sang model rẻ hơn/nhanh hơn để đảm bảo tính sẵn sàng.
* **Tầng 2: Rule-based & Semantic Search.** Nếu toàn bộ AI Provider sụp đổ, hệ thống chuyển về chế độ "Tìm kiếm theo từ khóa". Sử dụng **Vector DB** để gợi ý các mẫu báo cáo (SQL Templates) có sẵn thay vì tự sinh SQL mới.
* **Tầng 3: Circuit Breaker.** Nếu tỷ lệ lỗi hệ thống > 20%, hệ thống tự động ngắt kết nối với LLM và hiển thị thông báo: *"Hệ thống AI đang bảo trì. Bạn có thể sử dụng các báo cáo tiêu chuẩn hoặc quay lại sau 15 phút."*
* **Tầng 4: Human Escalation.** Với các lỗi liên quan đến Schema Inconsistency mà Agent không tự sửa được, tự động tạo một Ticket (Jira/Linear) kèm theo log lỗi gửi trực tiếp cho đội Data Engineer (DE) xử lý.

---

### 5. Đánh giá tính khả thi

* **Queue/Circuit Breaker:** Rất cần thiết cho môi trường Retail vì traffic thường biến động mạnh theo các đợt Sale/Lễ hội.
* **Retry Policy:** Chỉ nên áp dụng cho lỗi mạng (Network error). Tuyệt đối không retry với lỗi 400 (Bad Request) để tránh lãng phí token.
* **Human-in-the-loop:** Ở giai đoạn MVP, mọi update về `mappings.yaml` từ Agent nên được con người phê duyệt (Approve) trước khi merge vào Git để đảm bảo an toàn.
