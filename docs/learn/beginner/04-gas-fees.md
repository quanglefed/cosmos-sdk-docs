---
sidebar_position: 1
---

# Gas và Phí

:::note Tóm tắt
Tài liệu này mô tả các chiến lược mặc định để xử lý gas và phí trong một ứng dụng Cosmos SDK.
:::

:::note Tài liệu cần đọc trước

* [Cấu trúc của một ứng dụng Cosmos SDK](./00-app-anatomy.md)

:::

## Giới thiệu về `Gas` và `Fees`

Trong Cosmos SDK, `gas` là một đơn vị đặc biệt được dùng để theo dõi mức tiêu thụ tài nguyên trong quá trình thực thi. `gas` thường được tiêu thụ bất cứ khi nào có thao tác đọc và ghi vào store, nhưng cũng có thể được tiêu thụ nếu cần thực hiện các phép tính tốn kém. Nó phục vụ hai mục đích chính:

* Đảm bảo các block không tiêu thụ quá nhiều tài nguyên và được hoàn thiện. Điều này được triển khai mặc định trong Cosmos SDK qua [block gas meter](#block-gas-meter).
* Ngăn chặn spam và lạm dụng từ người dùng cuối. Để đạt mục tiêu này, `gas` được tiêu thụ trong quá trình thực thi [`message`](../../build/building-modules/02-messages-and-queries.md#messages) thường được định giá, dẫn đến một `fee` (`fees = gas * gas-prices`). `fees` thường phải được trả bởi người gửi `message`. Lưu ý rằng Cosmos SDK không thực thi định giá `gas` theo mặc định, vì có thể có các cách khác để ngăn spam (ví dụ: sơ đồ băng thông). Tuy nhiên, hầu hết ứng dụng triển khai cơ chế `fee` để ngăn spam bằng cách sử dụng [`AnteHandler`](#antehandler).

## Gas Meter

Trong Cosmos SDK, `gas` là một alias đơn giản cho `uint64` và được quản lý bởi một đối tượng gọi là _gas meter_. Gas meter triển khai interface `GasMeter`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/store/types/gas.go#L40-L51
```

trong đó:

* `GasConsumed()` trả về lượng gas đã được tiêu thụ bởi instance gas meter.
* `GasConsumedToLimit()` trả về lượng gas đã được tiêu thụ bởi instance gas meter, hoặc giới hạn nếu giới hạn đó đã đạt.
* `GasRemaining()` trả về lượng gas còn lại trong GasMeter.
* `Limit()` trả về giới hạn của instance gas meter. `0` nếu gas meter là vô hạn.
* `ConsumeGas(amount Gas, descriptor string)` tiêu thụ lượng `gas` được cung cấp. Nếu `gas` tràn, nó panic với thông báo `descriptor`. Nếu gas meter không phải vô hạn, nó panic nếu `gas` tiêu thụ vượt quá giới hạn.
* `RefundGas()` khấu trừ lượng đã cho từ gas đã tiêu thụ. Chức năng này cho phép hoàn tiền gas vào pool gas giao dịch hoặc block để các chuỗi tương thích EVM có thể hỗ trợ đầy đủ interface go-ethereum StateDB.
* `IsPastLimit()` trả về `true` nếu lượng gas đã tiêu thụ bởi instance gas meter nghiêm ngặt cao hơn giới hạn, ngược lại `false`.
* `IsOutOfGas()` trả về `true` nếu lượng gas đã tiêu thụ bởi instance gas meter lớn hơn hoặc bằng giới hạn, ngược lại `false`.

Gas meter thường được giữ trong [`ctx`](../advanced/02-context.md), và việc tiêu thụ gas được thực hiện theo mẫu sau:

```go
ctx.GasMeter().ConsumeGas(amount, "description")
```

Theo mặc định, Cosmos SDK sử dụng hai gas meter khác nhau, [main gas meter](#main-gas-meter) và [block gas meter](#block-gas-meter).

### Main Gas Meter

`ctx.GasMeter()` là main gas meter của ứng dụng. Main gas meter được khởi tạo trong `FinalizeBlock` qua `setFinalizeBlockState`, sau đó theo dõi mức tiêu thụ gas trong các chuỗi thực thi dẫn đến chuyển đổi trạng thái, tức là những chuỗi được kích hoạt ban đầu bởi [`FinalizeBlock`](../advanced/00-baseapp.md#finalizeblock). Ở đầu mỗi lần thực thi giao dịch, main gas meter **phải được đặt về 0** trong [`AnteHandler`](#antehandler), để nó có thể theo dõi mức tiêu thụ gas theo từng giao dịch.

Việc tiêu thụ gas có thể được thực hiện thủ công, thường bởi nhà phát triển module trong [`BeginBlocker`, `EndBlocker`](../../build/building-modules/06-beginblock-endblock.md) hoặc [`Msg` service](../../build/building-modules/03-msg-services.md), nhưng hầu hết thời gian nó được thực hiện tự động bất cứ khi nào có thao tác đọc hoặc ghi vào store. Logic tiêu thụ gas tự động này được triển khai trong một store đặc biệt gọi là [`GasKv`](../advanced/04-store.md#gaskv-store).

### Block Gas Meter

`ctx.BlockGasMeter()` là gas meter được dùng để theo dõi mức tiêu thụ gas trên mỗi block và đảm bảo nó không vượt quá một giới hạn nhất định.

Trong giai đoạn genesis, mức tiêu thụ gas là vô hạn để phục vụ các giao dịch khởi tạo.

```go
app.finalizeBlockState.SetContext(app.finalizeBlockState.Context().WithBlockGasMeter(storetypes.NewInfiniteGasMeter()))
```

Sau block genesis, block gas meter được đặt thành một giá trị hữu hạn bởi SDK. Sự chuyển đổi này được tạo điều kiện bởi consensus engine (ví dụ: CometBFT) gọi hàm `RequestFinalizeBlock`, lần lượt kích hoạt phương thức `FinalizeBlock` của SDK. Trong `FinalizeBlock`, `internalFinalizeBlock` được thực thi, thực hiện các cập nhật trạng thái và thực thi hàm cần thiết. Block gas meter, được khởi tạo mỗi lần với giới hạn hữu hạn, sau đó được tích hợp vào context để thực thi giao dịch, đảm bảo mức tiêu thụ gas không vượt quá giới hạn gas của block và được reset ở cuối mỗi block.

Các module trong Cosmos SDK có thể tiêu thụ block gas tại bất kỳ thời điểm nào trong quá trình thực thi bằng cách sử dụng `ctx`. Việc tiêu thụ gas này chủ yếu xảy ra trong các thao tác đọc/ghi trạng thái và xử lý giao dịch. Block gas meter, có thể truy cập qua `ctx.BlockGasMeter()`, giám sát tổng mức sử dụng gas trong một block, thực thi giới hạn gas để ngăn tính toán quá mức.

```go
gasMeter := app.getBlockGasMeter(app.finalizeBlockState.Context())
app.finalizeBlockState.SetContext(app.finalizeBlockState.Context().WithBlockGasMeter(gasMeter))
```

Phần trên cho thấy cơ chế chung để đặt block gas meter với giới hạn hữu hạn dựa trên các tham số đồng thuận của block.

## AnteHandler

`AnteHandler` được chạy cho mỗi giao dịch trong `CheckTx` và `FinalizeBlock`, trước khi phương thức Protobuf `Msg` service cho mỗi `sdk.Msg` trong giao dịch được thực thi.

AnteHandler không được triển khai trong core Cosmos SDK mà trong một module. Tuy vậy, hầu hết ứng dụng ngày nay sử dụng triển khai mặc định được định nghĩa trong [module `auth`](https://github.com/cosmos/cosmos-sdk/tree/main/x/auth). Đây là những gì `anteHandler` dự định thực hiện trong một ứng dụng Cosmos SDK thông thường:

* Xác minh rằng các giao dịch có đúng loại. Các loại giao dịch được định nghĩa trong module triển khai `anteHandler`, và chúng tuân theo interface giao dịch:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/types/tx_msg.go#L53-L58
```

  Điều này cho phép các nhà phát triển làm việc với nhiều loại giao dịch khác nhau cho ứng dụng của họ. Trong module `auth` mặc định, loại giao dịch mặc định là `Tx`:

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/proto/cosmos/tx/v1beta1/tx.proto#L15-L28
```

* Xác minh chữ ký cho mỗi [`message`](../../build/building-modules/02-messages-and-queries.md#messages) chứa trong giao dịch. Mỗi `message` nên được ký bởi một hoặc nhiều người gửi, và các chữ ký này phải được xác minh trong `anteHandler`.
* Trong `CheckTx`, xác minh rằng gas price được cung cấp cùng giao dịch lớn hơn `min-gas-prices` cục bộ (nhắc lại, gas-prices có thể được suy ra từ phương trình: `fees = gas * gas-prices`). `min-gas-prices` là tham số cục bộ cho mỗi full-node và được dùng trong `CheckTx` để loại bỏ các giao dịch không cung cấp đủ phí tối thiểu. Điều này đảm bảo mempool không thể bị spam bởi các giao dịch rác.
* Xác minh rằng người gửi giao dịch có đủ tiền để trang trải `fees`. Khi người dùng cuối tạo giao dịch, họ phải chỉ định 2 trong 3 tham số sau (tham số thứ ba là ngầm định): `fees`, `gas` và `gas-prices`. Điều này cho biết họ sẵn lòng trả bao nhiêu để các node thực thi giao dịch của họ. Giá trị `gas` được cung cấp được lưu trong tham số `GasWanted` để dùng sau.
* Đặt `newCtx.GasMeter` về 0, với giới hạn là `GasWanted`. **Bước này rất quan trọng**, vì nó không chỉ đảm bảo giao dịch không thể tiêu thụ gas vô hạn, mà còn đảm bảo `ctx.GasMeter` được reset giữa mỗi giao dịch (`ctx` được đặt thành `newCtx` sau khi `anteHandler` chạy, và `anteHandler` chạy mỗi lần một giao dịch thực thi).

Như đã giải thích ở trên, `anteHandler` trả về giới hạn tối đa của `gas` mà giao dịch có thể tiêu thụ trong quá trình thực thi, được gọi là `GasWanted`. Lượng thực tế tiêu thụ cuối cùng được gọi là `GasUsed`, và do đó chúng ta phải có `GasUsed =< GasWanted`. Cả `GasWanted` và `GasUsed` đều được chuyển tiếp đến consensus engine bên dưới khi [`FinalizeBlock`](../advanced/00-baseapp.md#finalizeblock) trả về.
