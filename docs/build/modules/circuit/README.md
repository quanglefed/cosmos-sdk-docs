# `x/circuit`

## Khái niệm

Circuit Breaker là một module nhằm tránh việc chain phải halt/tắt khi xuất hiện
một lỗ hổng; thay vào đó, module cho phép vô hiệu hoá (disable) các message cụ
thể hoặc toàn bộ message. Khi vận hành một chain, nếu chain chỉ phục vụ mục đích
riêng của ứng dụng (app-specific) thì việc halt ít gây tác hại hơn; nhưng nếu có
nhiều ứng dụng được xây dựng trên chain thì việc halt sẽ rất tốn kém do gây gián
đoạn cho các ứng dụng đó.

Circuit Breaker hoạt động dựa trên ý tưởng rằng một địa chỉ hoặc một tập địa chỉ
có quyền chặn message không được thực thi và/hoặc không được đưa vào mempool.
Bất kỳ địa chỉ nào có quyền cũng có thể reset circuit breaker cho message.

Giao dịch sẽ được kiểm tra và có thể bị từ chối tại hai điểm:

* Trong [ante handler](https://docs.cosmos.network/main/learn/advanced/baseapp#antehandler) `CircuitBreakerDecorator`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/x/circuit/v0.1.0/x/circuit/ante/circuit.go#L27-L41
``` 

* Bằng một [kiểm tra ở message router](https://docs.cosmos.network/main/learn/advanced/baseapp#msg-service-router):

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.1/baseapp/msg_service_router.go#L104-L115
``` 

::::note
`CircuitBreakerDecorator` hoạt động cho phần lớn use-case, nhưng [không kiểm tra các message “bên trong” của một giao dịch](https://docs.cosmos.network/main/learn/beginner/tx-lifecycle#antehandler).
Điều này nghĩa là một số giao dịch (như giao dịch `x/authz` hoặc một số giao dịch `x/gov`) có thể vượt qua ante handler.
**Điều này không ảnh hưởng tới circuit breaker** vì kiểm tra ở message router vẫn sẽ làm giao dịch thất bại.
Đánh đổi này nhằm tránh đưa thêm phụ thuộc (dependency) vào module `x/circuit`.
Các chain có thể định nghĩa lại `CircuitBreakerDecorator` để kiểm tra inner message nếu muốn.
::::

## State

### Accounts

* AccountPermissions `0x1 | account_address  -> ProtocolBuffer(CircuitBreakerPermissions)`

```go
type level int32

const (
    // LEVEL_NONE_UNSPECIFIED indicates that the account will have no circuit
    // breaker permissions.
    LEVEL_NONE_UNSPECIFIED = iota
    // LEVEL_SOME_MSGS indicates that the account will have permission to
    // trip or reset the circuit breaker for some Msg type URLs. If this level
    // is chosen, a non-empty list of Msg type URLs must be provided in
    // limit_type_urls.
    LEVEL_SOME_MSGS
    // LEVEL_ALL_MSGS indicates that the account can trip or reset the circuit
    // breaker for Msg's of all type URLs.
    LEVEL_ALL_MSGS 
    // LEVEL_SUPER_ADMIN indicates that the account can take all circuit breaker
    // actions and can grant permissions to other accounts.
    LEVEL_SUPER_ADMIN
)

type Access struct {
	level int32 
	msgs []string // if full permission, msgs can be empty
}
```

### Danh sách vô hiệu hoá (Disable List)

Danh sách type URL đang bị vô hiệu hoá.

* DisableList `0x2 | msg_type_url -> []byte{}` <!--- should this be stored in json to skip encoding and decoding each block, does it matter?-->

## Chuyển trạng thái (State transitions)

### Authorize

Authorize được gọi bởi authority của module (mặc định là tài khoản module governance)
hoặc bất kỳ tài khoản nào có `LEVEL_SUPER_ADMIN` để cấp quyền vô hiệu hoá/bật message
cho một tài khoản khác. Có ba mức quyền có thể cấp. `LEVEL_SOME_MSGS` giới hạn số
message có thể bị vô hiệu hoá. `LEVEL_ALL_MSGS` cho phép vô hiệu hoá mọi message.
`LEVEL_SUPER_ADMIN` cho phép một tài khoản thực hiện mọi hành động circuit breaker,
bao gồm uỷ quyền và huỷ uỷ quyền tài khoản khác.

```protobuf
  // AuthorizeCircuitBreaker allows a super-admin to grant (or revoke) another
  // account's circuit breaker permissions.
  rpc AuthorizeCircuitBreaker(MsgAuthorizeCircuitBreaker) returns (MsgAuthorizeCircuitBreakerResponse);
```

### Trip

Trip được gọi bởi tài khoản đã được uỷ quyền để vô hiệu hoá thực thi message cho
một msgURL cụ thể. Nếu để trống, mọi message sẽ bị vô hiệu hoá.

```protobuf
  // TripCircuitBreaker pauses processing of Msg's in the state machine.
  rpc TripCircuitBreaker(MsgTripCircuitBreaker) returns (MsgTripCircuitBreakerResponse);
```

### Reset

Reset được gọi bởi tài khoản đã được uỷ quyền để bật lại thực thi cho một msgURL
cụ thể đã bị vô hiệu hoá. Nếu để trống, mọi message đang bị vô hiệu hoá sẽ được bật lại.

```protobuf
  // ResetCircuitBreaker resumes processing of Msg's in the state machine that
  // have been paused using TripCircuitBreaker.
  rpc ResetCircuitBreaker(MsgResetCircuitBreaker) returns (MsgResetCircuitBreakerResponse);
```

## Message

### MsgAuthorizeCircuitBreaker

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/main/proto/cosmos/circuit/v1/tx.proto#L25-L75
```

Message này dự kiến sẽ thất bại nếu:

* granter không phải tài khoản có permission level `LEVEL_SUPER_ADMIN` hoặc không phải module authority

### MsgTripCircuitBreaker

```protobuf reference 
https://github.com/cosmos/cosmos-sdk/blob/main/proto/cosmos/circuit/v1/tx.proto#L77-L93
```

Message này dự kiến sẽ thất bại nếu:

* signer không có permission level đủ để vô hiệu hoá message type URL được chỉ định

### MsgResetCircuitBreaker

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/main/proto/cosmos/circuit/v1/tx.proto#L95-109
```

Message này dự kiến sẽ thất bại nếu:

* type URL không đang bị vô hiệu hoá

## Events - liệt kê và mô tả tag

Module circuit phát ra các event sau:

### Message Events

#### MsgAuthorizeCircuitBreaker

| Type    | Attribute Key | Attribute Value           |
|---------|---------------|---------------------------|
| string  | granter       | {granterAddress}          |
| string  | grantee       | {granteeAddress}          |
| string  | permission    | {granteePermissions}      |
| message | module        | circuit                   |
| message | action        | authorize_circuit_breaker |

#### MsgTripCircuitBreaker

| Type     | Attribute Key | Attribute Value      |
|----------|---------------|----------------------|
| string   | authority     | {authorityAddress}   |
| []string | msg_urls      | []string{msg_urls}   |
| message  | module        | circuit              |
| message  | action        | trip_circuit_breaker |

#### ResetCircuitBreaker

| Type     | Attribute Key | Attribute Value       |
|----------|---------------|-----------------------|
| string   | authority     | {authorityAddress}    |
| []string | msg_urls      | []string{msg_urls}    |
| message  | module        | circuit               |
| message  | action        | reset_circuit_breaker |

## Keys - danh sách prefix key

* `AccountPermissionPrefix` - `0x01`
* `DisableListPrefix` -  `0x02`

## Client - liệt kê và mô tả CLI, gRPC, REST

## Ví dụ: dùng các lệnh CLI của Circuit Breaker

Phần này cung cấp các ví dụ thực tế khi dùng module Circuit Breaker qua dòng lệnh (CLI).
Các ví dụ minh hoạ cách uỷ quyền tài khoản, vô hiệu hoá (trip) các loại message cụ thể,
và bật lại (reset) chúng khi cần.

### Truy vấn quyền Circuit Breaker

Kiểm tra quyền circuit breaker hiện tại của một tài khoản:

```bash
# Query permissions for a specific account
<appd> query circuit account-permissions <account_address>

# Example:
simd query circuit account-permissions cosmos1...
```

Kiểm tra các loại message hiện đang bị vô hiệu hoá:

```bash
# Query all disabled message types
<appd> query circuit disabled-list

# Example:
simd query circuit disabled-list
```

### Uỷ quyền một tài khoản làm Circuit Breaker

Chỉ super-admin hoặc authority của module (thường là tài khoản module governance)
mới có thể cấp quyền circuit breaker cho tài khoản khác:

```bash
# Grant LEVEL_ALL_MSGS permission (can disable any message type)
<appd> tx circuit authorize <grantee_address> --level=ALL_MSGS --from=<super_admin_key> --gas=auto --gas-adjustment=1.5

# Grant LEVEL_SOME_MSGS permission (can only disable specific message types)
<appd> tx circuit authorize <grantee_address> --level=SOME_MSGS --limit-type-urls="/cosmos.bank.v1beta1.MsgSend,/cosmos.staking.v1beta1.MsgDelegate" --from=<super_admin_key> --gas=auto --gas-adjustment=1.5

# Grant LEVEL_SUPER_ADMIN permission (can disable messages and authorize other accounts)
<appd> tx circuit authorize <grantee_address> --level=SUPER_ADMIN --from=<super_admin_key> --gas=auto --gas-adjustment=1.5
```

### Vô hiệu hoá xử lý message (Trip)

Vô hiệu hoá các loại message cụ thể để ngăn thực thi (yêu cầu đã được uỷ quyền):

```bash
# Disable a single message type
<appd> tx circuit trip --type-urls="/cosmos.bank.v1beta1.MsgSend" --from=<authorized_key> --gas=auto --gas-adjustment=1.5

# Disable multiple message types
<appd> tx circuit trip --type-urls="/cosmos.bank.v1beta1.MsgSend,/cosmos.staking.v1beta1.MsgDelegate" --from=<authorized_key> --gas=auto --gas-adjustment=1.5

# Disable all message types (emergency measure)
<appd> tx circuit trip --from=<authorized_key> --gas=auto --gas-adjustment=1.5
```

### Bật lại xử lý message (Reset)

Bật lại các loại message đã bị vô hiệu hoá (yêu cầu đã được uỷ quyền):

```bash
# Re-enable a single message type
<appd> tx circuit reset --type-urls="/cosmos.bank.v1beta1.MsgSend" --from=<authorized_key> --gas=auto --gas-adjustment=1.5

# Re-enable multiple message types
<appd> tx circuit reset --type-urls="/cosmos.bank.v1beta1.MsgSend,/cosmos.staking.v1beta1.MsgDelegate" --from=<authorized_key> --gas=auto --gas-adjustment=1.5

# Re-enable all disabled message types
<appd> tx circuit reset --from=<authorized_key> --gas=auto --gas-adjustment=1.5
```

### Cách dùng trong tình huống khẩn cấp

Khi có lỗ hổng nghiêm trọng trong một loại message cụ thể:

1. Nhanh chóng vô hiệu hoá message bị lỗi:

   ```bash
   <appd> tx circuit trip --type-urls="/cosmos.vulnerable.v1beta1.MsgVulnerable" --from=<authorized_key> --gas=auto --gas-adjustment=1.5
   ```

2. Sau khi triển khai bản sửa lỗi, bật lại message đó:

   ```bash
   <appd> tx circuit reset --type-urls="/cosmos.vulnerable.v1beta1.MsgVulnerable" --from=<authorized_key> --gas=auto --gas-adjustment=1.5
   ```

Điều này cho phép chain vô hiệu hoá “đúng chỗ” chức năng có vấn đề mà không cần
halt toàn bộ chain, tạo thời gian để developer triển khai và phát hành bản sửa.

