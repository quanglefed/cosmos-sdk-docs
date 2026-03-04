---
sidebar_position: 1
---

# `runtime` Là Gì?

Package `runtime` trong Cosmos SDK cung cấp một framework linh hoạt để cấu hình và quản lý các ứng dụng blockchain. Nó đóng vai trò là nền tảng để tạo ra các ứng dụng blockchain theo dạng module bằng cách sử dụng phương pháp cấu hình khai báo.

## Tổng Quan

Package runtime hoạt động như một wrapper bao quanh `BaseApp` và `ModuleManager`, cung cấp một phương pháp kết hợp trong đó các ứng dụng có thể được cấu hình cả theo cách khai báo thông qua file cấu hình lẫn theo cách lập trình thông qua các phương thức truyền thống. Đây là một lớp trừu tượng giữa `baseapp` và các module ứng dụng, đơn giản hóa quá trình xây dựng ứng dụng Cosmos SDK.

## Các Thành Phần Cốt Lõi

### Cấu Trúc App

Struct App của runtime chứa một số thành phần chính:

```go
type App struct {
    *baseapp.BaseApp
    ModuleManager    *module.Manager
    configurator     module.Configurator
    config           *runtimev1alpha1.Module
    storeKeys        []storetypes.StoreKey
    // ... các trường khác
}
```

Các ứng dụng Cosmos SDK nên nhúng struct `*runtime.App` để tận dụng module runtime.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/simapp/app_di.go#L60-L61
```

### Cấu Hình

Module runtime được cấu hình bằng App Wiring. Đối tượng cấu hình chính là [message `Module`](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/proto/cosmos/app/runtime/v1alpha1/module.proto), hỗ trợ các cài đặt chính sau:

* `app_name`: Tên của ứng dụng
* `begin_blockers`: Danh sách tên module để gọi trong BeginBlock
* `end_blockers`: Danh sách tên module để gọi trong EndBlock
* `init_genesis`: Thứ tự khởi tạo module trong quá trình genesis
* `export_genesis`: Thứ tự để xuất dữ liệu genesis của module
* `pre_blockers`: Các module để thực thi trước khi xử lý block

Tìm hiểu thêm về wiring `runtime` trong [phần tiếp theo](./01-app-go-di.md).

#### Cấu Hình Store

Theo mặc định, module runtime sử dụng tên module làm store key. Tuy nhiên, nó cung cấp cấu hình store key linh hoạt thông qua:

* `override_store_keys`: Cho phép tùy chỉnh store key của module
* `skip_store_keys`: Chỉ định các store key cần bỏ qua trong quá trình xây dựng keeper

Ví dụ cấu hình:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/simapp/app_config.go#L133-L138
```

## Các Tính Năng Chính

### 1. Tích Hợp BaseApp và Các Thành Phần Cốt Lõi SDK

Module runtime tích hợp với `BaseApp` và các thành phần cốt lõi SDK khác để cung cấp trải nghiệm liền mạch cho các nhà phát triển.

Nhà phát triển chỉ cần nhúng struct `runtime.App` vào ứng dụng của mình để tận dụng module runtime. Việc cấu hình module manager và các thành phần cốt lõi khác được xử lý nội bộ qua [`AppBuilder`](#4-xây-dựng-ứng-dụng).

### 2. Đăng Ký Module

Runtime có hỗ trợ tích hợp sẵn cho [các module hỗ trợ `depinject`](../building-modules/15-depinject.md). Các module như vậy có thể được đăng ký thông qua file cấu hình (thường được đặt tên là `app_config.go`), không cần code bổ sung.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/simapp/app_config.go#L210-L216
```

Ngoài ra, package runtime tạo điều kiện cho việc đăng ký module thủ công thông qua phương thức `RegisterModules`. Đây là điểm tích hợp chính cho các module không được đăng ký qua cấu hình.

:::warning
Ngay cả khi sử dụng đăng ký thủ công, module vẫn nên được cấu hình trong message `Module` trong AppConfig.
:::

```go
func (a *App) RegisterModules(modules ...module.AppModule) error
```

SDK khuyến nghị sử dụng cách tiếp cận khai báo với `depinject` để đăng ký module bất cứ khi nào có thể.

### 3. Đăng Ký Dịch Vụ

Runtime đăng ký tất cả [các dịch vụ cốt lõi](https://pkg.go.dev/cosmossdk.io/core) cần thiết bởi các module. Các dịch vụ này bao gồm `store`, `event manager`, `context` và `logger`. Runtime đảm bảo rằng các dịch vụ được phân phạm vi cho các module tương ứng trong quá trình wiring.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/runtime/module.go#L201-L235
```

Ngoài ra, runtime cung cấp đăng ký tự động các dịch vụ thiết yếu khác (tức là các route gRPC) có sẵn cho App:

* Dịch vụ AutoCLI Query
* Dịch vụ Reflection
* Các dịch vụ module tùy chỉnh

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/runtime/builder.go#L52-L54
```

### 4. Xây Dựng Ứng Dụng

Kiểu `AppBuilder` cung cấp một cách có cấu trúc để xây dựng ứng dụng:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/runtime/builder.go#L14-L19
```

Các bước xây dựng chính:

1. Tải cấu hình
2. Đăng ký module
3. Thiết lập dịch vụ
4. Gắn store
5. Cấu hình router

Ứng dụng chỉ cần gọi `AppBuilder.Build` để tạo một ứng dụng được cấu hình đầy đủ (`runtime.App`).

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/runtime/builder.go#L26-L57
```

Thông tin thêm về xây dựng ứng dụng có thể tìm thấy trong [phần tiếp theo](./02-app-building.md).

## Các Thực Hành Tốt Nhất

1. **Thứ Tự Module**: Cân nhắc kỹ thứ tự của các module trong begin_blockers, end_blockers và pre_blockers.
2. **Store Keys**: Chỉ sử dụng override_store_keys khi cần thiết để duy trì sự rõ ràng.
3. **Thứ Tự Genesis**: Duy trì thứ tự khởi tạo đúng trong init_genesis.
4. **Quản Lý Migration**: Sử dụng order_migrations để kiểm soát các đường dẫn nâng cấp.

### Các Cân Nhắc Migration

Khi nâng cấp giữa các phiên bản:

1. Xem xét thứ tự migration được chỉ định trong `order_migrations`.
2. Đảm bảo tất cả các module cần thiết được bao gồm trong cấu hình.
3. Xác thực các cấu hình store key.
4. Kiểm thử kỹ đường dẫn nâng cấp.
