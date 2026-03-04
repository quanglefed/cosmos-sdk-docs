---
sidebar_position: 1
---

# Vòng Đời Truy Vấn

:::note Tóm tắt
Tài liệu này mô tả vòng đời của một truy vấn trong một ứng dụng Cosmos SDK, từ giao diện người dùng đến các store của ứng dụng và trở lại. Truy vấn được gọi là `MyQuery`.
:::

:::note Tài liệu cần đọc trước

* [Vòng đời giao dịch](./01-tx-lifecycle.md)
:::

## Tạo truy vấn

Một [**truy vấn**](../../build/building-modules/02-messages-and-queries.md#queries) là một yêu cầu thông tin được tạo bởi người dùng cuối của ứng dụng qua giao diện và được xử lý bởi một full-node. Người dùng có thể truy vấn thông tin về mạng lưới, bản thân ứng dụng, và trạng thái ứng dụng trực tiếp từ các store hoặc module của ứng dụng. Lưu ý rằng truy vấn khác với [giao dịch](../advanced/01-transactions.md) (xem vòng đời [tại đây](./01-tx-lifecycle.md)), đặc biệt là chúng không yêu cầu đồng thuận để được xử lý (vì chúng không kích hoạt chuyển đổi trạng thái); chúng có thể được xử lý hoàn toàn bởi một full-node duy nhất.

Để giải thích vòng đời truy vấn, giả sử truy vấn `MyQuery` đang yêu cầu danh sách các ủy quyền (delegation) được thực hiện bởi một địa chỉ delegator nhất định trong ứng dụng tên là `simapp`. Module [`staking`](../../../../x/staking/README.md) xử lý truy vấn này. Nhưng trước tiên, có một vài cách `MyQuery` có thể được tạo bởi người dùng.

### CLI

Giao diện chính của một ứng dụng là command-line interface. Người dùng kết nối đến một full-node và chạy CLI trực tiếp từ máy của họ — CLI tương tác trực tiếp với full-node. Để tạo `MyQuery` từ terminal, người dùng gõ lệnh sau:

```bash
simd query staking delegations <delegatorAddress>
```

Lệnh truy vấn này được định nghĩa bởi nhà phát triển module [`staking`](../../../../x/staking/README.md) và được thêm vào danh sách subcommand bởi nhà phát triển ứng dụng khi tạo CLI.

Lưu ý rằng định dạng chung như sau:

```bash
simd query [moduleName] [command] <arguments> --flag <flagArg>
```

Để cung cấp các giá trị như `--node` (full-node mà CLI kết nối đến), người dùng có thể dùng file cấu hình [`app.toml`](../../user/run-node/01-run-node.md#configuring-the-node-using-apptoml-and-configtoml) để đặt chúng hoặc cung cấp dưới dạng flags.

CLI hiểu một tập hợp lệnh cụ thể, được định nghĩa theo cấu trúc phân cấp bởi nhà phát triển ứng dụng: từ [lệnh gốc](../advanced/07-cli.md#root-command) (`simd`), loại lệnh (`Myquery`), module chứa lệnh (`staking`), và bản thân lệnh (`delegations`). Do đó, CLI biết chính xác module nào xử lý lệnh này và chuyển tiếp cuộc gọi trực tiếp đến đó.

### gRPC

Một giao diện khác mà người dùng có thể thực hiện truy vấn là các yêu cầu [gRPC](https://grpc.io) đến một [gRPC server](../advanced/06-grpc_rest.md#grpc-server). Các endpoint được định nghĩa là các phương thức service [Protocol Buffers](https://developers.google.com/protocol-buffers) bên trong các file `.proto`, được viết bằng ngôn ngữ định nghĩa interface độc lập với ngôn ngữ lập trình (IDL) của Protobuf. Hệ sinh thái Protobuf đã phát triển các công cụ để tạo code từ các file `*.proto` sang nhiều ngôn ngữ. Các công cụ này cho phép xây dựng gRPC client dễ dàng.

Một công cụ như vậy là [grpcurl](https://github.com/fullstorydev/grpcurl), và một yêu cầu gRPC cho `MyQuery` bằng client này trông như sau:

```bash
grpcurl \
    -plaintext                                           # Chúng ta muốn kết quả dạng văn bản thuần
    -import-path ./proto \                               # Import các file .proto này
    -proto ./proto/cosmos/staking/v1beta1/query.proto \  # Tìm trong file .proto này cho Query protobuf service
    -d '{"address":"$MY_DELEGATOR"}' \                   # Tham số truy vấn
    localhost:9090 \                                     # Endpoint gRPC server
    cosmos.staking.v1beta1.Query/Delegations             # Tên phương thức service đầy đủ
```

### REST

Một giao diện khác để thực hiện truy vấn là qua HTTP Request đến một [REST server](../advanced/06-grpc_rest.md#rest-server). REST server được tạo tự động hoàn toàn từ Protobuf service, sử dụng [gRPC-gateway](https://github.com/grpc-ecosystem/grpc-gateway).

Một ví dụ HTTP request cho `MyQuery` trông như sau:

```bash
GET http://localhost:1317/cosmos/staking/v1beta1/delegators/{delegatorAddr}/delegations
```

## Cách CLI xử lý truy vấn

Các ví dụ trên cho thấy cách người dùng bên ngoài có thể tương tác với một node bằng cách truy vấn trạng thái của nó. Để hiểu chi tiết hơn về vòng đời chính xác của một truy vấn, hãy xem cách CLI chuẩn bị truy vấn và cách node xử lý nó. Các tương tác từ góc độ người dùng có phần khác nhau, nhưng các hàm bên dưới gần như giống hệt nhau vì chúng là các triển khai của cùng một lệnh được định nghĩa bởi nhà phát triển module. Bước xử lý này xảy ra trong CLI, gRPC hoặc REST server, và liên quan nhiều đến `client.Context`.

### Context

Thứ đầu tiên được tạo trong quá trình thực thi lệnh CLI là `client.Context`. `client.Context` là một đối tượng lưu trữ tất cả dữ liệu cần thiết để xử lý một yêu cầu ở phía người dùng. Cụ thể, `client.Context` lưu trữ:

* **Codec**: [Bộ mã hóa/giải mã](../advanced/05-encoding.md) được ứng dụng sử dụng, dùng để marshal các tham số và truy vấn trước khi thực hiện yêu cầu CometBFT RPC và unmarshal phản hồi trả về thành đối tượng JSON. Codec mặc định mà CLI sử dụng là Protobuf.
* **Account Decoder**: Bộ giải mã tài khoản từ module [`auth`](../../../../x/auth/README.md), dịch `[]byte` thành các tài khoản.
* **RPC Client**: CometBFT RPC Client, hay node, mà các yêu cầu được chuyển tiếp đến.
* **Keyring**: [Key Manager](../beginner/03-accounts.md#keyring) dùng để ký giao dịch và xử lý các thao tác khác với key.
* **Output Writer**: [Writer](https://pkg.go.dev/io/#Writer) dùng để xuất phản hồi.
* **Cấu hình**: Các flag được người dùng cấu hình cho lệnh này, bao gồm `--height`, chỉ định chiều cao blockchain để truy vấn, và `--indent`, chỉ định thêm thụt lề vào phản hồi JSON.

`client.Context` cũng chứa các hàm khác nhau như `Query()`, dùng để lấy RPC Client và thực hiện lời gọi ABCI để chuyển tiếp truy vấn đến full-node.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/client/context.go#L27-70
```

Vai trò chính của `client.Context` là lưu trữ dữ liệu được dùng trong quá trình tương tác với người dùng cuối và cung cấp các phương thức để tương tác với dữ liệu này — nó được dùng trước và sau khi truy vấn được xử lý bởi full-node. Cụ thể, khi xử lý `MyQuery`, `client.Context` được dùng để mã hóa các tham số truy vấn, lấy full-node, và ghi đầu ra. Trước khi được chuyển tiếp đến full-node, truy vấn cần được mã hóa thành dạng `[]byte`, vì các full-node là application-agnostic và không hiểu các kiểu cụ thể.

### Tạo tham số và route

Tại thời điểm này, người dùng đã tạo một lệnh CLI với tất cả dữ liệu họ muốn đưa vào truy vấn. `client.Context` tồn tại để hỗ trợ phần còn lại của hành trình `MyQuery`. Bước tiếp theo là phân tích lệnh hoặc yêu cầu, trích xuất các tham số và mã hóa mọi thứ.

#### Mã hóa (Encoding)

Trong trường hợp của chúng ta (truy vấn delegation của một địa chỉ), `MyQuery` chứa một [địa chỉ](./03-accounts.md#addresses) `delegatorAddress` là đối số duy nhất. Tuy nhiên, yêu cầu chỉ có thể chứa `[]byte`, vì cuối cùng nó được chuyển tiếp đến consensus engine (ví dụ: CometBFT) của một full-node không có kiến thức cố hữu về các kiểu ứng dụng. Do đó, `codec` của `client.Context` được dùng để marshal địa chỉ.

Đây là code cho lệnh CLI:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/x/staking/client/cli/query.go#L315-L318
```

#### Tạo gRPC Query Client

Cosmos SDK tận dụng code được tạo từ Protobuf service để thực hiện truy vấn. `MyQuery` service của module `staking` tạo ra một `queryClient` mà CLI sử dụng để thực hiện truy vấn. Đây là code liên quan:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/x/staking/client/cli/query.go#L308-L343
```

Ẩn bên dưới, `client.Context` có hàm `Query()` được dùng để lấy node đã cấu hình sẵn và chuyển tiếp truy vấn đến nó; hàm nhận tên phương thức service đầy đủ của truy vấn làm đường dẫn (trong trường hợp của chúng ta: `/cosmos.staking.v1beta1.Query/Delegations`), và các tham số. Đầu tiên nó lấy RPC Client (được gọi là [**node**](../advanced/03-node.md)) mà người dùng đã cấu hình để chuyển tiếp truy vấn này đến, và tạo `ABCIQueryOptions` (các tham số được định dạng cho lời gọi ABCI). Sau đó node được dùng để thực hiện lời gọi ABCI, `ABCIQueryWithOptions()`.

Đây là code:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/client/query.go#L79-L113
```

## RPC

Với lời gọi `ABCIQueryWithOptions()`, `MyQuery` được nhận bởi một [full-node](../advanced/05-encoding.md) và xử lý yêu cầu. Lưu ý rằng, trong khi RPC được thực hiện đến consensus engine (ví dụ: CometBFT) của một full-node, các truy vấn không phải là một phần của đồng thuận và do đó không được broadcast đến phần còn lại của mạng lưới, vì chúng không yêu cầu bất cứ điều gì mà mạng cần đồng thuận.

Đọc thêm về ABCI Client và CometBFT RPC trong [tài liệu CometBFT](https://docs.cometbft.com/v0.37/spec/rpc/).

## Xử lý truy vấn của ứng dụng

Khi một truy vấn được full-node nhận sau khi được chuyển tiếp từ consensus engine bên dưới, tại thời điểm đó nó đang được xử lý trong môi trường hiểu các kiểu dành riêng cho ứng dụng và có bản sao trạng thái. [`baseapp`](../advanced/00-baseapp.md) triển khai hàm ABCI [`Query()`](../advanced/00-baseapp.md#query) và xử lý các truy vấn gRPC. Route của truy vấn được phân tích và khớp với tên phương thức service đầy đủ của một service method hiện có (thường trong một trong các module), sau đó `baseapp` chuyển tiếp yêu cầu đến module liên quan.

Vì `MyQuery` có tên phương thức service Protobuf đầy đủ từ module `staking` (nhớ lại `/cosmos.staking.v1beta1.Query/Delegations`), `baseapp` trước tiên phân tích đường dẫn, sau đó dùng `GRPCQueryRouter` nội bộ của mình để lấy gRPC handler tương ứng, và định tuyến truy vấn đến module. gRPC handler chịu trách nhiệm nhận dạng truy vấn này, lấy các giá trị phù hợp từ các store của ứng dụng và trả về phản hồi. Đọc thêm về query service [tại đây](../../build/building-modules/04-query-services.md).

Khi kết quả được nhận từ querier, `baseapp` bắt đầu quá trình trả về phản hồi cho người dùng.

## Phản hồi

Vì `Query()` là một hàm ABCI, `baseapp` trả về phản hồi dưới dạng kiểu [`abci.QueryResponse`](https://docs.cometbft.com/main/spec/abci/abci++_methods#query). Routine `Query()` của `client.Context` nhận phản hồi và xử lý nó.

### Phản hồi CLI

[`codec`](../advanced/05-encoding.md) của ứng dụng được dùng để unmarshal phản hồi thành JSON và `client.Context` in đầu ra ra command line, áp dụng các cấu hình như loại đầu ra (text, JSON hoặc YAML).

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/client/context.go#L350-L357
```

Vậy là xong! Kết quả của truy vấn được xuất ra console bởi CLI.
