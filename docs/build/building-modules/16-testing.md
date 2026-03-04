---
sidebar_position: 1
---

# Testing (Kiểm Thử)

Cosmos SDK chứa các loại [kiểm thử](https://martinfowler.com/articles/practical-test-pyramid.html) khác nhau. Các kiểm thử này có các mục tiêu khác nhau và được sử dụng ở các giai đoạn khác nhau trong chu kỳ phát triển. Chúng tôi khuyến nghị, như một nguyên tắc chung, sử dụng kiểm thử ở tất cả các giai đoạn của chu kỳ phát triển. Khuyến nghị, là nhà phát triển chain, kiểm thử ứng dụng và module của bạn theo cách tương tự như SDK.

Lý do đằng sau kiểm thử có thể tìm thấy trong [ADR-59](https://docs.cosmos.network/main/build/architecture/adr-059-test-scopes).

## Unit Tests (Kiểm Thử Đơn Vị)

Unit test là loại kiểm thử thấp nhất trong [kim tự tháp kiểm thử](https://martinfowler.com/articles/practical-test-pyramid.html). Tất cả các gói và module nên có độ phủ unit test. Các module nên mock các phụ thuộc của chúng: điều này có nghĩa là mock các keeper.

SDK sử dụng `mockgen` để tạo mock cho các keeper:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/scripts/mockgen.sh#L3-L6
```

Bạn có thể đọc thêm về mockgen [tại đây](https://go.uber.org/mock).

### Ví Dụ

Làm ví dụ, chúng ta sẽ đi qua [các keeper test](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/x/gov/keeper/keeper_test.go) của module `x/gov`.

Module `x/gov` có kiểu `Keeper`, yêu cầu một số phụ thuộc bên ngoài (tức là các import bên ngoài `x/gov` để hoạt động đúng).

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/x/gov/keeper/keeper.go#L22-L24
```

Để chỉ kiểm thử `x/gov`, chúng ta mock các [expected keeper](https://docs.cosmos.network/v0.46/building-modules/keeper.html#type-definition) và khởi tạo `Keeper` với các phụ thuộc được mock. Lưu ý rằng chúng ta có thể cần cấu hình các phụ thuộc được mock để trả về các giá trị mong đợi:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/x/gov/keeper/common_test.go#L68-L82
```

Điều này cho phép chúng ta kiểm thử module `x/gov` mà không cần import các module khác.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/x/gov/keeper/keeper_test.go#L3-L42
```

Sau đó chúng ta có thể tạo các unit test sử dụng phiên bản `Keeper` mới được tạo.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/x/gov/keeper/keeper_test.go#L83-L107
```

## Integration Tests (Kiểm Thử Tích Hợp)

Integration test ở cấp độ thứ hai của [kim tự tháp kiểm thử](https://martinfowler.com/articles/practical-test-pyramid.html). Trong SDK, chúng ta đặt các integration test dưới [`/tests/integrations`](https://github.com/cosmos/cosmos-sdk/tree/main/tests/integration).

Mục tiêu của các integration test này là kiểm thử cách một thành phần tương tác với các phụ thuộc khác. So với unit test, integration test không mock phụ thuộc. Thay vào đó, chúng sử dụng các phụ thuộc trực tiếp của thành phần. Điều này khác với end-to-end test, kiểm thử thành phần với một ứng dụng đầy đủ.

Integration test tương tác với module được kiểm thử thông qua các `Msg` và `Query` service được định nghĩa. Kết quả của test có thể được xác minh bằng cách kiểm tra trạng thái của ứng dụng, kiểm tra các sự kiện được phát ra hoặc phản hồi. Khuyến nghị kết hợp hai trong số các phương pháp này để xác minh kết quả của test.

SDK cung cấp các helper nhỏ để nhanh chóng thiết lập một integration test. Các helper này có thể tìm thấy tại <https://github.com/cosmos/cosmos-sdk/blob/main/testutil/integration>.

### Ví Dụ

```go reference
https://github.com/cosmos/cosmos-sdk/blob/a2f73a7dd37bea0ab303792c55fa1e4e1db3b898/testutil/integration/example_test.go#L30-L116
```

## Deterministic và Regression Tests (Kiểm Thử Xác Định và Hồi Quy)

Các test được viết cho các query trong Cosmos SDK có annotation Protobuf `module_query_safe`.

Mỗi query được kiểm thử bằng 2 phương pháp:

* Sử dụng property-based testing với thư viện [`rapid`](https://pkg.go.dev/pgregory.net/rapid@v0.5.3). Thuộc tính được kiểm thử là phản hồi query và mức tiêu thụ gas giống nhau sau 1000 lần gọi query.
* Regression test được viết với các phản hồi và gas hardcode, và xác minh chúng không thay đổi sau 1000 lần gọi và giữa các phiên bản SDK patch.

Dưới đây là ví dụ về regression test:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/tests/integration/bank/keeper/deterministic_test.go#L143-L160
```

## Simulations (Mô Phỏng)

Simulation cũng sử dụng một ứng dụng tối thiểu, được xây dựng với [`depinject`](../packages/01-depinject.md):

:::note
Bạn cũng có thể sử dụng `AppConfig` `configurator` để tạo `AppConfig` [inline](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/x/slashing/app_test.go#L54-L62). Không có sự khác biệt giữa hai cách này, hãy sử dụng cách nào bạn thích.
:::

Dưới đây là ví dụ cho simulation `x/gov/`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/x/gov/simulation/operations_test.go#L415-L441
```

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/x/gov/simulation/operations_test.go#L94-L136
```

## End-to-end Tests (Kiểm Thử Đầu Cuối)

End-to-end test ở đỉnh của [kim tự tháp kiểm thử](https://martinfowler.com/articles/practical-test-pyramid.html). Chúng phải kiểm thử toàn bộ luồng ứng dụng, từ góc nhìn người dùng (ví dụ: CLI test). Chúng nằm dưới [`/tests/e2e`](https://github.com/cosmos/cosmos-sdk/tree/main/tests/e2e).

Với mục đích đó, SDK đang sử dụng `simapp` nhưng bạn nên sử dụng ứng dụng của riêng bạn (`appd`). Dưới đây là một số ví dụ:

* SDK E2E tests: <https://github.com/cosmos/cosmos-sdk/tree/main/tests/e2e>.
* Cosmos Hub E2E tests: <https://github.com/cosmos/gaia/tree/main/tests/e2e>.
* Osmosis E2E tests: <https://github.com/osmosis-labs/osmosis/tree/main/tests/e2e>.

:::note warning
SDK đang trong quá trình tạo các E2E test, như được định nghĩa trong [ADR-59](https://docs.cosmos.network/main/build/architecture/adr-059-test-scopes). Trang này cuối cùng sẽ được cập nhật với các ví dụ tốt hơn.
:::

## Tìm Hiểu Thêm

Tìm hiểu thêm về phạm vi kiểm thử trong [ADR-59](https://docs.cosmos.network/main/build/architecture/adr-059-test-scopes).
