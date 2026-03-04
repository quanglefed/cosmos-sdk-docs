---
sidebar_position: 1
---

# Tương Tác Với Node

:::note Tóm tắt
Có nhiều cách để tương tác với một node: sử dụng CLI, sử dụng gRPC hoặc sử dụng các REST endpoint.
:::

:::note Đọc Trước

* [Các Endpoint gRPC, REST và CometBFT](../../learn/advanced/06-grpc_rest.md)
* [Chạy Một Node](./01-run-node.md)

:::

## Sử Dụng CLI

Bây giờ chain của bạn đang chạy, đã đến lúc thử gửi token từ tài khoản đầu tiên bạn tạo đến tài khoản thứ hai. Trong một cửa sổ terminal mới, bắt đầu bằng cách chạy lệnh truy vấn sau:

```bash
simd query bank balances $MY_VALIDATOR_ADDRESS
```

Bạn sẽ thấy số dư hiện tại của tài khoản bạn đã tạo. Bây giờ, tạo một tài khoản thứ hai:

```bash
simd keys add recipient --keyring-backend test

# Đặt địa chỉ được tạo ra vào một biến để sử dụng sau.
RECIPIENT=$(simd keys show recipient -a --keyring-backend test)
```

Lệnh trên tạo ra một cặp khóa cục bộ chưa được đăng ký trên chain. Một tài khoản được tạo khi lần đầu nó nhận token từ tài khoản khác. Bây giờ, chạy lệnh sau để gửi token đến tài khoản `recipient`:

```bash
simd tx bank send $MY_VALIDATOR_ADDRESS $RECIPIENT 1000000stake --chain-id my-test-chain --keyring-backend test

# Kiểm tra tài khoản recipient đã nhận được token chưa.
simd query bank balances $RECIPIENT
```

Cuối cùng, ủy thác một số token stake đến validator:

```bash
simd tx staking delegate $(simd keys show my_validator --bech val -a --keyring-backend test) 500stake --from recipient --chain-id my-test-chain --keyring-backend test

# Truy vấn tổng số delegation cho `validator`.
simd query staking delegations-to $(simd keys show my_validator --bech val -a --keyring-backend test)
```

Bạn sẽ thấy hai delegation: cái đầu tiên từ `gentx`, cái thứ hai bạn vừa thực hiện từ tài khoản `recipient`.

## Sử Dụng gRPC

Hệ sinh thái Protobuf đã phát triển các công cụ cho nhiều trường hợp sử dụng, bao gồm tạo code từ các file `*.proto` sang nhiều ngôn ngữ. Chúng ta sẽ chỉ trình bày ba lựa chọn:

* `grpcurl` cho debug và kiểm thử chung,
* lập trình qua Go,
* CosmJS cho các nhà phát triển JavaScript/TypeScript.

### grpcurl

