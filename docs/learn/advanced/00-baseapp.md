---
sidebar_position: 1
---

# BaseApp

:::note Tóm tắt
Tài liệu này mô tả `BaseApp`, lớp trừu tượng triển khai các chức năng cốt lõi của một ứng dụng Cosmos SDK.
:::

:::note Tài liệu cần đọc trước

* [Cấu trúc của một ứng dụng Cosmos SDK](../beginner/00-app-anatomy.md)
* [Vòng đời của một giao dịch Cosmos SDK](../beginner/01-tx-lifecycle.md)

:::

## Giới thiệu

`BaseApp` là kiểu cơ sở triển khai phần cốt lõi của một ứng dụng Cosmos SDK, cụ thể là:

* [Giao diện Blockchain Ứng dụng (ABCI)](#main-abci-messages), để state-machine giao tiếp với consensus engine bên dưới (ví dụ: CometBFT).
* [Service Router](#service-routers), để định tuyến các message và query đến module phù hợp.
* Các [trạng thái](#state-updates) khác nhau, vì state-machine có thể có các trạng thái biến động (volatile states) khác nhau được cập nhật dựa trên ABCI message nhận được.

Mục tiêu của `BaseApp` là cung cấp lớp nền tảng cho một ứng dụng Cosmos SDK mà các nhà phát triển có thể dễ dàng mở rộng để xây dựng ứng dụng tùy chỉnh của riêng mình. Thông thường, các nhà phát triển sẽ tạo một kiểu tùy chỉnh cho ứng dụng của họ, như sau:

```go
type App struct {
  // tham chiếu đến BaseApp
  *baseapp.BaseApp

  // danh sách các store key của ứng dụng

  // danh sách các keeper của ứng dụng

  // module manager
}
```

Việc mở rộng ứng dụng với `BaseApp` cho phép ứng dụng đó truy cập tất cả các phương thức của `BaseApp`. Điều này cho phép các nhà phát triển kết hợp ứng dụng tùy chỉnh của họ với các module mong muốn, mà không cần phải lo lắng về công việc phức tạp khi tự triển khai ABCI, service router và logic quản lý trạng thái.

## Định nghĩa kiểu

Kiểu `BaseApp` chứa nhiều tham số quan trọng cho bất kỳ ứng dụng nào dựa trên Cosmos SDK.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/baseapp/baseapp.go#L64-L201
```

Hãy cùng xem qua các thành phần quan trọng nhất.

> **Lưu ý**: Không phải tất cả các tham số đều được mô tả, chỉ những tham số quan trọng nhất. Tham khảo định nghĩa kiểu để xem danh sách đầy đủ.

Đầu tiên là các tham số quan trọng được khởi tạo trong quá trình khởi động ứng dụng:

* [`CommitMultiStore`](./04-store.md#commitmultistore): Đây là store chính của ứng dụng, lưu giữ trạng thái chuẩn (canonical state) được commit vào [cuối mỗi block](#commit). Store này **không** được cache, nghĩa là nó không được dùng để cập nhật các trạng thái biến động (chưa được commit) của ứng dụng. `CommitMultiStore` là một multi-store, tức là một store của các store. Mỗi module trong ứng dụng sử dụng một hoặc nhiều `KVStore` trong multi-store để lưu trữ phần trạng thái của nó.
* Database: `db` được dùng bởi `CommitMultiStore` để xử lý việc lưu trữ dữ liệu lâu dài.
* [`Msg` Service Router](#msg-service-router): `msgServiceRouter` tạo điều kiện định tuyến các yêu cầu `sdk.Msg` đến `Msg` service của module phù hợp để xử lý. Ở đây, `sdk.Msg` đề cập đến thành phần giao dịch cần được xử lý bởi một service để cập nhật trạng thái ứng dụng, không phải ABCI message — là thứ triển khai giao diện giữa ứng dụng và consensus engine bên dưới.
* [gRPC Query Router](#grpc-query-router): `grpcQueryRouter` tạo điều kiện định tuyến các gRPC query đến module phù hợp để xử lý. Các query này không phải là ABCI message, nhưng được chuyển tiếp đến gRPC `Query` service của module liên quan.
* [`TxDecoder`](https://pkg.go.dev/github.com/cosmos/cosmos-sdk/types#TxDecoder): Được dùng để giải mã các byte giao dịch thô do CometBFT engine bên dưới chuyển tiếp.
* [`AnteHandler`](#antehandler): Handler này được dùng để xử lý xác minh chữ ký, thanh toán phí, và các kiểm tra trước khi thực thi message khi nhận được giao dịch. Nó được thực thi trong quá trình [`CheckTx/RecheckTx`](#checktx) và [`FinalizeBlock`](#finalizeblock).
* [`InitChainer`](../beginner/00-app-anatomy.md#initchainer), [`PreBlocker`](../beginner/00-app-anatomy.md#preblocker), [`BeginBlocker` và `EndBlocker`](../beginner/00-app-anatomy.md#beginblocker-and-endblocker): Đây là các hàm được thực thi khi ứng dụng nhận được ABCI message `InitChain` và `FinalizeBlock` từ CometBFT engine bên dưới.

Tiếp theo là các tham số dùng để định nghĩa [các trạng thái biến động](#state-updates) (tức là cached states):

* `checkState`: Trạng thái này được cập nhật trong [`CheckTx`](#checktx) và được reset khi [`Commit`](#commit).
* `finalizeBlockState`: Trạng thái này được cập nhật trong [`FinalizeBlock`](#finalizeblock), được set về `nil` khi [`Commit`](#commit) và được khởi tạo lại trong `FinalizeBlock`.
* `processProposalState`: Trạng thái này được cập nhật trong [`ProcessProposal`](#process-proposal).
* `prepareProposalState`: Trạng thái này được cập nhật trong [`PrepareProposal`](#prepare-proposal).

Cuối cùng là một số tham số quan trọng khác:

* `voteInfos`: Tham số này mang danh sách các validator có precommit bị thiếu, do họ không bỏ phiếu hoặc do proposer không đưa phiếu của họ vào. Thông tin này được truyền qua [Context](./02-context.md) và có thể được ứng dụng sử dụng cho nhiều mục đích như phạt validator vắng mặt.
* `minGasPrices`: Tham số này xác định mức giá gas tối thiểu mà node chấp nhận. Đây là tham số **cục bộ**, nghĩa là mỗi full-node có thể đặt `minGasPrices` khác nhau. Nó được dùng trong `AnteHandler` trong quá trình [`CheckTx`](#checktx), chủ yếu như một cơ chế chống spam. Giao dịch chỉ được vào [mempool](https://github.com/cometbft/cometbft/blob/v0.37.x/spec/abci/abci++_basic_concepts.md#mempool-methods) nếu gas price của giao dịch lớn hơn một trong các mức giá gas tối thiểu trong `minGasPrices` (ví dụ: nếu `minGasPrices == 1uatom,1photon`, thì `gas-price` của giao dịch phải lớn hơn `1uatom` HOẶC `1photon`).
* `appVersion`: Phiên bản của ứng dụng. Nó được đặt trong [hàm constructor của ứng dụng](../beginner/00-app-anatomy.md#constructor-function).

## Constructor

```go
func NewBaseApp(
  name string, logger log.Logger, db dbm.DB, txDecoder sdk.TxDecoder, options ...func(*BaseApp),
) *BaseApp {

  // ...
}
```

Hàm constructor của `BaseApp` khá đơn giản. Điều đáng chú ý duy nhất là khả năng cung cấp thêm [`options`](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/baseapp/options.go) cho `BaseApp`, chúng sẽ được thực thi theo thứ tự. Các `options` thường là các hàm `setter` cho các tham số quan trọng, chẳng hạn như `SetPruning()` để đặt tùy chọn pruning hoặc `SetMinGasPrices()` để đặt `min-gas-prices` cho node.

Tất nhiên, các nhà phát triển có thể thêm `options` tùy chỉnh dựa trên nhu cầu của ứng dụng.

## Cập nhật trạng thái

`BaseApp` duy trì bốn trạng thái biến động chính và một trạng thái gốc (root) hay trạng thái chính (main state). Trạng thái chính là trạng thái chuẩn của ứng dụng, còn các trạng thái biến động — `checkState`, `prepareProposalState`, `processProposalState` và `finalizeBlockState` — được dùng để xử lý các chuyển đổi trạng thái trung gian giữa các lần [`Commit`](#commit) trên trạng thái chính.

Về mặt nội tại, chỉ có một `CommitMultiStore` duy nhất mà chúng ta gọi là trạng thái gốc (root) hoặc trạng thái chính. Từ trạng thái gốc này, chúng ta tạo ra bốn trạng thái biến động bằng cơ chế gọi là _store branching_ (thực hiện bởi hàm `CacheWrap`). Các kiểu có thể được minh họa như sau:

![Các kiểu](./baseapp_state.png)

### Cập nhật trạng thái InitChain

Trong `InitChain`, bốn trạng thái biến động — `checkState`, `prepareProposalState`, `processProposalState` và `finalizeBlockState` — được tạo bằng cách branch từ `CommitMultiStore` gốc. Mọi thao tác đọc và ghi tiếp theo đều xảy ra trên các phiên bản đã được branch của `CommitMultiStore`. Để tránh các round-trip không cần thiết đến trạng thái chính, tất cả các thao tác đọc trên branched store đều được cache.

![InitChain](./baseapp_state-initchain.png)

### Cập nhật trạng thái CheckTx

Trong `CheckTx`, `checkState` — được xây dựng dựa trên trạng thái đã commit gần nhất từ root store — được dùng cho mọi thao tác đọc và ghi. Ở đây chúng ta chỉ thực thi `AnteHandler` và xác minh rằng service router tồn tại cho mỗi message trong giao dịch. Lưu ý: khi thực thi `AnteHandler`, chúng ta branch thêm từ `checkState` đã được branch sẵn. Điều này có tác dụng phụ là nếu `AnteHandler` thất bại, các chuyển đổi trạng thái sẽ không được phản ánh vào `checkState` — tức là `checkState` chỉ được cập nhật khi thành công.

![CheckTx](./baseapp_state-checktx.png)

### Cập nhật trạng thái PrepareProposal

Trong `PrepareProposal`, `prepareProposalState` được tạo bằng cách branch từ `CommitMultiStore` gốc. `prepareProposalState` được dùng cho mọi thao tác đọc và ghi xảy ra trong giai đoạn `PrepareProposal`. Hàm này sử dụng phương thức `Select()` của mempool để duyệt qua các giao dịch. `runTx` sau đó được gọi để mã hóa và xác thực từng giao dịch, rồi `AnteHandler` được thực thi. Nếu thành công, các giao dịch hợp lệ được trả về kèm theo các event, tag và dữ liệu được tạo ra trong quá trình thực thi proposal. Hành vi được mô tả là của handler mặc định; các ứng dụng có thể linh hoạt định nghĩa [custom mempool handler](https://docs.cosmos.network/main/build/building-apps/app-mempool) riêng.

![ProcessProposal](./baseapp_state-prepareproposal.png)

### Cập nhật trạng thái ProcessProposal

Trong `ProcessProposal`, `processProposalState` được tạo dựa trên trạng thái đã commit gần nhất từ root store và được dùng để xử lý một proposal đã ký nhận được từ một validator. Trong trạng thái này, `runTx` được gọi, `AnteHandler` được thực thi, và context sử dụng trong trạng thái này được xây dựng với thông tin từ header và trạng thái chính, bao gồm cả mức giá gas tối thiểu cũng được đặt vào. Một lần nữa, hành vi được mô tả là của handler mặc định; các ứng dụng có thể linh hoạt định nghĩa [custom mempool handler](https://docs.cosmos.network/main/build/building-apps/app-mempool) riêng.

![ProcessProposal](./baseapp_state-processproposal.png)

### Cập nhật trạng thái FinalizeBlock

Trong `FinalizeBlock`, `finalizeBlockState` được thiết lập để sử dụng trong quá trình thực thi giao dịch và endblock. `finalizeBlockState` được xây dựng dựa trên trạng thái đã commit gần nhất từ root store và được branch ra. Lưu ý: `finalizeBlockState` được đặt về `nil` khi [`Commit`](#commit).

Luồng trạng thái trong quá trình thực thi giao dịch gần giống hệt `CheckTx`, ngoại trừ các chuyển đổi trạng thái xảy ra trên `finalizeBlockState` và các message trong giao dịch được thực thi. Tương tự `CheckTx`, các chuyển đổi trạng thái xảy ra trên một trạng thái được branch hai lần — `finalizeBlockState`. Việc thực thi message thành công dẫn đến các thay đổi được ghi vào `finalizeBlockState`. Lưu ý: nếu thực thi message thất bại, các chuyển đổi trạng thái từ AnteHandler vẫn được giữ lại.

### Cập nhật trạng thái Commit

Trong quá trình `Commit`, tất cả các chuyển đổi trạng thái xảy ra trong `finalizeBlockState` cuối cùng được ghi vào `CommitMultiStore` gốc, từ đó được commit xuống đĩa và tạo ra một app root hash mới. Các chuyển đổi trạng thái này hiện được coi là cuối cùng. Cuối cùng, `checkState` được đặt thành trạng thái mới vừa được commit và `finalizeBlockState` được đặt về `nil` để được reset lại trong `FinalizeBlock`.

![Commit](./baseapp_state-commit.png)

## ParamStore

Trong `InitChain`, `RequestInitChain` cung cấp `ConsensusParams` chứa các tham số liên quan đến thực thi block như gas tối đa, kích thước tối đa, cũng như các tham số bằng chứng (evidence). Nếu các tham số này khác nil, chúng được đặt vào `ParamStore` của BaseApp. Phía sau, `ParamStore` được quản lý bởi module `x/consensus_params`. Điều này cho phép các tham số được điều chỉnh thông qua governance on-chain.

## Service Router

Khi ứng dụng nhận được các message và query, chúng phải được định tuyến đến module phù hợp để xử lý. Việc định tuyến được thực hiện thông qua `BaseApp`, có chứa `msgServiceRouter` cho message và `grpcQueryRouter` cho query.

### `Msg` Service Router

[`sdk.Msg`s](../../build/building-modules/02-messages-and-queries.md#messages) cần được định tuyến sau khi được trích xuất từ các giao dịch được gửi đến từ CometBFT engine bên dưới qua các ABCI message [`CheckTx`](#checktx) và [`FinalizeBlock`](#finalizeblock). Để làm vậy, `BaseApp` có một `msgServiceRouter` ánh xạ các service method được định danh đầy đủ (fully-qualified) (`string`, được định nghĩa trong Protobuf `Msg` service của mỗi module) đến triển khai `MsgServer` tương ứng của module đó.

[`msgServiceRouter` mặc định được tích hợp trong `BaseApp`](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/baseapp/msg_service_router.go) là stateless. Tuy nhiên, một số ứng dụng có thể muốn dùng các cơ chế định tuyến có trạng thái (stateful), chẳng hạn như cho phép governance vô hiệu hóa một số route nhất định hoặc trỏ chúng đến module mới cho mục đích nâng cấp. Vì lý do này, `sdk.Context` cũng được truyền vào mỗi [route handler bên trong `msgServiceRouter`](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/baseapp/msg_service_router.go#L35-L36). Với một stateless router không muốn sử dụng điều này, bạn có thể bỏ qua `ctx`.

`msgServiceRouter` của ứng dụng được khởi tạo với tất cả các route bằng [module manager](../../build/building-modules/01-module-manager.md#manager) của ứng dụng (thông qua phương thức `RegisterServices`), module manager này được khởi tạo với tất cả các module của ứng dụng trong [constructor](../beginner/00-app-anatomy.md#constructor-function) của ứng dụng.

### gRPC Query Router

Tương tự `sdk.Msg`, [`query`](../../build/building-modules/02-messages-and-queries.md#queries) cũng cần được định tuyến đến [`Query` service](../../build/building-modules/04-query-services.md) của module phù hợp. Để làm vậy, `BaseApp` có một `grpcQueryRouter` ánh xạ các service method được định danh đầy đủ (fully-qualified) của module (`string`, được định nghĩa trong Protobuf `Query` gRPC của chúng) đến triển khai `QueryServer` tương ứng. `grpcQueryRouter` được gọi trong các giai đoạn đầu của quá trình xử lý query, có thể là bằng cách gửi gRPC query trực tiếp đến gRPC endpoint, hoặc thông qua [ABCI message `Query`](#query) trên CometBFT RPC endpoint.

Giống như `msgServiceRouter`, `grpcQueryRouter` được khởi tạo với tất cả các query route bằng [module manager](../../build/building-modules/01-module-manager.md) của ứng dụng (thông qua phương thức `RegisterServices`), module manager này được khởi tạo với tất cả các module của ứng dụng trong [constructor](../beginner/00-app-anatomy.md#app-constructor) của ứng dụng.

## Các ABCI 2.0 Message chính

[Application-Blockchain Interface](https://github.com/cometbft/cometbft/blob/v0.37.x/spec/abci/abci++_basic_concepts.md) (ABCI) là giao diện chung kết nối một state-machine với consensus engine để tạo thành một full-node hoạt động được. Nó có thể được bọc bằng bất kỳ ngôn ngữ nào và cần được triển khai bởi mỗi blockchain chuyên dụng (application-specific blockchain) xây dựng trên consensus engine tương thích ABCI như CometBFT.

Consensus engine xử lý hai nhiệm vụ chính:

* Logic mạng, chủ yếu bao gồm việc gossip (truyền đi) các phần block, giao dịch và phiếu bầu đồng thuận.
* Logic đồng thuận, dẫn đến việc sắp xếp giao dịch theo thứ tự xác định dưới dạng các block.

Consensus engine **không** có trách nhiệm định nghĩa trạng thái hay tính hợp lệ của giao dịch. Nhìn chung, các giao dịch được consensus engine xử lý dưới dạng `[]bytes` và được chuyển tiếp đến ứng dụng qua ABCI để được giải mã và xử lý. Tại các thời điểm quan trọng trong quá trình mạng và đồng thuận (ví dụ: đầu block, commit block, nhận được giao dịch chưa được xác nhận...), consensus engine phát ra các ABCI message để state-machine hành động theo.

Các nhà phát triển xây dựng trên Cosmos SDK không cần tự triển khai ABCI, vì `BaseApp` đã có sẵn triển khai tích hợp của giao diện này. Hãy cùng xem qua các ABCI message chính mà `BaseApp` triển khai:

* [`Prepare Proposal`](#prepare-proposal)
* [`Process Proposal`](#process-proposal)
* [`CheckTx`](#checktx)
* [`FinalizeBlock`](#finalizeblock)
* [`ExtendVote`](#extendvote)
* [`VerifyVoteExtension`](#verifyvoteextension)

### Prepare Proposal

Hàm `PrepareProposal` là một trong các phương thức mới được giới thiệu trong Application Blockchain Interface (ABCI++) của CometBFT và là một phần quan trọng trong hệ thống quản trị tổng thể của ứng dụng. Trong Cosmos SDK, nó cho phép ứng dụng có sự kiểm soát chi tiết hơn đối với các giao dịch được xử lý và đảm bảo rằng chỉ có các giao dịch hợp lệ mới được commit lên blockchain.

Đây là cách hàm `PrepareProposal` có thể được triển khai:

1. Trích xuất các `sdk.Msg` từ giao dịch.
2. Thực hiện kiểm tra _có trạng thái (stateful)_ bằng cách gọi `Validate()` trên mỗi `sdk.Msg`. Điều này được thực hiện sau các kiểm tra _không có trạng thái (stateless)_ vì kiểm tra stateful tốn nhiều tài nguyên tính toán hơn. Nếu `Validate()` thất bại, `PrepareProposal` trả về trước khi chạy thêm kiểm tra, giúp tiết kiệm tài nguyên.
3. Thực hiện bất kỳ kiểm tra bổ sung nào dành riêng cho ứng dụng, chẳng hạn như kiểm tra số dư tài khoản, hoặc đảm bảo rằng một số điều kiện nhất định được đáp ứng trước khi giao dịch được đề xuất.
4. Trả về các giao dịch đã cập nhật để consensus engine xử lý.

Lưu ý rằng, không giống như `CheckTx()`, `PrepareProposal` xử lý `sdk.Msg`, do đó nó có thể trực tiếp cập nhật trạng thái. Tuy nhiên, khác với `FinalizeBlock()`, nó không commit các cập nhật trạng thái đó. Cần thận trọng khi sử dụng `PrepareProposal` vì code không đúng có thể ảnh hưởng đến tính liveness (hoạt động liên tục) của mạng.

Cần lưu ý rằng `PrepareProposal` bổ sung cho phương thức `ProcessProposal` — được thực thi sau phương thức này. Sự kết hợp của hai phương thức này có nghĩa là có thể đảm bảo rằng không có giao dịch không hợp lệ nào từng được commit. Hơn nữa, thiết lập như vậy có thể mở ra các trường hợp sử dụng thú vị khác như Oracle, giải mã ngưỡng (threshold decryption) và nhiều hơn nữa.

`PrepareProposal` trả về phản hồi cho consensus engine bên dưới có kiểu [`abci.CheckTxResponse`](https://github.com/cometbft/cometbft/blob/v0.37.x/spec/abci/abci++_methods.md#processproposal). Phản hồi chứa:

* `Code (uint32)`: Mã phản hồi. `0` nếu thành công.
* `Data ([]byte)`: Byte kết quả, nếu có.
* `Log (string)`: Đầu ra của logger ứng dụng. Có thể không xác định (non-deterministic).
* `Info (string)`: Thông tin bổ sung. Có thể không xác định.

### Process Proposal

Hàm `ProcessProposal` được BaseApp gọi như một phần của luồng ABCI message và được thực thi trong giai đoạn `FinalizeBlock` của quá trình đồng thuận. Mục đích của hàm này là trao cho ứng dụng nhiều quyền kiểm soát hơn đối với việc xác thực block, cho phép ứng dụng kiểm tra tất cả các giao dịch trong một block được đề xuất trước khi validator gửi prevote cho block đó. Nó cho phép validator thực hiện công việc phụ thuộc vào ứng dụng trong một block được đề xuất, tạo điều kiện cho các tính năng như thực thi block ngay lập tức, và cho phép ứng dụng từ chối các block không hợp lệ.

Hàm `ProcessProposal` thực hiện một số nhiệm vụ chính, bao gồm:

1. Xác thực block được đề xuất bằng cách kiểm tra tất cả các giao dịch trong đó.
2. Kiểm tra block được đề xuất so với trạng thái hiện tại của ứng dụng, để đảm bảo rằng nó hợp lệ và có thể được thực thi.
3. Cập nhật trạng thái của ứng dụng dựa trên proposal, nếu nó hợp lệ và vượt qua tất cả các kiểm tra.
4. Trả về phản hồi cho CometBFT chỉ ra kết quả xử lý proposal.

`ProcessProposal` là một phần quan trọng trong hệ thống quản trị tổng thể của ứng dụng. Nó được dùng để quản lý các tham số mạng và các khía cạnh quan trọng khác của hoạt động mạng. Nó cũng đảm bảo thuộc tính coherence được tuân thủ, tức là tất cả các validator trung thực phải chấp nhận proposal từ một proposer trung thực.

Cần lưu ý rằng `ProcessProposal` bổ sung cho phương thức `PrepareProposal` — cho phép ứng dụng có sự kiểm soát giao dịch chi tiết hơn bằng cách cho phép sắp xếp lại, bỏ, trì hoãn, chỉnh sửa và thậm chí thêm giao dịch theo nhu cầu. Sự kết hợp của hai phương thức này có nghĩa là có thể đảm bảo rằng không có giao dịch không hợp lệ nào từng được commit. Hơn nữa, thiết lập như vậy có thể mở ra các trường hợp sử dụng thú vị khác như Oracle, giải mã ngưỡng và nhiều hơn nữa.

CometBFT gọi nó khi nhận được một proposal và thuật toán CometBFT chưa khóa vào một giá trị nào. Ứng dụng không thể chỉnh sửa proposal tại thời điểm này nhưng có thể từ chối nó nếu không hợp lệ. Nếu xảy ra điều đó, CometBFT sẽ prevote `nil` cho proposal, điều này có hàm ý lớn đến liveness của CometBFT. Theo quy tắc chung, ứng dụng NÊN chấp nhận một proposal đã được chuẩn bị được truyền qua `ProcessProposal`, ngay cả khi một phần của proposal không hợp lệ (ví dụ: một giao dịch không hợp lệ); ứng dụng có thể bỏ qua phần không hợp lệ của proposal đã chuẩn bị tại thời điểm thực thi block.

Tuy nhiên, các nhà phát triển phải thận trọng hơn khi sử dụng các phương thức này. Code không đúng có thể ảnh hưởng đến liveness vì CometBFT không thể nhận được 2/3 precommit hợp lệ để finalize một block.

`ProcessProposal` trả về phản hồi cho consensus engine bên dưới có kiểu [`abci.CheckTxResponse`](https://github.com/cometbft/cometbft/blob/v0.37.x/spec/abci/abci++_methods.md#processproposal). Phản hồi chứa:

* `Code (uint32)`: Mã phản hồi. `0` nếu thành công.
* `Data ([]byte)`: Byte kết quả, nếu có.
* `Log (string)`: Đầu ra của logger ứng dụng. Có thể không xác định.
* `Info (string)`: Thông tin bổ sung. Có thể không xác định.

### CheckTx

`CheckTx` được gửi bởi consensus engine bên dưới khi một giao dịch chưa được xác nhận (tức là chưa được đưa vào một block hợp lệ) được nhận bởi full-node. Vai trò của `CheckTx` là bảo vệ mempool của full-node (nơi lưu trữ các giao dịch chưa được xác nhận cho đến khi chúng được đưa vào block) khỏi các giao dịch spam. Các giao dịch chưa được xác nhận chỉ được chuyển tiếp đến các peer nếu chúng vượt qua `CheckTx`.

`CheckTx()` có thể thực hiện cả kiểm tra _có trạng thái_ lẫn _không có trạng thái_, nhưng các nhà phát triển nên cố gắng làm cho các kiểm tra **nhẹ nhàng (lightweight)** vì phí gas không được tính cho tài nguyên (CPU, tải dữ liệu...) sử dụng trong `CheckTx`.

Trong Cosmos SDK, sau khi [giải mã giao dịch](./05-encoding.md), `CheckTx()` được triển khai để thực hiện các kiểm tra sau:

1. Trích xuất các `sdk.Msg` từ giao dịch.
2. **Tùy chọn** thực hiện kiểm tra _không có trạng thái_ bằng cách gọi `ValidateBasic()` trên mỗi `sdk.Msg`. Điều này được thực hiện trước tiên vì kiểm tra stateless tốn ít tài nguyên tính toán hơn kiểm tra stateful. Nếu `ValidateBasic()` thất bại, `CheckTx` trả về trước khi chạy kiểm tra stateful, giúp tiết kiệm tài nguyên. Kiểm tra này vẫn được thực hiện đối với các message chưa được chuyển sang cơ chế xác thực message mới được định nghĩa trong [RFC 001](https://docs.cosmos.network/main/rfc/rfc-001-tx-validation) và vẫn còn phương thức `ValidateBasic()`.
3. Thực hiện các kiểm tra _có trạng thái_ không liên quan đến module trên [tài khoản](../beginner/03-accounts.md). Bước này chủ yếu là kiểm tra rằng chữ ký `sdk.Msg` hợp lệ, đủ phí được cung cấp và tài khoản gửi có đủ tiền để trả phí đó. Lưu ý rằng không có đếm [`gas`](../beginner/04-gas-fees.md) chính xác nào ở đây vì `sdk.Msg` không được xử lý. Thông thường, [`AnteHandler`](../beginner/04-gas-fees.md#antehandler) sẽ kiểm tra rằng `gas` được cung cấp cùng giao dịch lớn hơn một lượng gas tham chiếu tối thiểu dựa trên kích thước giao dịch thô, nhằm tránh spam với các giao dịch cung cấp 0 gas.

`CheckTx` **không** xử lý `sdk.Msg` — chúng chỉ cần được xử lý khi trạng thái chuẩn cần được cập nhật, điều này xảy ra trong `FinalizeBlock`.

Bước 2 và 3 được thực hiện bởi [`AnteHandler`](../beginner/04-gas-fees.md#antehandler) trong hàm [`RunTx()`](#runtx-antehandler-and-runmsgs), mà `CheckTx()` gọi với mode `runTxModeCheck`. Trong mỗi bước của `CheckTx()`, một [trạng thái biến động](#state-updates) đặc biệt gọi là `checkState` được cập nhật. Trạng thái này được dùng để theo dõi các thay đổi tạm thời kích hoạt bởi các lần gọi `CheckTx()` của mỗi giao dịch mà không làm thay đổi [trạng thái chuẩn chính](#main-state). Ví dụ, khi một giao dịch đi qua `CheckTx()`, phí giao dịch được trừ từ tài khoản của người gửi trong `checkState`. Nếu một giao dịch thứ hai được nhận từ cùng tài khoản trước khi giao dịch đầu tiên được xử lý, và tài khoản đã tiêu hết tiền trong `checkState` trong giao dịch đầu tiên, thì giao dịch thứ hai sẽ thất bại `CheckTx()` và bị từ chối. Dù thế nào, tài khoản của người gửi sẽ không thực sự trả phí cho đến khi giao dịch thực sự được đưa vào một block, vì `checkState` không bao giờ được commit vào trạng thái chính. `checkState` được reset về trạng thái mới nhất của trạng thái chính mỗi khi một block được [commit](#commit).

`CheckTx` trả về phản hồi cho consensus engine bên dưới có kiểu [`abci.CheckTxResponse`](https://github.com/cometbft/cometbft/blob/v0.37.x/spec/abci/abci++_methods.md#checktx). Phản hồi chứa:

* `Code (uint32)`: Mã phản hồi. `0` nếu thành công.
* `Data ([]byte)`: Byte kết quả, nếu có.
* `Log (string)`: Đầu ra của logger ứng dụng. Có thể không xác định.
* `Info (string)`: Thông tin bổ sung. Có thể không xác định.
* `GasWanted (int64)`: Lượng gas yêu cầu cho giao dịch. Được cung cấp bởi người dùng khi họ tạo giao dịch.
* `GasUsed (int64)`: Lượng gas tiêu thụ bởi giao dịch. Trong `CheckTx`, giá trị này được tính bằng cách nhân chi phí tiêu chuẩn của một byte giao dịch với kích thước của giao dịch thô. Ví dụ:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/x/auth/ante/basic.go#L104
```

* `Events ([]cmn.KVPair)`: Tag Key-Value để lọc và lập chỉ mục giao dịch (ví dụ: theo tài khoản). Xem thêm tại [`event`](./08-events.md).
* `Codespace (string)`: Namespace cho Code.

#### RecheckTx

Sau `Commit`, `CheckTx` được chạy lại trên tất cả các giao dịch còn lại trong mempool cục bộ của node, ngoại trừ các giao dịch đã được đưa vào block. Để ngăn mempool kiểm tra lại tất cả giao dịch mỗi khi một block được commit, có thể đặt tùy chọn cấu hình `mempool.recheck=false`. Kể từ Tendermint v0.32.1, một tham số `Type` bổ sung được cung cấp cho hàm `CheckTx` để cho biết giao dịch đến là mới (`CheckTxType_New`) hay là kiểm tra lại (`CheckTxType_Recheck`). Điều này cho phép bỏ qua một số kiểm tra như xác minh chữ ký trong `CheckTxType_Recheck`.

## RunTx, AnteHandler, RunMsgs, PostHandler

### RunTx

`RunTx` được gọi từ `CheckTx`/`FinalizeBlock` để xử lý giao dịch, với `execModeCheck` hoặc `execModeFinalize` là tham số để phân biệt hai chế độ thực thi. Lưu ý rằng khi `RunTx` nhận được một giao dịch, giao dịch đó đã được giải mã.

Điều đầu tiên `RunTx` thực hiện khi được gọi là lấy `CacheMultiStore` của `context` bằng cách gọi hàm `getContextForTx()` với mode phù hợp (hoặc `runTxModeCheck` hoặc `execModeFinalize`). `CacheMultiStore` này là một branch của store chính, có chức năng cache (cho các yêu cầu query), được khởi tạo trong `FinalizeBlock` cho việc thực thi giao dịch và trong `Commit` của block trước cho `CheckTx`. Sau đó, hai `defer func()` được gọi để quản lý [`gas`](../beginner/04-gas-fees.md). Chúng được thực thi khi `runTx` trả về và đảm bảo `gas` thực sự được tiêu thụ, đồng thời sẽ ném lỗi nếu có.

Sau đó, `RunTx()` gọi `ValidateBasic()`, khi có sẵn và để tương thích ngược, trên mỗi `sdk.Msg` trong `Tx`, thực hiện các kiểm tra hợp lệ _stateless_ sơ bộ. Nếu bất kỳ `sdk.Msg` nào không vượt qua `ValidateBasic()`, `RunTx()` trả về với lỗi.

Tiếp theo, [`anteHandler`](#antehandler) của ứng dụng được chạy (nếu tồn tại). Để chuẩn bị cho bước này, cả `context` của `checkState`/`finalizeBlockState` và `CacheMultiStore` của `context` đều được branch bằng hàm `cacheTxContext()`.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/baseapp/baseapp.go#L706-L722
```

Điều này cho phép `RunTx` không commit các thay đổi được thực hiện với trạng thái trong quá trình thực thi `anteHandler` nếu nó thất bại. Nó cũng ngăn module triển khai `anteHandler` ghi vào trạng thái, đây là một phần quan trọng của [object-capabilities](./10-ocap.md) trong Cosmos SDK.

Cuối cùng, hàm [`RunMsgs()`](#runmsgs) được gọi để xử lý các `sdk.Msg` trong `Tx`. Để chuẩn bị cho bước này, giống như với `anteHandler`, cả `context` của `checkState`/`finalizeBlockState` và `CacheMultiStore` của `context` đều được branch bằng hàm `cacheTxContext()`.

### AnteHandler

`AnteHandler` là handler đặc biệt triển khai interface `AnteHandler` và được dùng để xác thực giao dịch trước khi các message nội bộ của giao dịch được xử lý.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/types/handler.go#L3-L5
```

`AnteHandler` về lý thuyết là tùy chọn, nhưng vẫn là thành phần rất quan trọng của các mạng blockchain công cộng. Nó phục vụ 3 mục đích chính:

* Là tuyến phòng thủ đầu tiên chống spam và tuyến thứ hai (tuyến đầu tiên là mempool) chống replay giao dịch với việc trừ phí và kiểm tra [`sequence`](./01-transactions.md#transaction-generation).
* Thực hiện các kiểm tra hợp lệ _stateful_ sơ bộ như đảm bảo chữ ký hợp lệ hoặc người gửi có đủ tiền để trả phí.
* Đóng vai trò trong việc khuyến khích các stakeholder thông qua việc thu phí giao dịch.

`BaseApp` chứa một `anteHandler` như tham số được khởi tạo trong [constructor của ứng dụng](../beginner/00-app-anatomy.md#application-constructor). `anteHandler` được dùng rộng rãi nhất là [`auth` module](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/x/auth/ante/ante.go).

Nhấp vào [đây](../beginner/04-gas-fees.md#antehandler) để biết thêm về `anteHandler`.

### RunMsgs

`RunMsgs` được gọi từ `RunTx` với `runTxModeCheck` là tham số để kiểm tra sự tồn tại của route cho mỗi message trong giao dịch, và với `execModeFinalize` để thực sự xử lý các `sdk.Msg`.

Đầu tiên, nó lấy tên kiểu được định danh đầy đủ (fully-qualified type name) của `sdk.Msg` bằng cách kiểm tra `type_url` của Protobuf `Any` đại diện cho `sdk.Msg`. Sau đó, sử dụng [`msgServiceRouter`](#msg-service-router) của ứng dụng, nó kiểm tra sự tồn tại của phương thức `Msg` service liên quan đến `type_url` đó. Tại thời điểm này, nếu `mode == runTxModeCheck`, `RunMsgs` trả về. Ngược lại, nếu `mode == execModeFinalize`, RPC của [`Msg` service](../../build/building-modules/03-msg-services.md) được thực thi, trước khi `RunMsgs` trả về.

### PostHandler

`PostHandler` tương tự `AnteHandler`, nhưng như tên gọi, nó thực thi logic xử lý giao dịch tùy chỉnh sau khi [`RunMsgs`](#runmsgs) được gọi. `PostHandler` nhận `Result` của `RunMsgs` để cho phép hành vi tùy chỉnh này.

Giống như `AnteHandler`, `PostHandler` về lý thuyết là tùy chọn.

Các trường hợp sử dụng khác như hoàn lại gas chưa dùng cũng có thể được kích hoạt bởi `PostHandler`.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/x/auth/posthandler/post.go#L1-L15
```

Lưu ý: khi `PostHandler` thất bại, trạng thái từ `runMsgs` cũng bị revert, khiến giao dịch thất bại hoàn toàn.

## Các ABCI Message khác

### InitChain

[ABCI message `InitChain`](https://github.com/cometbft/cometbft/blob/v0.37.x/spec/abci/abci++_basic_concepts.md#method-overview) được gửi từ CometBFT engine bên dưới khi chuỗi được khởi động lần đầu. Nó chủ yếu được dùng để **khởi tạo** các tham số và trạng thái như:

* [Consensus Parameters](https://github.com/cometbft/cometbft/blob/v0.37.x/spec/abci/abci++_app_requirements.md#consensus-parameters) thông qua `setConsensusParams`.
* [`checkState` và `finalizeBlockState`](#state-updates) thông qua `setState`.
* [Block gas meter](../beginner/04-gas-fees.md#block-gas-meter), với gas vô hạn để xử lý các giao dịch genesis.

Cuối cùng, phương thức `InitChain(req abci.InitChainRequest)` của `BaseApp` gọi [`initChainer()`](../beginner/00-app-anatomy.md#initchainer) của ứng dụng để khởi tạo trạng thái chính của ứng dụng từ `genesis file` và, nếu được định nghĩa, gọi hàm [`InitGenesis`](../../build/building-modules/08-genesis.md#initgenesis) của từng module trong ứng dụng.

### FinalizeBlock

[ABCI message `FinalizeBlock`](https://github.com/cometbft/cometbft/blob/v0.38.x/spec/abci/abci++_basic_concepts.md#method-overview) được gửi từ CometBFT engine bên dưới khi một block proposal được tạo bởi proposer đúng đắn được nhận. Các lần gọi `BeginBlock`, `DeliverTx` và `EndBlock` trước đây là các phương thức private trên struct BaseApp.

```go reference 
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/baseapp/abci.go#L869
```

#### PreBlock

* Chạy [`preBlocker()`](../beginner/00-app-anatomy.md#preblocker) của ứng dụng, chủ yếu chạy phương thức [`PreBlocker()`](../../build/building-modules/17-preblock.md#preblock) của mỗi module.

#### BeginBlock

* Khởi tạo [`finalizeBlockState`](#state-updates) với header mới nhất bằng cách sử dụng `req abci.FinalizeBlockRequest` được truyền vào như tham số thông qua hàm `setState`.

  ```go reference
  https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/baseapp/baseapp.go#L746-L770
  ```

  Hàm này cũng reset [main gas meter](../beginner/04-gas-fees.md#main-gas-meter).

* Khởi tạo [block gas meter](../beginner/04-gas-fees.md#block-gas-meter) với giới hạn `maxGas`. `gas` tiêu thụ trong block không thể vượt quá `maxGas`. Tham số này được định nghĩa trong consensus parameters của ứng dụng.
* Chạy [`beginBlocker()`](../beginner/00-app-anatomy.md#beginblocker-and-endblocker) của ứng dụng, chủ yếu chạy phương thức [`BeginBlocker()`](../../build/building-modules/06-beginblock-endblock.md#beginblock) của mỗi module.
* Đặt [`VoteInfos`](https://github.com/cometbft/cometbft/blob/v0.37.x/spec/abci/abci++_methods.md#voteinfo) của ứng dụng, tức là danh sách các validator có _precommit_ cho block trước đã được proposer của block hiện tại đưa vào. Thông tin này được truyền vào [`Context`](./02-context.md) để có thể sử dụng trong quá trình thực thi giao dịch và EndBlock.

#### Thực thi giao dịch

Khi consensus engine bên dưới nhận được một block proposal, mỗi giao dịch trong block cần được ứng dụng xử lý. Vì vậy, consensus engine bên dưới gửi các giao dịch trong message FinalizeBlock đến ứng dụng cho mỗi giao dịch theo thứ tự tuần tự.

Trước khi giao dịch đầu tiên của một block nhất định được xử lý, một [trạng thái biến động](#state-updates) gọi là `finalizeBlockState` được khởi tạo trong FinalizeBlock. Trạng thái này được cập nhật mỗi khi một giao dịch được xử lý qua `FinalizeBlock`, và được commit vào [trạng thái chính](#main-state) khi block được [commit](#commit), sau đó nó được đặt về `nil`.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/baseapp/baseapp.go#L772-L807
```

Việc thực thi giao dịch trong `FinalizeBlock` thực hiện **các bước hoàn toàn giống với `CheckTx`**, với một lưu ý nhỏ ở bước 3 và thêm một bước thứ năm:

1. `AnteHandler` **không** kiểm tra xem `gas-prices` của giao dịch có đủ không. Đó là vì giá trị `min-gas-prices` mà `gas-prices` được kiểm tra là cục bộ đối với node, và do đó điều đủ cho một full-node có thể không đủ với một full-node khác. Điều này có nghĩa là proposer có thể bao gồm các giao dịch miễn phí, mặc dù họ không có động lực để làm vậy vì họ kiếm được tiền thưởng trên tổng phí của block mà họ đề xuất.
2. Đối với mỗi `sdk.Msg` trong giao dịch, định tuyến đến Protobuf [`Msg` service](../../build/building-modules/03-msg-services.md) của module phù hợp. Các kiểm tra _stateful_ bổ sung được thực hiện, và branched multistore trong `context` của `finalizeBlockState` được cập nhật bởi `keeper` của module. Nếu `Msg` service trả về thành công, branched multistore trong `context` được ghi vào `CacheMultiStore` của `finalizeBlockState`.

Trong bước thứ năm bổ sung được mô tả ở (2), mỗi thao tác đọc/ghi vào store tăng giá trị của `GasConsumed`. Bạn có thể tìm thấy chi phí mặc định của mỗi thao tác:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/store/types/gas.go#L230-L241
```

Bất cứ lúc nào, nếu `GasConsumed > GasWanted`, hàm trả về với `Code != 0` và việc thực thi thất bại.

Mỗi giao dịch trả về phản hồi cho consensus engine bên dưới có kiểu [`abci.ExecTxResult`](https://github.com/cometbft/cometbft/blob/v0.38.0-rc1/spec/abci/abci%2B%2B_methods.md#exectxresult). Phản hồi chứa:

* `Code (uint32)`: Mã phản hồi. `0` nếu thành công.
* `Data ([]byte)`: Byte kết quả, nếu có.
* `Log (string)`: Đầu ra của logger ứng dụng. Có thể không xác định.
* `Info (string)`: Thông tin bổ sung. Có thể không xác định.
* `GasWanted (int64)`: Lượng gas yêu cầu cho giao dịch. Được cung cấp bởi người dùng khi họ tạo giao dịch.
* `GasUsed (int64)`: Lượng gas tiêu thụ bởi giao dịch. Trong quá trình thực thi giao dịch, giá trị này được tính bằng cách nhân chi phí tiêu chuẩn của một byte giao dịch với kích thước của giao dịch thô, cộng với lượng gas mỗi khi có thao tác đọc/ghi vào store.
* `Events ([]cmn.KVPair)`: Tag Key-Value để lọc và lập chỉ mục giao dịch (ví dụ: theo tài khoản). Xem thêm tại [`event`](./08-events.md).
* `Codespace (string)`: Namespace cho Code.

#### EndBlock

EndBlock được chạy sau khi quá trình thực thi giao dịch hoàn tất. Nó cho phép các nhà phát triển có logic được thực thi vào cuối mỗi block. Trong Cosmos SDK, phương thức EndBlock() chính là để chạy EndBlocker() của ứng dụng, chủ yếu chạy phương thức EndBlocker() của mỗi module trong ứng dụng.

```go reference 
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/baseapp/baseapp.go#L811-L833
```

### Commit

[ABCI message `Commit`](https://github.com/cometbft/cometbft/blob/v0.37.x/spec/abci/abci++_basic_concepts.md#method-overview) được gửi từ CometBFT engine bên dưới sau khi full-node nhận được _precommit_ từ 2/3+ validator (theo trọng số biểu quyết). Về phía `BaseApp`, hàm `Commit(res abci.CommitResponse)` được triển khai để commit tất cả các chuyển đổi trạng thái hợp lệ xảy ra trong `FinalizeBlock` và reset trạng thái cho block tiếp theo.

Để commit các chuyển đổi trạng thái, hàm `Commit` gọi hàm `Write()` trên `finalizeBlockState.ms`, trong đó `finalizeBlockState.ms` là một branched multistore của store chính `app.cms`. Sau đó, hàm `Commit` đặt `checkState` thành header mới nhất (lấy từ `finalizeBlockState.ctx.BlockHeader`) và `finalizeBlockState` thành `nil`.

Cuối cùng, `Commit` trả về hash của commitment của `app.cms` lại cho consensus engine bên dưới. Hash này được dùng làm tham chiếu trong header của block tiếp theo.

### Info

[ABCI message `Info`](https://github.com/cometbft/cometbft/blob/v0.37.x/spec/abci/abci++_basic_concepts.md#info-methods) là một query đơn giản từ consensus engine bên dưới, đáng chú ý là được dùng để đồng bộ consensus engine với ứng dụng trong quá trình bắt tay (handshake) xảy ra khi khởi động. Khi được gọi, hàm `Info(res abci.InfoResponse)` từ `BaseApp` sẽ trả về tên, phiên bản của ứng dụng và hash của commit gần nhất của `app.cms`.

### Query

[ABCI message `Query`](https://github.com/cometbft/cometbft/blob/v0.37.x/spec/abci/abci++_basic_concepts.md#info-methods) được dùng để phục vụ các query nhận từ consensus engine bên dưới, bao gồm các query nhận qua RPC như CometBFT RPC. Trước đây nó là điểm vào (entrypoint) chính để xây dựng giao diện với ứng dụng, nhưng với sự ra đời của [gRPC query](../../build/building-modules/04-query-services.md) trong Cosmos SDK v0.40, cách sử dụng của nó bị hạn chế hơn. Ứng dụng phải tuân thủ một số quy tắc khi triển khai phương thức `Query`, được trình bày [ở đây](https://github.com/cometbft/cometbft/blob/v0.37.x/spec/abci/abci++_app_requirements.md#query).

Mỗi `query` của CometBFT đi kèm với một `path`, là một `string` biểu thị cần query gì. Nếu `path` khớp với một service method gRPC được định danh đầy đủ, thì `BaseApp` sẽ chuyển query đến `grpcQueryRouter` và để nó xử lý như giải thích [ở trên](#grpc-query-router). Nếu không, `path` đại diện cho một query chưa (được) xử lý bởi gRPC router. `BaseApp` tách chuỗi `path` bằng ký tự phân tách `/`. Theo quy ước, phần tử đầu tiên của chuỗi được tách (`split[0]`) chứa danh mục của `query` (`app`, `p2p`, `store` hoặc `custom`). Triển khai `Query(req abci.QueryRequest)` của `BaseApp` là một dispatcher đơn giản phục vụ 4 danh mục query chính này:

* Các query liên quan đến ứng dụng như query phiên bản ứng dụng, được phục vụ thông qua phương thức `handleQueryApp`.
* Các query trực tiếp đến multistore, được phục vụ bởi phương thức `handlerQueryStore`. Các query trực tiếp này khác với custom query đi qua `app.queryRouter`, và chủ yếu được sử dụng bởi các nhà cung cấp dịch vụ bên thứ ba như block explorer.
* Các P2P query, được phục vụ thông qua phương thức `handleQueryP2P`. Các query này trả về `app.addrPeerFilter` hoặc `app.ipPeerFilter` chứa danh sách các peer được lọc theo địa chỉ hoặc IP tương ứng. Các danh sách này được khởi tạo đầu tiên thông qua `options` trong [constructor](#constructor) của `BaseApp`.

### ExtendVote

`ExtendVote` cho phép ứng dụng mở rộng một phiếu pre-commit với dữ liệu tùy ý. Quá trình này KHÔNG cần phải xác định (deterministic) và dữ liệu trả về có thể là duy nhất cho từng tiến trình validator.

Trong Cosmos SDK, điều này được triển khai là NoOp:

```go reference 
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/baseapp/abci_utils.go#L444-L450
```

### VerifyVoteExtension

`VerifyVoteExtension` cho phép ứng dụng xác minh rằng dữ liệu được trả về bởi `ExtendVote` là hợp lệ. Quá trình này PHẢI là xác định (deterministic). Hơn nữa, giá trị của `ResponseVerifyVoteExtension.status` PHẢI phụ thuộc hoàn toàn vào các tham số được truyền vào trong lần gọi `RequestVerifyVoteExtension`, và trạng thái ứng dụng đã được commit gần nhất.

Trong Cosmos SDK, điều này được triển khai là NoOp:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/baseapp/abci_utils.go#L452-L458
```
