# ADR 010: AnteHandler Mô-đun Hóa

## Changelog

* 31 tháng 8 năm 2019: Bản nháp đầu tiên
* 14 tháng 9 năm 2021: Được thay thế bởi ADR-045

## Trạng Thái

ĐÃ ĐƯỢC THAY THẾ bởi ADR-045

## Bối Cảnh

Thiết kế AnteHandler hiện tại cho phép người dùng sử dụng AnteHandler mặc định được cung cấp trong `x/auth` hoặc tự xây dựng AnteHandler từ đầu. Lý tưởng nhất, chức năng AnteHandler được chia thành nhiều hàm mô-đun, có thể kết hợp cùng với các hàm ante tùy chỉnh để người dùng không phải viết lại logic antehandler phổ biến khi muốn triển khai hành vi tùy chỉnh.

Ví dụ, giả sử người dùng muốn triển khai một số logic xác minh chữ ký tùy chỉnh. Trong codebase hiện tại, người dùng phải tự viết Antehandler từ đầu, về cơ bản là tái triển khai nhiều code tương tự và sau đó đặt antehandler tùy chỉnh nguyên khối của riêng họ trong baseapp. Thay vào đó, chúng ta muốn cho phép người dùng chỉ định hành vi tùy chỉnh khi cần thiết và kết hợp với chức năng ante-handler mặc định theo cách mô-đun và linh hoạt nhất có thể.

## Đề Xuất

### AnteHandler Theo Module

