# ADR 050: SIGN_MODE_TEXTUAL

## Changelog

* 06 tháng 12 năm 2021: Bản nháp đầu tiên.
* 07 tháng 2 năm 2022: Nhóm Ledger đọc và concept-ACK.
* 16 tháng 5 năm 2022: Chuyển trạng thái sang Accepted.
* 11 tháng 8 năm 2022: Yêu cầu ký qua raw bytes của tx.
* 07 tháng 9 năm 2022: Thêm renderer tùy chỉnh cho `Msg`.
* 18 tháng 9 năm 2022: Định dạng có cấu trúc thay vì các dòng văn bản.
* 23 tháng 11 năm 2022: Chỉ định mã hóa CBOR.
* 06 tháng 12 năm 2022: Thứ tự lại các màn hình envelope.
* 23 tháng 1 năm 2023: Chuyển Screen.Text sang Title+Content.
* 07 tháng 3 năm 2023: Thay đổi SignDoc từ mảng sang struct chứa mảng.
* 20 tháng 3 năm 2023: Giới thiệu phiên bản spec được khởi tạo thành 0.

## Trạng Thái

Accepted. Triển khai đã bắt đầu. Một số chi tiết nhỏ về value renderer vẫn cần được tinh chỉnh.

Phiên bản spec: 0.

## Tóm Tắt

ADR này chỉ định SIGN_MODE_TEXTUAL, một chế độ ký dựa trên chuỗi mới nhắm đến việc ký với các thiết bị phần cứng.

## Bối Cảnh

