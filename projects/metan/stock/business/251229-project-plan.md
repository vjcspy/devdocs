# Project Plan

## Plan

Xây dựng một application có các chức năng chính:

1. Dựa vào dữ liệu đã thu thập để build ra các features.
2. Dùng AI sử dụng features này để dự đoán giá trong phiên của VN30 (hợp đồng phái sinh VN30F1M). Lưu ý là chỉ nắm giữ trong phiên, bắt buộc sẽ bán khi đặt target profit, chạm stop loss hoặc cuối phiên.
Cụ thể AI model cần sẽ đánh giá, tức là sẽ mở lệnh và đóng lệnh tại các thời điểm phù hợp trong ngày, đưa ra hành động với 3 trường hợp:

- Vào vị thế và dự đoán Giá tăng X%
- Vào vị thế và dự đoán Giá giảm X%
- Không vào vị thế
