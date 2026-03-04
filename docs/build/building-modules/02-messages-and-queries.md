---
sidebar_position: 1
---

# Messages và Queries

:::note Tóm tắt
`Msg` và `Queries` là hai đối tượng chính được xử lý bởi các module. Hầu hết các thành phần core được định nghĩa trong một module, như `Msg` services, `keeper` và `Query` services, đều tồn tại để xử lý các `message` và `query`.
:::

:::note Yêu Cầu Đọc Trước

* [Giới thiệu về Cosmos SDK Modules](./00-intro.md)

:::

## Messages

`Msg` là các đối tượng có mục đích cuối cùng là kích hoạt các chuyển đổi trạng thái. Chúng được bọc trong các [giao dịch](../../learn/advanced/01-transactions.md), mỗi giao dịch có thể chứa một hoặc nhiều `Msg`.

Khi một giao dịch được chuyển tiếp từ consensus engine bên dưới đến ứng dụng Cosmos SDK, nó được giải mã trước tiên bởi [`BaseApp`](../../learn/advanced/00-baseapp.md). Sau đó, mỗi message chứa trong giao dịch được trích xuất và định tuyến đến module thích hợp thông qua `MsgServiceRouter` của `BaseApp` để có thể được xử lý bởi [`Msg` service](./03-msg-services.md) của module. Để giải thích chi tiết hơn về vòng đời của một giao dịch, nhấn vào [đây](../../learn/beginner/01-tx-lifecycle.md).

### `Msg` Services

