---
sidebar_position: 1
---

# Module Genesis

:::note Tóm tắt
Các module thường xử lý một tập con trạng thái và do đó, chúng cần định nghĩa tập con liên quan trong genesis file cũng như các phương thức để khởi tạo, xác thực và xuất nó.
:::

:::note Yêu Cầu Đọc Trước

* [Module Manager](./01-module-manager.md)
* [Keepers](./06-keeper.md)

:::

## Định Nghĩa Kiểu

Tập con trạng thái genesis được định nghĩa bởi một module nhất định thường được định nghĩa trong file `genesis.proto` ([thêm thông tin](../../learn/advanced/05-encoding.md#gogoproto) về cách định nghĩa các protobuf message). Struct định nghĩa tập con trạng thái genesis của module thường được gọi là `GenesisState` và chứa tất cả các giá trị liên quan đến module cần được khởi tạo trong quá trình genesis.

Xem ví dụ về định nghĩa protobuf message `GenesisState` từ module `auth`:

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/proto/cosmos/auth/v1beta1/genesis.proto
```

Tiếp theo chúng ta trình bày các phương thức chính liên quan đến genesis mà nhà phát triển module cần triển khai để module của họ có thể được sử dụng trong các ứng dụng Cosmos SDK.

### `DefaultGenesis`

Phương thức `DefaultGenesis()` là một hàm đơn giản gọi hàm constructor của `GenesisState` với giá trị mặc định cho mỗi tham số. Xem ví dụ từ module `auth`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/auth/module.go#L63-L67
```

### `ValidateGenesis`

Phương thức `ValidateGenesis(data GenesisState)` được gọi để xác minh rằng `genesisState` được cung cấp là đúng. Nó nên thực hiện các kiểm tra hợp lệ trên từng tham số được liệt kê trong `GenesisState`. Xem ví dụ từ module `auth`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/auth/types/genesis.go#L62-L75
```

## Các Phương Thức Genesis Khác

Ngoài các phương thức liên quan trực tiếp đến `GenesisState`, nhà phát triển module dự kiến phải triển khai hai phương thức khác như một phần của [`AppModuleGenesis` interface](./01-module-manager.md#appmodulegenesis) (chỉ khi module cần khởi tạo một tập con trạng thái trong genesis). Các phương thức này là [`InitGenesis`](#initgenesis) và [`ExportGenesis`](#exportgenesis).

### `InitGenesis`

Phương thức `InitGenesis` được thực thi trong [`InitChain`](../../learn/advanced/00-baseapp.md#initchain) khi ứng dụng khởi động lần đầu. Với một `GenesisState`, nó khởi tạo tập con trạng thái được quản lý bởi module bằng cách sử dụng hàm setter của [`keeper`](./06-keeper.md) của module trên mỗi tham số trong `GenesisState`.

[Module manager](./01-module-manager.md#manager) của ứng dụng chịu trách nhiệm gọi phương thức `InitGenesis` của từng module trong ứng dụng theo thứ tự. Thứ tự này được thiết lập bởi nhà phát triển ứng dụng thông qua `SetOrderGenesisMethod` của manager, được gọi trong [hàm constructor của ứng dụng](../../learn/beginner/00-app-anatomy.md#constructor-function).

Xem ví dụ về `InitGenesis` từ module `auth`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/auth/keeper/genesis.go#L8-L35
```

### `ExportGenesis`

Phương thức `ExportGenesis` được thực thi bất cứ khi nào có export trạng thái. Nó lấy phiên bản mới nhất đã biết của tập con trạng thái được quản lý bởi module và tạo ra một `GenesisState` mới từ đó. Điều này chủ yếu được sử dụng khi chain cần được nâng cấp thông qua hard fork.

Xem ví dụ về `ExportGenesis` từ module `auth`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/auth/keeper/genesis.go#L37-L49
```

### GenesisTxHandler

`GenesisTxHandler` là cách cho các module gửi các chuyển đổi trạng thái trước block đầu tiên. Điều này được sử dụng bởi `x/genutil` để gửi các genesis transaction cho các validator được thêm vào staking.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/core/genesis/txhandler.go#L3-L6
```
