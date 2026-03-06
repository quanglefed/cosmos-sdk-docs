# ADR 059: Phạm Vi Test

## Changelog

* 2022-08-02: Bản nháp đầu tiên
* 2023-03-02: Thêm chi tiết cho integration test
* 2023-03-23: Thêm chi tiết cho E2E test

## Trạng Thái

ĐỀ XUẤT Đã Triển Khai Một Phần

## Tóm Tắt

Công việc gần đây trong SDK nhằm phá vỡ go module nguyên khối đã làm nổi bật những thiếu sót và không nhất quán trong mô hình test của chúng ta. ADR này làm rõ ngôn ngữ chung để nói về phạm vi test và đề xuất trạng thái lý tưởng của test ở mỗi phạm vi.

## Bối Cảnh

[ADR-053: Go Module Refactoring](./adr-053-go-module-refactoring.md) thể hiện mong muốn của chúng ta về một SDK bao gồm nhiều Go module được phiên bản độc lập, và [ADR-057: App Wiring](./adr-057-app-wiring.md) cung cấp phương pháp để phá vỡ các inter-module dependency thông qua dependency injection.

Chúng ta xác định ba phạm vi test: **unit**, **integration**, **e2e** (end to end), tìm cách xác định ranh giới của mỗi phạm vi, những hạn chế của chúng và trạng thái lý tưởng của chúng trong SDK.

### Unit Test

Unit test kiểm tra code chứa trong một module đơn hoặc package đơn riêng biệt với phần còn lại của code base. Chúng ta xác định hai cấp độ:

* **Minh họa**: kiểm tra một phần nguyên tử của module trong cô lập — có thể thiết lập fixture/mocking của các phần khác của module.
* **Hành trình**: kiểm tra toàn bộ chức năng của module với các dependency được mock.

#### Hạn Chế

Một số module được kết hợp chặt chẽ ngoài giai đoạn test. Báo cáo dependency gần đây cho `bank -> auth` tìm thấy 274 lần sử dụng tổng cộng `auth` trong `bank`, 50 trong code production và 224 trong test.

### Integration Test

Integration test định nghĩa và kiểm tra các mối quan hệ giữa số lượng tùy ý các module và/hoặc các hệ thống con ứng dụng.

Wiring cho integration test được cung cấp bởi `depinject` và một số [helper code](https://github.com/cosmos/cosmos-sdk/blob/2bec9d2021918650d3938c3ab242f84289daef80/testutil/sims/app_helpers.go#L95) khởi động một ứng dụng đang chạy. Một phần của ứng dụng đang chạy sau đó có thể được test.

#### Hạn Chế

Thiết lập trạng thái input cụ thể có thể phức tạp hơn vì ứng dụng bắt đầu từ trạng thái không. Các test cũng có thể dễ bị ảnh hưởng hơn.

### Simulations

Simulations (hay còn gọi là generative testing) là trường hợp đặc biệt của integration test, nơi các module operation ngẫu nhiên có tính xác định được thực thi đối với simapp đang chạy, xây dựng các block trên chain cho đến khi đạt đến chiều cao được chỉ định.

### E2E Test

End-to-end test kiểm tra toàn bộ hệ thống trong môi trường xấp xỉ với production càng nhiều càng tốt.

#### Hạn Chế

Nhìn chung, các hạn chế của end to end test là chi phí điều phối và tính toán. Cần scaffold để khởi động và chạy môi trường prod-like và quá trình này mất nhiều thời gian hơn nhiều để bắt đầu và chạy so với unit hoặc integration test.

## Quyết Định

Chúng ta chấp nhận các phạm vi test này và xác định các điểm quyết định sau cho mỗi phạm vi.

| Phạm Vi     | Kiểu Ứng Dụng       | Mock? |
| ----------- | ------------------- | ----- |
| Unit        | Không               | Có    |
| Integration | integration helpers | Một số |
| Simulation  | minimal app         | Không |
| E2E         | minimal app         | Không |

### Unit Test

* Tất cả module phải có unit test coverage với mock.
* Các test minh họa nên nhiều hơn hành trình trong unit test.
* Unit test nên nhiều hơn integration test.
* Unit test không được giới thiệu các dependency bổ sung ngoài những gì đã có trong code production.

### Integration Test

* Tất cả integration test phải được đặt trong `/tests/integration`.
* Khuyến nghị sử dụng số lượng module nhỏ nhất có thể khi khởi động ứng dụng.
* Integration test nên nhiều hơn e2e test.

### Simulations

Simulations phải sử dụng minimal application và được đặt trong `/x/{moduleName}/simulation`.

### E2E Test

* Các e2e test hiện có phải được migrate sang integration test.
* E2E test runner phải chuyển từ in-process Tendermint sang runner được cung cấp bởi Docker qua [dockertest](https://github.com/ory/dockertest).
* Phải viết các E2E test kiểm tra network upgrade đầy đủ.

## Hậu Quả

### Tích Cực

* Test coverage được tăng.
* Tổ chức test được cải thiện.
* Kích thước dependency graph trong module giảm.
* simapp được xóa như một dependency khỏi các module.
* Thời gian chạy CI giảm sau khi chuyển đổi khỏi in-process Tendermint.

### Tiêu Cực

* Một số trùng lặp logic test giữa unit và integration test trong quá trình chuyển đổi.

## Tài Liệu Tham Khảo

* [ADR-053](./adr-053-go-module-refactoring.md)
* [ADR-057](./adr-057-app-wiring.md)