Một cách tiếp cận là sử dụng [ModuleManager](https://pkg.go.dev/github.com/cosmos/cosmos-sdk/types/module) và để mỗi module triển khai antehandler riêng nếu nó yêu cầu logic antehandler tùy chỉnh. ModuleManager sau đó có thể được truyền vào một thứ tự AnteHandler tương tự như cách nó có thứ tự cho BeginBlocker và EndBlocker. ModuleManager trả về một hàm AnteHandler duy nhất nhận tx và chạy `AnteHandle` của mỗi module theo thứ tự được chỉ định. AnteHandler của module manager được đặt làm AnteHandler của baseapp.

Ưu điểm:

1. Đơn giản để triển khai
2. Sử dụng kiến trúc ModuleManager hiện tại

Nhược điểm:

1. Cải thiện tính chi tiết nhưng vẫn không thể chi tiết hơn cơ sở theo module. Ví dụ, nếu hàm `AnteHandle` của auth chịu trách nhiệm xác thực memo và chữ ký, người dùng không thể hoán đổi chức năng kiểm tra chữ ký trong khi giữ phần còn lại của chức năng `AnteHandle` của auth.
2. Module AnteHandler được chạy lần lượt. Không có cách nào để một AnteHandler bao hoặc "trang trí" AnteHandler khác.

### Mẫu Decorator

Dự án [weave](https://github.com/iov-one/weave) đạt được tính mô-đun của AnteHandler thông qua mẫu decorator. Interface được thiết kế như sau:

```go
// Decorator bao một Handler để cung cấp chức năng phổ biến
// như xác thực hoặc xử lý phí cho nhiều Handler
type Decorator interface {
	Check(ctx Context, store KVStore, tx Tx, next Checker) (*CheckResult, error)
	Deliver(ctx Context, store KVStore, tx Tx, next Deliverer) (*DeliverResult, error)
}
```

Mỗi decorator hoạt động như một hàm antehandler được mô-đun hóa của Cosmos SDK, nhưng nó có thể nhận đối số `next` có thể là decorator khác hoặc Handler (không nhận đối số next). Các decorator này có thể được kết nối chuỗi với nhau, một decorator được truyền vào như đối số `next` của decorator trước đó trong chuỗi. Chuỗi kết thúc ở Router có thể nhận tx và định tuyến đến msg handler phù hợp.

Lợi ích chính của cách tiếp cận này là một Decorator có thể bao logic nội bộ của nó xung quanh Checker/Deliverer tiếp theo. Một Weave Decorator có thể làm như sau:

```go
// Hàm Deliver của Decorator ví dụ
func (example Decorator) Deliver(ctx Context, store KVStore, tx Tx, next Deliverer) {
    // Thực hiện một số logic tiền xử lý

    res, err := next.Deliver(ctx, store, tx)

    // Thực hiện một số logic hậu xử lý dựa trên kết quả và lỗi
}
```

Ưu điểm:

1. Weave Decorator có thể bao quanh decorator/handler tiếp theo trong chuỗi. Khả năng cả tiền xử lý và hậu xử lý có thể hữu ích trong một số cài đặt nhất định.
2. Cung cấp cấu trúc mô-đun lồng nhau không thể thực hiện được trong giải pháp trên, đồng thời cho phép cấu trúc tuyến tính lần lượt như giải pháp trên.

Nhược điểm:

1. Khó hiểu ngay từ cái nhìn đầu tiên về các cập nhật trạng thái sẽ xảy ra sau khi Decorator chạy với `ctx`, `store` và `tx` cho trước. Một Decorator có thể có số lượng tùy ý các Decorator lồng nhau được gọi trong thân hàm của nó, mỗi cái có thể thực hiện tiền xử lý và hậu xử lý trước khi gọi decorator tiếp theo trong chuỗi. Do đó để hiểu Decorator đang làm gì, người ta cũng phải hiểu mọi decorator khác trong chuỗi cũng đang làm gì. Điều này có thể khá phức tạp để lý luận. Cách tiếp cận tuyến tính lần lượt trong khi kém mạnh mẽ hơn, có thể dễ lý luận hơn nhiều.

### Micro-Function Được Kết Nối Chuỗi

Lợi ích của cách tiếp cận Weave là các Decorator có thể rất ngắn gọn, khi được kết nối chuỗi cho phép tùy biến tối đa. Tuy nhiên, cấu trúc lồng nhau có thể khá phức tạp và do đó khó lý luận.

Một cách tiếp cận khác là chia chức năng AnteHandler thành các "micro-function" có phạm vi chặt chẽ, trong khi vẫn bảo toàn thứ tự lần lượt từ cách tiếp cận ModuleManager.

Sau đó chúng ta có thể có cách để kết nối chuỗi các micro-function này để chúng chạy lần lượt. Các module có thể định nghĩa nhiều ante micro-function và cũng cung cấp một AnteHandler mặc định theo module triển khai thứ tự mặc định, được đề xuất cho các micro-function này.

Người dùng có thể sắp xếp AnteHandler dễ dàng bằng cách đơn giản là sử dụng ModuleManager. ModuleManager sẽ nhận danh sách AnteHandler và trả về một AnteHandler duy nhất chạy mỗi AnteHandler theo thứ tự danh sách được cung cấp. Nếu người dùng hài lòng với thứ tự mặc định của mỗi module, điều này đơn giản như cung cấp danh sách với antehandler của mỗi module (hoàn toàn giống như BeginBlocker và EndBlocker).

Tuy nhiên, nếu người dùng muốn thay đổi thứ tự hoặc thêm, sửa đổi hoặc xóa ante micro-function theo bất kỳ cách nào, họ luôn có thể định nghĩa micro-function ante riêng và thêm chúng rõ ràng vào danh sách được truyền vào module manager.

### Simple Decorator

Cách tiếp cận này lấy cảm hứng từ thiết kế decorator của Weave trong khi cố gắng giảm thiểu số lượng thay đổi breaking đối với Cosmos SDK và tối đa hóa sự đơn giản. Giống như weave decorator, cách tiếp cận này cho phép một `AnteDecorator` bao AnteHandler tiếp theo để thực hiện tiền xử lý và hậu xử lý trên kết quả. Điều này hữu ích vì các decorator có thể thực hiện defer/cleanup sau khi AnteHandler trả về cũng như thực hiện một số thiết lập trước. Khác với Weave decorator, các hàm `AnteDecorator` này chỉ có thể bao quanh AnteHandler thay vì toàn bộ đường thực thi handler. Điều này là cố ý vì chúng ta muốn các decorator từ các module khác nhau thực hiện xác thực trên `tx`. Tuy nhiên, chúng ta không muốn các decorator có khả năng bao và sửa đổi kết quả của `MsgHandler`.

Ngoài ra, cách tiếp cận này sẽ không phá vỡ bất kỳ API core nào của Cosmos SDK. Vì chúng ta vẫn bảo toàn khái niệm AnteHandler và vẫn đặt một AnteHandler duy nhất trong baseapp, decorator chỉ là một cách tiếp cận bổ sung có sẵn cho người dùng muốn tùy biến nhiều hơn.

```go
// An AnteDecorator bao một AnteHandler và có thể thực hiện tiền xử lý và hậu xử lý trên AnteHandler tiếp theo
type AnteDecorator interface {
    AnteHandle(ctx Context, tx Tx, simulate bool, next AnteHandler) (newCtx Context, err error)
}
```

```go
// ChainAnteDecorators sẽ liên kết đệ quy tất cả AnteDecorator trong chuỗi và trả về hàm AnteHandler cuối cùng
// Điều này được thực hiện để bảo toàn khả năng đặt một hàm AnteHandler duy nhất trong baseapp.
func ChainAnteDecorators(chain ...AnteDecorator) AnteHandler {
    if len(chain) == 1 {
        return func(ctx Context, tx Tx, simulate bool) {
            chain[0].AnteHandle(ctx, tx, simulate, nil)
        }
    }
    return func(ctx Context, tx Tx, simulate bool) {
        chain[0].AnteHandle(ctx, tx, simulate, ChainAnteDecorators(chain[1:]))
    }
}
```

#### Code Ví Dụ

Định nghĩa các hàm AnteDecorator

```go
// Thiết lập GasMeter, bắt OutOfGasPanic và xử lý phù hợp
type SetUpContextDecorator struct{}

func (sud SetUpContextDecorator) AnteHandle(ctx Context, tx Tx, simulate bool, next AnteHandler) (newCtx Context, err error) {
    ctx.GasMeter = NewGasMeter(tx.Gas)

    defer func() {
        // phục hồi từ panic OutOfGas và xử lý phù hợp
    }

    return next(ctx, tx, simulate)
}

// Decorator Xác Minh Chữ Ký. Xác minh Chữ Ký và tiếp tục
type SigVerifyDecorator struct{}

func (svd SigVerifyDecorator) AnteHandle(ctx Context, tx Tx, simulate bool, next AnteHandler) (newCtx Context, err error) {
    // xác minh sigs. Trả về lỗi nếu không hợp lệ

    // gọi antehandler tiếp theo nếu sigs ổn
    return next(ctx, tx, simulate)
}

// Decorator Do Người Dùng Định Nghĩa. Có thể chọn tiền xử lý và hậu xử lý trên AnteHandler
type UserDefinedDecorator struct{
    // các trường tùy chỉnh
}

func (udd UserDefinedDecorator) AnteHandle(ctx Context, tx Tx, simulate bool, next AnteHandler) (newCtx Context, err error) {
    // logic tiền xử lý

    ctx, err = next(ctx, tx, simulate)

    // logic hậu xử lý
}
```

Liên kết AnteDecorator để tạo AnteHandler cuối cùng. Đặt AnteHandler này trong baseapp.

```go
// Tạo antehandler cuối cùng bằng cách kết nối chuỗi các decorator
antehandler := ChainAnteDecorators(NewSetUpContextDecorator(), NewSigVerifyDecorator(), NewUserDefinedDecorator())

// Đặt Antehandler đã kết nối chuỗi trong baseapp
bapp.SetAnteHandler(antehandler)
```

Ưu điểm:

1. Cho phép một decorator tiền xử lý và hậu xử lý AnteHandler tiếp theo, tương tự như thiết kế Weave.
2. Không cần phá vỡ API baseapp. Người dùng vẫn có thể đặt một AnteHandler duy nhất nếu họ chọn.

Nhược điểm:

1. Mẫu Decorator có thể có cấu trúc lồng nhau sâu khó hiểu, điều này được giảm thiểu bằng cách liệt kê rõ ràng thứ tự decorator trong hàm `ChainAnteDecorators`.
2. Không sử dụng thiết kế ModuleManager. Vì điều này đã được sử dụng cho BeginBlocker/EndBlocker, đề xuất này có vẻ không phù hợp với mẫu thiết kế đó.

## Hậu Quả

Vì ưu và nhược điểm được viết cho mỗi cách tiếp cận, chúng được bỏ qua trong phần này.

## Tài Liệu Tham Khảo

* [#4572](https://github.com/cosmos/cosmos-sdk/issues/4572): Vấn đề AnteHandler Mô-đun Hóa
* [#4582](https://github.com/cosmos/cosmos-sdk/pull/4583): Triển khai đầu tiên của cách tiếp cận AnteHandler Theo Module
* [Code Weave Decorator](https://github.com/iov-one/weave/blob/master/handler.go#L35)
* [Video Thiết Kế Weave](https://vimeo.com/showcase/6189877)
