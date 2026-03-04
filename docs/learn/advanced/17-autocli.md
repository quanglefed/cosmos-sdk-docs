---
sidebar_position: 1
---

# AutoCLI

:::note Tóm tắt
Tài liệu này mô tả chi tiết cách xây dựng giao diện CLI và REST cho một module. Các ví dụ từ nhiều module Cosmos SDK khác nhau được đưa vào.
:::

:::note Tài liệu cần đọc trước

* [CLI](https://docs.cosmos.network/main/core/cli)

:::

Package `autocli` (còn được gọi là `client/v2`) là một [thư viện Go](https://pkg.go.dev/cosmossdk.io/client/v2/autocli) để tạo giao diện CLI (command line interface) cho các ứng dụng dựa trên Cosmos SDK. Nó cung cấp một cách đơn giản để thêm lệnh CLI vào ứng dụng của bạn bằng cách tự động tạo chúng dựa trên các định nghĩa gRPC service. Autocli tạo các lệnh CLI và flags trực tiếp từ các message protobuf, bao gồm options, tham số đầu vào và tham số đầu ra. Điều này có nghĩa là bạn có thể dễ dàng thêm giao diện CLI vào ứng dụng mà không cần tạo và quản lý lệnh thủ công.

## Tổng quan

`autocli` tạo ra lệnh CLI và flags cho mỗi phương thức được định nghĩa trong gRPC service của bạn. Theo mặc định, nó tạo lệnh cho mỗi gRPC service. Các lệnh được đặt tên dựa trên tên của phương thức service.

Ví dụ, với định nghĩa protobuf sau cho một service:

```protobuf
service MyService {
  rpc MyMethod(MyRequest) returns (MyResponse) {}
}
```

`autocli` sẽ tạo một lệnh tên `my-method` cho phương thức `MyMethod`. Lệnh sẽ có flags cho mỗi trường trong message `MyRequest`.

Có thể tùy chỉnh việc tạo giao dịch và truy vấn bằng cách định nghĩa options cho mỗi service.

## Kết nối ứng dụng (Application Wiring)

Các bước sử dụng AutoCLI:

1. Đảm bảo rằng các module của ứng dụng triển khai interface `appmodule.AppModule`.
2. (Tùy chọn) Cấu hình cách `autocli` tạo lệnh bằng cách triển khai phương thức `func (am AppModule) AutoCLIOptions() *autocliv1.ModuleOptions` trên module.
3. Sử dụng struct `autocli.AppOptions` để chỉ định các module bạn đã định nghĩa. Nếu đang dùng `depinject`, nó có thể tự động tạo một instance của `autocli.AppOptions` dựa trên cấu hình ứng dụng.
4. Sử dụng phương thức `EnhanceRootCommand()` do `autocli` cung cấp để thêm các lệnh CLI cho các module được chỉ định vào lệnh gốc.

:::tip
AutoCLI chỉ bổ sung thêm (additive only), nghĩa là việc _enhance_ lệnh gốc chỉ thêm các subcommand chưa được đăng ký. Điều này có nghĩa là bạn có thể dùng AutoCLI cùng với các lệnh tùy chỉnh khác trong ứng dụng.
:::

Đây là ví dụ về cách dùng `autocli` trong ứng dụng của bạn:

``` go
// Định nghĩa các module của ứng dụng
testModules := map[string]appmodule.AppModule{
    "testModule": &TestModule{},
}

// Định nghĩa autocli AppOptions
autoCliOpts := autocli.AppOptions{
    Modules: testModules,
}

// Tạo lệnh gốc
rootCmd := &cobra.Command{
    Use: "app",
}

if err := appOptions.EnhanceRootCommand(rootCmd); err != nil {
    return err
}

// Chạy lệnh gốc
if err := rootCmd.Execute(); err != nil {
    return err
}
```

### Keyring

`autocli` sử dụng keyring để giải quyết tên khóa và ký giao dịch.

:::tip
AutoCLI cung cấp UX tốt hơn CLI thông thường vì nó cho phép giải quyết tên khóa trực tiếp từ keyring trong tất cả giao dịch và lệnh.

```sh
<appd> q bank balances alice
<appd> tx bank send alice bob 1000denom
```

:::

Keyring được dùng để giải quyết tên và ký giao dịch được cung cấp thông qua `client.Context`. Keyring sau đó được chuyển đổi sang interface `client/v2/autocli/keyring`. Nếu không có keyring nào được cung cấp, lệnh được tạo bởi `autocli` sẽ không thể ký giao dịch, nhưng vẫn có thể truy vấn chuỗi.

:::tip
Keyring của Cosmos SDK triển khai interface `client/v2/autocli/keyring`, nhờ vào wrapper sau:

```go
keyring.NewAutoCLIKeyring(kb)
```

:::

## Ký giao dịch (Signing)

`autocli` hỗ trợ ký giao dịch với keyring. [Annotation protobuf `cosmos.msg.v1.signer`](https://docs.cosmos.network/main/build/building-modules/protobuf-annotations) định nghĩa trường signer của message. Trường này được tự động điền khi sử dụng flag `--from` hoặc định nghĩa signer là positional argument.

:::warning
AutoCLI hiện tại chỉ hỗ trợ một signer duy nhất cho mỗi giao dịch.
:::

## Kết nối module & Tùy chỉnh

Phương thức `AutoCLIOptions()` trên module của bạn cho phép chỉ định các lệnh tùy chỉnh, sub-command hoặc flags cho mỗi service — tương tự như một instance `cobra.Command` — trong struct `RpcCommandOptions`. Việc định nghĩa các options này sẽ tùy chỉnh hành vi tạo lệnh của `autocli`, vốn mặc định tạo một lệnh cho mỗi phương thức trong gRPC service.

```go
*autocliv1.RpcCommandOptions{
  RpcMethod: "Params", // Tên của gRPC service
  Use:       "params", // Cú pháp lệnh hiển thị trong phần help
  Short:     "Truy vấn các tham số của quy trình governance", // Mô tả ngắn của lệnh
  Long:      "Truy vấn các tham số của quy trình governance. Chỉ định loại tham số cụ thể (voting|tallying|deposit) để lọc kết quả.", // Mô tả dài của lệnh
  PositionalArgs: []*autocliv1.PositionalArgDescriptor{
    {ProtoField: "params_type", Optional: true}, // Chuyển một flag thành positional argument
  },
}
```

:::tip
AutoCLI có thể tạo gov proposal của bất kỳ tx nào chỉ bằng cách đặt trường `GovProposal` thành `true` trong struct `autocli.RpcCommandOptions`. Người dùng có thể dùng flag `--no-proposal` để tắt tạo proposal (hữu ích khi authority không phải là module gov trên chuỗi đó).
:::

### Chỉ định Subcommands

Theo mặc định, `autocli` tạo một lệnh cho mỗi phương thức trong gRPC service của bạn. Tuy nhiên, bạn có thể chỉ định subcommands để nhóm các lệnh liên quan lại với nhau bằng cách sử dụng struct `autocliv1.ServiceCommandDescriptor`.

Ví dụ dưới đây cho thấy cách sử dụng struct `autocliv1.ServiceCommandDescriptor` để nhóm các lệnh liên quan và chỉ định subcommands trong gRPC service bằng cách định nghĩa một instance của `autocliv1.ModuleOptions` trong file `autocli.go`.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-beta.0/x/gov/autocli.go#L94-L97
```

### Positional Arguments

Theo mặc định `autocli` tạo một flag cho mỗi trường trong message protobuf. Tuy nhiên, bạn có thể chọn sử dụng positional arguments thay vì flags cho một số trường nhất định.

Để thêm positional arguments vào một lệnh, sử dụng struct `autocliv1.PositionalArgDescriptor` và chỉ định tham số `ProtoField` là tên của trường protobuf cần dùng làm positional argument. Ngoài ra, nếu tham số là biến độ dài, bạn có thể đặt `Varargs` thành `true`. Điều này chỉ có thể áp dụng cho tham số positional cuối cùng và `ProtoField` phải là trường repeated.

Đây là ví dụ về cách định nghĩa positional argument cho phương thức `Account` của service `auth`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-beta.0/x/auth/autocli.go#L25-L30
```

Sau đó lệnh có thể được dùng như sau, thay vì phải chỉ định flag `--address`:

```bash
<appd> query auth account cosmos1abcd...xyz
```

#### Trường phẳng hóa trong Positional Arguments

AutoCLI cũng hỗ trợ phẳng hóa các trường message lồng nhau làm positional arguments. Điều này cho phép truy cập các trường lồng nhau bằng ký hiệu dấu chấm trong tham số `ProtoField`. Điều này đặc biệt hữu ích khi bạn muốn trực tiếp đặt các trường message lồng nhau làm positional arguments.

Ví dụ, với cấu trúc message lồng nhau sau:

```protobuf
message Permissions {
    string level = 1;
    repeated string limit_type_urls = 2;
}

message MsgAuthorizeCircuitBreaker {
    string grantee = 1;
    Permissions permissions = 2;
}
```

Bạn có thể phẳng hóa các trường trong cấu hình AutoCLI:

```go
{
    RpcMethod: "AuthorizeCircuitBreaker",
    Use:       "authorize <grantee> <level> <msg_type_urls>",
    PositionalArgs: []*autocliv1.PositionalArgDescriptor{
        {ProtoField: "grantee"},
        {ProtoField: "permissions.level"},
        {ProtoField: "permissions.limit_type_urls"},
    },
}
```

Điều này cho phép người dùng cung cấp giá trị cho các trường lồng nhau trực tiếp dưới dạng positional arguments:

```bash
<appd> tx circuit authorize cosmos1... super-admin "/cosmos.bank.v1beta1.MsgSend,/cosmos.bank.v1beta1.MsgMultiSend"
```

Thay vì phải cung cấp cấu trúc JSON phức tạp cho các trường lồng nhau, việc phẳng hóa giúp CLI thân thiện hơn với người dùng bằng cách cho phép truy cập trực tiếp vào các trường lồng nhau.

#### Tùy chỉnh tên Flag

Theo mặc định, `autocli` tạo tên flag dựa trên tên các trường trong message protobuf. Tuy nhiên, bạn có thể tùy chỉnh tên flag bằng cách cung cấp `FlagOptions`. Tham số này cho phép chỉ định tên tùy chỉnh cho các flag dựa trên tên của các trường message.

Ví dụ, với message có các trường `test` và `test1`, bạn có thể dùng các tùy chọn đặt tên sau để tùy chỉnh flags:

``` go
autocliv1.RpcCommandOptions{ 
    FlagOptions: map[string]*autocliv1.FlagOptions{ 
        "test": { Name: "custom_name", }, 
        "test1": { Name: "other_name", }, 
    }, 
}
```

`FlagsOptions` được định nghĩa giống như sub-command trong phương thức `AutoCLIOptions()` trên module của bạn.

### Kết hợp AutoCLI với các lệnh khác trong module

AutoCLI có thể được dùng cùng với các lệnh khác trong một module. Ví dụ, module `gov` dùng AutoCLI để tạo lệnh cho subcommand `query`, nhưng cũng định nghĩa các lệnh tùy chỉnh cho subcommand `proposer`.

Để bật hành vi này, đặt trường `EnhanceCustomCommand` thành `true` trong `AutoCLIOptions()` cho loại lệnh (queries và/hoặc transactions) bạn muốn mở rộng.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/fa4d87ef7e6d87aaccc94c337ffd2fe90fcb7a9d/x/gov/autocli.go#L98
```

Nếu không đặt thành true, `AutoCLI` sẽ không tạo lệnh cho module nếu đã có lệnh được đăng ký cho module đó (khi `GetTxCmd()` hoặc `GetQueryCmd()` đã được định nghĩa).

### Bỏ qua một lệnh

AutoCLI tự động bỏ qua các lệnh không được hỗ trợ khi [annotation protobuf `cosmos_proto.method_added_in`](https://docs.cosmos.network/main/build/building-modules/protobuf-annotations) có mặt.

Ngoài ra, một lệnh có thể bị bỏ qua thủ công bằng cách dùng `autocliv1.RpcCommandOptions`:

```go
*autocliv1.RpcCommandOptions{
  RpcMethod: "Params", // Tên của gRPC service
  Skip: true,
}
```

### Dùng AutoCLI cho lệnh không phải module

Có thể dùng `AutoCLI` cho các lệnh không phải module. Thủ thuật là vẫn triển khai interface `appmodule.Module` và thêm nó vào map `appOptions.ModuleOptions`.

Ví dụ, đây là cách SDK thực hiện cho các lệnh gRPC của `cometbft`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/client/v2.0.0-beta.1/client/grpc/cmtservice/autocli.go#L52-L71
```

## Tóm tắt

`autocli` cho phép bạn tạo CLI cho các ứng dụng dựa trên Cosmos SDK mà không cần boilerplate cobra. Nó cho phép bạn dễ dàng tạo lệnh CLI và flags từ các message protobuf, và cung cấp nhiều tùy chọn để tùy chỉnh hành vi của ứng dụng CLI.
