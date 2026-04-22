## WORKSHEET 0: PROBLEM STATEMENT

### Domain: Bán lẻ
Vincom Retail (Vận hành TTTM):

    Vincom Center: TTTM cao cấp, vị trí trung tâm.
    Vincom Mega Mall: TTTM quy mô lớn, tích hợp "all-in-one".
    Vincom Plaza: TTTM phổ thông, tập trung tại các tỉnh thành.
    Vincom+: Trung tâm mua sắm tiện ích tại các khu vực dân cư.

### Painpoint:
- Để xây dựng được 1 báo cáo thì cần đội DA và DE kết hợp
- Đội DA DE chịu trách nhiệm cho rất nhiều phòng ban => Khi nhiều phòng ban request data product, customer cùng một lúc
- Các sản phẩm/ website lấy data cho các phòng ban chỉ ở các trường, filter cố định

### Giải pháp:
Agent hỗ trợ thu thập data từ nhiều api của nhiều nền tảng
Agent có khả năng debug khi Field Mismatch/Wrong Field , response Payload Mismatch, Data Structure Inconsistency 
Agent có khả năng hỗ trợ lấy qua cho nhiều phòng ban  dưới dạng CSV

### Kết luận:
Tối ưu thời gian lấy data cho các phòng ban có nhu cầu
Giúp DE giảm workload maintain các DB 

### Dự định:
Agent có khả năng connect platform tạo Dashboard
Custom chart theo phong ban và chức vụ của phòng ban hoặc người sử dụng
