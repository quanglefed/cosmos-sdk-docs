---
sidebar_position: 1
---

# `Msg` Services

:::note Tóm tắt
Một Protobuf `Msg` service xử lý các [messages](./02-messages-and-queries.md#messages). Các Protobuf `Msg` service đặc thù cho module mà chúng được định nghĩa, và chỉ xử lý các message được định nghĩa trong module đó. Chúng được gọi từ `BaseApp` trong quá trình [`DeliverTx`](../../learn/advanced/00-baseapp.md#delivertx).
:::

:::note Yêu Cầu Đọc Trước

* [Module Manager](./01-module-manager.md)
* [Messages và Queries](./02-messages-and-queries.md)

:::

## Triển Khai `Msg` service của module

Mỗi module nên định nghĩa một Protobuf `Msg` service, chịu trách nhiệm xử lý các yêu cầu (triển khai `sdk.Msg`) và trả về các phản hồi.

Như được mô tả thêm trong [ADR 031](../../../architecture/adr-031-msg-service.md), cách tiếp cận này có ưu điểm là chỉ định rõ ràng các kiểu trả về và tạo mã server và client.

Protobuf tạo ra một interface `MsgServer` dựa trên định nghĩa của `Msg` service. Vai trò của nhà phát triển module là triển khai interface này, bằng cách triển khai logic chuyển đổi trạng thái phải xảy ra khi nhận mỗi `sdk.Msg`. Ví dụ, đây là interface `MsgServer` được tạo ra cho `x/bank`, cung cấp hai `sdk.Msg`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/bank/types/tx.pb.go#L550-L568
```

Khi có thể, [`Keeper`](./06-keeper.md) hiện có của module nên triển khai `MsgServer`, nếu không thì có thể tạo một struct `msgServer` nhúng `Keeper`, thường trong `./keeper/msg_server.go`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/bank/keeper/msg_server.go#L17-L19
```

Các phương thức `msgServer` có thể lấy `sdk.Context` từ tham số `context.Context` bằng cách sử dụng phương thức `sdk.UnwrapSDKContext`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/bank/keeper/msg_server.go#L56
```

Việc xử lý `sdk.Msg` thường theo 3 bước sau:

### Xác Thực (Validation)

Message server phải thực hiện tất cả các xác thực cần thiết (cả *stateful* và *stateless*) để đảm bảo `message` hợp lệ. `signer` bị tính phí gas cho chi phí xác thực này.

Ví dụ, một phương thức `msgServer` cho message `transfer` nên kiểm tra rằng tài khoản gửi có đủ tiền để thực sự thực hiện chuyển khoản.

Khuyến nghị triển khai tất cả các kiểm tra xác thực trong một hàm riêng biệt truyền các giá trị trạng thái làm đối số. Việc triển khai này đơn giản hóa kiểm thử. Như mong đợi, các hàm xác thực tốn kém sẽ tính thêm gas. Ví dụ:

```go
ValidateMsgA(msg MsgA, now Time, gm GasMeter) error {
	if now.Before(msg.Expire) {
		return sdkerrors.ErrInvalidRequest.Wrap("msg expired")
	}
	gm.ConsumeGas(1000, "signature verification")
	return signatureVerification(msg.Prover, msg.Data)
}
```

:::warning
Trước đây, phương thức `ValidateBasic` được sử dụng để thực hiện các kiểm tra xác thực đơn giản và không trạng thái.
Cách xác thực này đã lỗi thời, nghĩa là `msgServer` phải thực hiện tất cả các kiểm tra xác thực.
:::

### Chuyển Đổi Trạng Thái (State Transition)

Sau khi xác thực thành công, phương thức `msgServer` sử dụng các hàm [`keeper`](./06-keeper.md) để truy cập trạng thái và thực hiện chuyển đổi trạng thái.

### Sự Kiện (Events)

Trước khi trả về, các phương thức `msgServer` thường phát ra một hoặc nhiều [sự kiện](../../learn/advanced/08-events.md) bằng cách sử dụng `EventManager` được giữ trong `ctx`. Sử dụng hàm `EmitTypedEvent` mới sử dụng các kiểu sự kiện dựa trên protobuf:

```go
ctx.EventManager().EmitTypedEvent(
	&group.EventABC{Key1: Value1, Key2: Value2})
```

hoặc hàm `EmitEvent` cũ hơn:

```go
ctx.EventManager().EmitEvent(
	sdk.NewEvent(
		eventType,  // vd: sdk.EventTypeMessage cho message, types.CustomEventType cho sự kiện tùy chỉnh
		sdk.NewAttribute(key1, value1),
		sdk.NewAttribute(key2, value2),
	),
)
```

Các sự kiện này được chuyển tiếp lại cho consensus engine bên dưới và có thể được sử dụng bởi các nhà cung cấp dịch vụ để triển khai các dịch vụ xung quanh ứng dụng. Nhấn vào [đây](../../learn/advanced/08-events.md) để tìm hiểu thêm về sự kiện.

Phương thức `msgServer` được gọi trả về phản hồi `proto.Message` và một `error`. Các giá trị trả về này sau đó được bọc vào `*sdk.Result` hoặc `error` bằng cách sử dụng `sdk.WrapServiceResult(ctx context.Context, res proto.Message, err error)`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/baseapp/msg_service_router.go#L160
```

Phương thức này đảm nhận việc marshal tham số `res` sang protobuf và đính kèm bất kỳ sự kiện nào trên `ctx.EventManager()` vào `sdk.Result`.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/proto/cosmos/base/abci/v1beta1/abci.proto#L93-L113
```

Sơ đồ này cho thấy cấu trúc điển hình của một Protobuf `Msg` service, và cách message lan truyền qua module.

![Luồng giao dịch](https://raw.githubusercontent.com/cosmos/cosmos-sdk/release/v0.46.x/docs/uml/svg/transaction_flow.svg)

## Telemetry

Các [chỉ số telemetry](../../learn/advanced/09-telemetry.md) mới có thể được tạo từ các phương thức `msgServer` khi xử lý các message.

Đây là ví dụ từ module `x/auth/vesting`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/auth/vesting/msg_server.go#L76-L88
```
