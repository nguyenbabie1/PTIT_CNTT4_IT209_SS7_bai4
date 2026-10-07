# Bài 4: Cấu hình Reverse Proxy Nginx cho ứng dụng Spring Boot

- Họ tên: Đoàn Trung Nguyên
- Mã lớp: HN-K24-CNTT4
- Mã sinh viên: B24DTCN435
- Máy chủ: NguyenbabieVM
- Ngày thực hiện: 07/10/2026
- Đường dẫn nộp: `homework/session_07/ex4/`

## 1. Mục tiêu và mô hình định tuyến

Nginx lắng nghe tại cổng 80, phục vụ trang tĩnh trong `/var/www/html/` và chuyển các yêu cầu có tiền tố `/api/` đến Spring Boot tại `127.0.0.1:8082`.

| Yêu cầu | Xử lý |
| --- | --- |
| `/` | Phục vụ `/var/www/html/index.html` |
| `/api/health` | Chuyển tiếp đến `http://127.0.0.1:8082/health` |

## 2. Trang tĩnh và cấu hình Nginx

Trang `/var/www/html/index.html` hiển thị họ tên, mã lớp và mã sinh viên. Kết quả curl ở mục 5 xác nhận các thông tin này.

Tệp cấu hình đi kèm: `spring-proxy.conf`, đặt tại `/etc/nginx/sites-available/spring-proxy.conf` trên VPS. Tệp đi kèm tái hiện cấu hình đã hướng dẫn trong quá trình thực hành, không phải bản tải trực tiếp từ VPS.

Trong cấu hình:

- `root /var/www/html;` và `index index.html;` xác định trang tĩnh.
- `location /` sử dụng `try_files $uri $uri/ =404;`.
- `location /api/` sử dụng `proxy_pass http://127.0.0.1:8082/;`.
- Dấu `/` cuối `proxy_pass` khiến tiền tố `/api/` được thay bằng `/`, do đó backend nhận `/health`.
- Các header `Host`, `X-Real-IP`, `X-Forwarded-For` và `X-Forwarded-Proto` được chuyển tiếp đến backend.

Kích hoạt site bằng liên kết mềm:

```bash
sudo ln -s /etc/nginx/sites-available/spring-proxy.conf /etc/nginx/sites-enabled/spring-proxy.conf
```

Gỡ liên kết site `default` nếu nó đang chiếm cấu hình mặc định trên cổng 80. Kiểm tra cú pháp thành công trước khi reload:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

## 3. Chuẩn bị backend Spring Boot

Backend sử dụng Java 17, Gradle và Spring Web. Phiên bản ghi nhận trong log: Spring Boot 4.1.1; Gradle 9.7.1.

File `src/main/resources/application.properties`:

```properties
spring.application.name=spring-backend
server.port=8082
```

Controller trong package `com.example.backend`:

```java
package com.example.backend;

import java.util.Map;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HealthController {
    @GetMapping("/health")
    public Map<String, String> health() {
        return Map.of(
            "status", "UP",
            "application", "spring-backend",
            "student", "Đoàn Trung Nguyên"
        );
    }
}
```

Build và chạy từ thư mục `/root/spring-backend`:

```bash
./gradlew --no-daemon bootJar
nohup java -jar build/libs/spring-backend-0.0.1-SNAPSHOT.jar > app.log 2>&1 &
```

Trích kết quả thực tế:

```text
BUILD SUCCESSFUL in 1m 6s
4 actionable tasks: 4 executed
```

```text
2026-10-07T03:49:46.640Z  INFO 5438 --- [spring-backend] [           main] o.s.boot.tomcat.TomcatWebServer          : Tomcat started on port 8082 (http) with context path '/'
2026-10-07T03:49:46.653Z  INFO 5438 --- [spring-backend] [           main] c.e.backend.SpringBackendApplication     : Started SpringBackendApplication in 2.087 seconds (process running for 2.674)
```

## 4. Kiểm tra cú pháp Nginx

Lệnh:

```bash
sudo nginx -t
```

Đầu ra thực tế:

```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

## 5. Kiểm tra trang tĩnh

Lệnh:

```bash
curl -I http://localhost/
```

Đầu ra thực tế:

```text
HTTP/1.1 200 OK
Server: nginx/1.24.0 (Ubuntu)
Date: Wed, 07 Oct 2026 03:34:01 GMT
Content-Type: text/html
Content-Length: 964
Last-Modified: Wed, 07 Oct 2026 03:33:17 GMT
Connection: keep-alive
ETag: "6ac5bd7d-3c4"
Accept-Ranges: bytes
```

Lệnh `curl -I` chỉ lấy header. Để kiểm tra nội dung, sử dụng:

```bash
curl -i http://localhost/
```

Kết quả thực tế trả `HTTP/1.1 200 OK`. Trích phần nội dung HTML:

```html
<h1>Bài thực hành Nginx Reverse Proxy</h1>
<p><strong>Họ tên:</strong> Đoàn Trung Nguyên</p>
<p><strong>Mã lớp:</strong> HN-K24-CNTT4</p>
<p><strong>Mã sinh viên:</strong> B24DTCN435</p>
<p>Trang tĩnh được phục vụ trực tiếp bởi Nginx.</p>
<a href="/api/health">Kiểm tra API backend</a>
```

## 6. Kiểm tra API trực tiếp và qua Reverse Proxy

### Truy cập trực tiếp backend

```bash
curl -i http://127.0.0.1:8082/health
```

Đầu ra thực tế:

```text
HTTP/1.1 200
Content-Type: application/json
Content-Length: 79
Date: Wed, 07 Oct 2026 03:50:13 GMT

{"status":"UP","student":"Đoàn Trung Nguyên","application":"spring-backend"}
```

### Truy cập qua Nginx

```bash
curl -i http://localhost/api/health
```

Đầu ra thực tế:

```text
HTTP/1.1 200
Server: nginx/1.24.0 (Ubuntu)
Date: Wed, 07 Oct 2026 03:50:13 GMT
Content-Type: application/json
Content-Length: 79
Connection: keep-alive

{"status":"UP","student":"Đoàn Trung Nguyên","application":"spring-backend"}
```

## 7. Lỗi gặp phải và cách khắc phục

Ban đầu `/api/health` trả `502 Bad Gateway`. Kiểm tra trực tiếp `127.0.0.1:8082/health` báo không kết nối được; `ss` không ghi nhận tiến trình lắng nghe cổng 8082. Log Nginx ghi:

```text
connect() failed (111: Connection refused) while connecting to upstream
```

Nguyên nhân: backend chưa chạy. Cách xử lý: cài Java 17, tạo ứng dụng Spring Boot có `/health`, build bằng Gradle và chạy tại cổng 8082. Sau đó cả hai đường truy cập API đều trả HTTP 200.

JSON trả về là dữ liệu phản hồi, không phải lệnh shell. Dán JSON vào terminal gây `command not found`; lỗi nhập lệnh này không phải lỗi của Nginx hay Spring Boot.

## 8. Kết luận

- Cú pháp Nginx hợp lệ.
- Trang `/` trả HTTP 200 và hiển thị thông tin học viên.
- Backend trực tiếp và `/api/health` qua Nginx đều trả HTTP 200 cùng JSON trạng thái `UP`.
- Lỗi 502 đã được khắc phục trong các lần kiểm tra ghi nhận.

Các phép thử trong báo cáo được thực hiện trên VPS qua localhost; chưa có kết quả truy cập từ máy ngoài bằng IP công khai. Backend chạy bằng `nohup` và chưa được cấu hình tự khởi động sau khi VPS reboot.
