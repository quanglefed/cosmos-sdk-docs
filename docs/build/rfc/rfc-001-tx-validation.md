# RFC 001: Xác Thực Transaction

## Changelog

* 2023-03-12: Đề xuất

## Background (Bối Cảnh)

Xác thực Transaction rất quan trọng cho một state machine hoạt động. Trong Cosmos SDK có hai luồng xác thực, một luồng bên ngoài message server và luồng kia bên trong. Luồng bên ngoài message server là hàm `ValidateBasic`. Nó được gọi trong antehandler trên cả `CheckTx` và `DeliverTx`. Có overhead và đôi khi trùng lặp xác thực trong hai luồng này. Việc xác thực bổ sung này cung cấp một kiểm tra thêm trước khi vào mempool.

Với sự ngừng sử dụng của [`GetSigners`](https://github.com/cosmos/cosmos-sdk/issues/11275) chúng ta có tùy chọn xóa [sdk.Msg](https://github.com/cosmos/cosmos-sdk/blob/16a5404f8e00ddcf8857c8a55dca2f7c109c29bc/types/tx_msg.go#L16) và hàm `ValidateBasic`.

Với sự tách biệt giữa CometBFT và Cosmos-SDK, thiếu sự kiểm soát đối với những transaction nào được broadcast và đưa vào block. Việc xác thực bổ sung trong antehandler nhằm giúp ích trong trường hợp này. Trong hầu hết các trường hợp, transaction đang hoặc nên được mô phỏng đối với một node để xác thực. Với luồng này, các transaction sẽ được xử lý giống nhau.

## Proposal (Đề Xuất)

Chấp nhận RFC này sẽ chuyển xác thực trong `ValidateBasic` sang message server trong các module, cập nhật các tutorial và tài liệu để xóa đề cập đến việc sử dụng `ValidateBasic` thay bằng xử lý tất cả xác thực cho một message tại nơi nó được thực thi.

Chúng ta vẫn có thể và sẽ hỗ trợ hàm `Validatebasic` cho người dùng và cung cấp một extension interface của hàm khi `sdk.Msg` bị ngừng sử dụng.

> Lưu ý: Đây là cách các message được xử lý trong các VM như Ethereum và CosmWasm.

### Consequences (Hậu Quả)

Hậu quả của việc cập nhật luồng transaction là các transaction có thể đã thất bại trước đây với luồng `ValidateBasic` bây giờ sẽ được đưa vào block và tính phí.
