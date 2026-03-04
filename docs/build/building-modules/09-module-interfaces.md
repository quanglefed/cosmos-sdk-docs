---
sidebar_position: 1
---

# Module Interfaces

:::note Tóm tắt
Tài liệu này trình bày chi tiết cách xây dựng giao diện CLI và REST cho một module. Có bao gồm các ví dụ từ các module Cosmos SDK khác nhau.
:::

:::note Yêu Cầu Đọc Trước

* [Giới thiệu về Building Modules](./00-intro.md)

:::

## CLI

Một trong những giao diện chính của ứng dụng là [giao diện dòng lệnh](../../learn/advanced/07-cli.md). Điểm vào này thêm các lệnh từ các module của ứng dụng, cho phép người dùng cuối tạo [**message**](./02-messages-and-queries.md#messages) được bọc trong giao dịch và [**query**](./02-messages-and-queries.md#queries). Các file CLI thường được tìm thấy trong thư mục `./client/cli` của module.

### Lệnh Giao Dịch (Transaction Commands)

Để tạo các message kích hoạt thay đổi trạng thái, người dùng cuối phải tạo các [giao dịch](../../learn/advanced/01-transactions.md) bọc và giao các message. Một lệnh giao dịch tạo ra một giao dịch bao gồm một hoặc nhiều message.

Các lệnh giao dịch thường có file `tx.go` riêng nằm trong thư mục `./client/cli` của module. Các lệnh được chỉ định trong các hàm getter và tên của hàm nên bao gồm tên của lệnh.

Dưới đây là ví dụ từ module `x/bank`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/bank/client/cli/tx.go#L37-L76
```

Trong ví dụ, `NewSendTxCmd()` tạo và trả về lệnh giao dịch cho một giao dịch bọc và giao `MsgSend`. `MsgSend` là message dùng để gửi token từ tài khoản này sang tài khoản khác.

Nhìn chung, hàm getter thực hiện những việc sau:

* **Xây dựng lệnh:** Đọc [Tài liệu Cobra](https://pkg.go.dev/github.com/spf13/cobra) để biết thêm thông tin chi tiết về cách tạo lệnh.
    * **Use:** Chỉ định định dạng đầu vào người dùng cần để gọi lệnh. Trong ví dụ trên, `send` là tên lệnh giao dịch và `[from_key_or_address]`, `[to_address]`, và `[amount]` là các đối số.
    * **Args:** Số lượng đối số người dùng cung cấp. Trong trường hợp này, có đúng ba: `[from_key_or_address]`, `[to_address]`, và `[amount]`.
    * **Short và Long:** Mô tả cho lệnh. Cần có mô tả `Short`. Mô tả `Long` có thể được sử dụng để cung cấp thông tin bổ sung hiển thị khi người dùng thêm cờ `--help`.
    * **RunE:** Định nghĩa một hàm có thể trả về lỗi. Đây là hàm được gọi khi lệnh được thực thi. Hàm này đóng gói tất cả logic để tạo một giao dịch mới.
        * Hàm thường bắt đầu bằng việc lấy `clientCtx`, có thể thực hiện bằng `client.GetClientTxContext(cmd)`. `clientCtx` chứa thông tin liên quan đến xử lý giao dịch, bao gồm thông tin về người dùng. Trong ví dụ này, `clientCtx` được dùng để lấy địa chỉ của người gửi bằng cách gọi `clientCtx.GetFromAddress()`.
        * Nếu áp dụng, các đối số của lệnh được phân tích. Trong ví dụ này, cả hai đối số `[to_address]` và `[amount]` đều được phân tích.
        * Một [message](./02-messages-and-queries.md) được tạo bằng cách sử dụng các đối số đã phân tích và thông tin từ `clientCtx`. Hàm constructor của loại message được gọi trực tiếp. Trong trường hợp này, `types.NewMsgSend(fromAddr, toAddr, amount)`. Nên gọi các [phương thức xác thực message](../building-modules/03-msg-services.md#Validation) cần thiết trước khi phát sóng message.
        * Tùy thuộc vào những gì người dùng muốn, giao dịch được tạo ngoại tuyến hoặc được ký và phát sóng đến node đã cấu hình trước bằng `tx.GenerateOrBroadcastTxCLI(clientCtx, flags, msg)`.
* **Thêm transaction flags:** Tất cả các lệnh giao dịch phải thêm một bộ [flag](#flags) giao dịch. Các transaction flag được sử dụng để thu thập thêm thông tin từ người dùng (ví dụ: số tiền phí người dùng sẵn sàng trả). Các transaction flag được thêm vào lệnh đã xây dựng bằng `AddTxFlagsToCmd(cmd)`.
* **Trả về lệnh:** Cuối cùng, lệnh giao dịch được trả về.

Mỗi module có thể triển khai `NewTxCmd()`, tổng hợp tất cả các lệnh giao dịch của module. Dưới đây là ví dụ từ module `x/bank`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/bank/client/cli/tx.go#L20-L35
```

Mỗi module sau đó cũng có thể triển khai phương thức `GetTxCmd()` đơn giản là trả về `NewTxCmd()`. Điều này cho phép lệnh gốc dễ dàng tổng hợp tất cả các lệnh giao dịch cho mỗi module. Dưới đây là ví dụ:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/bank/module.go#L84-L86
```

### Lệnh Query (Query Commands)

:::warning
Phần này đang được viết lại. Hãy tham khảo [AutoCLI](https://docs.cosmos.network/main/core/autocli) trong khi phần này đang được cập nhật.
:::

## gRPC

[gRPC](https://grpc.io/) là một framework Gọi Thủ Tục Từ Xa (Remote Procedure Call). RPC là cách ưu tiên để các client bên ngoài như ví và sàn giao dịch tương tác với một blockchain.

Ngoài việc cung cấp đường dẫn query ABCI, Cosmos SDK cung cấp một máy chủ proxy gRPC định tuyến các yêu cầu query gRPC đến các yêu cầu query ABCI.

Để làm điều đó, các module phải triển khai `RegisterGRPCGatewayRoutes(clientCtx client.Context, mux *runtime.ServeMux)` trên `AppModuleBasic` để kết nối các yêu cầu gRPC client với handler đúng bên trong module.

Dưới đây là ví dụ từ module `x/auth`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/auth/module.go#L71-L76
```

## gRPC-gateway REST

Các ứng dụng cần hỗ trợ các web service sử dụng HTTP request (ví dụ: ví web như [Keplr](https://keplr.app)). [grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway) dịch các lời gọi REST sang gRPC, có thể hữu ích cho các client không sử dụng gRPC.

Các module muốn hiển thị REST query nên thêm annotation `google.api.http` vào các phương thức `rpc` của họ, như ví dụ dưới đây từ module `x/auth`:

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/proto/cosmos/auth/v1beta1/query.proto#L14-L89
```

gRPC gateway được khởi động trong cùng tiến trình với ứng dụng và CometBFT. Nó có thể được bật hoặc tắt bằng cách đặt cấu hình gRPC `enable` trong [`app.toml`](../../user/run-node/01-run-node.md#configuring-the-node-using-apptoml-and-configtoml).

Cosmos SDK cung cấp lệnh để tạo tài liệu [Swagger](https://swagger.io/) (`protoc-gen-swagger`). Đặt `swagger` trong [`app.toml`](../../user/run-node/01-run-node.md#configuring-the-node-using-apptoml-and-configtoml) xác định xem tài liệu swagger có nên được đăng ký tự động hay không.