[grpcurl](https://github.com/fullstorydev/grpcurl) giống như `curl` nhưng dành cho gRPC. Làm theo hướng dẫn trong link trước để cài đặt.

```bash
grpcurl -plaintext localhost:9090 list
```

Bạn sẽ thấy danh sách các dịch vụ gRPC, như `cosmos.bank.v1beta1.Query`. Để lấy mô tả của dịch vụ:

```bash
grpcurl -plaintext \
    localhost:9090 \
    describe cosmos.bank.v1beta1.Query
```

Để thực thi một lệnh gọi RPC:

```bash
grpcurl \
    -plaintext \
    -d "{\"address\":\"$MY_VALIDATOR_ADDRESS\"}" \
    localhost:9090 \
    cosmos.bank.v1beta1.Query/AllBalances
```

#### Truy Vấn Dữ Liệu Lịch Sử Sử Dụng grpcurl

```bash
grpcurl \
    -plaintext \
    -H "x-cosmos-block-height: 123" \
    -d "{\"address\":\"$MY_VALIDATOR_ADDRESS\"}" \
    localhost:9090 \
    cosmos.bank.v1beta1.Query/AllBalances
```

### Lập Trình Qua Go

```go
package main

import (
    "context"
    "fmt"

    "google.golang.org/grpc"

    "github.com/cosmos/cosmos-sdk/codec"
    sdk "github.com/cosmos/cosmos-sdk/types"
    banktypes "github.com/cosmos/cosmos-sdk/x/bank/types"
)

func queryState() error {
    myAddress, err := sdk.AccAddressFromBech32("cosmos1...")
    if err != nil {
        return err
    }

    grpcConn, err := grpc.Dial(
        "127.0.0.1:9090",
        grpc.WithInsecure(),
		grpc.WithDefaultCallOptions(grpc.ForceCodec(codec.NewProtoCodec(nil).GRPCCodec())),
	)
    if err != nil {
        return err
    }
    defer grpcConn.Close()

    bankClient := banktypes.NewQueryClient(grpcConn)
    bankRes, err := bankClient.Balance(
        context.Background(),
        &banktypes.QueryBalanceRequest{Address: myAddress.String(), Denom: "stake"},
    )
    if err != nil {
        return err
    }

    fmt.Println(bankRes.GetBalance())
    return nil
}

func main() {
    if err := queryState(); err != nil {
        panic(err)
    }
}
```

#### Truy Vấn Dữ Liệu Lịch Sử Sử Dụng Go

```go
package main

import (
	"context"
	"fmt"

	"google.golang.org/grpc"
	"google.golang.org/grpc/metadata"

	"github.com/cosmos/cosmos-sdk/codec"
	sdk "github.com/cosmos/cosmos-sdk/types"
	grpctypes "github.com/cosmos/cosmos-sdk/types/grpc"
	banktypes "github.com/cosmos/cosmos-sdk/x/bank/types"
)

func queryState() error {
	myAddress, err := sdk.AccAddressFromBech32("cosmos1yerherx4d43gj5wa3zl5vflj9d4pln42n7kuzu")
	if err != nil {
		return err
	}

	grpcConn, err := grpc.Dial(
		"127.0.0.1:9090",
		grpc.WithInsecure(),
		grpc.WithDefaultCallOptions(grpc.ForceCodec(codec.NewProtoCodec(nil).GRPCCodec())),
	)
	if err != nil {
		return err
	}
	defer grpcConn.Close()

	bankClient := banktypes.NewQueryClient(grpcConn)

	var header metadata.MD
	_, err = bankClient.Balance(
		metadata.AppendToOutgoingContext(context.Background(), grpctypes.GRPCBlockHeightHeader, "12"),
		&banktypes.QueryBalanceRequest{Address: myAddress.String(), Denom: "stake"},
		grpc.Header(&header),
	)
	if err != nil {
		return err
	}
	blockHeight := header.Get(grpctypes.GRPCBlockHeightHeader)

	fmt.Println(blockHeight)
	return nil
}

func main() {
    if err := queryState(); err != nil {
        panic(err)
    }
}
```

### CosmJS

Tài liệu CosmJS có thể tìm thấy tại [https://cosmos.github.io/cosmjs](https://cosmos.github.io/cosmjs). Tính đến tháng 1 năm 2021, tài liệu CosmJS vẫn đang trong quá trình hoàn thiện.

## Sử Dụng Các REST Endpoint

Như được mô tả trong [hướng dẫn gRPC](../../learn/advanced/06-grpc_rest.md), tất cả các dịch vụ gRPC trên Cosmos SDK được cung cấp qua gRPC-gateway. Ví dụ, REST endpoint cho `cosmos.bank.v1beta1.Query/AllBalances` là `GET /cosmos/bank/v1beta1/balances/{address}`.

REST endpoint không được bật theo mặc định. Để bật, chỉnh sửa phần `api` trong `~/.simapp/config/app.toml`:

```toml
enable = true
```

Ví dụ lệnh `curl` truy vấn số dư:

```bash
curl \
    -X GET \
    -H "Content-Type: application/json" \
    http://localhost:1317/cosmos/bank/v1beta1/balances/$MY_VALIDATOR_ADDRESS
```

Danh sách tất cả các REST endpoint có sẵn dưới dạng Swagger tại `localhost:1317/swagger`.

### Truy Vấn Dữ Liệu Lịch Sử Sử Dụng REST

```bash
curl \
    -X GET \
    -H "Content-Type: application/json" \
    -H "x-cosmos-block-height: 123" \
    http://localhost:1317/cosmos/bank/v1beta1/balances/$MY_VALIDATOR_ADDRESS
```

### Cross-Origin Resource Sharing (CORS)

[Chính sách CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) không được bật theo mặc định. Nếu muốn sử dụng rest-server trong môi trường công cộng, khuyến nghị dùng reverse proxy như [nginx](https://www.nginx.com/). Để kiểm thử và phát triển, có trường `enabled-unsafe-cors` trong [`app.toml`](../../user/run-node/01-run-node.md#configuring-the-node-using-apptoml-and-configtoml).
