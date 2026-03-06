---
sidebar_position: 1
---

# `x/auth/tx`

:::note Tài liệu cần đọc trước

* [Transactions](https://docs.cosmos.network/main/core/transactions#transaction-generation)
* [Encoding](https://docs.cosmos.network/main/core/encoding#transaction-encoding)

:::

## Tóm tắt

Tài liệu này mô tả gói `x/auth/tx` của Cosmos SDK.

Gói này là phần triển khai trong Cosmos SDK cho các interface `client.TxConfig`,
`client.TxBuilder`, `client.TxEncoder` và `client.TxDecoder`.

## Nội dung

* [Transactions](#transactions)
  * [`TxConfig`](#txconfig)
  * [`TxBuilder`](#txbuilder)
  * [`TxEncoder`/ `TxDecoder`](#txencoder-txdecoder)
* [Client](#client)
  * [CLI](#cli)
  * [gRPC](#grpc)

## Transactions

### `TxConfig`

`client.TxConfig` định nghĩa một interface mà client có thể dùng để tạo ra kiểu
giao dịch cụ thể (concrete transaction type) do ứng dụng quy định.
Interface này cung cấp một tập phương thức để tạo `client.TxBuilder`.

```go reference
https://github.com/cosmos/cosmos-sdk/tree/release/v0.50.x/client/tx_config.go#L25-L31
```

Hiện thực mặc định của `client.TxConfig` được khởi tạo bởi `NewTxConfig` trong
mô-đun `x/auth/tx`.

```go reference
https://github.com/cosmos/cosmos-sdk/tree/release/v0.50.x/x/auth/tx/config.go#L22-L28
```

### `TxBuilder`

```go reference
https://github.com/cosmos/cosmos-sdk/tree/release/v0.50.x/client/tx_config.go#L33-L50
```

Interface [`client.TxBuilder`](https://docs.cosmos.network/main/core/transactions#transaction-generation)
cũng được `x/auth/tx` triển khai.
Bạn có thể lấy một `client.TxBuilder` qua `TxConfig.NewTxBuilder()`.

### `TxEncoder`/ `TxDecoder`

Xem thêm thông tin về `TxEncoder` và `TxDecoder` [tại đây](https://docs.cosmos.network/main/core/encoding#transaction-encoding).

## Client

### CLI

#### Query

Mô-đun `x/auth/tx` cung cấp lệnh CLI để truy vấn một giao dịch bất kỳ dựa trên
hash, sequence của giao dịch, hoặc chữ ký.

Nếu không có tham số bổ sung, lệnh sẽ truy vấn theo hash của giao dịch.

```shell
simd query tx DFE87B78A630C0EFDF76C80CD24C997E252792E0317502AE1A02B9809F0D8685
```

Khi truy vấn một giao dịch từ một tài khoản theo sequence, dùng cờ `--type=acc_seq`:

```shell
simd query tx --type=acc_seq cosmos1u69uyr6v9qwe6zaaeaqly2h6wnedac0xpxq325/1
```

Khi truy vấn theo chữ ký, dùng cờ `--type=signature`:

```shell
simd query tx --type=signature Ofjvgrqi8twZfqVDmYIhqwRLQjZZ40XbxEamk/veH3gQpRF0hL2PH4ejRaDzAX+2WChnaWNQJQ41ekToIi5Wqw==
```

Khi truy vấn theo events, dùng cờ `--type=events`:

```shell
simd query txs --events 'message.sender=cosmos...' --page 1 --limit 30
```

Mô-đun `x/auth/block` cung cấp lệnh CLI để truy vấn một block bất kỳ theo hash,
height hoặc events.

Khi truy vấn block theo hash, dùng cờ `--type=hash`:

```shell
simd query block --type=hash DFE87B78A630C0EFDF76C80CD24C997E252792E0317502AE1A02B9809F0D8685
```

Khi truy vấn block theo height, dùng cờ `--type=height`:

```shell
simd query block --type=height 1357
```

Khi truy vấn block theo events, dùng cờ `--query`:

```shell
simd query blocks --query 'message.sender=cosmos...' --page 1 --limit 30
```

#### Transactions

Mô-đun `x/auth/tx` cung cấp các lệnh CLI tiện lợi để encode/decode giao dịch.

#### `encode`

Lệnh `encode` dùng để encode một giao dịch được tạo với cờ `--generate-only` hoặc
đã được ký bằng lệnh `sign`.
Giao dịch sẽ được serialize sang Protobuf và trả về dưới dạng base64.

```bash
$ simd tx encode tx.json
Co8BCowBChwvY29zbW9zLmJhbmsudjFiZXRhMS5Nc2dTZW5kEmwKLWNvc21vczFsNnZzcWhoN3Jud3N5cjJreXozampnM3FkdWF6OGd3Z3lsODI3NRItY29zbW9zMTU4c2FsZHlnOHBteHU3Znd2dDBkNng3amVzd3A0Z3d5a2xrNnkzGgwKBXN0YWtlEgMxMDASBhIEEMCaDA==
$ simd tx encode tx.signed.json
```

Xem thêm thông tin về lệnh `encode` bằng cách chạy `simd tx encode --help`.

#### `decode`

Lệnh `decode` dùng để giải mã (decode) một giao dịch đã được encode bởi lệnh `encode`.

```bash
simd tx decode Co8BCowBChwvY29zbW9zLmJhbmsudjFiZXRhMS5Nc2dTZW5kEmwKLWNvc21vczFsNnZzcWhoN3Jud3N5cjJreXozampnM3FkdWF6OGd3Z3lsODI3NRItY29zbW9zMTU4c2FsZHlnOHBteHU3Znd2dDBkNng3amVzd3A0Z3d5a2xrNnkzGgwKBXN0YWtlEgMxMDASBhIEEMCaDA==
```

Xem thêm thông tin về lệnh `decode` bằng cách chạy `simd tx decode --help`.

### gRPC

Người dùng có thể truy vấn mô-đun `x/auth/tx` thông qua các gRPC endpoint.

#### `TxDecode`

Endpoint `TxDecode` cho phép decode một giao dịch.

```shell
cosmos.tx.v1beta1.Service/TxDecode
```

Ví dụ:

```shell
grpcurl -plaintext \
    -d '{"tx_bytes":"Co8BCowBChwvY29zbW9zLmJhbmsudjFiZXRhMS5Nc2dTZW5kEmwKLWNvc21vczFsNnZzcWhoN3Jud3N5cjJreXozampnM3FkdWF6OGd3Z3lsODI3NRItY29zbW9zMTU4c2FsZHlnOHBteHU3Znd2dDBkNng3amVzd3A0Z3d5a2xrNnkzGgwKBXN0YWtlEgMxMDASBhIEEMCaDA=="}' \
    localhost:9090 \
    cosmos.tx.v1beta1.Service/TxDecode
```

Ví dụ output:

```json
{
  "tx": {
    "body": {
      "messages": [
        {"@type":"/cosmos.bank.v1beta1.MsgSend","amount":[{"denom":"stake","amount":"100"}],"fromAddress":"cosmos1l6vsqhh7rnwsyr2kyz3jjg3qduaz8gwgyl8275","toAddress":"cosmos158saldyg8pmxu7fwvt0d6x7jeswp4gwyklk6y3"}
      ]
    },
    "authInfo": {
      "fee": {
        "gasLimit": "200000"
      }
    }
  }
}
```

#### `TxEncode`

Endpoint `TxEncode` cho phép encode một giao dịch.

```shell
cosmos.tx.v1beta1.Service/TxEncode
```

Ví dụ:

```shell
grpcurl -plaintext \
    -d '{"tx": {
    "body": {
      "messages": [
        {"@type":"/cosmos.bank.v1beta1.MsgSend","amount":[{"denom":"stake","amount":"100"}],"fromAddress":"cosmos1l6vsqhh7rnwsyr2kyz3jjg3qduaz8gwgyl8275","toAddress":"cosmos158saldyg8pmxu7fwvt0d6x7jeswp4gwyklk6y3"}
      ]
    },
    "authInfo": {
      "fee": {
        "gasLimit": "200000"
      }
    }
  }}' \
    localhost:9090 \
    cosmos.tx.v1beta1.Service/TxEncode
```

Ví dụ output:

```json
{
  "txBytes": "Co8BCowBChwvY29zbW9zLmJhbmsudjFiZXRhMS5Nc2dTZW5kEmwKLWNvc21vczFsNnZzcWhoN3Jud3N5cjJreXozampnM3FkdWF6OGd3Z3lsODI3NRItY29zbW9zMTU4c2FsZHlnOHBteHU3Znd2dDBkNng3amVzd3A0Z3d5a2xrNnkzGgwKBXN0YWtlEgMxMDASBhIEEMCaDA=="
}
```

#### `TxDecodeAmino`

Endpoint `TxDecodeAmino` cho phép decode một giao dịch amino.

```shell
cosmos.tx.v1beta1.Service/TxDecodeAmino
```

Ví dụ:

```shell
grpcurl -plaintext \
    -d '{"amino_binary": "KCgWqQpvqKNhmgotY29zbW9zMXRzeno3cDJ6Z2Q3dnZrYWh5ZnJlNHduNXh5dTgwcnB0ZzZ2OWg1Ei1jb3Ntb3MxdHN6ejdwMnpnZDd2dmthaHlmcmU0d241eHl1ODBycHRnNnY5aDUaCwoFc3Rha2USAjEwEhEKCwoFc3Rha2USAjEwEMCaDCIGZm9vYmFy"}' \
    localhost:9090 \
    cosmos.tx.v1beta1.Service/TxDecodeAmino
```

Ví dụ output:

```json
{
  "aminoJson": "{\"type\":\"cosmos-sdk/StdTx\",\"value\":{\"msg\":[{\"type\":\"cosmos-sdk/MsgSend\",\"value\":{\"from_address\":\"cosmos1tszz7p2zgd7vvkahyfre4wn5xyu80rptg6v9h5\",\"to_address\":\"cosmos1tszz7p2zgd7vvkahyfre4wn5xyu80rptg6v9h5\",\"amount\":[{\"denom\":\"stake\",\"amount\":\"10\"}]}}],\"fee\":{\"amount\":[{\"denom\":\"stake\",\"amount\":\"10\"}],\"gas\":\"200000\"},\"signatures\":null,\"memo\":\"foobar\",\"timeout_height\":\"0\"}}"
}
```

#### `TxEncodeAmino`

Endpoint `TxEncodeAmino` cho phép encode một giao dịch amino.

```shell
cosmos.tx.v1beta1.Service/TxEncodeAmino
```

Ví dụ:

```shell
grpcurl -plaintext \
    -d '{"amino_json":"{\"type\":\"cosmos-sdk/StdTx\",\"value\":{\"msg\":[{\"type\":\"cosmos-sdk/MsgSend\",\"value\":{\"from_address\":\"cosmos1tszz7p2zgd7vvkahyfre4wn5xyu80rptg6v9h5\",\"to_address\":\"cosmos1tszz7p2zgd7vvkahyfre4wn5xyu80rptg6v9h5\",\"amount\":[{\"denom\":\"stake\",\"amount\":\"10\"}]}}],\"fee\":{\"amount\":[{\"denom\":\"stake\",\"amount\":\"10\"}],\"gas\":\"200000\"},\"signatures\":null,\"memo\":\"foobar\",\"timeout_height\":\"0\"}}"}' \
    localhost:9090 \
    cosmos.tx.v1beta1.Service/TxEncodeAmino
```

Ví dụ output:

```json
{
  "amino_binary": "KCgWqQpvqKNhmgotY29zbW9zMXRzeno3cDJ6Z2Q3dnZrYWh5ZnJlNHduNXh5dTgwcnB0ZzZ2OWg1Ei1jb3Ntb3MxdHN6ejdwMnpnZDd2dmthaHlmcmU0d241eHl1ODBycHRnNnY5aDUaCwoFc3Rha2USAjEwEhEKCwoFc3Rha2USAjEwEMCaDCIGZm9vYmFy"
}
```

