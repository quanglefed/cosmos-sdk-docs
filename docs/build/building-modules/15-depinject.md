---
sidebar_position: 1
---

# Modules sẵn sàng cho depinject

:::note Yêu Cầu Đọc Trước

* [Tài liệu Depinject](../packages/01-depinject.md)

:::

[`depinject`](../packages/01-depinject.md) được sử dụng để kết nối bất kỳ module nào trong `app.go`. Tất cả các module core đã được cấu hình để hỗ trợ dependency injection.

Để hoạt động với `depinject`, một module phải định nghĩa cấu hình và yêu cầu của nó để `depinject` có thể cung cấp các phụ thuộc đúng.

Tóm lại, là nhà phát triển module, cần thực hiện các bước sau:

1. Định nghĩa cấu hình module bằng Protobuf
2. Định nghĩa các phụ thuộc module trong `x/{moduleName}/module.go`

Nhà phát triển chain sau đó có thể sử dụng module bằng cách thực hiện hai bước sau:

1. Cấu hình module trong `app_config.go` hoặc `app.yaml`
2. Inject module trong `app.go`

## Cấu Hình Module

Cấu hình khả dụng của module được định nghĩa trong file Protobuf, nằm tại `{moduleName}/module/v1/module.proto`.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/proto/cosmos/group/module/v1/module.proto
```

* `go_import` phải trỏ đến gói Go của module tùy chỉnh.
* Các trường message định nghĩa cấu hình module. Cấu hình đó có thể được đặt trong file `app_config.go` / `app.yaml` cho nhà phát triển chain để cấu hình module. Lấy `group` làm ví dụ, nhà phát triển chain có thể quyết định, nhờ vào `uint64 max_metadata_len`, độ dài metadata tối đa được phép cho một group proposal là bao nhiêu.

  ```go reference
  https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/simapp/app_config.go#L228-L234
  ```

Message đó được tạo bằng cách sử dụng [`pulsar`](https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/scripts/protocgen-pulsar.sh) (bằng cách chạy `make proto-gen`). Trong trường hợp module `group`, file này được tạo tại đây: https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/api/cosmos/group/module/v1/module.pulsar.go.

Phần liên quan đến cấu hình module là:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/api/cosmos/group/module/v1/module.pulsar.go#L515-L527
```

:::note
Pulsar là tùy chọn. [`protoc-gen-go`](https://developers.google.com/protocol-buffers/docs/reference/go-generated) chính thức cũng có thể được sử dụng.
:::

## Định Nghĩa Phụ Thuộc

Khi proto cấu hình đã được định nghĩa, `module.go` của module phải định nghĩa các phụ thuộc mà module yêu cầu. Boilerplate tương tự nhau cho tất cả các module.

:::warning
Tất cả các phương thức, struct và các trường của chúng phải là public cho `depinject`.
:::

1. Import gói được tạo từ cấu hình module:

    ```go reference
    https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/group/module/module.go#L12-L14
    ```

    Định nghĩa hàm `init()` để định nghĩa `providers` của cấu hình module: Điều này đăng ký message cấu hình module và việc kết nối của module.

    ```go reference
    https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/group/module/module.go#L194-L199
    ```

2. Đảm bảo rằng module triển khai interface `appmodule.AppModule`:

    ```go reference
    https://github.com/cosmos/cosmos-sdk/blob/v0.47.0/x/group/module/module.go#L58-L64
    ```

3. Định nghĩa một struct nhúng `depinject.In` và định nghĩa các đầu vào của module (tức là các phụ thuộc của module):
   * `depinject` cung cấp các phụ thuộc đúng cho module.
   * `depinject` cũng kiểm tra rằng tất cả các phụ thuộc được cung cấp.

    :::tip
    Để làm cho một phụ thuộc trở thành tùy chọn, thêm struct tag `optional:"true"`.
    :::

    ```go reference
    https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/group/module/module.go#L201-L211
    ```

4. Định nghĩa các đầu ra của module với một struct public nhúng `depinject.Out`: Các đầu ra của module là các phụ thuộc mà module cung cấp cho các module khác. Thường là bản thân module và keeper của nó.

    ```go reference
    https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/group/module/module.go#L213-L218
    ```

5. Tạo một hàm tên `ProvideModule` (như được gọi trong bước 1.) và sử dụng các đầu vào để khởi tạo các đầu ra của module.

  ```go reference
  https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/group/module/module.go#L220-L235
  ```

Hàm `ProvideModule` nên trả về một phiên bản của `cosmossdk.io/core/appmodule.AppModule` triển khai một hoặc nhiều extension interface của app module để khởi tạo module.

Dưới đây là cấu hình app wiring hoàn chỉnh cho `group`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/group/module/module.go#L194-L235
```

Module giờ đây đã sẵn sàng để được sử dụng với `depinject` bởi nhà phát triển chain.

## Tích Hợp vào Ứng Dụng

App Wiring được thực hiện trong `app_config.go` / `app.yaml` và `app_di.go` và được giải thích chi tiết trong [tổng quan về `app_di.go`](../building-apps/01-app-go-di.md).
