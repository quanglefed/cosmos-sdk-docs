# ADR 057: App Wiring

## Changelog

* 2022-05-04: Bản nháp đầu tiên
* 2022-08-19: Cập nhật

## Trạng Thái

ĐỀ XUẤT Đã Triển Khai

## Tóm Tắt

Để giúp việc xây dựng các module và ứng dụng Cosmos SDK dễ dàng hơn, chúng tôi đề xuất một hệ thống app wiring mới dựa trên dependency injection và cấu hình ứng dụng declarative để thay thế code `app.go` hiện tại.

## Bối Cảnh

Một số yếu tố đã khiến SDK và các ứng dụng SDK ở trạng thái hiện tại khó bảo trì. Triệu chứng của trạng thái phức tạp hiện tại là [`simapp/app.go`](https://github.com/cosmos/cosmos-sdk/blob/c3edbb22cab8678c35e21fe0253919996b780c01/simapp/app.go) chứa gần 100 dòng import và hơn 600 dòng code boilerplate được sao chép vào mỗi dự án mới.

Lượng boilerplate lớn cần để khởi động một ứng dụng đã khiến việc phát hành các go module được phiên bản độc lập cho các module Cosmos SDK gặp khó khăn như mô tả trong [ADR 053: Go Module Refactoring](./adr-053-go-module-refactoring.md).

Ngoài ra, `app.go` cũng phơi bày một bề mặt lớn cho các thay đổi breaking vì hầu hết các module tự khởi tạo với các tham số vị trí, buộc thay đổi breaking bất cứ khi nào một tham số mới được thêm vào.

## Quyết Định

Để cải thiện tình trạng hiện tại, một mô hình "app wiring" mới đã được thiết kế để thay thế `app.go` bao gồm:

* Cấu hình declarative của các module trong ứng dụng có thể được serialize thành JSON hoặc YAML.
* Framework dependency-injection (DI) để khởi tạo ứng dụng từ cấu hình.

### Dependency Injection

Khi xem xét code trong `app.go`, hầu hết code chỉ đơn giản khởi tạo các module với các dependency được cung cấp bởi framework hoặc bởi các module khác. Thay vì yêu cầu nhà phát triển giải quyết dependency thủ công, một module sẽ thông báo cho DI container những dependency mà nó cần và container sẽ tìm cách cung cấp chúng.

Module [depinject](https://pkg.go.dev/github.com/cosmos/cosmos-sdk/depinject) của Cosmos SDK có các tính năng:

* Giải quyết và cung cấp dependency thông qua functional constructor, ví dụ: `func(need SomeDep) (AnotherDep, error)`.
* Cấu trúc `In` và `Out` cho dependency injection hỗ trợ dependency `optional`.
* Grouped-dependencies (nhiều-trên-mỗi-container) thông qua tag interface `ManyPerContainerType`.
* Dependency phạm vi module thông qua `ModuleKey` (mỗi module nhận một dependency duy nhất).
* One-per-module dependencies thông qua tag interface `OnePerModuleType`.

### Cấu Hình Ứng Dụng Declarative

Để kết hợp các module vào ứng dụng, một cấu hình ứng dụng declarative sẽ được sử dụng dựa trên protobuf:

```protobuf
package cosmos.app.v1;

message Config {
  repeated ModuleConfig modules = 1;
}

message ModuleConfig {
  string name = 1;
  google.protobuf.Any config = 2;
}
```

Cấu hình cho mỗi module là một message protobuf. Các module được xác định và tải dựa trên URL type protobuf của đối tượng cấu hình của chúng.

Ví dụ cấu hình ứng dụng YAML:

```yaml
modules:
  - name: baseapp
    config:
      "@type": cosmos.baseapp.module.v1.Module
      begin_blockers: [staking, auth, bank]
      end_blockers: [bank, auth, staking]
      init_genesis: [bank, auth, staking]
  - name: auth
    config:
      "@type": cosmos.auth.module.v1.Module
      bech32_prefix: "foo"
  - name: bank
    config:
      "@type": cosmos.bank.module.v1.Module
  - name: staking
    config:
      "@type": cosmos.staking.module.v1.Module
```

### Đăng Ký Module & Protobuf

Module phải đăng ký bản thân với API sau:

```go
func Register(configTypeName protoreflect.FullName, option ...Option) { ... }

func Provide(providers ...interface{}) Option { ... }
func Types(types ...TypeInfo) Option { ... }
```

Ví dụ:

```go
func init() {
    appmodule.Register("cosmos.bank.module.v1.Module",
        appmodule.Provide(provideBankModule),
    )
}

func ProvideBankModule(config *bankmodulev1.Module, inputs Inputs) (Outputs, error) { ... }
```

### `app.go` Mới

Với thiết lập này, `app.go` có thể trông như sau:

```go
package main

import (
    _ "github.com/cosmos/cosmos-sdk/x/auth/module"
    _ "github.com/cosmos/cosmos-sdk/x/bank/module"
    _ "github.com/cosmos/cosmos-sdk/x/staking/module"
    "github.com/cosmos/cosmos-sdk/core/app"
)

//go:embed app.yaml
var appConfigYAML []byte

func main() {
    app.Run(app.LoadYAML(appConfigYAML))
}
```

### Phiên Bản Semantic Module

Khi tạo các module SDK được phiên bản semantic trong các go module độc lập, thay đổi vi phạm state machine đối với module nên được xử lý như sau:

* Tăng phiên bản major semantic, và
* Tạo một kiểu config protobuf module được phiên bản semantic mới.

Ví dụ, nếu chúng ta có module bank trong go module `github.com/cosmos/cosmos-sdk/x/bank` với kiểu config module `cosmos.bank.module.v1.Module`, và chúng ta muốn thực hiện thay đổi vi phạm state machine:

* Tạo go module mới `github.com/cosmos/cosmos-sdk/x/bank/v2`,
* Với kiểu config protobuf module `cosmos.bank.module.v2.Module`.

## Hậu Quả

### Tương Thích Ngược

Các module hoạt động với hệ thống app wiring mới không cần loại bỏ các mô hình đăng ký `AppModule` và `NewKeeper` hiện có.

### Tích Cực

* Việc kết nối các ứng dụng mới sẽ đơn giản hơn, ngắn gọn hơn và ít lỗi hơn.
* Dễ phát triển và test các module SDK độc lập mà không cần tái tạo simapp.
* Có thể tải động các module và nâng cấp chain mà không cần dừng có phối hợp.
* Framework dependency injection cung cấp lý luận tự động hơn về các dependency trong dự án.

### Tiêu Cực

* Có thể gây nhầm lẫn khi thiếu dependency, mặc dù các thông báo lỗi và trực quan hóa GraphViz có thể giúp ích.

## Tài Liệu Tham Khảo

* https://github.com/cosmos/cosmos-sdk/blob/c3edbb22cab8678c35e21fe0253919996b780c01/simapp/app.go
* https://github.com/uber-go/dig
* [ADR 063: Core Module API](./adr-063-core-module-api.md)
