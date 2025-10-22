# Lỗ Hổng Wildcard Web Cache Deception Trong ChatGPT

## Tóm Tắt

Lỗ hổng "wildcard" web cache deception trong ChatGPT (2024) cho phép kẻ tấn công chiếm quyền tài khoản bằng cách khai thác sự không đồng bộ trong phân tích URL giữa CDN (Cloudflare) và máy chủ gốc. Sử dụng path traversal confusion (%2F..%2F), kẻ tấn công lưu trữ token xác thực từ endpoint nhạy cảm (/api/auth/session) trong cache, truy xuất để kiểm soát tài khoản. Lỗ hổng được vá sau khi báo cáo, nhưng nhấn mạnh rủi ro từ quy tắc cache lỏng lẻo.
<img width="802" height="847" alt="image" src="https://github.com/user-attachments/assets/6b4e50e1-72df-42b7-b43b-7efbad921943" />

## Chi Tiết Lỗ Hổng

### Phát Hiện
Lỗ hổng xuất hiện trong tính năng "share" của ChatGPT (https://chat.openai.com/share/CHAT-UUID). Phản hồi có header "Cf-Cache-Status: HIT", cho thấy Cloudflare cache nội dung dù URL không có phần mở rộng tĩnh. Kiểm tra cho thấy quy tắc cache áp dụng cho mọi đường dẫn dưới /share/*, kể cả không tồn tại, tạo điều kiện cho tấn công.

### Cơ Chế Khai Thác
Lỗ hổng khai thác sự khác biệt trong xử lý URL-encoded path traversal:
- **CDN (Cloudflare)**: Không giải mã hoặc chuẩn hóa %2F..%2F (tức /../), coi URL như nằm dưới /share/*, nên lưu cache phản hồi.
- **Máy chủ gốc**: Giải mã %2F..%2F thành /../, dẫn đến truy cập endpoint nhạy cảm như /api/auth/session, trả về token xác thực.

**Payload mẫu**: https://chat.openai.com/share/%2F..%2Fapi/auth/session?cachebuster=123
- CDN lưu phản hồi dưới khóa /share/%2F..%2Fapi/auth/session.
- Máy chủ gốc trả dữ liệu từ /api/auth/session.
- Tham số ?cachebuster=123 tạo khóa cache duy nhất khi thử nghiệm.

### Quy Trình Tấn Công
1. Kẻ tấn công gửi URL độc hại đến nạn nhân, ngụy trang là link chia sẻ.
2. Nạn nhân (đã đăng nhập) truy cập, CDN chuyển tiếp yêu cầu, máy chủ gốc trả token, và CDN lưu vào cache.
3. Kẻ tấn công truy cập cùng URL, nhận cache hit chứa token của nạn nhân.
4. Token cho phép chiếm quyền tài khoản, xem chat, thông tin thanh toán.

### Tính "Wildcard"
Khác với deception thông thường (giới hạn endpoint cụ thể), lỗ hổng này cho phép nhắm bất kỳ endpoint nào bằng path traversal, do quy tắc cache /share/* quá rộng.

### So Sánh Với Lỗ Hổng Trước
Năm 2023, lỗ hổng tương tự trong ChatGPT dùng phần mở rộng giả (.css) để lừa cache. Biến thể 2024 dùng path traversal (%2F..%2F), vượt qua bản vá cũ, nhưng đều dựa trên sự không đồng bộ CDN-origin.

## Tác Động
Lỗ hổng cho phép rò rỉ token xác thực, dẫn đến chiếm quyền tài khoản, truy cập dữ liệu nhạy cảm (chat, thanh toán). Các hệ thống dùng CDN với quy tắc cache lỏng lẻo đều có nguy cơ tương tự.

## Khắc Phục
- **Quy tắc cache**: Loại bỏ wildcard (/share/*), kiểm tra Content-Type nghiêm ngặt.
- **Đồng bộ phân tích**: CDN và máy chủ gốc phải xử lý URL-encoded đồng nhất.
- **Header**: Áp dụng Cache-Control: no-store cho endpoint nhạy cảm.
- **Chặn traversal**: Ngăn chặn ../ trong URL.

## Kết Luận
Lỗ hổng wildcard web cache deception trong ChatGPT cho thấy nguy cơ từ quy tắc cache lỏng lẻo và sự không đồng bộ trong phân tích URL. Các hệ thống AI cần kiểm tra bảo mật kỹ lưỡng để tránh rủi ro tương tự.
