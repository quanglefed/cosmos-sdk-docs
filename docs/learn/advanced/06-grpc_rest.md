---
sidebar_position: 1
---

# Các Endpoint gRPC, REST và CometBFT

:::note Tóm tắt
Tài liệu này trình bày tổng quan về tất cả các endpoint mà một node cung cấp: gRPC, REST và một số endpoint khác.
:::

## Tổng quan về Tất cả Endpoint

Mỗi node cung cấp các endpoint sau cho người dùng tương tác với node, mỗi endpoint được phục vụ trên một cổng khác nhau. Chi tiết về cách cấu hình từng endpoint được cung cấp trong phần riêng của từng endpoint.

* gRPC server (cổng mặc định: `9090`),
* REST server (cổng mặc định: `1317`),
* CometBFT RPC endpoint (cổng mặc định: `26657`).

:::tip
Node cũng cung cấp một số endpoint khác, chẳng hạn như CometBFT P2P endpoint hoặc [Prometheus endpoint](https://docs.cometbft.com/v0.37/core/metrics), không liên quan trực tiếp đến Cosmos SDK. Hãy tham khảo [tài liệu CometBFT](https://docs.cometbft.com/v0.37/core/configuration) để biết thêm thông tin về các endpoint này.
:::

:::note
Tất cả các endpoint mặc định là localhost và phải được sửa đổi để mở ra internet công cộng.
:::

## gRPC Server

Trong Cosmos SDK, Protobuf là thư viện [mã hóa](./05-encoding.md) chính. Điều này mang lại nhiều công cụ dựa trên Protobuf có thể được tích hợp vào Cosmos SDK. Một trong số đó là [gRPC](https://grpc.io), một framework RPC hiệu suất cao mã nguồn mở hiện đại có hỗ trợ client tốt trong nhiều ngôn ngữ.

Mỗi module cung cấp một [Protobuf `Query` service](../../build/building-modules/02-messages-and-queries.md#queries) định nghĩa các query trạng thái. Các `Query` service và transaction service dùng để phát sóng giao dịch được kết nối với gRPC server thông qua hàm sau bên trong ứng dụng:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/server/types/app.go#L46-L48
```

Lưu ý: Không thể cung cấp bất kỳ endpoint [Protobuf `Msg` service](../../build/building-modules/02-messages-and-queries.md#messages) nào qua gRPC. Giao dịch phải được tạo và ký bằng CLI hoặc theo cách lập trình trước khi có thể được phát sóng bằng gRPC. Xem [Tạo, Ký và Phát sóng Giao dịch](../../user/run-node/03-txs.md) để biết thêm thông tin.

`grpc.Server` là một gRPC server cụ thể, khởi tạo và phục vụ tất cả các gRPC query request và broadcast transaction request. Server này có thể được cấu hình trong `~/.simapp/config/app.toml`:

* Trường `grpc.enable = true|false` xác định xem gRPC server có nên được bật hay không. Mặc định là `true`.
* Trường `grpc.address = {string}` xác định `ip:port` mà server nên bind đến. Mặc định là `localhost:9090`.

:::tip
`~/.simapp` là thư mục lưu trữ cấu hình và database của node. Theo mặc định, nó được đặt thành `~/.{app_name}`.
:::

Sau khi gRPC server được khởi động, bạn có thể gửi request đến nó bằng gRPC client. Một số ví dụ được cung cấp trong hướng dẫn [Tương tác với Node](../../user/run-node/02-interact-node.md#using-grpc).

Tổng quan về tất cả các gRPC endpoint sẵn có đi kèm với Cosmos SDK được cung cấp trong [Protobuf documentation](https://buf.build/cosmos/cosmos-sdk).

## REST Server

Cosmos SDK hỗ trợ các REST route thông qua gRPC-gateway.

Tất cả các route được cấu hình trong các trường sau trong `~/.simapp/config/app.toml`:

* Trường `api.enable = true|false` xác định xem REST server có nên được bật hay không. Mặc định là `false`.
* Trường `api.address = {string}` xác định `ip:port` mà server nên bind đến. Mặc định là `tcp://localhost:1317`.
* Một số tùy chọn cấu hình API bổ sung được định nghĩa trong `~/.simapp/config/app.toml`, cùng với các chú thích, hãy tham khảo trực tiếp file đó.

### REST Routes của gRPC-gateway

Nếu vì nhiều lý do bạn không thể dùng gRPC (ví dụ: bạn đang xây dựng ứng dụng web, và trình duyệt không hỗ trợ HTTP2 mà gRPC được xây dựng trên đó), thì Cosmos SDK cung cấp các REST route thông qua gRPC-gateway.

[gRPC-gateway](https://grpc-ecosystem.github.io/grpc-gateway/) là công cụ cung cấp các gRPC endpoint dưới dạng REST endpoint. Đối với mỗi gRPC endpoint được định nghĩa trong một Protobuf `Query` service, Cosmos SDK cung cấp REST endpoint tương đương. Ví dụ, query số dư có thể được thực hiện thông qua gRPC endpoint `/cosmos.bank.v1beta1.QueryAllBalances`, hoặc thay thế bằng gRPC-gateway REST endpoint `"/cosmos/bank/v1beta1/balances/{address}"`: cả hai đều trả về cùng kết quả. Đối với mỗi phương thức RPC được định nghĩa trong một Protobuf `Query` service, REST endpoint tương ứng được định nghĩa như một tùy chọn:

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/proto/cosmos/bank/v1beta1/query.proto#L23-L30
```

Đối với các nhà phát triển ứng dụng, các REST route của gRPC-gateway cần được kết nối với REST server, điều này được thực hiện bằng cách gọi hàm `RegisterGRPCGatewayRoutes` trên ModuleManager.

### Swagger

Một file đặc tả [Swagger](https://swagger.io/) (hoặc OpenAPIv2) được cung cấp tại route `/swagger` trên API server. Swagger là một đặc tả mở mô tả các endpoint API mà server phục vụ, bao gồm mô tả, tham số đầu vào, kiểu trả về và nhiều hơn nữa về từng endpoint.

Việc bật endpoint `/swagger` có thể được cấu hình trong `~/.simapp/config/app.toml` thông qua trường `api.swagger`, mặc định là false.

Đối với các nhà phát triển ứng dụng, bạn có thể muốn tạo định nghĩa Swagger của riêng mình dựa trên các module tùy chỉnh. [Script tạo Swagger của Cosmos SDK](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/scripts/protoc-swagger-gen.sh) là điểm khởi đầu tốt.

## CometBFT RPC

Độc lập với Cosmos SDK, CometBFT cũng cung cấp RPC server. RPC server này có thể được cấu hình bằng cách điều chỉnh các tham số trong bảng `rpc` của `~/.simapp/config/config.toml`, địa chỉ lắng nghe mặc định là `tcp://localhost:26657`. Đặc tả OpenAPI của tất cả các CometBFT RPC endpoint có sẵn [ở đây](https://docs.cometbft.com/main/rpc/).

Một số CometBFT RPC endpoint liên quan trực tiếp đến Cosmos SDK:

* `/abci_query`: endpoint này sẽ query ứng dụng về trạng thái. Là tham số `path`, bạn có thể gửi các chuỗi sau:
    * bất kỳ service method gRPC được định danh đầy đủ nào, chẳng hạn như `/cosmos.bank.v1beta1.Query/AllBalances`. Trường `data` sau đó phải bao gồm (các) tham số request của phương thức được mã hóa dưới dạng bytes bằng Protobuf.
    * `/app/simulate`: sẽ mô phỏng một giao dịch và trả về một số thông tin như gas đã sử dụng.
    * `/app/version`: sẽ trả về phiên bản của ứng dụng.
    * `/store/{storeName}/key`: sẽ query trực tiếp store được đặt tên cho dữ liệu liên kết với key được biểu diễn trong tham số `data`.
    * `/store/{storeName}/subspace`: sẽ query trực tiếp store được đặt tên cho các cặp key/value mà key có giá trị của tham số `data` làm tiền tố.
    * `/p2p/filter/addr/{port}`: sẽ trả về danh sách được lọc của các P2P peer của node theo cổng địa chỉ.
    * `/p2p/filter/id/{id}`: sẽ trả về danh sách được lọc của các P2P peer của node theo ID.
* `/broadcast_tx_{sync,async,commit}`: 3 endpoint này sẽ phát sóng giao dịch đến các peer khác. CLI, gRPC và REST cung cấp [cách để phát sóng giao dịch](./01-transactions.md#broadcasting-the-transaction), nhưng tất cả đều dùng 3 CometBFT RPC này bên dưới.

## Bảng so sánh

| Tên           | Ưu điểm                                                                                                                                                                       | Nhược điểm                                                                                                    |
| -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| gRPC           | - có thể dùng stub được tạo từ code trong nhiều ngôn ngữ <br /> - hỗ trợ streaming và giao tiếp hai chiều (HTTP2) <br /> - kích thước binary wire nhỏ, truyền tải nhanh hơn | - dựa trên HTTP2, không dùng được trên trình duyệt <br /> - đường cong học tập (chủ yếu do Protobuf)         |
| REST           | - phổ biến rộng rãi <br/> - thư viện client trong mọi ngôn ngữ, triển khai nhanh hơn <br />                                                                                  | - chỉ hỗ trợ giao tiếp request-response đơn chiều (HTTP1.1) <br/> - kích thước message lớn hơn (JSON)        |
| CometBFT RPC | - dễ sử dụng                                                                                                                                                                  | - kích thước message lớn hơn (JSON)                                                                           |
