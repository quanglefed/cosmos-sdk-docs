# ADR 022: Xử Lý Panic Tùy Chỉnh trong BaseApp

## Nhật Ký Thay Đổi

* 24 tháng 4 năm 2020: Bản nháp đầu tiên
* 14 tháng 9 năm 2021: Được thay thế bởi ADR-045

## Trạng Thái

BỊ THAY THẾ bởi ADR-045

## Bối Cảnh

Triển khai hiện tại của BaseApp không cho phép các nhà phát triển viết các handler lỗi tùy chỉnh trong quá trình khôi phục panic tại phương thức [runTx()](https://github.com/cosmos/cosmos-sdk/blob/bad4ca75f58b182f600396ca350ad844c18fc80b/baseapp/baseapp.go#L539). Chúng tôi cho rằng phương thức này có thể linh hoạt hơn và có thể cung cấp cho người dùng Cosmos SDK nhiều tùy chọn tùy chỉnh hơn mà không cần phải viết lại toàn bộ BaseApp. Ngoài ra, có một trường hợp đặc biệt cho việc xử lý lỗi `sdk.ErrorOutOfGas`, trường hợp đó có thể được xử lý theo cách "tiêu chuẩn" (middleware) cùng với các trường hợp khác.

Chúng tôi đề xuất giải pháp middleware có thể giúp các nhà phát triển triển khai các trường hợp sau:

* Thêm ghi nhật ký bên ngoài (chẳng hạn gửi báo cáo đến các dịch vụ bên ngoài như [Sentry](https://sentry.io));
* Gọi panic cho các trường hợp lỗi cụ thể;

Nó cũng sẽ làm cho trường hợp `OutOfGas` và trường hợp `default` trở thành một trong các middleware. Trường hợp `Default` bọc đối tượng khôi phục thành một lỗi và ghi nhật ký ([ví dụ triển khai middleware](#recovery-middleware)).

Dự án của chúng tôi có một dịch vụ sidecar chạy cùng với node blockchain (máy ảo hợp đồng thông minh). Điều thiết yếu là kết nối node <-> sidecar phải ổn định cho việc xử lý TX. Vì vậy khi giao tiếp bị gián đoạn, chúng tôi cần crash node và khởi động lại nó khi vấn đề được giải quyết. Hành vi đó làm cho quá trình thực thi máy trạng thái của node có tính xác định. Vì tất cả các panic của keeper đều bị bắt bởi handler `defer()` của runTx, chúng tôi phải điều chỉnh mã BaseApp để tùy chỉnh nó.

## Quyết Định

### Thiết Kế

#### Tổng Quan

Thay vì mã hóa cứng xử lý lỗi tùy chỉnh vào BaseApp, chúng tôi đề xuất sử dụng một tập hợp các middleware có thể được tùy chỉnh từ bên ngoài và cho phép các nhà phát triển sử dụng nhiều handler lỗi tùy chỉnh tùy ý. Triển khai với các test có thể được tìm thấy [tại đây](https://github.com/cosmos/cosmos-sdk/pull/6053).

#### Chi Tiết Triển Khai

##### Recovery Handler

Kiểu `RecoveryHandler` mới được thêm vào. Đối số đầu vào `recoveryObj` là một đối tượng được trả về bởi hàm Go tiêu chuẩn `recover()` từ gói `builtin`.

```go
type RecoveryHandler func(recoveryObj interface{}) error
```

Handler nên xác nhận kiểu (hoặc các phương thức khác) một đối tượng để xác định xem đối tượng đó có nên được xử lý hay không. `nil` nên được trả về nếu đối tượng đầu vào không thể được xử lý bởi `RecoveryHandler` đó (không phải kiểu mục tiêu của handler). Lỗi không phải `nil` nên được trả về nếu đối tượng đầu vào đã được xử lý và việc thực thi chuỗi middleware nên dừng lại.

Một ví dụ:

```go
func exampleErrHandler(recoveryObj interface{}) error {
    err, ok := recoveryObj.(error)
    if !ok { return nil }

    if someSpecificError.Is(err) {
        panic(customPanicMsg)
    } else {
        return nil
    }
}
```

Ví dụ này ngắt quá trình thực thi ứng dụng, nhưng nó cũng có thể bổ sung ngữ cảnh lỗi như handler `OutOfGas`.

##### Recovery Middleware

Chúng tôi cũng thêm kiểu middleware (decorator). Kiểu hàm này bọc `RecoveryHandler` và trả về middleware tiếp theo trong chuỗi thực thi cùng với `error` của handler. Kiểu được sử dụng để tách việc xử lý đối tượng `recovery()` thực tế khỏi xử lý chuỗi middleware.

```go
type recoveryMiddleware func(recoveryObj interface{}) (recoveryMiddleware, error)

func newRecoveryMiddleware(handler RecoveryHandler, next recoveryMiddleware) recoveryMiddleware {
    return func(recoveryObj interface{}) (recoveryMiddleware, error) {
        if err := handler(recoveryObj); err != nil {
            return nil, err
        }
        return next, nil
    }
}
```

Hàm nhận đối tượng `recoveryObj` và trả về:

* (middleware `recoveryMiddleware` tiếp theo, `nil`) nếu đối tượng không được xử lý (không phải kiểu mục tiêu) bởi `RecoveryHandler`;
* (`nil`, `error` không phải nil) nếu đối tượng đầu vào đã được xử lý và các middleware khác trong chuỗi không nên được thực thi;
* (`nil`, `nil`) trong trường hợp hành vi không hợp lệ. Có thể khôi phục panic không được xử lý đúng cách; điều này có thể tránh bằng cách luôn sử dụng `default` làm middleware ngoài cùng bên phải trong chuỗi (luôn trả về `error`);

Ví dụ middleware `OutOfGas`:

```go
func newOutOfGasRecoveryMiddleware(gasWanted uint64, ctx sdk.Context, next recoveryMiddleware) recoveryMiddleware {
    handler := func(recoveryObj interface{}) error {
        err, ok := recoveryObj.(sdk.ErrorOutOfGas)
        if !ok { return nil }

        return errorsmod.Wrap(
            sdkerrors.ErrOutOfGas, fmt.Sprintf(
                "out of gas in location: %v; gasWanted: %d, gasUsed: %d", err.Descriptor, gasWanted, ctx.GasMeter().GasConsumed(),
            ),
        )
    }

    return newRecoveryMiddleware(handler, next)
}
```

Ví dụ middleware `Default`:

```go
func newDefaultRecoveryMiddleware() recoveryMiddleware {
    handler := func(recoveryObj interface{}) error {
        return errorsmod.Wrap(
            sdkerrors.ErrPanic, fmt.Sprintf("recovered: %v\nstack:\n%v", recoveryObj, string(debug.Stack())),
        )
    }

    return newRecoveryMiddleware(handler, nil)
}
```

##### Xử Lý Recovery

Xử lý chuỗi middleware cơ bản sẽ trông như thế này:

```go
func processRecovery(recoveryObj interface{}, middleware recoveryMiddleware) error {
	if middleware == nil { return nil }

	next, err := middleware(recoveryObj)
	if err != nil { return err }
	if next == nil { return nil }

	return processRecovery(recoveryObj, next)
}
```

Theo cách đó chúng ta có thể tạo một chuỗi middleware được thực thi từ trái sang phải, middleware ngoài cùng bên phải là handler `default` phải trả về một `error`.

##### Thay Đổi BaseApp

Chuỗi middleware `default` phải tồn tại trong đối tượng `BaseApp`. Các sửa đổi với `Baseapp`:

```go
type BaseApp struct {
    // ...
    runTxRecoveryMiddleware recoveryMiddleware
}

func NewBaseApp(...) {
    // ...
    app.runTxRecoveryMiddleware = newDefaultRecoveryMiddleware()
}

func (app *BaseApp) runTx(...) {
    // ...
    defer func() {
        if r := recover(); r != nil {
            recoveryMW := newOutOfGasRecoveryMiddleware(gasWanted, ctx, app.runTxRecoveryMiddleware)
            err, result = processRecovery(r, recoveryMW), nil
        }

        gInfo = sdk.GasInfo{GasWanted: gasWanted, GasUsed: ctx.GasMeter().GasConsumed()}
    }()
    // ...
}
```

Các nhà phát triển có thể thêm `RecoveryHandler` tùy chỉnh của họ bằng cách cung cấp `AddRunTxRecoveryHandler` như một tham số tùy chọn BaseApp cho constructor `NewBaseapp`:

```go
func (app *BaseApp) AddRunTxRecoveryHandler(handlers ...RecoveryHandler) {
    for _, h := range handlers {
        app.runTxRecoveryMiddleware = newRecoveryMiddleware(h, app.runTxRecoveryMiddleware)
    }
}
```

Phương thức này sẽ thêm các handler vào đầu chuỗi hiện có.

## Hậu Quả

### Tích Cực

* Các nhà phát triển dự án dựa trên Cosmos SDK có thể thêm các handler panic tùy chỉnh để:
    * thêm ngữ cảnh lỗi cho các nguồn panic tùy chỉnh (panic bên trong các keeper tùy chỉnh);
    * phát ra `panic()`: truyền đối tượng khôi phục tới lõi Tendermint;
    * các xử lý cần thiết khác;
* Các nhà phát triển có thể sử dụng triển khai `BaseApp` tiêu chuẩn của Cosmos SDK, thay vì phải viết lại trong các dự án của họ;
* Giải pháp đề xuất không phá vỡ luồng `runTx()` "tiêu chuẩn" hiện tại;

### Tiêu Cực

* Giới thiệu các thay đổi vào thiết kế mô hình thực thi.

### Trung Lập

* Handler lỗi `OutOfGas` trở thành một trong các middleware;
* Handler panic mặc định trở thành một trong các middleware;

## Tham Khảo

* [PR-6053 với giải pháp đề xuất](https://github.com/cosmos/cosmos-sdk/pull/6053)
* [Giải pháp tương tự. ADR-010 Modular AnteHandler](https://github.com/cosmos/cosmos-sdk/blob/main/docs/architecture/adr-010-modular-antehandler.md)
