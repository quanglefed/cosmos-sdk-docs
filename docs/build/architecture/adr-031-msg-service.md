# ADR 031: Protobuf Msg Services

## Nhật Ký Thay Đổi

* 05-10-2020: Bản nháp đầu tiên
* 21-04-2021: Xóa `ServiceMsg` để tuân theo spec của `Any` của Protobuf, xem [#9063](https://github.com/cosmos/cosmos-sdk/issues/9063).

## Trạng Thái

Đã Chấp Nhận

## Tóm Tắt

Chúng ta muốn tận dụng định nghĩa `service` của protobuf để định nghĩa `Msg`, điều này sẽ mang lại cho chúng ta những cải tiến UX dành cho nhà phát triển đáng kể về mặt code được tạo ra và thực tế là các kiểu trả về giờ sẽ được định nghĩa rõ ràng.

## Bối Cảnh

Hiện tại các handler `Msg` trong Cosmos SDK có các giá trị trả về được đặt trong trường `data` của phản hồi. Tuy nhiên, các giá trị trả về này không được chỉ định ở bất kỳ đâu ngoài code handler golang.

Trong các cuộc thảo luận ban đầu, [đã được đề xuất](https://docs.google.com/document/d/1eEgYgvgZqLE45vETjhwIw4VOqK-5hwQtZtjVbiXnIGc/edit) rằng các kiểu trả về `Msg` được nắm bắt bằng cách sử dụng trường mở rộng protobuf, ví dụ:

```protobuf
package cosmos.gov;

message MsgSubmitProposal
	option (cosmos_proto.msg_return) = "uint64";
	string delegator_address = 1;
	string validator_address = 2;
	repeated sdk.Coin amount = 3;
}
```

Tuy nhiên điều này chưa bao giờ được áp dụng.

Có một giá trị trả về được chỉ định rõ ràng cho `Msg` sẽ cải thiện UX của client. Ví dụ, trong `x/gov`, `MsgSubmitProposal` trả về ID đề xuất dưới dạng big-endian `uint64`. Điều này không thực sự được ghi lại ở đâu và các client cần biết nội bộ của Cosmos SDK để phân tích giá trị đó và trả về cho người dùng.

Ngoài ra, có thể có các trường hợp chúng ta muốn sử dụng các giá trị trả về này theo cách lập trình. Ví dụ, https://github.com/cosmos/cosmos-sdk/issues/7093 đề xuất một phương thức để thực hiện Ocaps liên module sử dụng router `Msg`. Một kiểu trả về được định nghĩa rõ ràng sẽ cải thiện UX nhà phát triển cho cách tiếp cận này.

Ngoài ra, việc đăng ký handler của các kiểu `Msg` có xu hướng thêm một chút mã boilerplate trên các keeper và thường được thực hiện thông qua type switch thủ công. Điều này không nhất thiết là xấu, nhưng nó thêm overhead khi tạo các module.

## Quyết Định

Chúng ta quyết định sử dụng định nghĩa `service` của protobuf để định nghĩa `Msg` cũng như code được tạo ra bởi chúng như là sự thay thế cho các handler `Msg`.

Dưới đây chúng ta định nghĩa điều này sẽ trông như thế nào đối với message `SubmitProposal` từ module `x/gov`. Chúng ta bắt đầu với định nghĩa `service` `Msg`:

```protobuf
package cosmos.gov;

service Msg {
  rpc SubmitProposal(MsgSubmitProposal) returns (MsgSubmitProposalResponse);
}

// Lưu ý rằng để tương thích ngược, điều này sử dụng MsgSubmitProposal làm loại yêu cầu
// thay vì MsgSubmitProposalRequest chuẩn hơn
message MsgSubmitProposal {
  google.protobuf.Any content = 1;
  string proposer = 2;
}

message MsgSubmitProposalResponse {
  uint64 proposal_id;
}
```

Mặc dù điều này thường được sử dụng nhất cho gRPC, việc quá tải các định nghĩa `service` của protobuf như thế này không vi phạm ý định của [spec protobuf](https://developers.google.com/protocol-buffers/docs/proto3#services) nói rằng:
> Nếu bạn không muốn sử dụng gRPC, cũng có thể sử dụng protocol buffers với triển khai RPC riêng của bạn.

Với cách tiếp cận này, chúng ta sẽ nhận được interface `MsgServer` được tạo tự động:

Ngoài việc chỉ định rõ ràng các kiểu trả về, điều này có lợi ích là tạo ra code client và server. Ở phía server, điều này gần giống như một phương thức keeper được tạo tự động và có thể cuối cùng được sử dụng thay vì keeper (xem [\#7093](https://github.com/cosmos/cosmos-sdk/issues/7093)):

```go
package gov

type MsgServer interface {
  SubmitProposal(context.Context, *MsgSubmitProposal) (*MsgSubmitProposalResponse, error)
}
```

Ở phía client, các nhà phát triển có thể tận dụng điều này bằng cách tạo các triển khai RPC đóng gói logic giao dịch. Các thư viện Protobuf sử dụng callback không đồng bộ, như [protobuf.js](https://github.com/protobufjs/protobuf.js#using-services) có thể sử dụng điều này để đăng ký callback cho các message cụ thể ngay cả với các giao dịch bao gồm nhiều `Msg`.

Mỗi phương thức service `Msg` nên có chính xác một tham số yêu cầu: kiểu `Msg` tương ứng của nó. Ví dụ, phương thức service `Msg` `/cosmos.gov.v1beta1.Msg/SubmitProposal` ở trên có chính xác một tham số yêu cầu, cụ thể là kiểu `Msg` `/cosmos.gov.v1beta1.MsgSubmitProposal`. Điều quan trọng là người đọc hiểu rõ sự khác biệt về danh pháp giữa `Msg` service (một service Protobuf) và kiểu `Msg` (một message Protobuf), và sự khác biệt trong tên đầy đủ của chúng.

Quy ước này đã được quyết định thay vì tên `Msg...Request` chuẩn hơn chủ yếu vì lý do tương thích ngược, nhưng cũng để dễ đọc hơn trong `TxBody.messages` (xem [phần Encoding](#encoding) bên dưới): các giao dịch chứa `/cosmos.gov.MsgSubmitProposal` dễ đọc hơn những giao dịch chứa `/cosmos.gov.v1beta1.MsgSubmitProposalRequest`.

Một hệ quả của quy ước này là mỗi kiểu `Msg` chỉ có thể là tham số yêu cầu của một phương thức service `Msg`. Tuy nhiên, chúng ta coi giới hạn này là thực hành tốt về tính rõ ràng.

### Mã Hóa

Mã hóa các giao dịch được tạo ra với các service `Msg` không khác với mã hóa giao dịch Protobuf hiện tại như được định nghĩa trong [ADR-020](./adr-020-protobuf-transaction-encoding.md). Chúng ta mã hóa các kiểu `Msg` (chính xác là tham số yêu cầu của các phương thức service `Msg`) dưới dạng `Any` trong `Tx` liên quan đến việc đóng gói `Msg` được mã hóa nhị phân với URL kiểu của nó.

### Giải Mã

Vì các kiểu `Msg` được đóng gói vào `Any`, việc giải mã các message giao dịch được thực hiện bằng cách giải nén `Any` thành các kiểu `Msg`. Để biết thêm thông tin, vui lòng tham khảo [ADR-020](./adr-020-protobuf-transaction-encoding.md#transactions).

### Định Tuyến

Chúng ta đề xuất thêm `msg_service_router` vào BaseApp. Router này là một bản đồ key/value ánh xạ `type_url` của các kiểu `Msg` tới các handler phương thức service `Msg` tương ứng của chúng. Vì có ánh xạ 1-1 giữa các kiểu `Msg` và phương thức service `Msg`, `msg_service_router` có chính xác một mục cho mỗi phương thức service `Msg`.

Khi một giao dịch được xử lý bởi BaseApp (trong CheckTx hoặc DeliverTx), `TxBody.messages` của nó được giải mã dưới dạng `Msg`. `type_url` của mỗi `Msg` được khớp với một mục trong `msg_service_router`, và handler phương thức service `Msg` tương ứng được gọi.

Để tương thích ngược, các handler cũ không bị xóa. Nếu BaseApp nhận được `Msg` cũ mà không có mục tương ứng trong `msg_service_router`, nó sẽ được định tuyến qua phương thức `Route()` cũ của nó vào handler cũ.

### Cấu Hình Module

Trong [ADR 021](./adr-021-protobuf-query-encoding.md), chúng ta đã giới thiệu phương thức `RegisterQueryService` cho `AppModule` cho phép các module đăng ký các querier gRPC.

Để đăng ký các service `Msg`, chúng ta thử một cách tiếp cận có khả năng mở rộng hơn bằng cách chuyển đổi `RegisterQueryService` thành phương thức `RegisterServices` tổng quát hơn:

```go
type AppModule interface {
  RegisterServices(Configurator)
  ...
}

type Configurator interface {
  QueryServer() grpc.Server
  MsgServer() grpc.Server
}

// module ví dụ:
func (am AppModule) RegisterServices(cfg Configurator) {
	types.RegisterQueryServer(cfg.QueryServer(), keeper)
	types.RegisterMsgServer(cfg.MsgServer(), keeper)
}
```

Phương thức `RegisterServices` và interface `Configurator` được thiết kế để phát triển nhằm đáp ứng các trường hợp sử dụng được thảo luận trong [\#7093](https://github.com/cosmos/cosmos-sdk/issues/7093) và [\#7122](https://github.com/cosmos/cosmos-sdk/issues/7421).

Khi các service `Msg` được đăng ký, framework _nên_ xác minh rằng tất cả các kiểu `Msg` triển khai interface `sdk.Msg` và ném ra lỗi trong quá trình khởi tạo thay vì về sau khi giao dịch được xử lý.

### Triển Khai `Msg` Service

Giống như các query service, các phương thức service `Msg` có thể lấy `sdk.Context` từ tham số `context.Context` sử dụng phương thức `sdk.UnwrapSDKContext`:

```go
package gov

func (k Keeper) SubmitProposal(goCtx context.Context, params *types.MsgSubmitProposal) (*MsgSubmitProposalResponse, error) {
	ctx := sdk.UnwrapSDKContext(goCtx)
    ...
}
```

`sdk.Context` nên có `EventManager` đã được gắn bởi `msg_service_router` của BaseApp.

Định nghĩa handler riêng biệt không còn cần thiết với cách tiếp cận này.

## Hậu Quả

Thiết kế này thay đổi cách chức năng module được lộ và truy cập. Nó không dùng nữa interface `Handler` và `AppModule.Route` hiện tại để ủng hộ [Protocol Buffer Services](https://developers.google.com/protocol-buffers/docs/proto3#services) và Service Routing được mô tả ở trên. Điều này đơn giản hóa đáng kể code. Chúng ta không còn cần tạo các handler và keeper nữa. Việc sử dụng các client được tạo tự động của Protocol Buffer phân tách rõ ràng các giao diện giao tiếp giữa module và người dùng module. Logic kiểm soát (tức là các handler và keeper) không còn bị lộ nữa. Giao diện module có thể được coi như một hộp đen có thể truy cập thông qua API client. Đáng chú ý là các giao diện client cũng được tạo bởi Protocol Buffers.

Điều này cũng cho phép chúng ta thay đổi cách chúng ta thực hiện các test chức năng. Thay vì mock AppModules và Router, chúng ta sẽ mock client (server sẽ vẫn ẩn). Cụ thể hơn: chúng ta sẽ không bao giờ mock `moduleA.MsgServer` trong `moduleB`, mà là `moduleA.MsgClient`. Người ta có thể nghĩ về nó như làm việc với các dịch vụ bên ngoài (vd: DBs, hoặc các server online...). Chúng ta giả định rằng việc truyền giữa client và server được xử lý đúng bởi Protocol Buffers được tạo ra.

Cuối cùng, việc đóng module cho API client mở ra các mẫu OCAP mong muốn được thảo luận trong ADR-033. Vì triển khai và giao diện server bị ẩn, không ai có thể giữ "keepers"/server và sẽ buộc phải dựa vào giao diện client, điều này sẽ thúc đẩy các nhà phát triển về các mẫu đóng gói và kỹ thuật phần mềm đúng đắn.

### Ưu Điểm

* Truyền đạt kiểu trả về rõ ràng
* Đăng ký handler thủ công và marshal kiểu trả về không còn cần thiết, chỉ cần triển khai interface và đăng ký nó
* Giao diện giao tiếp được tạo tự động, nhà phát triển giờ có thể chỉ tập trung vào các phương thức chuyển đổi trạng thái - điều này sẽ cải thiện UX của cách tiếp cận [\#7093](https://github.com/cosmos/cosmos-sdk/issues/7093) (1) nếu chúng ta chọn áp dụng
* Code client được tạo ra có thể hữu ích cho các client và test
* Giảm và đơn giản hóa đáng kể code

### Nhược Điểm

* Sử dụng định nghĩa `service` bên ngoài bối cảnh gRPC có thể gây nhầm lẫn (nhưng không vi phạm spec proto3)

## Tham Khảo

* [Issue Github ban đầu \#7122](https://github.com/cosmos/cosmos-sdk/issues/7122)
* [Hướng dẫn ngôn ngữ proto 3: Định nghĩa Services](https://developers.google.com/protocol-buffers/docs/proto3#services)
* [Các thiết kế `Msg` trước `Any` ban đầu](https://docs.google.com/document/d/1eEgYgvgZqLE45vETjhwIw4VOqK-5hwQtZtjVbiXnIGc)
* [ADR 020](./adr-020-protobuf-transaction-encoding.md)
* [ADR 021](./adr-021-protobuf-query-encoding.md)
