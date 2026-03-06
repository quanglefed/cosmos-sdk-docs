---
sidebar_position: 14
---

# Tips (Tiền Bo Thêm)

:::note Tóm tắt
Cơ chế Tips cho phép người dùng trả phí giao dịch bằng token có đơn vị khác với token phí gốc của chuỗi, hữu ích cho các kịch bản cross-chain qua IBC.
:::

:::note Tài liệu cần đọc trước

* [Giao dịch (Transactions)](./01-transactions.md)
* [Vòng đời của một giao dịch](../beginner/01-tx-lifecycle.md)

:::

## Giới thiệu

Transaction Tips (tiền bo thêm giao dịch) là cơ chế trong Cosmos SDK cho phép người dùng trả phí giao dịch bằng đơn vị token khác với token phí gốc (native fee token) của chuỗi. Cơ chế này đặc biệt hữu ích cho các kịch bản cross-chain, khi người dùng muốn thực hiện hành động trên một chuỗi nhưng chỉ sở hữu token từ chuỗi khác (thường nhận được qua IBC).

## Các bên tham gia

**Tipper (người bo thêm)**: Người khởi tạo giao dịch muốn thực thi message trên chuỗi nhưng không có token gốc của chuỗi đó. Họ chỉ có token từ chuỗi khác (thường nhận qua IBC).

**Fee Payer (người trả phí)**: Bên relay và phát sóng giao dịch cuối cùng lên chuỗi đích, và sở hữu token gốc cần thiết để trả phí. Tipper không cần tin tưởng fee payer.

## Cách hoạt động

Quy trình Tips giao dịch diễn ra qua các bước sau:

1. Tipper gửi token từ chuỗi khác (qua IBC) để trang trải phí trên chuỗi đích.
2. Thay vì tạo giao dịch thông thường, tipper tạo tài liệu `AuxSignerData` chứa địa chỉ của họ, sign document, chế độ ký và chữ ký.
3. Tipper gửi giao dịch đã ký này đến fee relayer (fee payer), người chọn phí giao dịch và phát sóng.
4. SDK cung cấp cơ chế chuyển tip đã định nghĩa trước từ tipper sang fee payer để trang trải phí thực tế.

## Định nghĩa kiểu

### Tip

Trường `Tip` trong `AuthInfo` mô tả tiền bo thêm tùy chọn dùng cho phí giao dịch trả bằng đơn vị khác:

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/proto/cosmos/tx/v1beta1/tx.proto#L252-L264
```

* `amount`: Số tiền tip.
* `tipper`: Địa chỉ tài khoản trả tip.

### AuxSignerData

`AuxSignerData` là định dạng trung gian mà bên ký phụ trợ (ví dụ: tipper) tạo và gửi đến fee payer (người sẽ tạo và phát sóng giao dịch thực tế). `AuxSignerData` không phải giao dịch hợp lệ và sẽ bị node từ chối nếu gửi trực tiếp.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/proto/cosmos/tx/v1beta1/tx.proto#L266-L284
```

### SIGN_MODE_DIRECT_AUX

Chế độ ký `SIGN_MODE_DIRECT_AUX` được thiết kế cho các giao dịch có nhiều bên ký và hỗ trợ luồng Tips. Các bên ký phụ trợ (auxiliary signers) dùng chế độ này để chỉ ký lên `TxBody` và thông tin bên ký của chính họ, mà không cần ký lên phí. Fee payer sau đó thu thập chữ ký và tạo giao dịch cuối cùng.

Xem thêm [SIGN_MODE_DIRECT_AUX](./01-transactions.md#sign_mode_direct_aux) trong tài liệu Giao dịch.

## Bật Tips trên chuỗi

Để chuỗi hỗ trợ Tips, chain phải thêm `TipDecorator` vào posthandler của nó. Nếu chuỗi không bật tips, trường `Tip` trong `AuthInfo` sẽ bị bỏ qua.

## Trường hợp sử dụng

Giải pháp này phục vụ các kịch bản khi người dùng cross-chain muốn thực hiện hành động trên chuỗi khác mà không cần swap token trước — ví dụ: người dùng Osmosis bỏ phiếu cho proposal trên Cosmos Hub mà không cần sở hữu token ATOM.

## Trạng thái

:::warning
Trường `Tip` và các kiểu liên quan trong Cosmos SDK đã được đánh dấu deprecated trong các phiên bản gần đây. Các chain mới nên tham khảo tài liệu và issue cập nhật để xác định cơ chế thay thế được khuyến nghị.
:::
