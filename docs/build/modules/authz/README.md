---
sidebar_position: 1
---

# `x/authz`

## Tóm tắt

`x/authz` là một hiện thực module Cosmos SDK theo [ADR 30](https://github.com/cosmos/cosmos-sdk/blob/main/docs/architecture/adr-030-authz-module.md),
cho phép cấp (grant) các đặc quyền tuỳ ý từ một tài khoản (granter) sang một tài khoản khác (grantee).
Authorization phải được cấp cho từng phương thức Msg service (Msg service method) một cách riêng lẻ,
bằng một hiện thực của interface `Authorization`.

## Nội dung

* [Khái niệm](#khái-niệm)
  * [Authorization và Grant](#authorization-và-grant)
  * [Authorization tích hợp sẵn](#authorization-tích-hợp-sẵn)
  * [Gas](#gas)
* [State](#state)
  * [Grant](#grant)
  * [GrantQueue](#grantqueue)
* [Messages](#messages)
  * [MsgGrant](#msggrant)
  * [MsgRevoke](#msgrevoke)
  * [MsgExec](#msgexec)
* [Events](#events)
* [Client](#client)
  * [CLI](#cli)
  * [gRPC](#grpc)
  * [REST](#rest)

## Khái niệm

### Authorization và Grant

Module `x/authz` định nghĩa các interface và message để cấp authorization nhằm thực hiện
hành động thay mặt một tài khoản cho tài khoản khác. Thiết kế được mô tả trong
[ADR 030](https://github.com/cosmos/cosmos-sdk/blob/main/docs/architecture/adr-030-authz-module.md).

Một *grant* là một “hạn mức/cho phép” để grantee thực thi một Msg thay mặt granter.
Authorization là một interface cần được hiện thực bởi logic authorization cụ thể để validate và thực thi grant.
Authorization có thể mở rộng và có thể được định nghĩa cho bất kỳ phương thức Msg service nào, kể cả bên ngoài module nơi phương thức Msg đó được định nghĩa.
Xem ví dụ `SendAuthorization` ở phần tiếp theo để biết thêm.

**Lưu ý:** module authz khác với module [auth (authentication)](../auth/) chịu trách nhiệm xác định các kiểu transaction và account cơ sở.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/x/authz/authorizations.go#L11-L25
```

### Authorization tích hợp sẵn

Module `x/authz` của Cosmos SDK cung cấp các loại authorization sau:

#### GenericAuthorization

`GenericAuthorization` hiện thực interface `Authorization`, cho phép không hạn chế (unrestricted)
để thực thi Msg được cung cấp thay mặt tài khoản granter.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/authz/v1beta1/authz.proto#L14-L22
```

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/x/authz/generic_authorization.go#L16-L29
```

* `msg` lưu Msg type URL.

#### SendAuthorization

`SendAuthorization` hiện thực interface `Authorization` cho Msg `cosmos.bank.v1beta1.MsgSend`.

* Nhận một `SpendLimit` (dương) chỉ định lượng token tối đa mà grantee có thể chi. `SpendLimit` sẽ được cập nhật khi token được chi.
* Nhận một `AllowList` (tuỳ chọn) chỉ định grantee có thể gửi token tới các địa chỉ nào.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/bank/v1beta1/authz.proto#L11-L30
```

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/x/bank/types/send_authorization.go#L29-L62
```

* `spend_limit` theo dõi còn bao nhiêu coin còn lại trong authorization.
* `allow_list` chỉ định danh sách địa chỉ (tuỳ chọn) mà grantee có thể gửi token tới thay mặt granter.

#### StakeAuthorization

`StakeAuthorization` hiện thực interface `Authorization` cho các message trong [module staking](https://docs.cosmos.network/v0.53/build/modules/staking).
Nó nhận một `AuthorizationType` để chỉ định bạn muốn uỷ quyền cho delegating, undelegating hay redelegating (tức là các hành động này phải được uỷ quyền riêng).
Nó cũng nhận một `MaxTokens` (tuỳ chọn) để theo dõi giới hạn lượng token có thể delegate/undelegate/redelegate. Nếu để trống thì không giới hạn.
Ngoài ra, Msg này nhận `AllowList` hoặc `DenyList` để bạn chọn validator nào cho phép hoặc cấm grantee stake cùng.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/staking/v1beta1/authz.proto#L11-L35
```

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/x/staking/types/authz.go#L15-L35
```

### Gas

Để ngăn tấn công DoS, việc cấp `StakeAuthorization` bằng `x/authz` sẽ tốn gas.
`StakeAuthorization` cho phép bạn uỷ quyền cho tài khoản khác delegate/undelegate/redelegate tới validator.
Người uỷ quyền có thể định nghĩa danh sách validator cho phép hoặc cấm. Cosmos SDK sẽ lặp qua các danh sách
và tính 10 gas cho mỗi validator trong mỗi danh sách.

Vì state duy trì danh sách cho cặp granter/grantee có cùng expiration, ta phải lặp qua danh sách để xoá grant
(trong trường hợp revoke một `msgType` cụ thể) và tính 20 gas cho mỗi lần lặp.

## State

### Grant

Grant được định danh bằng cách kết hợp: địa chỉ granter (bytes), địa chỉ grantee (bytes),
và loại Authorization (type URL). Vì vậy ta chỉ cho phép một grant cho bộ ba (granter, grantee, Authorization).

* Grant: `0x01 | granter_address_len (1 byte) | granter_address_bytes | grantee_address_len (1 byte) | grantee_address_bytes |  msgType_bytes -> ProtocolBuffer(AuthorizationGrant)`

Đối tượng grant đóng gói một loại `Authorization` và một expiration timestamp:

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/authz/v1beta1/authz.proto#L24-L32
```

### GrantQueue

Ta duy trì một hàng đợi (queue) để pruning authz. Mỗi khi tạo grant, một item sẽ được thêm vào
`GrantQueue` với key gồm expiration, granter, grantee.

Trong `EndBlock` (chạy ở mọi block), ta liên tục kiểm tra và prune các grant hết hạn bằng cách tạo
prefix key từ blocktime hiện tại đã vượt expiration được lưu trong `GrantQueue`, lặp qua tất cả record
khớp từ `GrantQueue` và xoá chúng khỏi `GrantQueue` và store `Grant`.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/5f4ddc6f80f9707320eec42182184207fff3833a/x/authz/keeper/keeper.go#L378-L403
```

* GrantQueue: `0x02 | expiration_bytes | granter_address_len (1 byte) | granter_address_bytes | grantee_address_len (1 byte) | grantee_address_bytes -> ProtocolBuffer(GrantQueueItem)`

`expiration_bytes` là ngày hết hạn theo UTC theo định dạng `"2006-01-02T15:04:05.000000000"`.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/x/authz/keeper/keys.go#L77-L93
```

Đối tượng `GrantQueueItem` chứa danh sách type URL giữa granter và grantee sẽ hết hạn tại thời điểm được chỉ ra trong key.

## Messages

Phần này mô tả xử lý message cho module authz.

### MsgGrant

Tạo một authorization grant bằng message `MsgGrant`.
Nếu đã tồn tại grant cho bộ ba `(granter, grantee, Authorization)`, grant mới sẽ ghi đè grant cũ.
Để cập nhật hoặc gia hạn một grant hiện có, hãy tạo một grant mới với cùng bộ ba `(granter, grantee, Authorization)`.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/authz/v1beta1/tx.proto#L35-L45
```

Xử lý message nên thất bại nếu:

* granter và grantee có cùng địa chỉ.
* `Expiration` được cung cấp nhỏ hơn unix timestamp hiện tại (nhưng nếu không cung cấp `expiration` thì vẫn tạo grant vì `expiration` là tuỳ chọn).
* `Grant.Authorization` được cung cấp không được hiện thực.
* `Authorization.MsgTypeURL()` không được định nghĩa trong router (không có handler trong app router để xử lý Msg type đó).

### MsgRevoke

Một grant có thể bị xoá bằng message `MsgRevoke`.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/authz/v1beta1/tx.proto#L69-L78
```

Xử lý message nên thất bại nếu:

* granter và grantee có cùng địa chỉ.
* `MsgTypeUrl` được cung cấp rỗng.

LƯU Ý: Message `MsgExec` sẽ xoá grant nếu grant đã hết hạn.

### MsgExec

Khi grantee muốn thực thi một giao dịch thay mặt granter, họ phải gửi `MsgExec`.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/authz/v1beta1/tx.proto#L52-L63
```

Xử lý message nên thất bại nếu:

* `Authorization` được cung cấp không được hiện thực.
* grantee không có quyền chạy giao dịch.
* authorization được cấp đã hết hạn.

## Events

Module authz phát ra proto event được định nghĩa trong [tham chiếu Protobuf](https://buf.build/cosmos/cosmos-sdk/docs/main/cosmos.authz.v1beta1#cosmos.authz.v1beta1.EventGrant).

## Client

### CLI

Người dùng có thể truy vấn và tương tác với module `authz` bằng CLI.

#### Query

Các lệnh `query` cho phép truy vấn state của `authz`.

```bash
simd query authz --help
```

##### grants

Lệnh `grants` cho phép truy vấn grant theo cặp granter-grantee. Nếu đặt message type URL,
nó chỉ chọn grant cho message type đó.

```bash
simd query authz grants [granter-addr] [grantee-addr] [msg-type-url]? [flags]
```

Ví dụ:

```bash
simd query authz grants cosmos1.. cosmos1.. /cosmos.bank.v1beta1.MsgSend
```

Ví dụ output:

```bash
grants:
- authorization:
    '@type': /cosmos.bank.v1beta1.SendAuthorization
    spend_limit:
    - amount: "100"
      denom: stake
  expiration: "2022-01-01T00:00:00Z"
pagination: null
```

#### Transactions

Các lệnh `tx` cho phép tương tác với module `authz`.

```bash
simd tx authz --help
```

##### exec

Lệnh `exec` cho phép grantee thực thi một giao dịch thay mặt granter.

```bash
  simd tx authz exec [tx-json-file] --from [grantee] [flags]
```

Ví dụ:

```bash
simd tx authz exec tx.json --from=cosmos1..
```

##### grant

Lệnh `grant` cho phép granter cấp authorization cho grantee.

```bash
simd tx authz grant <grantee> <authorization_type="send"|"generic"|"delegate"|"unbond"|"redelegate"> --from <granter> [flags]
```

* `send` là authorization_type trỏ tới loại tích hợp sẵn `SendAuthorization`. Các cờ tuỳ biến là `spend-limit` (bắt buộc) và `allow-list` (tuỳ chọn), mô tả [tại đây](#sendauthorization)

Ví dụ:

```bash
    simd tx authz grant cosmos1.. send --spend-limit=100stake --allow-list=cosmos1...,cosmos2... --from=cosmos1..
```

* `generic` là authorization_type trỏ tới loại tích hợp sẵn `GenericAuthorization`. Cờ tuỳ biến là `msg-type` (bắt buộc), mô tả [tại đây](#genericauthorization).

> Lưu ý: `msg-type` là bất kỳ Cosmos SDK `Msg` type URL hợp lệ nào.

Ví dụ:

```bash
    simd tx authz grant cosmos1.. generic --msg-type=/cosmos.bank.v1beta1.MsgSend --from=cosmos1..
```

* `delegate`, `unbond`, `redelegate` là authorization_type trỏ tới loại tích hợp sẵn `StakeAuthorization`. Các cờ tuỳ biến gồm `spend-limit` (tuỳ chọn), `allowed-validators` (tuỳ chọn) và `deny-validators` (tuỳ chọn) mô tả [tại đây](#stakeauthorization).

> Lưu ý: `allowed-validators` và `deny-validators` không thể đồng thời rỗng. `spend-limit` đại diện cho `MaxTokens`.

Ví dụ:

```bash
simd tx authz grant cosmos1.. delegate --spend-limit=100stake --allowed-validators=cosmos...,cosmos... --deny-validators=cosmos... --from=cosmos1..
```

##### revoke

Lệnh `revoke` cho phép granter thu hồi authorization từ grantee.

```bash
simd tx authz revoke [grantee] [msg-type-url] --from=[granter] [flags]
```

Ví dụ:

```bash
simd tx authz revoke cosmos1.. /cosmos.bank.v1beta1.MsgSend --from=cosmos1..
```

### gRPC

Người dùng có thể truy vấn module `authz` qua các endpoint gRPC.

#### Grants

Endpoint `Grants` cho phép truy vấn grant theo cặp granter-grantee. Nếu đặt message type URL,
nó chỉ chọn grant cho message type đó.

```bash
cosmos.authz.v1beta1.Query/Grants
```

Ví dụ:

```bash
grpcurl -plaintext \
    -d '{"granter":"cosmos1..","grantee":"cosmos1..","msg_type_url":"/cosmos.bank.v1beta1.MsgSend"}' \
    localhost:9090 \
    cosmos.authz.v1beta1.Query/Grants
```

Ví dụ output:

```bash
{
  "grants": [
    {
      "authorization": {
        "@type": "/cosmos.bank.v1beta1.SendAuthorization",
        "spendLimit": [
          {
            "denom":"stake",
            "amount":"100"
          }
        ]
      },
      "expiration": "2022-01-01T00:00:00Z"
    }
  ]
}
```

### REST

Người dùng có thể truy vấn module `authz` qua các endpoint REST.

```bash
/cosmos/authz/v1beta1/grants
```

Ví dụ:

```bash
curl "localhost:1317/cosmos/authz/v1beta1/grants?granter=cosmos1..&grantee=cosmos1..&msg_type_url=/cosmos.bank.v1beta1.MsgSend"
```

Ví dụ output:

```bash
{
  "grants": [
    {
      "authorization": {
        "@type": "/cosmos.bank.v1beta1.SendAuthorization",
        "spend_limit": [
          {
            "denom": "stake",
            "amount": "100"
          }
        ]
      },
      "expiration": "2022-01-01T00:00:00Z"
    }
  ],
  "pagination": null
}
```

