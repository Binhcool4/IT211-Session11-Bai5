# Báo cáo: Chiến lược kiểm thử cho hệ thống Booking Microservice

# Phần 1: Phân tích logic

## 1. Thách thức kiểm thử trong kiến trúc microservices

Kiến trúc microservices phức tạp hơn monolithic vì hệ thống được chia thành nhiều service độc lập như:

```text
UserService
BookingService
NotificationService
```

Mỗi service có:

```text
Database riêng
API riêng
Logic riêng
Cách deploy riêng
```

Vì vậy, lỗi không chỉ nằm trong từng service mà còn có thể xuất hiện khi các service giao tiếp với nhau.

Ví dụ:

```text
BookingService gọi UserService để kiểm tra user.
BookingService tạo lịch hẹn.
BookingService gọi NotificationService để gửi email/SMS.
```

Nếu một API thay đổi response hoặc lỗi mạng xảy ra, BookingService có thể bị lỗi dù Unit Test vẫn passed.

---

## 2. Vì sao chỉ Unit Test là chưa đủ?

Unit Test chỉ kiểm tra từng hàm hoặc từng class riêng lẻ, thường dùng mock để giả lập dependency.

Ví dụ:

```text
Mock UserService trả về user hợp lệ.
Mock NotificationService gửi thông báo thành công.
```

Nhưng thực tế có thể xảy ra:

```text
UserService đổi format JSON.
NotificationService timeout.
Database thật hoạt động khác database giả lập.
API trả về lỗi 500.
Network chậm hoặc mất kết nối.
```

Những lỗi này Unit Test không phát hiện được.

Vì vậy, ngoài Unit Test cần có:

```text
Integration Test
Contract Test
End-to-End Test
Performance Test cơ bản
CI/CD Quality Gate
```

---

# Phần 2: Chiến lược kiểm thử toàn diện

## 1. Mô hình kiểm thử: Test Pyramid

Đề xuất áp dụng mô hình Test Pyramid:

```text
            End-to-End Test
          Integration Test
        Unit Test
```

Tỷ lệ đề xuất:

| Loại test | Tỷ lệ | Mục tiêu |
|---|---:|---|
| Unit Test | 60% | Kiểm tra logic nội bộ từng service |
| Integration Test | 30% | Kiểm tra service với DB, API, message, container |
| End-to-End Test | 10% | Kiểm tra luồng nghiệp vụ hoàn chỉnh |

---

## 2. Unit Test

Áp dụng cho từng service:

```text
UserService
BookingService
NotificationService
```

Mục tiêu:

```text
Test logic nghiệp vụ
Test happy path
Test unhappy path
Test validation
Test exception
```

Công cụ đề xuất:

```text
JUnit 5
Mockito
AssertJ
JaCoCo
```

Ví dụ cần test trong BookingService:

```text
Tạo booking thành công
Không cho booking khi user không tồn tại
Không cho booking trùng lịch
Không cho booking với thời gian quá khứ
Không cho booking khi service trả lỗi
```

---

## 3. Integration Test

Integration Test dùng để kiểm tra service với thành phần thật hơn, ví dụ database hoặc REST API.

Công cụ đề xuất:

```text
Spring Boot Test
@DataJpaTest
@WebMvcTest
Testcontainers
REST Assured
MockWebServer
```

Ví dụ:

```text
BookingService kết nối PostgreSQL thật bằng Testcontainers
Kiểm tra repository lưu booking đúng
Kiểm tra API POST /bookings trả về 201 Created
Kiểm tra API GET /bookings/{id} trả đúng dữ liệu
```

Với database, nên dùng:

```text
Testcontainers + PostgreSQL/MySQL
```

thay vì chỉ dùng H2, vì H2 có thể khác DB thật.

---

## 4. End-to-End Test

End-to-End Test kiểm tra toàn bộ luồng hệ thống.

Ví dụ luồng đặt lịch:

```text
User đăng nhập
User chọn lịch trống
BookingService tạo booking
NotificationService gửi thông báo
User nhận kết quả đặt lịch thành công
```

Công cụ đề xuất:

```text
REST Assured
Postman/Newman
Cypress
Selenium
Docker Compose
```

Không nên viết quá nhiều E2E Test vì:

```text
Chạy chậm
Dễ flaky
Khó debug
Tốn tài nguyên CI/CD
```

Chỉ nên chọn các luồng quan trọng nhất.

---

## 5. Chiến lược Testing Coverage với JaCoCo

Mỗi microservice cần có ngưỡng coverage riêng.

Mục tiêu đề xuất:

```text
Line Coverage >= 80%
Branch Coverage >= 70%
```

Áp dụng cho:

```text
UserService
BookingService
NotificationService
```

Ý nghĩa:

```text
Line Coverage kiểm tra số dòng code đã được chạy.
Branch Coverage kiểm tra các nhánh if/else, switch, exception.
```

Branch Coverage quan trọng vì microservices có nhiều logic rẽ nhánh như:

```text
User tồn tại / không tồn tại
Booking hợp lệ / không hợp lệ
Notification gửi thành công / thất bại
```

Nếu coverage thấp hơn ngưỡng, pipeline CI/CD phải fail.

---

## 6. Kiểm thử giao tiếp giữa các services

Trong microservices, lỗi thường xảy ra ở phần giao tiếp API.

Cần kiểm thử:

```text
Request body
Response body
Status code
Timeout
Error response
Backward compatibility
```

Đề xuất dùng Consumer-driven Contract Testing với Pact.

Ví dụ:

```text
BookingService là consumer của UserService.
UserService là provider.
BookingService định nghĩa contract mong muốn.
UserService phải đảm bảo response không phá contract đó.
```

Lợi ích:

```text
Phát hiện sớm API breaking change
Không cần chạy toàn bộ hệ thống
Phù hợp với microservices
Giảm phụ thuộc vào E2E Test
```

Ngoài Pact, có thể dùng:

```text
WireMock
MockWebServer
Spring Cloud Contract
```

---

## 7. Tích hợp vào CI/CD

Pipeline đề xuất:

```text
1. Developer push code
2. Run static analysis
3. Run Unit Test
4. Run JaCoCo coverage check
5. Run Integration Test
6. Run Contract Test
7. Build Docker image
8. Deploy lên test environment
9. Run End-to-End Test
10. Deploy staging/production
```

Thứ tự nên tối ưu tốc độ:

```text
Unit Test chạy trước vì nhanh
Integration Test chạy sau
E2E Test chạy cuối vì chậm nhất
```

Nếu fail ở bước nào thì dừng pipeline ngay.

---

## 8. Tổng kết chiến lược

Chiến lược kiểm thử cho Booking microservice cần kết hợp nhiều tầng:

```text
Unit Test: kiểm tra logic nhỏ
Integration Test: kiểm tra service với DB/API thật hơn
Contract Test: kiểm tra giao tiếp giữa services
End-to-End Test: kiểm tra luồng nghiệp vụ hoàn chỉnh
JaCoCo: kiểm soát coverage
CI/CD: tự động hóa chất lượng
```

Mục tiêu cuối cùng:

```text
Phát hiện lỗi sớm
Giảm kiểm thử thủ công
Đảm bảo service thay đổi không phá service khác
Duy trì tốc độ phát triển nhanh
Đạt coverage 80% line và 70% branch
Giảm lỗi nghiêm trọng trên staging
```
