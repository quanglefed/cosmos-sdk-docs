---
sidebar_position: 1
---

# Protocol Buffers

Cosmos SDK sử dụng protocol buffers một cách rộng rãi, tài liệu này nhằm cung cấp hướng dẫn về cách nó được sử dụng trong cosmos-sdk.

Để tạo file proto, Cosmos SDK sử dụng một docker image, image này được cung cấp để tất cả mọi người sử dụng. Phiên bản mới nhất là `ghcr.io/cosmos/proto-builder:0.17.0`

Dưới đây là ví dụ về các lệnh của Cosmos SDK để tạo, lint và định dạng các file protobuf có thể được tái sử dụng trong bất kỳ makefile nào của ứng dụng.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/Makefile#L411-L432
```

Script được sử dụng để tạo các file protobuf có thể tìm thấy trong thư mục `scripts/`.

```shell reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/scripts/protocgen.sh
```

## Buf

[Buf](https://buf.build) là một công cụ protobuf trừu tượng hóa nhu cầu sử dụng toolchain `protoc` phức tạp cùng với nhiều thứ khác đảm bảo bạn sử dụng protobuf theo phù hợp với phần lớn hệ sinh thái. Trong repository cosmos-sdk có một số file với tiền tố buf. Hãy bắt đầu từ thư mục cấp cao nhất và sau đó đi sâu vào các thư mục khác nhau.

### Workspace (Không Gian Làm Việc)

Ở thư mục cấp root, một workspace được định nghĩa bằng cách sử dụng [buf workspaces](https://docs.buf.build/configuration/v1/buf-work-yaml). Điều này hữu ích nếu có một hoặc nhiều thư mục chứa protobuf trong dự án của bạn.

Ví dụ Cosmos SDK:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/main/buf.work.yaml#L6-L9
```

### Thư Mục Proto

Tiếp theo là thư mục `proto/` nơi tất cả các file protobuf của chúng ta nằm. Trong đó có nhiều file buf khác nhau được định nghĩa, mỗi file phục vụ một mục đích khác nhau.

```bash
├── README.md
├── buf.gen.gogo.yaml
├── buf.gen.pulsar.yaml
├── buf.gen.swagger.yaml
├── buf.lock
├── buf.md
├── buf.yaml
├── cosmos
└── tendermint
```

Sơ đồ trên hiển thị tất cả các file và thư mục trong thư mục `proto/` của Cosmos SDK.

#### `buf.gen.gogo.yaml`

`buf.gen.gogo.yaml` định nghĩa cách các file protobuf nên được tạo để sử dụng trong module. File này sử dụng [gogoproto](https://github.com/gogo/protobuf), một generator riêng biệt từ google go-proto generator giúp làm việc với các đối tượng khác nhau thuận tiện hơn, và có các bước encode và decode hiệu quả hơn.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/main/proto/buf.gen.gogo.yaml#L1-L9
```

:::tip
Ví dụ về cách định nghĩa các file `gen` có thể tìm thấy [tại đây](https://docs.buf.build/generate/overview)
:::

#### `buf.gen.pulsar.yaml`

`buf.gen.pulsar.yaml` định nghĩa cách các file protobuf nên được tạo bằng cách sử dụng [golang apiv2 mới của protobuf](https://go.dev/blog/protobuf-apiv2). Generator này được sử dụng thay cho google go-proto generator vì nó có một số helper bổ sung cho các ứng dụng Cosmos SDK và sẽ có encode và decode hiệu quả hơn generator google go-proto. Bạn có thể theo dõi quá trình phát triển của generator này [tại đây](https://github.com/cosmos/cosmos-proto).

```go reference
https://github.com/cosmos/cosmos-sdk/blob/main/proto/buf.gen.pulsar.yaml#L1-L18
```

:::tip
Ví dụ về cách định nghĩa các file `gen` có thể tìm thấy [tại đây](https://docs.buf.build/generate/overview)
:::

#### `buf.gen.swagger.yaml`

`buf.gen.swagger.yaml` tạo tài liệu swagger cho các query và message của chain. Điều này chỉ định nghĩa các REST API endpoint được định nghĩa trong các query và msg server. Bạn có thể tìm ví dụ về điều này [tại đây](https://github.com/cosmos/cosmos-sdk/blob/main/proto/cosmos/bank/v1beta1/query.proto#L19)

```go reference
https://github.com/cosmos/cosmos-sdk/blob/main/proto/buf.gen.swagger.yaml#L1-L6
```

:::tip
Ví dụ về cách định nghĩa các file `gen` có thể tìm thấy [tại đây](https://docs.buf.build/generate/overview)
:::

#### `buf.lock`

Đây là file được tạo tự động dựa trên các phụ thuộc được yêu cầu bởi các file `.gen`. Không cần sao chép file hiện tại. Nếu bạn phụ thuộc vào các định nghĩa proto cosmos-sdk, một entry mới cho Cosmos SDK sẽ cần được cung cấp. Phụ thuộc bạn sẽ cần sử dụng là `buf.build/cosmos/cosmos-sdk`.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/main/proto/buf.lock#L1-L16
```

#### `buf.yaml`

`buf.yaml` định nghĩa [tên package của bạn](https://github.com/cosmos/cosmos-sdk/blob/main/proto/buf.yaml#L3), [breakage checker](https://docs.buf.build/breaking/overview) nào để sử dụng và cách [lint các file protobuf của bạn](https://buf.build/docs/tutorials/getting-started-with-buf-cli#lint-your-api).

```go reference
https://github.com/cosmos/cosmos-sdk/blob/main/proto/buf.yaml#L1-L24
```

Chúng tôi sử dụng nhiều linter khác nhau cho các file protobuf Cosmos SDK. Repo cũng kiểm tra điều này trong CI.

Tham chiếu đến github actions có thể tìm thấy [tại đây](https://github.com/cosmos/cosmos-sdk/blob/main/.github/workflows/proto.yml#L1-L32)

```go reference
https://github.com/cosmos/cosmos-sdk/blob/main/.github/workflows/proto.yml#L1-L32
```
