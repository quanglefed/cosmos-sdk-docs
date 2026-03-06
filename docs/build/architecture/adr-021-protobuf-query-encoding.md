# ADR 021: Mã Hóa Truy Vấn Protocol Buffer

## Nhật Ký Thay Đổi

* 27 tháng 3 năm 2020: Bản nháp đầu tiên

## Trạng Thái

Đã Chấp Nhận

## Bối Cảnh

ADR này là phần tiếp nối của động lực, thiết kế và bối cảnh được thiết lập trong [ADR 019](./adr-019-protobuf-state-encoding.md) và [ADR 020](./adr-020-protobuf-transaction-encoding.md), cụ thể là chúng ta hướng tới việc thiết kế lộ trình chuyển đổi Protocol Buffer cho phía client của Cosmos SDK.

ADR này tiếp nối từ [ADR 020](./adr-020-protobuf-transaction-encoding.md) để chỉ định mã hóa cho các truy vấn.

## Quyết Định

### Định Nghĩa Truy Vấn Tùy Chỉnh

Các module định nghĩa truy vấn tùy chỉnh thông qua định nghĩa `service` của protocol buffers. Các định nghĩa `service` này thường được liên kết và sử dụng bởi giao thức GRPC. Tuy nhiên, đặc tả protocol buffers chỉ ra rằng chúng có thể được sử dụng tổng quát hơn bởi bất kỳ giao thức yêu cầu/phản hồi nào sử dụng mã hóa protocol buffer. Do đó, chúng ta có thể sử dụng định nghĩa `service` để chỉ định các truy vấn ABCI tùy chỉnh và thậm chí tái sử dụng phần lớn cơ sở hạ tầng GRPC.

Mỗi module có truy vấn tùy chỉnh nên định nghĩa một service có tên chuẩn là `Query`:

```protobuf
// x/bank/types/types.proto

service Query {
  rpc QueryBalance(QueryBalanceParams) returns (cosmos_sdk.v1.Coin) { }
  rpc QueryAllBalances(QueryAllBalancesParams) returns (QueryAllBalancesResponse) { }
}
```

#### Xử Lý Các Kiểu Interface

Các module sử dụng kiểu interface và cần tính đa hình thực sự thường đẩy một `oneof` lên cấp app để cung cấp tập hợp các triển khai cụ thể của interface đó mà app hỗ trợ. Mặc dù các app có thể làm tương tự với truy vấn và triển khai một query service cấp app, nhưng khuyến nghị rằng các module nên cung cấp các phương thức truy vấn để lộ các interface này thông qua `google.protobuf.Any`. Có một lo ngại ở cấp giao dịch rằng chi phí của `Any` quá cao để biện minh cho việc sử dụng nó. Tuy nhiên, với truy vấn thì không đáng lo, và việc cung cấp các truy vấn cấp module tổng quát sử dụng `Any` không loại trừ các app khỏi việc cũng cung cấp các truy vấn cấp app sử dụng `oneof` cấp app.

Một ví dụ giả thuyết cho module `gov` sẽ trông như sau:

```protobuf
// x/gov/types/types.proto

import "google/protobuf/any.proto";

service Query {
  rpc GetProposal(GetProposalParams) returns (AnyProposal) { }
}

message AnyProposal {
  ProposalBase base = 1;
  google.protobuf.Any content = 2;
}
```

### Triển Khai Truy Vấn Tùy Chỉnh

