# Hijacking OAuth Flows via Cookie Tossing

**Tác giả:** Elliot Ward  
**Chia sẻ:**  
Tìm hiểu về **tấn công Cookie Tossing**, một kỹ thuật ít được biết đến để chiếm quyền điều khiển luồng OAuth và thực hiện chiếm đoạt tài khoản tại các Nhà cung cấp Danh tính (IdPs). Khám phá tác động, ví dụ thực tế và cách bảo vệ ứng dụng bằng tiền tố cookie **__Host__**.

## Nội dung bài viết
- [Cookie Tossing là gì?](#cookie-tossing-là-gì)
- [Khai thác Cookie Tossing](#khai-thác-cookie-tossing)
- [Thăm lại GitPod](#thăm-lại-gitpod)
- [Tiền tố cookie __Host__](#tiền-tố-cookie-__host__)

## Cookie Tossing là gì?

**Cookie Tossing** là kỹ thuật cho phép một **subdomain** (ví dụ: `securitylabs.snyk.io`) thiết lập cookie trên **domain cha** (ví dụ: `snyk.io`). Kỹ thuật này thường bị bỏ qua hoặc ít được biết đến, dẫn đến ít tài liệu nghiên cứu. Bài viết này giải thích chi tiết cách Cookie Tossing có thể chiếm quyền điều khiển luồng OAuth và gây ra chiếm đoạt tài khoản tại Nhà cung cấp Danh tính (IdP).

### HTTP Cookies là gì?

Theo chuẩn **RFC 6265**, **cookie** là một mẩu dữ liệu nhỏ được trao đổi giữa máy chủ và trình duyệt web của người dùng. Cookie rất quan trọng trong các ứng dụng web vì chúng:
- Lưu trữ dữ liệu giới hạn.
- Duy trì trạng thái (state) cho giao thức HTTP vốn không lưu trạng thái.
- Cho phép duy trì phiên người dùng, lưu trữ tùy chọn và cung cấp trải nghiệm cá nhân hóa.

#### Các thuộc tính và cờ của cookie

Cookie có các **thuộc tính (attributes)** và **cờ (flags)** định nghĩa hành vi và phạm vi của chúng. Dưới đây là các thuộc tính và cờ chính:

| Thuộc tính   | Mô tả | Ví dụ |
|-------------|-------|-------|
| Expires     | Đặt ngày và giờ hết hạn của cookie. | `Expires=Wed, 21 Oct 2024 07:28:00 GMT` |
| Max-Age     | Xác định thời gian sống của cookie (tính bằng giây). | `Max-Age=3600` (1 giờ) |
| Domain      | Chỉ định domain mà cookie có hiệu lực, cho phép các subdomain truy cập. | `Domain=.snyk.io` |
| Path        | Giới hạn cookie cho một đường dẫn cụ thể trong domain. | `Path=/account` |
| SameSite    | Kiểm soát việc gửi cookie trong các yêu cầu cross-site để bảo vệ chống CSRF. Giá trị: `Strict`, `Lax`, `None`. | `SameSite=Lax` |

| Cờ         | Mô tả | Ví dụ |
|------------|-------|-------|
| Secure     | Đảm bảo cookie chỉ được gửi qua HTTPS. | `Secure` |
| HttpOnly   | Ngăn cookie bị truy cập qua JavaScript, tăng cường bảo mật. | `HttpOnly` |

Những thuộc tính và cờ này xác định thời gian sống, phạm vi và bảo mật của cookie, giúp quản lý phiên người dùng một cách hiệu quả và an toàn.

### Cách thiết lập cookie

Cookie có thể được thiết lập bằng hai cách chính:

1. **Sử dụng header Set-Cookie trong phản hồi HTTP**:
```
HTTP/1.1 200 OK
Set-Cookie: userId=patch01; Expires=Wed, 21 Oct 2024 07:28:00 GMT; Domain=.snyk.io; Path=/; Secure; HttpOnly; SameSite=Lax
```

2. **Sử dụng JavaScript Cookie API**:
```javascript
document.cookie = "userId=patch01; expires=Wed, 21 Oct 2024 07:28:00 GMT; path=/; domain=.snyk.io; secure; samesite=lax";
```

Trong trình duyệt, cookie được lưu trữ dưới dạng **tuple** gồm **key**, **value** và **các thuộc tính**. Khi trình duyệt gửi cookie về máy chủ, chỉ **key** và **value** được gửi, không bao gồm các thuộc tính. Trình duyệt giới hạn số lượng cookie tối đa cho mỗi domain.

### Cookie Domains

Thuộc tính **Domain** xác định domain nào có thể truy cập cookie. Mặc định, cookie chỉ có hiệu lực với domain đã thiết lập nó. Tuy nhiên, thuộc tính **Domain** có thể mở rộng phạm vi truy cập. Ví dụ:
- Nếu `blog.snyk.io` đặt cookie với `Domain=.snyk.io`, cookie này sẽ khả dụng cho tất cả subdomain của `snyk.io` (như `app.snyk.io`, `snyk.io`).
- Ngược lại, domain cha (`snyk.io`) không thể đặt cookie chỉ dành riêng cho một subdomain cụ thể (như `blog.snyk.io`).

### Cookie Paths và Thứ tự

Thuộc tính **Path** xác định các URL mà cookie áp dụng. Mặc định, cookie có hiệu lực với đường dẫn của URL tạo ra nó và các thư mục con. Ví dụ:
- Cookie với `Path=/account` sẽ khả dụng cho `/account` và `/account/settings`.
- Cookie được ưu tiên theo **độ cụ thể** của **Path**: cookie với đường dẫn cụ thể hơn (như `/account/settings`) được gửi trước cookie với đường dẫn ít cụ thể hơn (như `/account`).

## Khai thác Cookie Tossing

**Cookie Tossing** lợi dụng thuộc tính **Domain** và **Path** để tấn công. Khi kẻ tấn công kiểm soát một subdomain (qua lỗ hổng XSS hoặc thiết kế của dịch vụ), họ có thể đặt cookie trên domain cha. Điều này có thể dẫn đến:
- Thiết lập cookie phiên của kẻ tấn công trên trình duyệt của nạn nhân cho các endpoint cụ thể.
- Ví dụ: Kẻ tấn công đặt cookie với `Domain=.snyk.io` và `Path=/api/payment`. Khi nạn nhân truy cập endpoint này, ứng dụng sẽ sử dụng cookie của kẻ tấn công thay vì cookie của nạn nhân, dẫn đến việc thông tin nhạy cảm (như phương thức thanh toán) bị thêm vào tài khoản của kẻ tấn công.

### Thách thức khi khai thác
- **CSRF tokens**: Nếu ứng dụng sử dụng token chống CSRF, yêu cầu hợp pháp từ nạn nhân có thể thất bại do token không khớp.
- Tuy nhiên, nhiều API dựa trên JSON không sử dụng CSRF tokens, dựa vào **Same Origin Policy (SOP)** và **CORS**. Điều này khiến chúng dễ bị tấn công từ subdomain.
- **SameSite** không bảo vệ chống lại Cookie Tossing vì subdomain được coi là cùng "site" theo định nghĩa của SameSite (kể cả với `Lax` hoặc `Strict`).

## Thăm lại GitPod

**GitPod** là một môi trường phát triển đám mây (Cloud Development Environment - CDE) cho phép triển khai môi trường phát triển nhanh chóng. Các môi trường này được lưu trữ trên subdomain của `gitpod.io`, và người dùng có thể thực thi JavaScript trên các subdomain này.

### Tấn công Cookie Tossing trên GitPod
Các nhà nghiên cứu đã kiểm tra luồng OAuth của GitPod với các nhà cung cấp như GitHub hoặc BitBucket. Bằng cách:
1. Tạo JavaScript trên một instance GitPod (như `redacted.ws–eu114.gitpod.io`) để đặt cookie `_gitpod_io_jwt2_` với giá trị phiên của kẻ tấn công, với đường dẫn:
   - `/api/authorize`
   - `/auth/bitbucket/callback`
2. Gửi URL của workspace chứa JavaScript độc hại đến nạn nhân.

Khi nạn nhân truy cập URL, cookie của kẻ tấn công được thiết lập. Khi nạn nhân kết nối tài khoản GitHub/BitBucket, luồng OAuth sẽ sử dụng cookie của kẻ tấn công, dẫn đến việc tài khoản Git của nạn nhân bị liên kết với tài khoản GitPod của kẻ tấn công. Điều này cho phép kẻ tấn công:
- Tạo workspace từ các kho mã nguồn của nạn nhân.
- Thay đổi mã nguồn hoặc đẩy commit mới.

### Kết quả
Lỗ hổng này được báo cáo cho GitPod vào ngày **26/06/2024** và được khắc phục vào ngày **01/07/2024** bằng cách sử dụng tiền tố cookie **__Host__**. Lỗ hổng được gán mã **CVE-2024-21583**.

## Tiền tố cookie __Host__

Tiền tố **__Host__** là giải pháp đơn giản để ngăn chặn Cookie Tossing. Khi sử dụng **__Host__**:
- Cookie chỉ có hiệu lực với domain đã thiết lập nó.
- Không thể sửa đổi thuộc tính **Domain** hoặc **Path**, ngăn subdomain đặt cookie trên domain cha hoặc nhắm vào đường dẫn cụ thể.

## Kết luận

**Cookie Tossing** là một lỗ hổng độc đáo và thường bị bỏ qua, ảnh hưởng đến các ứng dụng không sử dụng tiền tố **__Host__**. Kỹ thuật này có thể bị khai thác để chiếm quyền điều khiển các yêu cầu nhạy cảm, đặc biệt trong các luồng phức tạp như OAuth, dẫn đến việc lộ dữ liệu hoặc cấp quyền truy cập trái phép. Việc sử dụng **__Host__** là một biện pháp bảo vệ hiệu quả, nhưng vẫn còn hiếm trong thực tế, khiến nhiều ứng dụng, đặc biệt là những ứng dụng sử dụng subdomain, dễ bị tấn công.

**Lưu ý:** Để biết thêm chi tiết về giá cả hoặc dịch vụ của xAI, vui lòng truy cập [x.ai](https://x.ai).
