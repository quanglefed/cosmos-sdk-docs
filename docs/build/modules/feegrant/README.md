---
sidebar_position: 1
---

# `x/feegrant`

## Tóm tắt

Tài liệu này mô tả module fee grant. Để xem đầy đủ ADR, xem [Fee Grant ADR-029](https://github.com/cosmos/cosmos-sdk/blob/main/docs/architecture/adr-029-fee-grant-module.md).

Module này cho phép tài khoản cấp “hạn mức trả phí” (fee allowance) và cho phép sử dụng phí từ tài khoản của họ.
Grantee có thể thực thi bất kỳ giao dịch nào mà không cần duy trì đủ phí.

## Nội dung

* [Khái niệm](#khái-niệm)
* [State](#state)
  * [FeeAllowance](#feeallowance)
  * [FeeAllowanceQueue](#feeallowancequeue)
* [Messages](#messages)
  * [Msg/GrantAllowance](#msggrantallowance)
  * [Msg/RevokeAllowance](#msgrevokeallowance)
* [Events](#events)
* [Msg Server](#msg-server)
  * [MsgGrantAllowance](#msggrantallowance-1)
  * [MsgRevokeAllowance](#msgrevokeallowance-1)
  * [Thực thi fee allowance](#exec-fee-allowance)
* [Client](#client)
  * [CLI](#cli)
  * [gRPC](#grpc)

## Khái niệm

### Grant

`Grant` được lưu trong KVStore để ghi lại một grant với đầy đủ ngữ cảnh. Mỗi grant sẽ chứa `granter`,
`grantee` và loại `allowance` được cấp. `granter` là địa chỉ tài khoản cấp quyền cho `grantee`
(địa chỉ tài khoản thụ hưởng) để trả một phần hoặc toàn bộ phí giao dịch của `grantee` bằng tài
khoản của `granter`. `allowance` định nghĩa loại hạn mức trả phí (`BasicAllowance` hoặc `PeriodicAllowance`,
xem bên dưới) được cấp cho `grantee`. `allowance` nhận một interface triển khai `FeeAllowanceI`,
được encode dưới dạng `Any`. Mỗi cặp `granter`/`grantee` chỉ có thể có một fee grant; không cho phép self grant.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/feegrant/v1beta1/feegrant.proto#L83-L93
```

`FeeAllowanceI` như sau:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/x/feegrant/fees.go#L9-L32
```

### Các loại fee allowance

Hiện có các loại fee allowance sau:

* `BasicAllowance`
* `PeriodicAllowance`
* `AllowedMsgAllowance`

### BasicAllowance

`BasicAllowance` là quyền cho `grantee` sử dụng phí từ tài khoản `granter`. Nếu một trong các giới hạn
`spend_limit` hoặc `expiration` chạm ngưỡng, grant sẽ bị xoá khỏi state.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/feegrant/v1beta1/feegrant.proto#L15-L28
```

* `spend_limit` là giới hạn số coin cho phép dùng từ tài khoản `granter`. Nếu rỗng, hiểu là không giới hạn,
  `grantee` có thể dùng bất kỳ lượng coin khả dụng nào từ tài khoản `granter` trước thời điểm hết hạn.

* `expiration` chỉ định thời điểm tuỳ chọn khi allowance hết hạn. Nếu để trống, grant không hết hạn.

* Khi tạo grant với `spend_limit` và `expiration` đều trống, grant vẫn hợp lệ. Nó sẽ không hạn chế lượng coin
  `grantee` có thể dùng từ `granter` và cũng không có hết hạn. Cách duy nhất để hạn chế `grantee` là revoke grant.

### PeriodicAllowance

`PeriodicAllowance` là hạn mức trả phí lặp lại theo chu kỳ. Ta có thể chỉ định khi nào grant hết hạn,
cũng như khi nào chu kỳ được reset. Ta cũng có thể định nghĩa lượng coin tối đa có thể dùng trong một chu kỳ.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/feegrant/v1beta1/feegrant.proto#L34-L68
```

* `basic` là một instance của `BasicAllowance`, tuỳ chọn đối với periodic fee allowance. Nếu rỗng, grant sẽ không có `expiration` và không có `spend_limit`.
* `period` là độ dài chu kỳ; sau mỗi chu kỳ, `period_can_spend` sẽ được reset.
* `period_spend_limit` chỉ định lượng coin tối đa có thể chi trong một chu kỳ.
* `period_can_spend` là lượng coin còn lại có thể chi trước thời điểm period_reset.
* `period_reset` theo dõi thời điểm reset chu kỳ tiếp theo.

### AllowedMsgAllowance

`AllowedMsgAllowance` là một fee allowance; nó có thể là `BasicFeeAllowance` hoặc `PeriodicAllowance`,
nhưng bị giới hạn chỉ áp dụng cho các message mà granter cho phép.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/feegrant/v1beta1/feegrant.proto#L70-L81
```

* `allowance` là `BasicAllowance` hoặc `PeriodicAllowance`.
* `allowed_messages` là mảng message được phép thực thi theo allowance đã cấp.

### Cờ FeeGranter

Module `feegrant` giới thiệu cờ `FeeGranter` cho CLI để thực thi giao dịch với fee granter.
Khi cờ này được đặt, `clientCtx` sẽ gắn địa chỉ tài khoản granter vào các giao dịch được tạo qua CLI.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/client/cmd.go#L249-L260
```

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/client/tx/tx.go#L109-L109
```

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/x/auth/tx/builder.go#L275-L284
```

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/tx/v1beta1/tx.proto#L203-L224
```

Ví dụ lệnh:

```go
./simd tx gov submit-proposal --title="Test Proposal" --description="My awesome proposal" --type="Text" --from validator-key --fee-granter=cosmos1xh44hxt7spr67hqaa7nyx5gnutrz5fraw6grxn --chain-id=testnet --fees="10stake"
```

### Khấu trừ phí từ grant

Phí được trừ từ grant trong ante handler `x/auth`. Để hiểu thêm ante handler hoạt động thế nào,
hãy đọc [Auth Module AnteHandlers Guide](../auth/README.md#antehandlers).

### Gas

Để ngăn DoS, việc dùng `x/feegrant` với filter sẽ tốn gas. SDK phải đảm bảo các giao dịch của `grantee`
tuân theo filter do `granter` đặt. SDK làm điều này bằng cách lặp qua các allowed message trong filter
và tính 10 gas cho mỗi message được filter. Sau đó SDK lặp qua các message mà `grantee` đang gửi để đảm
bảo các message tuân theo filter, cũng tính 10 gas cho mỗi message. SDK sẽ dừng lặp và làm giao dịch thất
bại nếu phát hiện một message không tuân theo filter.

**CẢNH BÁO**: Gas được tính vào allowance được cấp. Hãy đảm bảo message của bạn tuân theo filter (nếu có)
trước khi gửi giao dịch dùng allowance.

### Pruning

Một queue trong state được duy trì với prefix theo expiration của grant và được kiểm tra ở EndBlock theo block time hiện tại cho mỗi block để prune.

## State

### FeeAllowance

Fee Allowance được định danh bằng cách kết hợp `Grantee` (địa chỉ tài khoản grantee) với `Granter` (địa chỉ tài khoản granter).

Grant fee allowance được lưu trong state như sau:

* Grant: `0x00 | grantee_addr_len (1 byte) | grantee_addr_bytes |  granter_addr_len (1 byte) | granter_addr_bytes -> ProtocolBuffer(Grant)`

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/x/feegrant/feegrant.pb.go#L222-L230
```

### FeeAllowanceQueue

Item trong Fee Allowances queue được định danh bằng cách kết hợp `FeeAllowancePrefixQueue` (tức 0x01),
`expiration`, `grantee` và `granter`. EndBlocker kiểm tra state `FeeAllowanceQueue` để tìm các grant hết hạn
và prune chúng khỏi `FeeAllowance` nếu tìm thấy.

Key của fee allowance queue được lưu như sau:

* Grant: `0x01 | expiration_bytes | grantee_addr_len (1 byte) | grantee_addr_bytes |  granter_addr_len (1 byte) | granter_addr_bytes -> EmptyBytes`

## Messages

### Msg/GrantAllowance

Fee allowance grant được tạo bằng message `MsgGrantAllowance`.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/feegrant/v1beta1/tx.proto#L25-L39
```

### Msg/RevokeAllowance

Một fee allowance grant có thể bị xoá bằng message `MsgRevokeAllowance`.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/feegrant/v1beta1/tx.proto#L41-L54
```

## Events

Module feegrant phát ra các event sau:

## Msg Server

### MsgGrantAllowance

| Type    | Attribute Key | Attribute Value  |
| ------- | ------------- | ---------------- |
| message | action        | set_feegrant     |
| message | granter       | {granterAddress} |
| message | grantee       | {granteeAddress} |

### MsgRevokeAllowance

| Type    | Attribute Key | Attribute Value  |
| ------- | ------------- | ---------------- |
| message | action        | revoke_feegrant  |
| message | granter       | {granterAddress} |
| message | grantee       | {granteeAddress} |

### Thực thi fee allowance

| Type    | Attribute Key | Attribute Value  |
| ------- | ------------- | ---------------- |
| message | action        | use_feegrant     |
| message | granter       | {granterAddress} |
| message | grantee       | {granteeAddress} |

### Prune fee allowance

| Type    | Attribute Key | Attribute Value  |
| ------- | ------------- | ---------------- |
| message | action        |  prune_feegrant  |
| message | pruner        | {prunerAddress}  |

## Client

### CLI

Người dùng có thể truy vấn và tương tác với module `feegrant` bằng CLI.

#### Query

Các lệnh `query` cho phép truy vấn state của `feegrant`.

```shell
simd query feegrant --help
```

##### grant

Lệnh `grant` cho phép truy vấn một grant theo cặp granter-grantee.

```shell
simd query feegrant grant [granter] [grantee] [flags]
```

Ví dụ:

```shell
simd query feegrant grant cosmos1.. cosmos1..
```

Ví dụ output:

```yml
allowance:
  '@type': /cosmos.feegrant.v1beta1.BasicAllowance
  expiration: null
  spend_limit:
  - amount: "100"
    denom: stake
grantee: cosmos1..
granter: cosmos1..
```

##### grants

Lệnh `grants` cho phép truy vấn tất cả grant của một grantee.

```shell
simd query feegrant grants [grantee] [flags]
```

Ví dụ:

```shell
simd query feegrant grants cosmos1..
```

Ví dụ output:

```yml
allowances:
- allowance:
    '@type': /cosmos.feegrant.v1beta1.BasicAllowance
    expiration: null
    spend_limit:
    - amount: "100"
      denom: stake
  grantee: cosmos1..
  granter: cosmos1..
pagination:
  next_key: null
  total: "0"
```

#### Transactions

Các lệnh `tx` cho phép tương tác với module `feegrant`.

```shell
simd tx feegrant --help
```

##### grant

Lệnh `grant` cho phép cấp fee allowance cho tài khoản khác. Fee allowance có thể có ngày hết hạn,
giới hạn tổng chi, và/hoặc giới hạn chi theo chu kỳ.

```shell
simd tx feegrant grant [granter] [grantee] [flags]
```

Ví dụ (giới hạn chi một lần):

```shell
simd tx feegrant grant cosmos1.. cosmos1.. --spend-limit 100stake
```

Ví dụ (giới hạn chi theo chu kỳ):

```shell
simd tx feegrant grant cosmos1.. cosmos1.. --period 3600 --period-limit 10stake
```

##### revoke

Lệnh `revoke` cho phép thu hồi một fee allowance đã cấp.

```shell
simd tx feegrant revoke [granter] [grantee] [flags]
```

Ví dụ:

```shell
simd tx feegrant revoke cosmos1.. cosmos1..
```

### gRPC

Người dùng có thể truy vấn module `feegrant` qua các endpoint gRPC.

#### Allowance

Endpoint `Allowance` cho phép truy vấn một fee allowance đã được cấp.

```shell
cosmos.feegrant.v1beta1.Query/Allowance
```

Ví dụ:

```shell
grpcurl -plaintext \
    -d '{"grantee":"cosmos1..","granter":"cosmos1.."}' \
    localhost:9090 \
    cosmos.feegrant.v1beta1.Query/Allowance
```

Ví dụ output:

```json
{
  "allowance": {
    "granter": "cosmos1..",
    "grantee": "cosmos1..",
    "allowance": {"@type":"/cosmos.feegrant.v1beta1.BasicAllowance","spendLimit":[{"denom":"stake","amount":"100"}]}
  }
}
```

#### Allowances

Endpoint `Allowances` cho phép truy vấn tất cả fee allowance đã cấp cho một grantee.

```shell
cosmos.feegrant.v1beta1.Query/Allowances
```

Ví dụ:

```shell
grpcurl -plaintext \
    -d '{"address":"cosmos1.."}' \
    localhost:9090 \
    cosmos.feegrant.v1beta1.Query/Allowances
```

Ví dụ output:

```json
{
  "allowances": [
    {
      "granter": "cosmos1..",
      "grantee": "cosmos1..",
      "allowance": {"@type":"/cosmos.feegrant.v1beta1.BasicAllowance","spendLimit":[{"denom":"stake","amount":"100"}]}
    }
  ],
  "pagination": {
    "total": "1"
  }
}
```

