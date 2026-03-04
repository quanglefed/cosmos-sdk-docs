---
sidebar_position: 1
---

# Events (Sự kiện)

:::note Tóm tắt
`Event` là các đối tượng chứa thông tin về quá trình thực thi của ứng dụng. Chúng chủ yếu được sử dụng bởi các nhà cung cấp dịch vụ như block explorer và ví để theo dõi quá trình thực thi của các message khác nhau và lập chỉ mục giao dịch.
:::

:::note Tài liệu cần đọc trước

* [Cấu trúc của một ứng dụng Cosmos SDK](../beginner/00-app-anatomy.md)
* [Tài liệu CometBFT về Events](https://docs.cometbft.com/v0.37/spec/abci/abci++_basic_concepts#events)

:::

## Events

Events được triển khai trong Cosmos SDK như một alias của kiểu ABCI `Event` và có dạng: `{eventType}.{attributeKey}={attributeValue}`.

```protobuf reference
https://github.com/cometbft/cometbft/blob/v0.37.0/proto/tendermint/abci/types.proto#L334-L343
```

Một Event chứa:

* Một `type` để phân loại Event ở cấp độ cao; ví dụ, Cosmos SDK dùng kiểu `"message"` để lọc Events theo `Msg`.
* Danh sách `attributes` là các cặp key-value cung cấp thêm thông tin về Event đã phân loại. Ví dụ, với kiểu `"message"`, ta có thể lọc Events theo cặp key-value bằng `message.action={some_action}`, `message.module={some_module}` hoặc `message.sender={some_sender}`.
* `msg_index` để xác định các message nào liên quan đến cùng một giao dịch.

:::tip
Để phân tích các giá trị attribute dưới dạng chuỗi, hãy đảm bảo thêm dấu `'` (nháy đơn) xung quanh mỗi giá trị attribute.
:::

_Typed Events_ là các [message](../../../architecture/adr-032-typed-events.md) được định nghĩa bằng Protobuf, được Cosmos SDK sử dụng để phát ra và query Events. Chúng được định nghĩa trong file `event.proto`, **theo từng module** và được đọc như `proto.Message`.
_Legacy Events_ được định nghĩa **theo từng module** trong file `/types/events.go` của module. Chúng được kích hoạt từ Protobuf [`Msg` service](../../build/building-modules/03-msg-services.md) của module bằng cách dùng [`EventManager`](#eventmanager).

Ngoài ra, mỗi module ghi lại các event của mình trong phần `Events` của spec (x/{moduleName}/`README.md`).

Cuối cùng, Events được trả về cho consensus engine bên dưới trong phản hồi của các ABCI message sau:

* [`BeginBlock`](./00-baseapp.md#beginblock)
* [`EndBlock`](./00-baseapp.md#endblock)
* [`CheckTx`](./00-baseapp.md#checktx)
* [`Transaction Execution`](./00-baseapp.md#transactionexecution)

### Ví dụ

Các ví dụ sau cho thấy cách query Events bằng Cosmos SDK.

| Event                                            | Mô tả                                                                                                                                                |
| ------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `tx.height=23`                                   | Query tất cả giao dịch ở chiều cao 23                                                                                                                |
| `message.action='/cosmos.bank.v1beta1.Msg/Send'` | Query tất cả giao dịch chứa x/bank `Send` [Service `Msg`](../../build/building-modules/03-msg-services.md). Lưu ý dấu `'` xung quanh giá trị.      |
| `message.module='bank'`                          | Query tất cả giao dịch chứa message từ module x/bank. Lưu ý dấu `'` xung quanh giá trị.                                                             |
| `create_validator.validator='cosmosval1...'`     | Event dành riêng cho x/staking, xem [x/staking SPEC](../../../../x/staking/README.md).                                                              |

## EventManager

Trong các ứng dụng Cosmos SDK, Events được quản lý bởi một lớp trừu tượng gọi là `EventManager`. Về mặt nội tại, `EventManager` theo dõi danh sách Events cho toàn bộ luồng thực thi của `FinalizeBlock` (tức là thực thi giao dịch, `BeginBlock`, `EndBlock`).

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/types/events.go#L18-L25
```

`EventManager` đi kèm với một tập hợp các phương thức hữu ích để quản lý Events. Phương thức được dùng nhiều nhất bởi các nhà phát triển module và ứng dụng là `EmitTypedEvent` hoặc `EmitEvent`, theo dõi một Event trong `EventManager`.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/types/events.go#L51-L60
```

Các nhà phát triển module nên xử lý việc phát ra Event thông qua `EventManager#EmitTypedEvent` hoặc `EventManager#EmitEvent` trong mỗi `Handler` của message và trong mỗi handler `BeginBlock`/`EndBlock`. `EventManager` được truy cập thông qua [`Context`](./02-context.md), nơi Event nên đã được đăng ký, và được phát ra như sau:

**Typed events:**

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/x/group/keeper/msg_server.go#L95-L97
```

**Legacy events:**

```go
ctx.EventManager().EmitEvent(
    sdk.NewEvent(eventType, sdk.NewAttribute(attributeKey, attributeValue)),
)
```

Xem tài liệu concept [`Msg` services](../../build/building-modules/03-msg-services.md) để có cái nhìn chi tiết hơn về cách triển khai Events và dùng `EventManager` trong các module.

## Đăng ký Events

Bạn có thể dùng [Websocket](https://docs.cometbft.com/v0.37/core/subscription) của CometBFT để đăng ký Events bằng cách gọi phương thức RPC `subscribe`:

```json
{
  "jsonrpc": "2.0",
  "method": "subscribe",
  "id": "0",
  "params": {
    "query": "tm.event='eventCategory' AND eventType.eventAttribute='attributeValue'"
  }
}
```

Các `eventCategory` chính bạn có thể đăng ký là:

* `NewBlock`: Chứa Events được kích hoạt trong `BeginBlock` và `EndBlock`.
* `Tx`: Chứa Events được kích hoạt trong `DeliverTx` (tức là xử lý giao dịch).
* `ValidatorSetUpdates`: Chứa các bản cập nhật tập hợp validator cho block.

Các Events này được kích hoạt từ package `state` sau khi một block được commit. Bạn có thể lấy danh sách đầy đủ các danh mục Event [trong tài liệu CometBFT Go](https://pkg.go.dev/github.com/cometbft/cometbft/types#pkg-constants).

Giá trị `type` và `attribute` của `query` cho phép bạn lọc Event cụ thể bạn đang tìm kiếm. Ví dụ, giao dịch `Mint` kích hoạt một Event kiểu `EventMint` và có `Id` và `Owner` làm `attributes` (như định nghĩa trong [file `events.proto` của module `NFT`](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/proto/cosmos/nft/v1beta1/event.proto#L21-L31)).

Đăng ký Event này sẽ được thực hiện như sau:

```json
{
  "jsonrpc": "2.0",
  "method": "subscribe",
  "id": "0",
  "params": {
    "query": "tm.event='Tx' AND mint.owner='ownerAddress'"
  }
}
```

trong đó `ownerAddress` là địa chỉ theo định dạng [`AccAddress`](../beginner/03-accounts.md#addresses).

Tương tự có thể áp dụng để đăng ký [legacy events](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/x/bank/types/events.go).

## Default Events (Sự kiện mặc định)

Có một số event được tự động phát ra cho tất cả message, trực tiếp từ `baseapp`.

* `message.action`: Tên của kiểu message.
* `message.sender`: Địa chỉ của bên ký message.
* `message.module`: Tên của module đã phát ra message.

:::tip
Tên module được `baseapp` giả định là phần tử thứ hai của message route: `"cosmos.bank.v1beta1.MsgSend" -> "bank"`. Trong trường hợp module không tuân theo đường dẫn message tiêu chuẩn (ví dụ: IBC), nên tiếp tục phát ra event tên module. `Baseapp` chỉ phát ra event đó nếu module chưa tự làm vậy.
:::