SIGN_MODE_DIRECT dựa trên Protobuf được giới thiệu trong [ADR-020](./adr-020-protobuf-transaction-encoding.md) và được dự định thay thế SIGN_MODE_LEGACY_AMINO_JSON trong hầu hết các tình huống, chẳng hạn như ví di động và CLI keyring. Tuy nhiên, ví phần cứng [Ledger](https://www.ledger.com/) vẫn đang sử dụng SIGN_MODE_LEGACY_AMINO_JSON để hiển thị sign bytes cho người dùng. Ví phần cứng không thể chuyển sang SIGN_MODE_DIRECT vì:

* SIGN_MODE_DIRECT là dạng nhị phân và do đó không phù hợp để hiển thị cho người dùng cuối. Về mặt kỹ thuật, ví phần cứng có thể hiển thị sign bytes cho người dùng. Nhưng điều này sẽ được coi là blind signing và là mối lo ngại bảo mật.
* Phần cứng không thể giải mã các sign bytes protobuf do hạn chế bộ nhớ, vì các định nghĩa Protobuf cần được nhúng vào thiết bị phần cứng.

Trong nỗ lực xóa Amino khỏi SDK, cần tạo một chế độ ký mới cho thiết bị phần cứng. [Các thảo luận ban đầu](https://github.com/cosmos/cosmos-sdk/issues/6513) đề xuất chế độ ký dựa trên văn bản, mà ADR này chính thức chỉ định.

## Quyết Định

Trong SIGN_MODE_TEXTUAL, một giao dịch được hiển thị thành biểu diễn văn bản, sau đó được gửi đến thiết bị hoặc hệ thống con an toàn để người dùng xem xét và ký. Không giống `SIGN_MODE_DIRECT`, dữ liệu được truyền có thể được giải mã đơn giản thành văn bản dễ đọc ngay cả trên các thiết bị có khả năng xử lý và hiển thị hạn chế.

Biểu diễn văn bản là một chuỗi các _màn hình_. Mỗi màn hình được dự định hiển thị toàn bộ (nếu có thể) ngay cả trên thiết bị nhỏ như Ledger. Một màn hình tương đương với một dòng văn bản ngắn. Khoảng 40 ký tự là mục tiêu tốt.

Văn bản có thể chứa toàn bộ dải điểm code Unicode, bao gồm ký tự điều khiển và nul. Thiết bị chịu trách nhiệm quyết định cách hiển thị các ký tự mà nó không thể hiển thị native.

Các màn hình có mức thụt lề không âm để báo hiệu các cấu trúc tổng hợp hoặc lồng nhau. Mức thụt lề không là cấp cao nhất.

Một số màn hình được đánh dấu là màn hình _expert_, chỉ hiển thị nếu người xem chọn opt in cho chi tiết bổ sung.

### Rendering Có Thể Đảo Ngược

Chúng ta yêu cầu việc rendering giao dịch phải có thể đảo ngược: phải có một hàm parsing sao cho với mọi giao dịch, khi được render thành biểu diễn văn bản, việc phân tích biểu diễn đó tạo ra một proto message tương đương với bản gốc.

### SignDoc

SignDoc cho `SIGN_MODE_TEXTUAL` được hình thành từ cấu trúc dữ liệu như:

```go
type Screen struct {
  Title string
  Content string
  Indent uint8
  Expert bool
}

type SignDocTextual struct {
  Screens []Screen
}
```

Chúng ta sẽ sử dụng [CBOR](https://cbor.io) ([RFC 8949](https://www.rfc-editor.org/rfc/rfc8949.html)) thay vì tuần tự hóa protobuf. Mã hóa được định nghĩa bởi CDDL sau:

```
sign_doc = {
  screens_key: [* screen],
}
screens_key = 1

screen = {
  ? title_key: tstr,
  ? content_key: tstr,
  ? indent_key: uint,
  ? expert_key: bool,
}
title_key = 1
content_key = 2
indent_key = 3
expert_key = 4
```

## Chi Tiết

### Mã Hóa Envelope Giao Dịch

"Envelope giao dịch" là tất cả dữ liệu trong giao dịch không nằm trong trường `TxBody.Messages`. `//` ký hiệu comment và không hiển thị trên thiết bị Ledger.

```
Chain ID: <string>
Account number: <uint64>
Sequence: <uint64>
Address: <string>
*Public Key: <Any>
This transaction has <int> Message(s)
> Message (<int>/<int>): <Any>
End of Message
Memo: <string>                    // Bỏ qua nếu không có memo
Fee: <coins>
*Fee payer: <string>              // Bỏ qua nếu không có fee_payer
*Fee granter: <string>            // Bỏ qua nếu không có fee_granter
Tip: <coins>                      // Bỏ qua nếu không có tip
Tipper: <string>
*Gas Limit: <uint64>
*Timeout Height: <uint64>         // Bỏ qua nếu không có timeout_height
*Hash of raw bytes: <hex_string>  // Ngăn chặn tx hash malleability
```

### Renderer `Msg` Tùy Chỉnh

Nhà phát triển ứng dụng có thể chọn không tuân theo renderer value mặc định cho `Msg` của riêng họ. Điều này được thực hiện bằng cách đặt tùy chọn Protobuf `cosmos.msg.textual.v1.expert_custom_renderer` thành một chuỗi không trống.

### Yêu Cầu Ký Qua Raw Bytes của `TxBody` và `AuthInfo`

Chúng ta yêu cầu người dùng ký qua các bytes này trong SIGN_MODE_TEXTUAL, cụ thể hơn qua chuỗi sau:

```
*Hash of raw bytes: <HEX(sha256(len(body_bytes) ++ body_bytes ++ len(auth_info_bytes) ++ auth_info_bytes))>
```

Điều này nhằm ngăn chặn transaction hash malleability.

## Cập Nhật Đặc Tả Hiện Tại

Chúng ta phân biệt hai loại cập nhật:

1. Cập nhật yêu cầu thay đổi ứng dụng nhúng thiết bị phần cứng.
2. Cập nhật chỉ sửa đổi envelope và value renderer.

Chúng ta định nghĩa một phiên bản spec, là số nguyên phải được tăng lên với mỗi cập nhật của cả hai loại.

## Hậu Quả

### Tương Thích Ngược

SIGN_MODE_TEXTUAL hoàn toàn bổ sung và không vi phạm khả năng tương thích ngược với các chế độ ký khác.

### Tích Cực

* Cách thân thiện với người dùng để ký trên thiết bị phần cứng.
* Sau khi SIGN_MODE_TEXTUAL được phát hành, SIGN_MODE_LEGACY_AMINO_JSON có thể bị deprecated và xóa bỏ.

### Tiêu Cực

* Một số trường vẫn được mã hóa theo cách không thể đọc được, chẳng hạn như public key ở dạng hex.
* Cần phát hành ứng dụng Ledger mới.

### Trung Lập

* Nếu giao dịch phức tạp, mảng chuỗi có thể dài tùy ý, và một số người dùng có thể bỏ qua một số màn hình.

## Tài Liệu Tham Khảo

* [Annex 1](./adr-050-sign-mode-textual-annex1.md)
* Thảo luận ban đầu: https://github.com/cosmos/cosmos-sdk/issues/6513
