---
sidebar_position: 1
---

# Cấu Trúc Của Một Ứng Dụng Cosmos SDK

:::note Tóm tắt
Tài liệu này mô tả các thành phần cốt lõi của một ứng dụng Cosmos SDK, được đề cập xuyên suốt tài liệu dưới dạng một ứng dụng mẫu có tên `app`.
:::

## Node Client

Daemon, hay còn gọi là [Full-Node Client](../advanced/03-node.md), là tiến trình cốt lõi của một blockchain dựa trên Cosmos SDK. Các thành viên tham gia mạng lưới chạy tiến trình này để khởi tạo state-machine của họ, kết nối với các full-node khác, và cập nhật state-machine khi có block mới đến.

```text
                ^  +-------------------------------+  ^
                |  |                               |  |
                |  |  State-machine = Application  |  |
                |  |                               |  |   Xây dựng với Cosmos SDK
                |  |            ^      +           |  |
                |  +----------- | ABCI | ----------+  v
                |  |            +      v           |  ^
                |  |                               |  |
Blockchain Node |  |           Consensus           |  |
                |  |                               |  |
                |  +-------------------------------+  |   CometBFT
                |  |                               |  |
                |  |           Networking          |  |
                |  |                               |  |
                v  +-------------------------------+  v
```

Full-node của blockchain được trình bày dưới dạng một binary, thường có hậu tố `-d` cho "daemon" (ví dụ: `appd` cho `app` hoặc `gaiad` cho `gaia`). Binary này được xây dựng bằng cách chạy một hàm [`main.go`](../advanced/03-node.md#main-function) đơn giản đặt trong `./cmd/appd/`. Thao tác này thường được thực hiện thông qua [Makefile](#dependencies-and-makefile).

Khi binary chính đã được xây dựng, node có thể được khởi động bằng cách chạy [lệnh `start`](../advanced/03-node.md#start-command). Hàm lệnh này chủ yếu thực hiện ba việc:

1. Tạo một instance của state-machine được định nghĩa trong [`app.go`](#core-application-file).
2. Khởi tạo state-machine với trạng thái mới nhất đã biết, được lấy từ `db` lưu trong thư mục `~/.app/data`. Tại thời điểm này, state-machine ở chiều cao `appBlockHeight`.
3. Tạo và khởi động một instance CometBFT mới. Trong số các thứ khác, node thực hiện handshake với các peer. Nó lấy `blockHeight` mới nhất từ chúng và phát lại các block để đồng bộ đến chiều cao đó nếu lớn hơn `appBlockHeight` cục bộ. Node bắt đầu từ genesis và CometBFT gửi thông báo `InitChain` qua ABCI đến `app`, kích hoạt [`InitChainer`](#initchainer).

:::note
Khi khởi động một instance CometBFT, file genesis là chiều cao `0` và trạng thái trong file genesis được commit tại chiều cao block `1`. Khi truy vấn trạng thái của node, truy vấn chiều cao block 0 sẽ trả về lỗi.
:::

## File Ứng Dụng Cốt Lõi

Nói chung, cốt lõi của state-machine được định nghĩa trong một file gọi là `app.go`. File này chủ yếu chứa **định nghĩa kiểu (type) của ứng dụng** và các hàm để **tạo và khởi tạo** nó.

### Định nghĩa kiểu của ứng dụng

Thứ đầu tiên được định nghĩa trong `app.go` là `type` của ứng dụng. Nó thường bao gồm các phần sau:

* **Nhúng [runtime.App](../../build/building-apps/00-runtime.md)**: Package runtime quản lý các thành phần cốt lõi và module của ứng dụng thông qua dependency injection. Nó cung cấp cấu hình khai báo cho quản lý module, lưu trữ trạng thái, và xử lý ABCI.
    * `Runtime` bọc `BaseApp`, nghĩa là khi một giao dịch được CometBFT chuyển tiếp đến ứng dụng, `app` sử dụng các phương thức của `runtime` để định tuyến chúng đến module thích hợp. `BaseApp` triển khai tất cả [ABCI methods](https://docs.cometbft.com/v0.38/spec/abci/) và [routing logic](../advanced/00-baseapp.md#service-routers).
    * Nó tự động cấu hình **[module manager](../../build/building-modules/01-module-manager.md#manager)** dựa trên cấu hình app wiring. Module manager tạo điều kiện cho các hoạt động liên quan đến các module này, như đăng ký [`Msg` service](../../build/building-modules/03-msg-services.md) và [gRPC `Query` service](#grpc-query-services), hoặc đặt thứ tự thực thi giữa các module cho các hàm như [`InitChainer`](#initchainer), [`PreBlocker`](#preblocker) và [`BeginBlocker` và `EndBlocker`](#beginblocker-and-endblocker).
* [**File cấu hình App Wiring**](../../build/building-apps/00-runtime.md): File cấu hình app wiring chứa danh sách các module của ứng dụng mà `runtime` phải khởi tạo. Việc khởi tạo các module được thực hiện bằng `depinject`. Nó cũng chứa thứ tự mà tất cả các phương thức `InitGenesis` và `Pre/Begin/EndBlocker` của các module nên được thực thi.
* **Tham chiếu đến một [`appCodec`](../advanced/05-encoding.md).** `appCodec` của ứng dụng được dùng để tuần tự hóa và giải tuần tự hóa các cấu trúc dữ liệu để lưu trữ, vì store chỉ có thể lưu `[]bytes`. Codec mặc định là [Protocol Buffers](../advanced/05-encoding.md).
* **Tham chiếu đến codec [`legacyAmino`](../advanced/05-encoding.md).** Một số phần của Cosmos SDK chưa được chuyển sang dùng `appCodec` và vẫn được hardcode để dùng Amino. Các phần khác dùng Amino tường minh vì lý do tương thích ngược. Vì những lý do này, ứng dụng vẫn giữ tham chiếu đến codec Amino cũ. Lưu ý rằng codec Amino sẽ bị xóa khỏi SDK trong các phiên bản sắp tới.

Xem ví dụ về định nghĩa kiểu ứng dụng từ `simapp`, ứng dụng của chính Cosmos SDK dùng cho demo và kiểm thử:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/simapp/app_di.go#L57-L90
```

### Hàm Constructor

Cũng được định nghĩa trong `app.go` là hàm constructor, hàm này xây dựng một ứng dụng mới theo kiểu được định nghĩa ở phần trên. Hàm phải thỏa mãn chữ ký `AppCreator` để được sử dụng trong [lệnh `start`](../advanced/03-node.md#start-command) của lệnh daemon của ứng dụng.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/server/types/app.go#L67-L69
```

Dưới đây là các hành động chính được thực hiện bởi hàm này:

* Khởi tạo một [`codec`](../advanced/05-encoding.md) mới và khởi tạo `codec` của mỗi module của ứng dụng bằng cách dùng [basic manager](../../build/building-modules/01-module-manager.md#basicmanager).
* Khởi tạo một ứng dụng mới với tham chiếu đến một instance `baseapp`, một codec, và tất cả các store key phù hợp.
* Khởi tạo tất cả các đối tượng [`keeper`](#keeper) được định nghĩa trong `type` của ứng dụng bằng cách dùng hàm `NewKeeper` của mỗi module. Lưu ý rằng keeper phải được khởi tạo theo đúng thứ tự, vì `NewKeeper` của một module có thể yêu cầu tham chiếu đến `keeper` của module khác.
* Khởi tạo [module manager](../../build/building-modules/01-module-manager.md#manager) của ứng dụng với đối tượng [`AppModule`](#application-module-interface) của mỗi module.
* Với module manager, khởi tạo các [`Msg` service](../advanced/00-baseapp.md#msg-services), [gRPC `Query` service](../advanced/00-baseapp.md#grpc-query-services), [legacy `Msg` route](../advanced/00-baseapp.md#routing) và [legacy query route](../advanced/00-baseapp.md#query-routing) của ứng dụng. Khi một giao dịch được CometBFT chuyển tiếp đến ứng dụng qua ABCI, nó được định tuyến đến [`Msg` service](#msg-services) của module phù hợp sử dụng các route được định nghĩa ở đây. Tương tự, khi một yêu cầu truy vấn gRPC được ứng dụng nhận, nó được định tuyến đến [`gRPC query service`](#grpc-query-services) của module phù hợp.
* Với module manager, đăng ký [các invariant của module ứng dụng](../../build/building-modules/07-invariants.md). Invariant là các biến (ví dụ: tổng cung của một token) được đánh giá vào cuối mỗi block. Quá trình kiểm tra invariant được thực hiện qua một module đặc biệt gọi là [`InvariantsRegistry`](../../build/building-modules/07-invariants.md#invariant-registry). Giá trị của invariant phải bằng một giá trị dự đoán được định nghĩa trong module. Nếu giá trị khác với giá trị dự đoán, logic đặc biệt được định nghĩa trong invariant registry sẽ được kích hoạt (thường là chuỗi bị dừng).
* Với module manager, đặt thứ tự thực thi giữa các hàm `InitGenesis`, `PreBlocker`, `BeginBlocker` và `EndBlocker` của mỗi [module của ứng dụng](#application-module-interface). Lưu ý rằng không phải tất cả module đều triển khai các hàm này.
* Đặt các tham số ứng dụng còn lại:
    * [`InitChainer`](#initchainer): dùng để khởi tạo ứng dụng khi nó lần đầu được khởi động.
    * [`PreBlocker`](#preblocker): được gọi trước BeginBlock.
    * [`BeginBlocker`, `EndBlocker`](#beginblocker-and-endblocker): được gọi ở đầu và cuối mỗi block.
    * [`anteHandler`](../advanced/00-baseapp.md#antehandler): dùng để xử lý phí và xác minh chữ ký.
* Mount các store.
* Trả về ứng dụng.

Lưu ý rằng hàm constructor chỉ tạo một instance của ứng dụng, trong khi trạng thái thực tế hoặc được mang từ thư mục `~/.app/data` nếu node được khởi động lại, hoặc được tạo từ file genesis nếu node được khởi động lần đầu.

Xem ví dụ về constructor ứng dụng từ `simapp`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/simapp/app.go#L190-L708
```

### InitChainer

`InitChainer` là một hàm khởi tạo trạng thái của ứng dụng từ một file genesis (tức là số dư token của các tài khoản genesis). Nó được gọi khi ứng dụng nhận thông báo `InitChain` từ CometBFT engine, điều này xảy ra khi node được khởi động tại `appBlockHeight == 0` (tức là khi genesis). Ứng dụng phải đặt `InitChainer` trong [constructor](#constructor-function) của nó qua phương thức [`SetInitChainer`](https://pkg.go.dev/github.com/cosmos/cosmos-sdk/baseapp#BaseApp.SetInitChainer).

Nói chung, `InitChainer` chủ yếu bao gồm hàm [`InitGenesis`](../../build/building-modules/08-genesis.md#initgenesis) của mỗi module trong ứng dụng. Điều này được thực hiện bằng cách gọi hàm `InitGenesis` của module manager, hàm này lần lượt gọi `InitGenesis` của mỗi module chứa trong nó. Lưu ý rằng thứ tự gọi hàm `InitGenesis` của các module phải được đặt trong module manager bằng phương thức `SetOrderInitGenesis`. Điều này được thực hiện trong [constructor của ứng dụng](#constructor-function) và `SetOrderInitGenesis` phải được gọi trước `SetInitChainer`.

Xem ví dụ về `InitChainer` từ `simapp`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/simapp/app.go#L765-L773
```

### PreBlocker

Có hai ngữ nghĩa xung quanh phương thức vòng đời mới này:

* Nó chạy trước `BeginBlocker` của tất cả các module.
* Nó có thể sửa đổi các tham số đồng thuận trong storage và thông báo cho caller thông qua giá trị trả về.

Khi nó trả về `ConsensusParamsChanged=true`, caller phải làm mới tham số đồng thuận trong finalize context:

```go
app.finalizeBlockState.ctx = app.finalizeBlockState.ctx.WithConsensusParams(app.GetConsensusParams())
```

Context mới phải được truyền cho tất cả các phương thức vòng đời khác.

### BeginBlocker và EndBlocker

Cosmos SDK cung cấp cho các nhà phát triển khả năng triển khai việc thực thi code tự động như một phần của ứng dụng của họ. Điều này được triển khai qua hai hàm gọi là `BeginBlocker` và `EndBlocker`. Chúng được gọi khi ứng dụng nhận thông báo `FinalizeBlock` từ CometBFT consensus engine, điều này xảy ra tương ứng ở đầu và cuối mỗi block. Ứng dụng phải đặt `BeginBlocker` và `EndBlocker` trong [constructor](#constructor-function) của nó qua các phương thức [`SetBeginBlocker`](https://pkg.go.dev/github.com/cosmos/cosmos-sdk/baseapp#BaseApp.SetBeginBlocker) và [`SetEndBlocker`](https://pkg.go.dev/github.com/cosmos/cosmos-sdk/baseapp#BaseApp.SetEndBlocker).

Nói chung, các hàm `BeginBlocker` và `EndBlocker` chủ yếu bao gồm các hàm [`BeginBlock` và `EndBlock`](../../build/building-modules/06-beginblock-endblock.md) của mỗi module trong ứng dụng. Điều này được thực hiện bằng cách gọi các hàm `BeginBlock` và `EndBlock` của module manager, hàm này lần lượt gọi `BeginBlock` và `EndBlock` của mỗi module. Thứ tự gọi phải được đặt trong module manager bằng các phương thức `SetOrderBeginBlockers` và `SetOrderEndBlockers`.

Cần lưu ý rằng các blockchain dành riêng cho ứng dụng là xác định (deterministic). Các nhà phát triển phải cẩn thận không đưa tính không xác định vào `BeginBlocker` hoặc `EndBlocker`, và cũng phải cẩn thận không làm chúng quá tốn kém về mặt tính toán, vì [gas](./04-gas-fees.md) không ràng buộc chi phí thực thi của `BeginBlocker` và `EndBlocker`.

Xem ví dụ về các hàm `BeginBlocker` và `EndBlocker` từ `simapp`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/simapp/app.go#L752-L759
```

### Đăng ký Codec

Cấu trúc `EncodingConfig` là phần quan trọng cuối cùng của file `app.go`. Mục tiêu của cấu trúc này là định nghĩa các codec sẽ được sử dụng xuyên suốt ứng dụng.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/simapp/params/encoding.go#L9-L16
```

Dưới đây là mô tả ý nghĩa của bốn trường:

* `InterfaceRegistry`: `InterfaceRegistry` được dùng bởi codec Protobuf để xử lý các interface được mã hóa và giải mã (còn gọi là "unpack") bằng [`google.protobuf.Any`](https://github.com/protocolbuffers/protobuf/blob/master/src/google/protobuf/any.proto). `Any` có thể được coi như một struct chứa `type_url` (tên của kiểu cụ thể triển khai interface) và `value` (các byte được mã hóa của nó). `InterfaceRegistry` cung cấp cơ chế để đăng ký các interface và các triển khai có thể được unpack an toàn từ `Any`. Mỗi module ứng dụng triển khai phương thức `RegisterInterfaces` có thể được dùng để đăng ký các interface và triển khai của module.
* `Codec`: Codec mặc định được dùng xuyên suốt Cosmos SDK. Nó bao gồm `BinaryCodec` dùng để mã hóa và giải mã trạng thái, và `JSONCodec` dùng để xuất dữ liệu cho người dùng (ví dụ: trong [CLI](#cli)). Mặc định SDK dùng Protobuf làm `Codec`.
* `TxConfig`: `TxConfig` định nghĩa một interface mà client có thể sử dụng để tạo kiểu giao dịch cụ thể do ứng dụng định nghĩa. Hiện tại, SDK xử lý hai loại giao dịch: `SIGN_MODE_DIRECT` (dùng Protobuf binary làm mã hóa truyền qua mạng) và `SIGN_MODE_LEGACY_AMINO_JSON` (phụ thuộc vào Amino).
* `Amino`: Một số phần cũ của Cosmos SDK vẫn dùng Amino để tương thích ngược. Mỗi module expose một phương thức `RegisterLegacyAmino` để đăng ký các kiểu cụ thể của module trong Amino. Codec `Amino` này không nên được nhà phát triển ứng dụng sử dụng nữa và sẽ bị xóa trong các phiên bản tương lai.

Một ứng dụng nên tạo encoding config riêng của mình. Xem ví dụ về `simappparams.EncodingConfig` từ `simapp`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/simapp/params/encoding.go#L11-L16
```

## Các Module

[Module](../../build/building-modules/00-intro.md) là trái tim và linh hồn của các ứng dụng Cosmos SDK. Chúng có thể được coi là các state-machine lồng nhau bên trong state-machine. Khi một giao dịch được CometBFT engine chuyển tiếp qua ABCI đến ứng dụng, nó được [`baseapp`](../advanced/00-baseapp.md) định tuyến đến module thích hợp để được xử lý. Mô hình này giúp các nhà phát triển dễ dàng xây dựng các state-machine phức tạp, vì hầu hết các module họ cần thường đã tồn tại sẵn. **Với các nhà phát triển, phần lớn công việc khi xây dựng một ứng dụng Cosmos SDK xoay quanh việc xây dựng các module tùy chỉnh mà ứng dụng cần nhưng chưa tồn tại, và tích hợp chúng với các module đã tồn tại thành một ứng dụng gắn kết.** Trong thư mục ứng dụng, thực hành tiêu chuẩn là lưu trữ module trong thư mục `x/` (không nhầm với thư mục `x/` của Cosmos SDK chứa các module đã được xây dựng sẵn).

### Giao Diện Module Ứng Dụng

Các module phải triển khai [các interface](../../build/building-modules/01-module-manager.md#application-module-interfaces) được định nghĩa trong Cosmos SDK, [`AppModuleBasic`](../../build/building-modules/01-module-manager.md#appmodulebasic) và [`AppModule`](../../build/building-modules/01-module-manager.md#appmodule). Interface trước triển khai các phần tử cơ bản không phụ thuộc của module như `codec`, trong khi interface sau xử lý phần lớn các phương thức của module (bao gồm các phương thức yêu cầu tham chiếu đến `keeper` của các module khác). Cả hai kiểu `AppModule` và `AppModuleBasic` đều được quy ước định nghĩa trong một file gọi là `module.go`.

`AppModule` expose một tập hợp các phương thức hữu ích trên module giúp kết hợp các module thành một ứng dụng gắn kết. Các phương thức này được gọi từ [`module manager`](../../build/building-modules/01-module-manager.md#manager), quản lý tập hợp các module của ứng dụng.

### `Msg` Services

Mỗi module ứng dụng định nghĩa hai [Protobuf service](https://developers.google.com/protocol-buffers/docs/proto#services): một `Msg` service để xử lý message, và một gRPC `Query` service để xử lý truy vấn. Nếu coi module như một state-machine, thì `Msg` service là tập hợp các phương thức RPC chuyển đổi trạng thái. Mỗi phương thức `Msg` service Protobuf có quan hệ 1:1 với một kiểu yêu cầu Protobuf, phải triển khai interface `sdk.Msg`. Lưu ý rằng các `sdk.Msg` được đóng gói trong [giao dịch](../advanced/01-transactions.md), và mỗi giao dịch chứa một hoặc nhiều message.

Khi một block giao dịch hợp lệ được full-node nhận, CometBFT chuyển tiếp từng giao dịch đến ứng dụng qua [`DeliverTx`](https://docs.cometbft.com/v0.37/spec/abci/abci++_app_requirements#specifics-of-responsedelivertx). Sau đó ứng dụng xử lý giao dịch:

1. Khi nhận được giao dịch, ứng dụng trước tiên giải mã (unmarshal) nó từ `[]byte`.
2. Sau đó, nó xác minh một số thứ về giao dịch như [thanh toán phí và chữ ký](./04-gas-fees.md#antehandler) trước khi trích xuất các `Msg` chứa trong giao dịch.
3. Các `sdk.Msg` được mã hóa bằng Protobuf [`Any`](#register-codec). Bằng cách phân tích `type_url` của mỗi `Any`, `msgServiceRouter` của baseapp định tuyến `sdk.Msg` đến `Msg` service của module tương ứng.
4. Nếu message được xử lý thành công, trạng thái được cập nhật.

Để biết thêm chi tiết, xem [vòng đời giao dịch](./01-tx-lifecycle.md).

Nhà phát triển module tạo các `Msg` service tùy chỉnh khi họ xây dựng module của mình. Thực hành thông thường là định nghĩa `Msg` Protobuf service trong file `tx.proto`. Ví dụ, module `x/bank` định nghĩa một service với hai phương thức để chuyển token:

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/proto/cosmos/bank/v1beta1/tx.proto#L13-L36
```

Các phương thức service sử dụng `keeper` để cập nhật trạng thái module.

Mỗi module cũng nên triển khai phương thức `RegisterServices` như một phần của [interface `AppModule`](#application-module-interface). Phương thức này nên gọi hàm `RegisterMsgServer` được cung cấp bởi code Protobuf được tạo ra.

### gRPC `Query` Services

gRPC `Query` service cho phép người dùng truy vấn trạng thái bằng [gRPC](https://grpc.io). Chúng được bật theo mặc định và có thể được cấu hình trong các trường `grpc.enable` và `grpc.address` bên trong [`app.toml`](../../user/run-node/01-run-node.md#configuring-the-node-using-apptoml-and-configtoml).

gRPC `Query` service được định nghĩa trong các file định nghĩa Protobuf của module, cụ thể trong `query.proto`. File định nghĩa `query.proto` expose một `Query` [Protobuf service](https://developers.google.com/protocol-buffers/docs/proto#services) duy nhất. Mỗi gRPC query endpoint tương ứng với một phương thức service, bắt đầu bằng từ khóa `rpc`, bên trong `Query` service.

Protobuf tạo ra interface `QueryServer` cho mỗi module, chứa tất cả các phương thức service. [`keeper`](#keeper) của module sau đó cần triển khai interface `QueryServer` này bằng cách cung cấp triển khai cụ thể của mỗi phương thức service.

Cuối cùng, mỗi module cũng nên triển khai phương thức `RegisterServices` như một phần của [interface `AppModule`](#application-module-interface). Phương thức này nên gọi hàm `RegisterQueryServer` được cung cấp bởi code Protobuf được tạo ra.

### Keeper

[`Keeper`](../../build/building-modules/06-keeper.md) là những người gác cổng của các store trong module. Để đọc hoặc ghi vào store của module, bắt buộc phải đi qua một trong các phương thức `keeper` của nó. Điều này được đảm bảo bởi mô hình [object-capability](../advanced/10-ocap.md) của Cosmos SDK. Chỉ các đối tượng giữ key đến store mới có thể truy cập nó, và chỉ `keeper` của module mới nên giữ key đến store của module đó.

`Keeper` thường được định nghĩa trong một file gọi là `keeper.go`. Nó chứa định nghĩa kiểu và các phương thức của `keeper`.

Định nghĩa kiểu của `keeper` thường bao gồm:

* **Key** đến các store của module trong multistore.
* Tham chiếu đến **`keeper` của các module khác**. Chỉ cần thiết nếu `keeper` cần truy cập store của module khác (để đọc hoặc ghi).
* Tham chiếu đến **codec** của ứng dụng. `keeper` cần nó để marshal các struct trước khi lưu, hoặc unmarshal khi lấy ra, vì store chỉ chấp nhận `[]bytes` làm giá trị.

Cùng với định nghĩa kiểu, thành phần quan trọng tiếp theo trong file `keeper.go` là hàm constructor `NewKeeper`. Hàm này khởi tạo một `keeper` mới theo kiểu được định nghĩa ở trên với `codec`, các `key` store và có thể tham chiếu đến `keeper` của các module khác làm tham số. Hàm `NewKeeper` được gọi từ [constructor của ứng dụng](#constructor-function). Phần còn lại của file định nghĩa các phương thức của `keeper`, chủ yếu là getter và setter.

### Giao Diện CLI, gRPC Services và REST

Mỗi module định nghĩa các lệnh command-line, gRPC service và REST route để được expose cho người dùng cuối qua [giao diện của ứng dụng](#application-interfaces). Điều này cho phép người dùng cuối tạo các message theo kiểu được định nghĩa trong module, hoặc truy vấn tập con trạng thái được quản lý bởi module.

#### CLI

Nói chung, [các lệnh liên quan đến module](../../build/building-modules/09-module-interfaces.md#cli) được định nghĩa trong một thư mục gọi là `client/cli` trong thư mục module. CLI chia lệnh thành hai loại, giao dịch và truy vấn, được định nghĩa trong `client/cli/tx.go` và `client/cli/query.go` tương ứng. Cả hai lệnh đều được xây dựng trên [Cobra Library](https://github.com/spf13/cobra):

* Lệnh giao dịch cho phép người dùng tạo giao dịch mới để có thể được đưa vào block và cuối cùng cập nhật trạng thái. Một lệnh nên được tạo cho mỗi [kiểu message](#message-types) được định nghĩa trong module. Lệnh gọi constructor của message với các tham số do người dùng cung cấp và bọc nó vào một giao dịch. Cosmos SDK xử lý việc ký và thêm các metadata giao dịch khác.
* Truy vấn cho phép người dùng truy vấn tập con trạng thái được định nghĩa bởi module.

#### gRPC

[gRPC](https://grpc.io) là framework RPC hiệu năng cao nguồn mở hiện đại có hỗ trợ nhiều ngôn ngữ. Đây là cách được khuyến nghị cho các client bên ngoài (như ví, trình duyệt và các dịch vụ backend khác) để tương tác với một node.

Mỗi module có thể expose các gRPC endpoint gọi là [service method](https://grpc.io/docs/what-is-grpc/core-concepts/#service-definition), được định nghĩa trong [file Protobuf `query.proto` của module](#grpc-query-services). Một service method được định nghĩa bởi tên, tham số đầu vào và phản hồi đầu ra. Module sau đó cần thực hiện các hành động sau:

* Định nghĩa phương thức `RegisterGRPCGatewayRoutes` trên `AppModuleBasic` để kết nối các yêu cầu gRPC của client đến handler đúng bên trong module.
* Với mỗi service method, định nghĩa một handler tương ứng, nằm trong file `keeper/grpc_query.go`.

#### gRPC-gateway REST Endpoints

Một số client bên ngoài có thể không muốn dùng gRPC. Trong trường hợp này, Cosmos SDK cung cấp dịch vụ gRPC gateway, expose mỗi gRPC service như một REST endpoint tương ứng. Tham khảo [tài liệu grpc-gateway](https://grpc-ecosystem.github.io/grpc-gateway/) để tìm hiểu thêm.

Các REST endpoint được định nghĩa trong các file Protobuf, cùng với gRPC service, sử dụng các Protobuf annotation. Các module muốn expose REST query nên thêm `google.api.http` annotation vào các phương thức `rpc` của chúng. Theo mặc định, tất cả REST endpoint được định nghĩa trong SDK có URL bắt đầu bằng tiền tố `/cosmos/`.

Cosmos SDK cũng cung cấp một development endpoint để tạo các file định nghĩa [Swagger](https://swagger.io/) cho các REST endpoint này. Endpoint này có thể được bật trong file cấu hình [`app.toml`](../../user/run-node/01-run-node.md#configuring-the-node-using-apptoml-and-configtoml), dưới key `api.swagger`.

## Giao Diện Ứng Dụng

[Các giao diện](#command-line-grpc-services-and-rest-interfaces) cho phép người dùng cuối tương tác với các full-node client. Điều này có nghĩa là truy vấn dữ liệu từ full-node hoặc tạo và gửi giao dịch mới để được full-node chuyển tiếp và cuối cùng được đưa vào block.

Giao diện chính là [Command-Line Interface](../advanced/07-cli.md). CLI của một ứng dụng Cosmos SDK được xây dựng bằng cách tổng hợp [các lệnh CLI](#cli) được định nghĩa trong mỗi module được ứng dụng sử dụng. CLI của một ứng dụng giống như daemon (ví dụ: `appd`) và được định nghĩa trong một file gọi là `appd/main.go`. File này chứa:

* **Hàm `main()`**, được thực thi để xây dựng interface client `appd`. Hàm này chuẩn bị mỗi lệnh và thêm chúng vào `rootCmd` trước khi build. Tại gốc của `appd`, hàm thêm các lệnh chung như `status`, `keys`, và `config`, lệnh truy vấn, lệnh tx và `rest-server`.
* **Lệnh truy vấn**, được thêm bằng cách gọi hàm `queryCmd`. Hàm này trả về một lệnh Cobra chứa các lệnh truy vấn được định nghĩa trong mỗi module của ứng dụng, cũng như một số lệnh truy vấn cấp thấp hơn như truy vấn block hoặc validator.
* **Lệnh giao dịch**, được thêm bằng cách gọi hàm `txCmd`. Tương tự `queryCmd`, hàm trả về một lệnh Cobra chứa các lệnh tx được định nghĩa trong mỗi module của ứng dụng, cũng như các lệnh tx cấp thấp hơn như ký hoặc broadcast giao dịch.

Xem ví dụ về file lệnh command-line chính của ứng dụng từ [Cosmos Hub](https://github.com/cosmos/gaia).

```go reference
https://github.com/cosmos/gaia/blob/26ae7c2/cmd/gaiad/cmd/root.go#L39-L80
```

## Dependencies và Makefile

Phần này là tùy chọn, vì các nhà phát triển được tự do chọn trình quản lý dependency và phương pháp build dự án. Tuy vậy, framework hiện được sử dụng nhiều nhất cho quản lý phiên bản là [`go.mod`](https://github.com/golang/go/wiki/Modules). Nó đảm bảo rằng mỗi thư viện được sử dụng xuyên suốt ứng dụng được import với đúng phiên bản.

Dưới đây là `go.mod` của [Cosmos Hub](https://github.com/cosmos/gaia), được cung cấp làm ví dụ.

```go reference
https://github.com/cosmos/gaia/blob/26ae7c2/go.mod#L1-L28
```

Để build ứng dụng, [Makefile](https://en.wikipedia.org/wiki/Makefile) thường được sử dụng. Makefile chủ yếu đảm bảo rằng `go.mod` được chạy trước khi build hai điểm vào của ứng dụng, [`Node Client`](#node-client) và [`Application Interface`](#application-interface).

Đây là ví dụ về [Makefile của Cosmos Hub](https://github.com/cosmos/gaia/blob/main/Makefile).
