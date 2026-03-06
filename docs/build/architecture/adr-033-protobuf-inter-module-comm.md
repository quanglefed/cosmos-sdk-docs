# ADR 033: Giao Tiếp Liên Module qua Protobuf

## Nhật Ký Thay Đổi

* 2020-10-05: Bản nháp đầu tiên

## Trạng Thái

Đề Xuất

## Tóm Tắt

ADR này đề xuất một cơ chế giao tiếp liên module an toàn và có khả năng kiểm tra bằng cách sử dụng các Protobuf `Msg` service như là định nghĩa giao diện, và một `MsgServiceRouter` để định tuyến các message tới handler đúng. Điều này cung cấp sự thay thế an toàn cho việc truy cập keeper trực tiếp giữa các module.

## Bối Cảnh

Hiện tại, Cosmos SDK cho phép giao tiếp liên module thông qua hai cơ chế chính:

1. **Truy cập Keeper trực tiếp**: Module A giữ một tham chiếu tới keeper của Module B và gọi trực tiếp các phương thức của nó. Điều này dẫn đến sự ghép nối chặt chẽ và khó thực thi kiểm soát truy cập đúng đắn.

2. **SDK.Msg qua handler**: Module A tạo ra một `sdk.Msg` và gửi nó qua `MsgServiceRouter` nhưng đây thường được thiết kế cho giao dịch từ người dùng bên ngoài, không phải giao tiếp liên module.

Các vấn đề với cách tiếp cận hiện tại:

* **Không có kiểm soát truy cập rõ ràng**: Các keeper thường lộ nhiều chức năng hơn mức cần thiết cho người dùng của chúng
* **Khó kiểm tra**: Việc mock keeper cho testing đòi hỏi kiến thức về chi tiết triển khai nội bộ
* **Ghép nối chặt chẽ**: Các thay đổi API keeper phá vỡ tất cả module phụ thuộc

## Quyết Định

Chúng ta đề xuất sử dụng `Msg` service Protobuf như là giao diện liên module chuẩn, với `MsgServiceRouter` định tuyến các message tới đúng handler ngay cả khi được gọi từ bên trong module.

### Giao Tiếp Liên Module Sử Dụng `Msg` Services

Module có thể gọi các `Msg` service của module khác bằng cách sử dụng một client được tạo tự động:

```go
// Trong module A, sử dụng `Msg` service của module B
bankMsgClient := bankv1beta1.NewMsgClient(interModuleRouter)
resp, err := bankMsgClient.Send(ctx, &bankv1beta1.MsgSend{
    FromAddress: moduleAAddress,
    ToAddress:   recipientAddress,
    Amount:      coins,
})
```

### Router Liên Module

Một `InterModuleRouter` mới sẽ được tạo ra cho phép các module gửi `Msg` tới nhau:

```go
type InterModuleRouter interface {
    // Invoke gọi handler tương ứng cho Msg được cung cấp
    Invoke(ctx context.Context, req sdk.Msg) (sdk.Msg, error)
}
```

### Kiểm Soát Truy Cập

Khi một module gọi một `Msg` service của module khác, caller được xác định dựa trên tài khoản module của module đó. Handler có thể sau đó kiểm tra xem caller có quyền thực hiện hành động được yêu cầu không:

```go
func (k Keeper) Send(ctx context.Context, msg *MsgSend) (*MsgSendResponse, error) {
    // Lấy caller module address từ context
    callerAddr := sdk.UnwrapSDKContext(ctx).CallerModuleAddress()
    
    // Xác thực caller có quyền gửi từ địa chỉ nguồn
    if !callerAddr.Equals(msg.FromAddress) {
        return nil, sdkerrors.ErrUnauthorized
    }
    
    // Xử lý việc gửi...
}
```

### Tích Hợp với `x/authz`

Cơ chế giao tiếp liên module tích hợp tốt với module `authz`. Một module có thể cấp ủy quyền cho module khác thực hiện hành động thay mặt cho nó:

```go
// Module A cấp quyền cho Module B gửi token thay mặt
authzKeeper.SaveGrant(ctx, moduleB.Address, moduleA.Address, &SendAuthorization{
    SpendLimit: coins,
})
```

## Hậu Quả

### Tích Cực

* Kiểm soát truy cập rõ ràng và có thể kiểm tra
* Ghép nối lỏng lẻo giữa các module thông qua các giao diện được định nghĩa rõ ràng
* Code được tạo tự động giảm boilerplate
* Tương thích với hệ thống ủy quyền hiện có

### Tiêu Cực

* Overhead của việc định tuyến message so với gọi keeper trực tiếp
* Yêu cầu tái cấu trúc các module hiện có để sử dụng giao diện mới

### Trung Lập

* Các module cần lộ `Msg` services như là giao diện công khai của chúng

## Tham Khảo

* [ADR 021: Mã Hóa Truy Vấn Protobuf](./adr-021-protobuf-query-encoding.md)
* [ADR 031: Protobuf Msg Services](./adr-031-msg-service.md)
* [Issue #7093](https://github.com/cosmos/cosmos-sdk/issues/7093)