Định nghĩa các Protobuf `Msg` service là cách được khuyến nghị để xử lý các message. Một Protobuf `Msg` service nên được tạo cho mỗi module, thường trong `tx.proto` (xem thêm về [quy ước và đặt tên](../../learn/advanced/05-encoding.md#faq)). Nó phải có một phương thức RPC service được định nghĩa cho mỗi message trong module.

Mỗi phương thức `Msg` service phải có đúng một đối số, phải triển khai interface `sdk.Msg`, và một phản hồi Protobuf. Quy ước đặt tên là gọi đối số RPC là `Msg<service-rpc-name>` và phản hồi RPC là `Msg<service-rpc-name>Response`. Ví dụ:

```protobuf
  rpc Send(MsgSend) returns (MsgSendResponse);
```

Xem ví dụ về định nghĩa `Msg` service từ module `x/bank`:

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/proto/cosmos/bank/v1beta1/tx.proto#L13-L36
```

### Interface `sdk.Msg`

`sdk.Msg` là một alias của `proto.Message`.

Để gắn phương thức `ValidateBasic()` vào một message, bạn phải thêm các phương thức vào kiểu tuân thủ `HasValidateBasic`.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/9c1e8b247cd47b5d3decda6e86fbc3bc996ee5d7/types/tx_msg.go#L84-L88
```

Trong phiên bản 0.50+, các signer từ lời gọi `GetSigners()` được tự động hóa qua annotation protobuf.

Đọc thêm về trường signer [tại đây](./05-protobuf-annotations.md).

```protobuf reference 
https://github.com/cosmos/cosmos-sdk/blob/e6848d99b55a65d014375b295bdd7f9641aac95e/proto/cosmos/bank/v1beta1/tx.proto#L40
```

Nếu cần signer tùy chỉnh thì có một con đường thay thế có thể thực hiện. Có thể định nghĩa một hàm trả về `signing.CustomGetSigner` cho một message cụ thể.

```go
func ProvideBankSendTransactionGetSigners() signing.CustomGetSigner {

			// Trích xuất signer từ chữ ký.
			signer, err := coretypes.LatestSigner(Tx).Sender(ethTx)
      if err != nil {
				return nil, err
			}

			// Trả về signer ở định dạng yêu cầu.
			return [][]byte{signer.Bytes()}, nil
}
```

Khi sử dụng dependency injection (depinject), điều này có thể được cung cấp cho ứng dụng thông qua phương thức provide.

```go
depinject.Provide(banktypes.ProvideBankSendTransactionGetSigners)
```

Cosmos SDK sử dụng các định nghĩa Protobuf để tạo mã client và server:

* Interface `MsgServer` định nghĩa API server cho `Msg` service và việc triển khai của nó được mô tả trong tài liệu [`Msg` services](./03-msg-services.md).
* Các struct được tạo ra cho tất cả các loại yêu cầu và phản hồi RPC.

Một phương thức `RegisterMsgServer` cũng được tạo ra và nên được sử dụng để đăng ký triển khai `MsgServer` của module trong phương thức `RegisterServices` từ [`AppModule` interface](./01-module-manager.md#appmodule).

Để các client (CLI và grpc-gateway) có các URL này được đăng ký, Cosmos SDK cung cấp hàm `RegisterMsgServiceDesc(registry codectypes.InterfaceRegistry, sd *grpc.ServiceDesc)` nên được gọi bên trong phương thức [`RegisterInterfaces`](01-module-manager.md#appmodulebasic) của module, sử dụng `&_Msg_serviceDesc` được tạo từ proto làm đối số `*grpc.ServiceDesc`.


## Queries

`query` là một yêu cầu thông tin được thực hiện bởi người dùng cuối của ứng dụng thông qua một interface và được xử lý bởi một full-node. Một `query` được nhận bởi full-node thông qua consensus engine của nó và được chuyển tiếp đến ứng dụng qua ABCI. Sau đó nó được định tuyến đến module thích hợp thông qua `QueryRouter` của `BaseApp` để có thể được xử lý bởi query service của module (./04-query-services.md). Để xem chi tiết hơn về vòng đời của một `query`, nhấn vào [đây](../../learn/beginner/02-query-lifecycle.md).

### gRPC Queries

Các query nên được định nghĩa bằng cách sử dụng [Protobuf services](https://developers.google.com/protocol-buffers/docs/proto#services). Một `Query` service nên được tạo cho mỗi module trong `query.proto`. Service này liệt kê các endpoint bắt đầu bằng `rpc`.

Dưới đây là ví dụ về định nghĩa `Query` service như vậy:

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/proto/cosmos/auth/v1beta1/query.proto#L14-L89
```

Là các `proto.Message`, các kiểu `Response` được tạo ra mặc định triển khai phương thức `String()` của [`fmt.Stringer`](https://pkg.go.dev/fmt#Stringer).

Một phương thức `RegisterQueryServer` cũng được tạo ra và nên được sử dụng để đăng ký query server của module trong phương thức `RegisterServices` từ [`AppModule` interface](./01-module-manager.md#appmodule).

### Legacy Queries (Queries Kế Thừa)

Trước khi Protobuf và gRPC được giới thiệu trong Cosmos SDK, thường không có đối tượng `query` cụ thể nào được định nghĩa bởi nhà phát triển module, trái ngược với `message`. Thay vào đó, Cosmos SDK áp dụng cách tiếp cận đơn giản hơn là sử dụng một `path` đơn giản để định nghĩa mỗi `query`. `path` chứa loại `query` và tất cả các đối số cần thiết để xử lý nó. Đối với hầu hết các query module, `path` nên có dạng như sau:

```text
queryCategory/queryRoute/queryType/arg1/arg2/...
```

trong đó:

* `queryCategory` là danh mục của `query`, thường là `custom` cho các query module. Nó được dùng để phân biệt giữa các loại query khác nhau trong [`phương thức Query`](../../learn/advanced/00-baseapp.md#query) của `BaseApp`.
* `queryRoute` được sử dụng bởi [`queryRouter`](../../learn/advanced/00-baseapp.md#query-routing) của `BaseApp` để ánh xạ `query` đến module của nó. Thường, `queryRoute` phải là tên của module.
* `queryType` được sử dụng bởi [`querier`](./04-query-services.md#legacy-queriers) của module để ánh xạ `query` đến hàm querier thích hợp trong module.
* `args` là các đối số thực tế cần thiết để xử lý `query`. Chúng được người dùng cuối điền vào. Lưu ý rằng đối với các query lớn hơn, bạn có thể muốn truyền đối số trong trường `Data` của yêu cầu `req` thay vì trong `path`.

`path` cho mỗi `query` phải được định nghĩa bởi nhà phát triển module trong [file giao diện dòng lệnh](./09-module-interfaces.md#query-commands) của module. Nhìn chung, có 3 thành phần chính mà nhà phát triển module cần triển khai để làm cho tập con trạng thái được định nghĩa bởi module của họ có thể query được:

* Một [`querier`](./04-query-services.md#legacy-queriers), để xử lý `query` sau khi nó đã được [định tuyến đến module](../../learn/advanced/00-baseapp.md#query-routing).
* [Các lệnh query](./09-module-interfaces.md#query-commands) trong file CLI của module, nơi `path` cho mỗi `query` được chỉ định.
* Các kiểu trả về của `query`. Thường được định nghĩa trong file `types/querier.go`, chúng chỉ định kiểu kết quả của mỗi `query` trong module. Các kiểu tùy chỉnh này phải triển khai phương thức `String()` của [`fmt.Stringer`](https://pkg.go.dev/fmt#Stringer).

### Store Queries

Store queries truy cập trực tiếp các store key. Chúng sử dụng `clientCtx.QueryABCI(req abci.QueryRequest)` để trả về `abci.QueryResponse` đầy đủ với Merkle proofs đi kèm.

Xem các ví dụ sau:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/baseapp/abci.go#L864-L894
```
