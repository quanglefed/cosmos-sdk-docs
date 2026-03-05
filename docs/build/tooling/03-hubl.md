---
sidebar_position: 1
---

# Hubl

`Hubl` là một công cụ cho phép bạn truy vấn bất kỳ blockchain nào dựa trên Cosmos SDK.
Nó tận dụng tính năng [AutoCLI](https://pkg.go.dev/github.com/cosmos/cosmos-sdk/client/v2@v2.0.0-20220916140313-c5245716b516/cli) mới <!-- TODO thay thế bằng tài liệu AutoCLI --> của Cosmos SDK.

## Cài Đặt

Hubl có thể được cài đặt bằng `go install`:

```shell
go install cosmossdk.io/tools/hubl/cmd/hubl@latest
```

Hoặc build từ source:

```shell
git clone --depth=1 https://github.com/cosmos/cosmos-sdk
make hubl
```

Binary sẽ nằm trong `tools/hubl`.

## Sử Dụng

```shell
hubl --help
```

### Thêm Chain

Để cấu hình một chain mới, chỉ cần chạy lệnh này sử dụng flag `--init` và tên của chain như được liệt kê trong chain registry (<https://github.com/cosmos/chain-registry>).

Nếu chain không được liệt kê trong chain registry, bạn có thể sử dụng bất kỳ tên duy nhất nào.

```shell
hubl init [chain-name]
hubl init regen
```

Cấu hình chain được lưu trữ trong `~/.hubl/config.toml`.

:::tip

Khi sử dụng gRPC endpoint không bảo mật, hãy thay đổi trường `insecure` thành `true` trong file config.

```toml
[chains]
[chains.regen]
[[chains.regen.trusted-grpc-endpoints]]
endpoint = 'localhost:9090'
insecure = true
```

Hoặc sử dụng flag `--insecure`:

```shell
hubl init regen --insecure
```

:::

### Query (Truy Vấn)

Để truy vấn một chain, bạn có thể sử dụng lệnh `query`.
Sau đó chỉ định module nào bạn muốn truy vấn và query chính nó.

```shell
hubl regen query auth module-accounts
```
