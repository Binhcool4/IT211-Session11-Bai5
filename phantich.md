# Phần 1: Phân tích logic

## Thách thức kiểm thử trong kiến trúc Microservices

So với kiến trúc Monolithic, kiến trúc Microservices có độ phức tạp cao hơn vì hệ thống được chia thành nhiều dịch vụ độc lập thay vì tập trung trong một ứng dụng duy nhất. Mỗi service có database riêng, được triển khai độc lập và giao tiếp với nhau thông qua API. Điều này làm phát sinh nhiều vấn đề trong quá trình kiểm thử.

Một thách thức lớn là việc kiểm tra giao tiếp giữa các service. Trong kiến trúc Monolithic, các module thường gọi trực tiếp các phương thức bên trong cùng hệ thống. Trong khi đó ở Microservices, việc giao tiếp phụ thuộc vào API hoặc network. Vì vậy có thể xuất hiện các lỗi như sai định dạng dữ liệu, lỗi kết nối hoặc thay đổi API giữa các service.

Ngoài ra, dữ liệu giữa các service cũng có thể không đồng bộ. Ví dụ khi BookingService tạo lịch hẹn thành công nhưng NotificationService gặp lỗi và không gửi được email xác nhận, hệ thống sẽ rơi vào trạng thái không nhất quán.

Một khó khăn khác là môi trường kiểm thử trở nên phức tạp hơn. Thay vì chỉ chạy một ứng dụng duy nhất, cần khởi động nhiều service cùng lúc và quản lý nhiều database khác nhau.

Chỉ sử dụng Unit Test là chưa đủ trong trường hợp này. Unit Test chỉ kiểm tra logic bên trong từng service một cách cô lập bằng Mock. Nó không thể kiểm tra việc giao tiếp thực tế giữa các service hoặc phát hiện các lỗi tích hợp. Một service có thể pass toàn bộ Unit Test nhưng vẫn gặp lỗi khi chạy chung với các service khác.

---

# Phần 2: Thiết kế chiến lược kiểm thử cho hệ thống Booking Microservices

## Mô hình kiểm thử

Đối với hệ thống Booking microservices, mô hình Test Pyramid là lựa chọn phù hợp vì giúp cân bằng giữa tốc độ và độ tin cậy của quá trình kiểm thử.

Tỷ lệ đề xuất:

- Unit Test: khoảng 70%
- Integration Test: khoảng 20%
- End-to-End Test: khoảng 10%

Unit Test chiếm phần lớn vì chạy nhanh, dễ bảo trì và giúp kiểm tra logic nghiệp vụ riêng của từng service.

Integration Test chiếm tỷ lệ trung bình để kiểm tra sự tương tác giữa các thành phần như database, API hoặc nhiều service khác nhau.

End-to-End Test chiếm tỷ lệ thấp vì thời gian chạy lâu và chi phí bảo trì cao.

---

## Công cụ và kỹ thuật

Ở tầng Unit Test có thể sử dụng:

- JUnit 5
- Mockito
- AssertJ

JUnit 5 dùng để xây dựng test, Mockito dùng để tạo Mock object và AssertJ giúp việc viết assertion dễ đọc hơn.

Ở tầng Integration Test có thể sử dụng:

- Spring `@DataJpaTest`
- Spring `@WebMvcTest`
- Testcontainers

Testcontainers cho phép tạo database thật bằng Docker để kiểm tra môi trường gần giống thực tế.

Ở tầng End-to-End Test có thể sử dụng:

- REST Assured
- Selenium
- Cypress

REST Assured phù hợp kiểm thử API, trong khi Selenium hoặc Cypress phù hợp với giao diện người dùng.

---

## Chiến lược cho Testing Coverage

Mỗi microservice sẽ tích hợp JaCoCo để đo độ phủ mã nguồn.

Thiết lập Quality Gate:

```text
Line Coverage >= 80%
Branch Coverage >= 70%
```

Việc đặt ngưỡng tối thiểu giúp đảm bảo các service đều được kiểm tra ở mức chấp nhận được và tránh tình trạng developer bỏ qua việc viết test.

Branch Coverage được áp dụng vì nó phản ánh tốt hơn chất lượng kiểm thử logic nghiệp vụ.

Nếu coverage thấp hơn mức yêu cầu thì pipeline CI/CD sẽ tự động thất bại.

---

## Giải pháp kiểm thử giao tiếp giữa các services

Đối với việc kiểm thử tương tác giữa các service, có thể kết hợp nhiều phương pháp.

Integration Test kết hợp Testcontainers giúp chạy nhiều service và database trong môi trường gần giống thực tế.

Consumer-driven Contract Testing sử dụng Pact giúp đảm bảo dữ liệu trao đổi giữa các service không bị thay đổi ngoài ý muốn.

Ví dụ:

- BookingService gửi dữ liệu cho NotificationService
- Nếu NotificationService thay đổi cấu trúc API
- Pact sẽ phát hiện ngay lỗi không tương thích

Ngoài ra cần có End-to-End Test để kiểm tra toàn bộ luồng xử lý:

```text
User tạo tài khoản
→ Đặt lịch
→ Lưu dữ liệu
→ Gửi thông báo
→ Hoàn tất
```

---

## Tích hợp vào CI/CD

Quy trình CI/CD nên tự động hóa toàn bộ quá trình kiểm thử để giảm thời gian kiểm tra thủ công.

Quy trình đề xuất:

```text
Developer push code
        ↓
Build project
        ↓
Chạy Unit Test
        ↓
Kiểm tra JaCoCo Coverage
        ↓
Chạy Integration Test
        ↓
Chạy Contract Test
        ↓
Chạy End-to-End Test
        ↓
Deploy staging
        ↓
Deploy production
```

Trong quy trình này, Unit Test sẽ chạy đầu tiên vì tốc độ nhanh và phản hồi sớm. Integration Test và Contract Test sẽ chạy tiếp theo để phát hiện lỗi giữa các service. End-to-End Test chạy cuối cùng để xác nhận toàn bộ hệ thống hoạt động ổn định.

Chiến lược này giúp giảm lỗi phát sinh, tăng mức độ tự động hóa và hỗ trợ hệ thống mở rộng dễ dàng trong tương lai.