Để triển khai query service, chúng ta có thể tái sử dụng plugin grpc của [gogo protobuf](https://github.com/cosmos/gogoproto) hiện có, plugin này cho một service có tên `Query` sẽ tạo ra một interface tên `QueryServer` như sau:

```go
type QueryServer interface {
	QueryBalance(context.Context, *QueryBalanceParams) (*types.Coin, error)
	QueryAllBalances(context.Context, *QueryAllBalancesParams) (*QueryAllBalancesResponse, error)
}
```

Các truy vấn tùy chỉnh cho module của chúng ta được triển khai bằng cách hiện thực interface này.

Tham số đầu tiên trong interface được tạo ra này là `context.Context` chung, trong khi các phương thức querier thường cần một thể hiện của `sdk.Context` để đọc từ store. Vì các giá trị tùy ý có thể được gắn vào `context.Context` sử dụng các phương thức `WithValue` và `Value`, Cosmos SDK nên cung cấp hàm `sdk.UnwrapSDKContext` để lấy `sdk.Context` từ `context.Context` được cung cấp.

Một ví dụ triển khai `QueryBalance` cho module bank như trên sẽ trông như thế này:

```go
type Querier struct {
	Keeper
}

func (q Querier) QueryBalance(ctx context.Context, params *types.QueryBalanceParams) (*sdk.Coin, error) {
	balance := q.GetBalance(sdk.UnwrapSDKContext(ctx), params.Address, params.Denom)
	return &balance, nil
}
```

### Đăng Ký và Định Tuyến Truy Vấn Tùy Chỉnh

Các triển khai query server như trên sẽ được đăng ký với `AppModule` sử dụng phương thức mới `RegisterQueryService(grpc.Server)` có thể được triển khai đơn giản như sau:

```go
// x/bank/module.go
func (am AppModule) RegisterQueryService(server grpc.Server) {
	types.RegisterQueryServer(server, keeper.Querier{am.keeper})
}
```

Bên dưới, một phương thức mới `RegisterService(sd *grpc.ServiceDesc, handler interface{})` sẽ được thêm vào `baseapp.QueryRouter` hiện có để thêm các truy vấn vào bảng định tuyến truy vấn tùy chỉnh (với phương thức định tuyến được mô tả bên dưới). Chữ ký của phương thức này khớp với phương thức `RegisterServer` hiện có trên kiểu `Server` của GRPC trong đó `handler` là triển khai query server tùy chỉnh được mô tả ở trên.

Các yêu cầu dạng GRPC được định tuyến bởi tên service (vd: `cosmos_sdk.x.bank.v1.Query`) và tên phương thức (vd: `QueryBalance`) kết hợp với `/` để tạo thành tên phương thức đầy đủ (vd: `/cosmos_sdk.x.bank.v1.Query/QueryBalance`). Điều này được dịch sang truy vấn ABCI là `custom/cosmos_sdk.x.bank.v1.Query/QueryBalance`. Các service handler đăng ký với `QueryRouter.RegisterService` sẽ được định tuyến theo cách này.

Ngoài tên phương thức, các yêu cầu GRPC mang theo payload được mã hóa protobuf, ánh xạ tự nhiên tới `RequestQuery.Data`, và nhận phản hồi hoặc lỗi được mã hóa protobuf. Do đó có một ánh xạ khá tự nhiên từ các phương thức rpc dạng GRPC sang cơ sở hạ tầng `sdk.Query` và `QueryRouter` hiện có.

Đặc tả cơ bản này cho phép chúng ta tái sử dụng định nghĩa `service` của protocol buffer cho các truy vấn ABCI tùy chỉnh, giảm đáng kể nhu cầu giải mã và mã hóa thủ công trong các phương thức truy vấn.

### Hỗ Trợ Giao Thức GRPC

Ngoài việc cung cấp đường dẫn truy vấn ABCI, chúng ta có thể dễ dàng cung cấp một proxy server GRPC định tuyến các yêu cầu theo giao thức GRPC sang các yêu cầu truy vấn ABCI bên dưới. Theo cách này, các client có thể sử dụng các triển khai GRPC hiện có của ngôn ngữ host của họ để thực hiện các truy vấn trực tiếp đối với các app Cosmos SDK sử dụng các định nghĩa `service` này. Để server này hoạt động, `QueryRouter` trên `BaseApp` sẽ cần lộ các service handler đã đăng ký với `QueryRouter.RegisterService` cho triển khai proxy server. Các node có thể khởi chạy proxy server trên một cổng riêng biệt trong cùng một tiến trình với app ABCI sử dụng cờ dòng lệnh.

### Truy Vấn REST và Tạo Swagger

[grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway) là một dự án dịch các lời gọi REST thành lời gọi GRPC sử dụng các chú thích đặc biệt trên các phương thức service. Các module muốn lộ truy vấn REST nên thêm chú thích `google.api.http` vào các phương thức `rpc` của chúng như trong ví dụ dưới đây.

```protobuf
// x/bank/types/types.proto

service Query {
  rpc QueryBalance(QueryBalanceParams) returns (cosmos_sdk.v1.Coin) {
    option (google.api.http) = {
      get: "/x/bank/v1/balance/{address}/{denom}"
    };
  }
  rpc QueryAllBalances(QueryAllBalancesParams) returns (QueryAllBalancesResponse) {
    option (google.api.http) = {
      get: "/x/bank/v1/balances/{address}"
    };
  }
}
```

grpc-gateway sẽ hoạt động trực tiếp với proxy GRPC mô tả ở trên, proxy này sẽ dịch các yêu cầu sang truy vấn ABCI bên dưới. grpc-gateway cũng có thể tự động tạo định nghĩa Swagger.

Trong triển khai hiện tại của truy vấn REST, mỗi module cần triển khai truy vấn REST thủ công ngoài các phương thức querier ABCI. Sử dụng cách tiếp cận grpc-gateway, sẽ không cần tạo các handler truy vấn REST riêng biệt, chỉ cần các query server như mô tả ở trên vì grpc-gateway xử lý việc dịch protobuf sang REST cũng như định nghĩa Swagger.

Cosmos SDK nên cung cấp các lệnh CLI cho các app để khởi động GRPC gateway hoặc trong một tiến trình riêng biệt hoặc cùng tiến trình với app ABCI, cũng như cung cấp lệnh để tạo các file `.proto` proxy grpc-gateway và file `swagger.json`.

### Sử Dụng Phía Client

Plugin grpc của gogo protobuf tạo ra các interface client ngoài các interface server. Với service `Query` định nghĩa ở trên, chúng ta sẽ nhận được interface `QueryClient` như:

```go
type QueryClient interface {
	QueryBalance(ctx context.Context, in *QueryBalanceParams, opts ...grpc.CallOption) (*types.Coin, error)
	QueryAllBalances(ctx context.Context, in *QueryAllBalancesParams, opts ...grpc.CallOption) (*QueryAllBalancesResponse, error)
}
```

Thông qua một bản vá nhỏ cho gogo protobuf ([gogo/protobuf#675](https://github.com/gogo/protobuf/pull/675)), chúng ta đã điều chỉnh codegen grpc để sử dụng interface thay vì kiểu cụ thể cho struct client được tạo ra. Điều này cho phép chúng ta cũng tái sử dụng cơ sở hạ tầng GRPC cho các truy vấn client ABCI.

`Context` sẽ nhận một phương thức mới `QueryConn` trả về `ClientConn` định tuyến lời gọi tới các truy vấn ABCI.

Các client (như các phương thức CLI) sau đó có thể gọi các phương thức truy vấn như thế này:

```go
clientCtx := client.NewContext()
queryClient := types.NewQueryClient(clientCtx.QueryConn())
params := &types.QueryBalanceParams{addr, denom}
result, err := queryClient.QueryBalance(gocontext.Background(), params)
```

### Kiểm Thử

Các test có thể tạo một query client trực tiếp từ các tham chiếu keeper và `sdk.Context` sử dụng `QueryServerTestHelper` như sau:

```go
queryHelper := baseapp.NewQueryServerTestHelper(ctx)
types.RegisterQueryServer(queryHelper, keeper.Querier{app.BankKeeper})
queryClient := types.NewQueryClient(queryHelper)
```

## Cải Tiến Tương Lai

## Hậu Quả

### Tích Cực

* Việc triển khai querier đơn giản hơn nhiều (không cần mã hóa/giải mã thủ công)
* Tạo query client dễ dàng (có thể sử dụng các công cụ grpc và swagger hiện có)
* Không cần triển khai truy vấn REST
* Các phương thức truy vấn an toàn về kiểu dữ liệu (được tạo bởi plugin grpc)
* Trong tương lai, sẽ ít phá vỡ các phương thức truy vấn hơn do các đảm bảo tương thích ngược được cung cấp bởi buf

### Tiêu Cực

* Tất cả các client sử dụng các truy vấn ABCI/REST hiện có sẽ cần được tái cấu trúc cho cả các đường dẫn truy vấn GRPC/REST mới lẫn dữ liệu được mã hóa protobuf/proto-json, nhưng điều này ít nhiều là không thể tránh khỏi trong việc tái cấu trúc protobuf

### Trung Lập

## Tham Khảo
