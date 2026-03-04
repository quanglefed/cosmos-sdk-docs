---
sidebar_position: 1
---

# Module Manager

:::note Tóm tắt
Các Cosmos SDK module cần triển khai [`AppModule` interfaces](#application-module-interfaces) để được quản lý bởi [module manager](#module-manager) của ứng dụng. Module manager đóng vai trò quan trọng trong [định tuyến `message` và `query`](../../learn/advanced/00-baseapp.md#routing), và cho phép nhà phát triển ứng dụng thiết lập thứ tự thực thi của các hàm như [`PreBlocker`](../../learn/beginner/00-app-anatomy.md#preblocker) và [`BeginBlocker` và `EndBlocker`](../../learn/beginner/00-app-anatomy.md#beginblocker-and-endblocker).
:::

:::note Yêu Cầu Đọc Trước

* [Giới thiệu về Cosmos SDK Modules](./00-intro.md)

:::

## Các Interface của Application Module

Các interface của application module tồn tại để tạo điều kiện kết hợp các module lại với nhau để tạo thành một ứng dụng Cosmos SDK có chức năng.

:::note

Khuyến nghị triển khai các interface từ gói `appmodule` của [Core API](https://docs.cosmos.network/main/architecture/adr-063-core-module-api). Điều này làm cho các module ít phụ thuộc hơn vào SDK.
Vì lý do kế thừa, các module vẫn có thể triển khai các interface từ gói `module` của SDK.
:::

Có 2 interface application module chính:

* [`appmodule.AppModule` / `module.AppModule`](#appmodule) cho các chức năng module phụ thuộc lẫn nhau (ngoại trừ các chức năng liên quan đến genesis).
* (kế thừa) [`module.AppModuleBasic`](#appmodulebasic) cho các chức năng module độc lập. Các module mới có thể sử dụng `module.CoreAppModuleBasicAdaptor` thay thế.

Các interface trên chủ yếu nhúng các interface nhỏ hơn (extension interface), định nghĩa các chức năng cụ thể:

* (kế thừa) `module.HasName`: Cho phép module cung cấp tên của chính nó cho mục đích kế thừa.
* (kế thừa) [`module.HasGenesisBasics`](#modulehasgenesisbasics): Interface kế thừa cho các phương thức genesis không trạng thái.
* [`module.HasGenesis`](#modulehasgenesis) cho các chức năng module liên quan đến genesis phụ thuộc lẫn nhau.
* [`module.HasABCIGenesis`](#modulehasabcigenesis) cho các chức năng module liên quan đến genesis phụ thuộc lẫn nhau.
* [`appmodule.HasGenesis` / `module.HasGenesis`](#appmodulehasgenesis): Extension interface cho các phương thức genesis có trạng thái.
* [`appmodule.HasPreBlocker`](#haspreblocker): Extension interface chứa thông tin về `AppModule` và `PreBlock`.
* [`appmodule.HasBeginBlocker`](#hasbeginblocker): Extension interface chứa thông tin về `AppModule` và `BeginBlock`.
* [`appmodule.HasEndBlocker`](#hasendblocker): Extension interface chứa thông tin về `AppModule` và `EndBlock`.
* [`appmodule.HasPrecommit`](#hasprecommit): Extension interface chứa thông tin về `AppModule` và `Precommit`.
* [`appmodule.HasPrepareCheckState`](#haspreparecheckstate): Extension interface chứa thông tin về `AppModule` và `PrepareCheckState`.
* [`appmodule.HasService` / `module.HasServices`](#hasservices): Extension interface cho các module đăng ký services.
* [`module.HasABCIEndBlock`](#hasabciendblock): Extension interface chứa thông tin về `AppModule`, `EndBlock` và trả về bộ validator đã cập nhật.
* (kế thừa) [`module.HasInvariants`](#hasinvariants): Extension interface để đăng ký invariants.
* (kế thừa) [`module.HasConsensusVersion`](#hasconsensusversion): Extension interface để khai báo phiên bản consensus của module.

Interface `AppModuleBasic` tồn tại để định nghĩa các phương thức độc lập của module, tức là những phương thức không phụ thuộc vào các module khác trong ứng dụng. Điều này cho phép xây dựng cấu trúc ứng dụng cơ bản từ sớm trong định nghĩa ứng dụng, thường trong hàm `init()` của [file ứng dụng chính](../../learn/beginner/00-app-anatomy.md#core-application-file).

Interface `AppModule` tồn tại để định nghĩa các phương thức module phụ thuộc lẫn nhau. Nhiều module cần tương tác với các module khác, thường thông qua [`keeper`](./06-keeper.md), có nghĩa là cần một interface nơi các module liệt kê các `keeper` và các phương thức khác yêu cầu tham chiếu đến đối tượng của module khác. Extension interface của `AppModule`, như `HasBeginBlocker` và `HasEndBlocker`, cũng cho phép module manager thiết lập thứ tự thực thi giữa các phương thức của module như `BeginBlock` và `EndBlock`, điều này quan trọng trong các trường hợp thứ tự thực thi giữa các module có ý nghĩa trong ngữ cảnh của ứng dụng.

Việc sử dụng extension interface cho phép các module chỉ định nghĩa các chức năng họ cần. Ví dụ, một module không cần `EndBlock` thì không cần định nghĩa interface `HasEndBlocker` và do đó không cần phương thức `EndBlock`. `AppModule` và `AppModuleGenesis` là các interface cố tình được giữ nhỏ gọn, có thể tận dụng các mẫu `Module` mà không phải định nghĩa nhiều hàm giữ chỗ.

### `AppModuleBasic`

:::note
Sử dụng `module.CoreAppModuleBasicAdaptor` thay thế để tạo một `AppModuleBasic` từ một `appmodule.AppModule`.
:::

Interface `AppModuleBasic` định nghĩa các phương thức độc lập mà các module cần triển khai.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/types/module/module.go#L56-L61
```

* `RegisterLegacyAminoCodec(*codec.LegacyAmino)`: Đăng ký `amino` codec cho module, dùng để marshal và unmarshal các struct sang/từ `[]byte` để lưu trữ trong `KVStore` của module.
* `RegisterInterfaces(codectypes.InterfaceRegistry)`: Đăng ký các loại interface của module và các triển khai cụ thể của chúng dưới dạng `proto.Message`.
* `RegisterGRPCGatewayRoutes(client.Context, *runtime.ServeMux)`: Đăng ký các route gRPC cho module.

Tất cả `AppModuleBasic` của một ứng dụng được quản lý bởi [`BasicManager`](#basicmanager).

### `HasName`

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/types/module/module.go#L66-L68
```

* `HasName` là một interface có phương thức `Name()`. Phương thức này trả về tên của module dưới dạng `string`.

### Genesis

:::tip
Để dễ dàng tạo một `AppModule` chỉ có chức năng genesis, sử dụng `module.GenesisOnlyAppModule`.
:::

#### `module.HasGenesisBasics`

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/types/module/module.go#L71-L74
```

Hãy xem qua các phương thức:

* `DefaultGenesis(codec.JSONCodec)`: Trả về một [`GenesisState`](./08-genesis.md#genesisstate) mặc định cho module, được marshal sang `json.RawMessage`. `GenesisState` mặc định cần được định nghĩa bởi nhà phát triển module và chủ yếu được sử dụng cho mục đích kiểm thử.
* `ValidateGenesis(codec.JSONCodec, client.TxEncodingConfig, json.RawMessage)`: Dùng để xác thực `GenesisState` được định nghĩa bởi một module, được cung cấp ở dạng `json.RawMessage`. Thường sẽ unmarshal `json` trước khi chạy hàm [`ValidateGenesis`](./08-genesis.md#validategenesis) tùy chỉnh do nhà phát triển module định nghĩa.

#### `module.HasGenesis`

`HasGenesis` là extension interface cho phép các module triển khai các chức năng genesis.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/6ce2505/types/module/module.go#L184-L189
```

#### `module.HasABCIGenesis`

`HasABCIGenesis` là extension interface cho phép các module triển khai các chức năng genesis và trả về các bản cập nhật bộ validator.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/6ce2505/types/module/module.go#L191-L196
```

#### `appmodule.HasGenesis`

:::warning
`appmodule.HasGenesis` đang trong giai đoạn thử nghiệm và nên được coi là không ổn định, khuyến nghị không sử dụng interface này vào lúc này.
:::

```go reference
https://github.com/cosmos/cosmos-sdk/blob/6ce2505/core/appmodule/genesis.go#L8-L25
```

### `AppModule`

Interface `AppModule` định nghĩa một module. Các module có thể khai báo chức năng của chúng bằng cách triển khai các extension interface. Các `AppModule` được quản lý bởi [module manager](#manager), kiểm tra xem module triển khai những extension interface nào.

#### `appmodule.AppModule`

```go reference
https://github.com/cosmos/cosmos-sdk/blob/6afece6/core/appmodule/module.go#L11-L20
```

#### `module.AppModule`

:::note
Trước đây interface `module.AppModule` chứa tất cả các phương thức được định nghĩa trong các extension interface. Điều này dẫn đến nhiều boilerplate cho các module không cần tất cả các chức năng.
:::

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/types/module/module.go#L199-L206
```

### `HasInvariants`

Interface này định nghĩa một phương thức. Nó cho phép kiểm tra xem một module có thể đăng ký invariants hay không.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/types/module/module.go#L211-L214
```

* `RegisterInvariants(sdk.InvariantRegistry)`: Đăng ký các [`invariant`](./07-invariants.md) của module. Nếu một invariant lệch khỏi giá trị dự đoán, [`InvariantRegistry`](./07-invariants.md#registry) sẽ kích hoạt logic thích hợp (thường xuyên nhất là chain sẽ bị dừng).

### `HasServices`

Interface này định nghĩa một phương thức. Nó cho phép kiểm tra xem một module có thể đăng ký services hay không.

#### `appmodule.HasService`

```go reference
https://github.com/cosmos/cosmos-sdk/blob/6afece6/core/appmodule/module.go#L22-L40
```

#### `module.HasServices`

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/types/module/module.go#L217-L220
```

* `RegisterServices(Configurator)`: Cho phép một module đăng ký services.

### `HasConsensusVersion`

Interface này định nghĩa một phương thức để kiểm tra phiên bản consensus của module.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/types/module/module.go#L223-L229
```

* `ConsensusVersion() uint64`: Trả về phiên bản consensus của module.

### `HasPreBlocker`

`HasPreBlocker` là extension interface từ `appmodule.AppModule`. Tất cả các module có phương thức `PreBlock` đều triển khai interface này.

### `HasBeginBlocker`

`HasBeginBlocker` là extension interface từ `appmodule.AppModule`. Tất cả các module có phương thức `BeginBlock` đều triển khai interface này.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/core/appmodule/module.go#L73-L80
```

* `BeginBlock(context.Context) error`: Phương thức này cho nhà phát triển module tùy chọn triển khai logic được tự động kích hoạt vào đầu mỗi block.

### `HasEndBlocker`

`HasEndBlocker` là extension interface từ `appmodule.AppModule`. Tất cả các module có phương thức `EndBlock` đều triển khai interface này. Nếu một module cần trả về các bản cập nhật bộ validator (staking), họ có thể sử dụng `HasABCIEndBlock`.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/core/appmodule/module.go#L83-L89
```

* `EndBlock(context.Context) error`: Phương thức này cho nhà phát triển module tùy chọn triển khai logic được tự động kích hoạt vào cuối mỗi block.

### `HasABCIEndBlock`

`HasABCIEndBlock` là extension interface từ `module.AppModule`. Tất cả các module có `EndBlock` trả về các bản cập nhật bộ validator đều triển khai interface này.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/types/module/module.go#L236-L239
```

* `EndBlock(context.Context) ([]abci.ValidatorUpdate, error)`: Phương thức này cho nhà phát triển module tùy chọn thông báo cho consensus engine bên dưới về các thay đổi bộ validator (ví dụ: module `staking`).

### `HasPrecommit`

`HasPrecommit` là extension interface từ `appmodule.AppModule`. Tất cả các module có phương thức `Precommit` đều triển khai interface này.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/core/appmodule/module.go#L50-L53
```

* `Precommit(context.Context)`: Phương thức này cho nhà phát triển module tùy chọn triển khai logic được tự động kích hoạt trong [`Commit`](../../learn/advanced/00-baseapp.md#commit) của mỗi block bằng cách sử dụng [`finalizeblockstate`](../../learn/advanced/00-baseapp.md#state-updates) của block sắp được commit. Triển khai rỗng nếu không cần kích hoạt logic nào trong `Commit` của mỗi block cho module này.

### `HasPrepareCheckState`

`HasPrepareCheckState` là extension interface từ `appmodule.AppModule`. Tất cả các module có phương thức `PrepareCheckState` đều triển khai interface này.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/core/appmodule/module.go#L44-L47
```

* `PrepareCheckState(context.Context)`: Phương thức này cho nhà phát triển module tùy chọn triển khai logic được tự động kích hoạt trong [`Commit`](../../learn/advanced/00-baseapp.md#commit) của mỗi block bằng cách sử dụng [`checkState`](../../learn/advanced/00-baseapp.md#state-updates) của block tiếp theo. Các ghi vào trạng thái này sẽ có mặt trong [`checkState`](../../learn/advanced/00-baseapp.md#state-updates) của block tiếp theo, và do đó phương thức này có thể được sử dụng để chuẩn bị `checkState` cho block tiếp theo.

### Triển Khai Các Interface của Application Module

Thông thường, các interface của application module được triển khai trong một file gọi là `module.go`, nằm trong thư mục của module (ví dụ: `./x/module/module.go`).

Hầu hết mọi module cần triển khai các interface `AppModuleBasic` và `AppModule`. Nếu module chỉ được sử dụng cho genesis, nó sẽ triển khai `AppModuleGenesis` thay vì `AppModule`. Kiểu cụ thể triển khai interface có thể thêm các tham số cần thiết cho việc triển khai các phương thức khác nhau của interface. Ví dụ, hàm `Route()` thường gọi hàm `NewMsgServerImpl(k keeper)` được định nghĩa trong `keeper/msg_server.go` và do đó cần truyền [`keeper`](./06-keeper.md) của module làm tham số.

```go
// ví dụ
type AppModule struct {
    AppModuleBasic
    keeper       Keeper
}
```

Trong ví dụ trên, bạn có thể thấy rằng kiểu cụ thể `AppModule` tham chiếu đến một `AppModuleBasic`, chứ không phải `AppModuleGenesis`. Đó là vì `AppModuleGenesis` chỉ cần được triển khai trong các module tập trung vào các chức năng liên quan đến genesis. Trong hầu hết các module, kiểu cụ thể `AppModule` sẽ có tham chiếu đến `AppModuleBasic` và triển khai trực tiếp hai phương thức bổ sung của `AppModuleGenesis` trong kiểu `AppModule`.

Nếu không cần tham số (thường là trường hợp với `AppModuleBasic`), chỉ cần khai báo một kiểu cụ thể rỗng như sau:

```go
type AppModuleBasic struct{}
```

## Module Managers

Module managers được sử dụng để quản lý các tập hợp `AppModuleBasic` và `AppModule`.

### `BasicManager`

`BasicManager` là một cấu trúc liệt kê tất cả `AppModuleBasic` của một ứng dụng:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/types/module/module.go#L77
```

Nó triển khai các phương thức sau:

* `NewBasicManager(modules ...AppModuleBasic)`: Hàm constructor. Nhận một danh sách `AppModuleBasic` của ứng dụng và xây dựng một `BasicManager` mới. Hàm này thường được gọi trong hàm `init()` của [`app.go`](../../learn/beginner/00-app-anatomy.md#core-application-file) để nhanh chóng khởi tạo các phần tử độc lập của các module của ứng dụng.
* `NewBasicManagerFromManager(manager *Manager, customModuleBasics map[string]AppModuleBasic)`: Hàm constructor. Tạo một `BasicManager` mới từ một `Manager`. `BasicManager` sẽ chứa tất cả `AppModuleBasic` từ `AppModule` manager bằng cách sử dụng `CoreAppModuleBasicAdaptor` bất cứ khi nào có thể. `AppModuleBasic` của module có thể được ghi đè bằng cách truyền một map AppModuleBasic tùy chỉnh.
* `RegisterLegacyAminoCodec(cdc *codec.LegacyAmino)`: Đăng ký các [`codec.LegacyAmino`](../../learn/advanced/05-encoding.md#amino) của từng `AppModuleBasic` của ứng dụng. Hàm này thường được gọi sớm trong quá trình [xây dựng ứng dụng](../../learn/beginner/00-app-anatomy.md#constructor).
* `RegisterInterfaces(registry codectypes.InterfaceRegistry)`: Đăng ký các loại interface và triển khai của từng `AppModuleBasic` của ứng dụng.
* `DefaultGenesis(cdc codec.JSONCodec)`: Cung cấp thông tin genesis mặc định cho các module trong ứng dụng bằng cách gọi hàm [`DefaultGenesis(cdc codec.JSONCodec)`](./08-genesis.md#defaultgenesis) của từng module. Chỉ gọi các module triển khai interface `HasGenesisBasics`.
* `ValidateGenesis(cdc codec.JSONCodec, txEncCfg client.TxEncodingConfig, genesis map[string]json.RawMessage)`: Xác thực thông tin genesis của các module bằng cách gọi hàm [`ValidateGenesis(codec.JSONCodec, client.TxEncodingConfig, json.RawMessage)`](./08-genesis.md#validategenesis) của các module triển khai interface `HasGenesisBasics`.
* `RegisterGRPCGatewayRoutes(clientCtx client.Context, rtr *runtime.ServeMux)`: Đăng ký các route gRPC cho các module.
* `AddTxCommands(rootTxCmd *cobra.Command)`: Thêm các lệnh giao dịch của modules (được định nghĩa là `GetTxCmd() *cobra.Command`) vào [`rootTxCommand`](../../learn/advanced/07-cli.md#transaction-commands) của ứng dụng.
* `AddQueryCommands(rootQueryCmd *cobra.Command)`: Thêm các lệnh query của modules (được định nghĩa là `GetQueryCmd() *cobra.Command`) vào [`rootQueryCommand`](../../learn/advanced/07-cli.md#query-commands) của ứng dụng.

### `Manager`

`Manager` là một cấu trúc chứa tất cả `AppModule` của một ứng dụng, và định nghĩa thứ tự thực thi giữa một số thành phần chính của các module này:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/types/module/module.go#L278-L288
```

Module manager được sử dụng xuyên suốt ứng dụng bất cứ khi nào cần thực hiện một hành động trên tập hợp các module. Nó triển khai các phương thức sau:

* `NewManager(modules ...AppModule)`: Hàm constructor. Nhận một danh sách các `AppModule` của ứng dụng và xây dựng một `Manager` mới. Thường được gọi từ hàm [constructor chính](../../learn/beginner/00-app-anatomy.md#constructor-function) của ứng dụng.
* `SetOrderInitGenesis(moduleNames ...string)`: Thiết lập thứ tự mà hàm [`InitGenesis`](./08-genesis.md#initgenesis) của mỗi module sẽ được gọi khi ứng dụng khởi động lần đầu. Để khởi tạo modules thành công, cần xem xét các phụ thuộc giữa module. Ví dụ, module `genutil` phải xảy ra sau module `staking` để các pool được khởi tạo đúng cách với token từ các tài khoản genesis.
* `SetOrderExportGenesis(moduleNames ...string)`: Thiết lập thứ tự mà hàm [`ExportGenesis`](./08-genesis.md#exportgenesis) của mỗi module sẽ được gọi trong trường hợp export.
* `SetOrderPreBlockers(moduleNames ...string)`: Thiết lập thứ tự mà hàm `PreBlock()` của mỗi module sẽ được gọi trước `BeginBlock()` của tất cả các module.
* `SetOrderBeginBlockers(moduleNames ...string)`: Thiết lập thứ tự mà hàm `BeginBlock()` của mỗi module sẽ được gọi vào đầu mỗi block.
* `SetOrderEndBlockers(moduleNames ...string)`: Thiết lập thứ tự mà hàm `EndBlock()` của mỗi module sẽ được gọi vào cuối mỗi block.
* `SetOrderPrecommiters(moduleNames ...string)`: Thiết lập thứ tự mà hàm `Precommit()` của mỗi module sẽ được gọi trong quá trình commit của mỗi block.
* `SetOrderPrepareCheckStaters(moduleNames ...string)`: Thiết lập thứ tự mà hàm `PrepareCheckState()` của mỗi module sẽ được gọi trong quá trình commit của mỗi block.
* `SetOrderMigrations(moduleNames ...string)`: Thiết lập thứ tự các migration cần chạy. Nếu không được đặt thì migration sẽ chạy theo thứ tự được định nghĩa trong `DefaultMigrationsOrder`.
* `RegisterInvariants(ir sdk.InvariantRegistry)`: Đăng ký các [invariant](./07-invariants.md) của module triển khai interface `HasInvariants`.
* `RegisterServices(cfg Configurator)`: Đăng ký các service của các module triển khai interface `HasServices`.
* `InitGenesis(ctx context.Context, cdc codec.JSONCodec, genesisData map[string]json.RawMessage)`: Gọi hàm [`InitGenesis`](./08-genesis.md#initgenesis) của mỗi module khi ứng dụng khởi động lần đầu, theo thứ tự được định nghĩa trong `OrderInitGenesis`. Trả về một `abci.InitChainResponse` cho consensus engine bên dưới, có thể chứa các bản cập nhật validator.
* `ExportGenesis(ctx context.Context, cdc codec.JSONCodec)`: Gọi hàm [`ExportGenesis`](./08-genesis.md#exportgenesis) của mỗi module theo thứ tự được định nghĩa trong `OrderExportGenesis`. Export tạo ra một file genesis từ trạng thái đã tồn tại trước đó, chủ yếu được sử dụng khi cần nâng cấp hard-fork của chain.
* `ExportGenesisForModules(ctx context.Context, cdc codec.JSONCodec, modulesToExport []string)`: Hoạt động giống như `ExportGenesis`, ngoại trừ nhận một danh sách các module cần export.
* `BeginBlock(ctx context.Context) error`: Vào đầu mỗi block, hàm này được gọi từ [`BaseApp`](../../learn/advanced/00-baseapp.md#beginblock) và lần lượt gọi hàm [`BeginBlock`](./06-beginblock-endblock.md) của mỗi module triển khai interface `appmodule.HasBeginBlocker`, theo thứ tự được định nghĩa trong `OrderBeginBlockers`.
* `EndBlock(ctx context.Context) error`: Vào cuối mỗi block, hàm này được gọi từ [`BaseApp`](../../learn/advanced/00-baseapp.md#endblock) và lần lượt gọi hàm [`EndBlock`](./06-beginblock-endblock.md) của mỗi module triển khai interface `appmodule.HasEndBlocker`, theo thứ tự được định nghĩa trong `OrderEndBlockers`.
* `EndBlock(context.Context) ([]abci.ValidatorUpdate, error)`: Vào cuối mỗi block, hàm này được gọi từ [`BaseApp`](../../learn/advanced/00-baseapp.md#endblock) và lần lượt gọi hàm [`EndBlock`](./06-beginblock-endblock.md) của mỗi module triển khai interface `module.HasABCIEndBlock`, theo thứ tự được định nghĩa trong `OrderEndBlockers`. Hàm trả về một `abci` chứa các sự kiện đã đề cập, cũng như các bản cập nhật bộ validator (nếu có).
* `Precommit(ctx context.Context)`: Trong [`Commit`](../../learn/advanced/00-baseapp.md#commit), hàm này được gọi từ `BaseApp` ngay trước khi [`deliverState`](../../learn/advanced/00-baseapp.md#state-updates) được ghi vào [`rootMultiStore`](../../learn/advanced/04-store.md#commitmultistore) bên dưới, và lần lượt gọi hàm `Precommit` của mỗi module triển khai interface `HasPrecommit`, theo thứ tự được định nghĩa trong `OrderPrecommiters`.
* `PrepareCheckState(ctx context.Context)`: Trong [`Commit`](../../learn/advanced/00-baseapp.md#commit), hàm này được gọi từ `BaseApp` ngay sau khi [`deliverState`](../../learn/advanced/00-baseapp.md#state-updates) được ghi vào [`rootMultiStore`](../../learn/advanced/04-store.md#commitmultistore) bên dưới, và lần lượt gọi hàm `PrepareCheckState` của mỗi module triển khai interface `HasPrepareCheckState`, theo thứ tự được định nghĩa trong `OrderPrepareCheckStaters`.

Dưới đây là ví dụ về tích hợp cụ thể trong `simapp`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/simapp/app.go#L510-L533
```

Đây là ví dụ tương tự từ `runtime` (gói cung cấp sức mạnh cho app di):

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/runtime/module.go#L63
```

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/runtime/module.go#L85
```
