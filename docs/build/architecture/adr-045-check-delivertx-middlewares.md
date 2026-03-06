# ADR 045: BaseApp `{Check,Deliver}Tx` như Middleware

## Changelog

* 20.08.2021: Bản nháp đầu tiên.
* 07.12.2021: Cập nhật interface `tx.Handler`.
* 17.05.2022: ADR bị từ bỏ, vì middleware bị coi là quá khó để lý luận về.

## Trạng Thái

ĐÃ BỊ TỪ BỎ. Thay thế đang được thảo luận trong [#11955](https://github.com/cosmos/cosmos-sdk/issues/11955).

## Tóm Tắt

ADR này thay thế thiết kế `runTx` và antehandlers hiện tại của BaseApp bằng thiết kế dựa trên middleware.

## Bối Cảnh

Triển khai ABCI `{Check,Deliver}Tx()` của BaseApp và phương thức `Simulate()` của chính nó gọi phương thức `runTx` bên dưới, đầu tiên chạy các antehandler, sau đó thực thi các `Msg`. Tuy nhiên, các use case [transaction Tips](https://github.com/cosmos/cosmos-sdk/issues/9406) và [hoàn lại gas chưa sử dụng](https://github.com/cosmos/cosmos-sdk/issues/2150) yêu cầu logic tùy chỉnh chạy sau khi thực thi `Msg`. Hiện tại không có cách nào để thực hiện điều này.

Giải pháp đơn giản sẽ là thêm các post-`Msg` hook vào BaseApp. Tuy nhiên, nhóm Cosmos SDK đang nghĩ song song về bức tranh lớn hơn là làm cho app wiring đơn giản hơn ([#9181](https://github.com/cosmos/cosmos-sdk/discussions/9182)), bao gồm việc làm cho BaseApp nhẹ hơn và mô-đun hóa hơn.

## Quyết Định

Chúng ta quyết định chuyển đổi triển khai ABCI `{Check,Deliver}Tx` của Baseapp và các phương thức `Simulate` của nó để sử dụng thiết kế dựa trên middleware.

Hai interface sau là cơ sở của thiết kế middleware, và được định nghĩa trong `types/tx`:

```go
type Handler interface {
    CheckTx(ctx context.Context, req Request, checkReq RequestCheckTx) (Response, ResponseCheckTx, error)
    DeliverTx(ctx context.Context, req Request) (Response, error)
    SimulateTx(ctx context.Context, req Request (Response, error)
}

type Middleware func(Handler) Handler
```

BaseApp giữ tham chiếu đến `tx.Handler`:

```go
type BaseApp  struct {
    // các trường khác
    txHandler tx.Handler
}
```

Các phương thức ABCI `{Check,Deliver}Tx()` và `Simulate()` của Baseapp chỉ đơn giản gọi `app.txHandler.{Check,Deliver,Simulate}Tx()` với các đối số liên quan.

### Triển Khai Middleware

Trong thực tế, các middleware được tạo bởi hàm Go nhận các tham số cần thiết cho middleware và trả về một `tx.Middleware`:

```go
type myTxHandler struct {
    next tx.Handler
}

func NewMyMiddleware(arg1, arg2) tx.Middleware {
    return func (txh tx.Handler) tx.Handler {
        return myTxHandler{next: txh}
    }
}

func (h myTxHandler) CheckTx(ctx context.Context, req Request, checkReq RequestcheckTx) (Response, ResponseCheckTx, error) {
    // logic tiền xử lý CheckTx
    res, checkRes, err := txh.next.CheckTx(ctx, req, checkReq)
    // logic hậu xử lý CheckTx
    return res, checkRes, err
}
```

### Kết Hợp Middleware

Chúng ta định nghĩa hàm `ComposeMiddlewares` để kết hợp các middleware. Nó nhận handler cơ sở làm đối số đầu tiên, và các middleware theo thứ tự "ngoài vào trong":

```go
txHandler := middleware.ComposeMiddlewares(H, A, B, C)
```

### Middleware do Cosmos SDK Duy Trì

| Middleware              | Mô Tả |
| ----------------------- | ------ |
| RunMsgsTxHandler        | Handler `tx.Handler` cơ sở. Thay thế `runMsgs` của baseapp cũ, thực thi các `Msg` của giao dịch. |
| TxDecoderMiddleware     | Nhận raw bytes của giao dịch và decode chúng thành `sdk.Tx`. |
| {Antehandlers}          | Mỗi antehandler được chuyển đổi thành middleware riêng. |
| IndexEventsTxMiddleware | Chọn sự kiện nào để lập chỉ mục trong Tendermint. |
| RecoveryTxMiddleware    | Phục hồi từ panic. |
| GasTxMiddleware         | Đặt GasMeter trên sdk.Context. |

### Điểm Tương Đồng và Khác Biệt giữa Antehandlers và Middleware

#### Tương Đồng với Antehandlers

* Được thiết kế để kết nối/kết hợp các phần nhỏ có tính mô-đun.
* Cho phép tái sử dụng code cho `{Check,Deliver}Tx` và `Simulate`.
* Được thiết lập trong `app.go` và dễ tùy chỉnh bởi nhà phát triển ứng dụng.

#### Khác Biệt với Antehandlers

* Các middleware có thể chạy trước và sau khi thực thi `Msg`.
* Middleware sử dụng `context.Context` chuẩn của Go, trong khi antehandlers sử dụng `sdk.Context`.

## Hậu Quả

### Tương Thích Ngược

Thay vì tạo chuỗi antehandler trong `app.go`, nhà phát triển ứng dụng cần tạo một middleware stack — thay đổi API.

### Tích Cực

* Cho phép logic tùy chỉnh chạy trước và sau khi thực thi `Msg`.
* Làm cho BaseApp nhẹ hơn.
* Các path riêng biệt cho `{Check,Deliver,Simulate}Tx`.

### Tiêu Cực

* Khó hiểu hơn khi nhìn lướt qua về những cập nhật trạng thái sẽ xảy ra.
* Thay đổi API cho nhà phát triển ứng dụng.

## Tài Liệu Tham Khảo

* Thảo luận ban đầu: https://github.com/cosmos/cosmos-sdk/issues/9585
