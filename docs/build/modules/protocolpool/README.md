---
sidebar_position: 1
---

# `x/protocolpool`

## Khái niệm

`x/protocolpool` là một module bổ trợ (supplemental) của Cosmos SDK xử lý các chức
năng liên quan tới quỹ community pool. Module cung cấp một module account riêng
cho community pool, giúp dễ theo dõi tài sản trong pool. Từ Cosmos SDK v0.53,
community funds có thể được theo dõi bằng module này thay vì module `x/distribution`.
Quỹ sẽ được migrate tự động từ community pool của `x/distribution` sang module
account của `x/protocolpool`.

Module này là `supplemental`, không bắt buộc để chạy một chain Cosmos SDK.
`x/protocolpool` nâng cao chức năng community pool do `x/distribution` cung cấp
và cho phép các module tuỳ biến mở rộng thêm community pool.

Lưu ý: _miễn là external community pool keeper (ở đây là `x/protocolpool`) được
wiring trong cấu hình DI, `x/distribution` sẽ tự động dùng nó như external pool._

## Giới hạn khi sử dụng

Các handler `x/distribution` sau sẽ trả lỗi khi module `protocolpool` được dùng
cùng `x/distribution`:

**QueryService**

* `CommunityPool`

**MsgService**

* `CommunityPoolSpend`
* `FundCommunityPool`

Nếu bạn có các dịch vụ phụ thuộc các chức năng này từ `x/distribution`, hãy cập
nhật chúng để dùng các chức năng tương đương của `x/protocolpool`.

## Chuyển trạng thái (State transitions)

### FundCommunityPool

FundCommunityPool có thể được gọi bởi bất kỳ tài khoản hợp lệ nào để gửi quỹ vào
module account của `x/protocolpool`.

```protobuf
  // FundCommunityPool defines a method to allow an account to directly
  // fund the community pool.
  rpc FundCommunityPool(MsgFundCommunityPool) returns (MsgFundCommunityPoolResponse);
```

### CommunityPoolSpend

CommunityPoolSpend có thể được gọi bởi authority của module (mặc định là tài khoản
module governance) hoặc bất kỳ tài khoản nào có uỷ quyền để chi quỹ từ module account
của `x/protocolpool` đến một địa chỉ nhận.

```protobuf
  // CommunityPoolSpend defines a governance  operation for sending tokens from
  // the community pool in the x/protocolpool module to another account, which
  // could be the governance module itself. The authority is defined in the
  // keeper.
  rpc CommunityPoolSpend(MsgCommunityPoolSpend) returns (MsgCommunityPoolSpendResponse);
```

### CreateContinuousFund

CreateContinuousFund là message dùng để khởi tạo một “quỹ liên tục” (continuous fund)
cho một người nhận cụ thể. Tỷ lệ % quỹ được đề xuất sẽ chỉ được phân phối khi người
nhận yêu cầu rút (withdraw). Việc phân phối quỹ tiếp tục cho tới khi hết thời gian
hết hạn (expiry time) hoặc yêu cầu continuous fund bị huỷ.
LƯU Ý: Tính năng này được thiết kế để hoạt động với bond denom mặc định của SDK.

```protobuf
  // CreateContinuousFund defines a method to distribute a percentage of funds to an address continuously.
  // This ContinuousFund can be indefinite or run until a given expiry time.
  // Funds come from validator block rewards from x/distribution, but may also come from
  // any user who funds the ProtocolPoolEscrow module account directly through x/bank.
  rpc CreateContinuousFund(MsgCreateContinuousFund) returns (MsgCreateContinuousFundResponse);
```

### CancelContinuousFund

CancelContinuousFund là message dùng để huỷ một đề xuất continuous fund hiện có cho
một người nhận cụ thể. Huỷ continuous fund sẽ dừng phân phối quỹ trong tương lai,
và đối tượng state sẽ bị xoá khỏi storage.

```protobuf
  // CancelContinuousFund defines a method for cancelling continuous fund.
  rpc CancelContinuousFund(MsgCancelContinuousFund) returns (MsgCancelContinuousFundResponse);
```

## Message

### MsgFundCommunityPool

Message này gửi coin trực tiếp từ người gửi vào community pool.

:::tip
Nếu bạn biết địa chỉ module account của `x/protocolpool`, bạn có thể dùng trực tiếp
giao dịch bank `send` thay thế.
:::

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/release/v0.53.x/proto/cosmos/protocolpool/v1/tx.proto#L43-L53
```

* Message sẽ thất bại nếu số coin không thể được chuyển từ người gửi sang module account `x/protocolpool`.

```go
func (k Keeper) FundCommunityPool(ctx context.Context, amount sdk.Coins, sender sdk.AccAddress) error {
	return k.bankKeeper.SendCoinsFromAccountToModule(ctx, sender, types.ModuleName, amount)
}
```

### MsgCommunityPoolSpend

Message này phân phối quỹ từ module account `x/protocolpool` tới người nhận bằng
phương thức keeper `DistributeFromCommunityPool`.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/release/v0.53.x/proto/cosmos/protocolpool/v1/tx.proto#L58-L69
```

Message sẽ thất bại nếu:

* Không thể chuyển coin tới người nhận từ module account `x/protocolpool`.
* Địa chỉ `recipient` bị hạn chế.

```go
func (k Keeper) DistributeFromCommunityPool(ctx context.Context, amount sdk.Coins, receiveAddr sdk.AccAddress) error {
	return k.bankKeeper.SendCoinsFromModuleToAccount(ctx, types.ModuleName, receiveAddr, amount)
}
```

### MsgCreateContinuousFund

Message này dùng để tạo continuous fund cho một người nhận cụ thể. Tỷ lệ % quỹ
được đề xuất chỉ được phân phối khi người nhận yêu cầu rút. Việc phân phối tiếp
tục cho tới khi hết hạn hoặc bị huỷ.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/release/v0.53.x/proto/cosmos/protocolpool/v1/tx.proto#L114-L130
```

Message sẽ thất bại nếu:

* Địa chỉ người nhận rỗng hoặc bị hạn chế.
* Tỷ lệ % bằng 0/âm/lớn hơn 1.
* Thời gian hết hạn nhỏ hơn thời gian block hiện tại.

:::warning
Nếu tạo hai đề xuất continuous fund cho cùng một địa chỉ, ContinuousFund trước đó
sẽ bị cập nhật bằng ContinuousFund mới.
:::

```go reference
https://github.com/cosmos/cosmos-sdk/blob/release/v0.53.x/x/protocolpool/keeper/msg_server.go#L103-L166
```

### MsgCancelContinuousFund

Message này dùng để huỷ một đề xuất continuous fund hiện có cho một người nhận cụ thể.
Khi đã huỷ, continuous fund sẽ không còn phân phối quỹ ở mỗi begin block, và đối
tượng state sẽ bị xoá.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/release/v0.53.x/x/protocolpool/proto/cosmos/protocolpool/v1/tx.proto#L136-L161
```

Message sẽ thất bại nếu:

* Địa chỉ người nhận rỗng hoặc bị hạn chế.
* ContinuousFund cho người nhận không tồn tại.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/release/v0.53.x/x/protocolpool/keeper/msg_server.go#L188-L226
```

## Client

Module này tận dụng `AutoCLI`.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/release/v0.53.x/x/protocolpool/autocli.go
```

