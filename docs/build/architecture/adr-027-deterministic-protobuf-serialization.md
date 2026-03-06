# ADR 027: Tuần Tự Hóa Protobuf Xác Định

## Nhật Ký Thay Đổi

* 07-08-2020: Bản nháp đầu tiên
* 01-09-2020: Làm rõ thêm các quy tắc

## Trạng Thái

Đề Xuất

## Tóm Tắt

Tuần tự hóa cấu trúc xác định hoàn toàn, hoạt động trên nhiều ngôn ngữ và client, là cần thiết khi ký các message. Chúng ta cần đảm bảo rằng bất cứ khi nào chúng ta tuần tự hóa một cấu trúc dữ liệu, dù bằng ngôn ngữ nào được hỗ trợ, các byte thô sẽ giống nhau. Tuần tự hóa [Protobuf](https://developers.google.com/protocol-buffers/docs/proto3) không phải là song ánh (tức là tồn tại một số lượng gần như vô hạn các biểu diễn nhị phân hợp lệ cho một tài liệu protobuf nhất định)<sup>1</sup>.

Tài liệu này mô tả một lược đồ tuần tự hóa xác định cho một tập con của các tài liệu protobuf, bao gồm trường hợp sử dụng này nhưng cũng có thể được tái sử dụng trong các trường hợp khác.

### Bối Cảnh

Để xác minh chữ ký trong Cosmos SDK, người ký và người xác minh cần đồng ý về cùng một tuần tự hóa của `SignDoc` như được định nghĩa trong [ADR-020](./adr-020-protobuf-transaction-encoding.md) mà không truyền tuần tự hóa.

Hiện tại, đối với chữ ký block, chúng ta đang sử dụng cách giải quyết: chúng ta tạo một thể hiện [TxRaw](https://github.com/cosmos/cosmos-sdk/blob/9e85e81e0e8140067dd893421290c191529c148c/proto/cosmos/tx/v1beta1/tx.proto#L30) mới (như được định nghĩa trong [adr-020-protobuf-transaction-encoding](https://github.com/cosmos/cosmos-sdk/blob/main/docs/architecture/adr-020-protobuf-transaction-encoding.md#transactions)) bằng cách chuyển đổi tất cả các trường [Tx](https://github.com/cosmos/cosmos-sdk/blob/9e85e81e0e8140067dd893421290c191529c148c/proto/cosmos/tx/v1beta1/tx.proto#L13) thành byte ở phía client. Điều này thêm một bước thủ công bổ sung khi gửi và ký giao dịch.

### Quyết Định

Lược đồ mã hóa sau đây được sử dụng bởi các ADR khác, và đặc biệt cho tuần tự hóa `SignDoc`.

## Đặc Tả

### Phạm Vi

ADR này định nghĩa một bộ tuần tự hóa protobuf3. Đầu ra là một tuần tự hóa protobuf hợp lệ, sao cho mọi trình phân tích cú pháp protobuf đều có thể phân tích nó.

Không hỗ trợ map trong phiên bản 1 do độ phức tạp của việc định nghĩa một tuần tự hóa xác định. Điều này có thể thay đổi trong tương lai. Các triển khai phải từ chối các tài liệu chứa map như đầu vào không hợp lệ.

### Nền Tảng - Mã Hóa Protobuf3

Hầu hết các kiểu số trong protobuf3 được mã hóa dưới dạng [varint](https://developers.google.com/protocol-buffers/docs/encoding#varints). Varint dài tối đa 10 byte, và vì mỗi byte varint có 7 bit dữ liệu, varint là biểu diễn của `uint70` (số nguyên không dấu 70-bit). Khi mã hóa, các giá trị số được chuyển đổi từ kiểu cơ sở sang `uint70`, và khi giải mã, `uint70` được phân tích được chuyển đổi sang kiểu số thích hợp.

Giá trị hợp lệ tối đa cho một varint tuân theo protobuf3 là `FF FF FF FF FF FF FF FF FF 7F` (tức là `2**70 -1`). Nếu kiểu trường là `{,u,s}int64`, 6 bit cao nhất của 70 bit sẽ bị bỏ trong quá trình giải mã, gây ra 6 bit thay đổi. Nếu kiểu trường là `{,u,s}int32`, 38 bit cao nhất của 70 bit sẽ bị bỏ trong quá trình giải mã, gây ra 38 bit thay đổi.

Trong số các nguồn gây ra tính không xác định khác, ADR này loại bỏ khả năng thay đổi mã hóa.

### Quy Tắc Tuần Tự Hóa

Tuần tự hóa dựa trên [mã hóa protobuf3](https://developers.google.com/protocol-buffers/docs/encoding) với các bổ sung sau:

1. Các trường phải được tuần tự hóa chỉ một lần theo thứ tự tăng dần
2. Các trường hoặc dữ liệu bổ sung không được thêm vào
3. [Giá trị mặc định](https://developers.google.com/protocol-buffers/docs/proto3#default) phải được bỏ qua
4. Các trường `repeated` của kiểu số vô hướng phải sử dụng [mã hóa packed](https://developers.google.com/protocol-buffers/docs/encoding#packed)
5. Mã hóa varint không được dài hơn cần thiết:
    * Không có byte không (trailing zero bytes) (trong little endian, tức là không có số không đứng đầu trong big endian). Theo quy tắc 3 ở trên, giá trị mặc định `0` phải được bỏ qua, vì vậy quy tắc này không áp dụng trong các trường hợp như vậy.
    * Giá trị tối đa cho một varint phải là `FF FF FF FF FF FF FF FF FF 01`. Nói cách khác, khi giải mã, 6 bit cao nhất của số nguyên không dấu 70-bit phải là `0`. (10-byte varints là 10 nhóm 7 bit, tức là 70 bit, trong đó chỉ có 70-6=64 bit thấp nhất là hữu ích.)
    * Giá trị tối đa cho các giá trị 32-bit trong mã hóa varint phải là `FF FF FF FF 0F` với một ngoại lệ (bên dưới). Nói cách khác, khi giải mã, 38 bit cao nhất của số nguyên không dấu 70-bit phải là `0`.
        * Một ngoại lệ cho điều trên là `int32` _âm_, phải được mã hóa sử dụng đầy đủ 10 byte để mở rộng dấu<sup>2</sup>.
    * Giá trị tối đa cho các giá trị Boolean trong mã hóa varint phải là `01` (tức là phải là `0` hoặc `1`). Theo quy tắc 3 ở trên, giá trị mặc định `0` phải được bỏ qua, vì vậy nếu một Boolean được bao gồm nó phải có giá trị `1`.

Trong khi các quy tắc số 1. và 2. nên khá đơn giản và mô tả hành vi mặc định của tất cả các bộ mã hóa protobuf mà tác giả biết, quy tắc thứ 3. thú vị hơn. Sau khi giải tuần tự hóa protobuf3, bạn không thể phân biệt giữa các trường chưa được đặt và các trường được đặt thành giá trị mặc định<sup>3</sup>. Tuy nhiên, ở cấp tuần tự hóa, có thể đặt các trường với giá trị rỗng hoặc bỏ qua chúng hoàn toàn. Đây là sự khác biệt đáng kể so với ví dụ JSON nơi một thuộc tính có thể rỗng (`""`, `0`), `null` hoặc không xác định, dẫn đến 3 tài liệu khác nhau.

Bỏ qua các trường được đặt thành giá trị mặc định là hợp lệ vì trình phân tích cú pháp phải gán giá trị mặc định cho các trường bị thiếu trong tuần tự hóa<sup>4</sup>. Đối với các kiểu vô hướng, việc bỏ qua các giá trị mặc định được yêu cầu bởi spec<sup>5</sup>. Đối với các trường `repeated`, không tuần tự hóa chúng là cách duy nhất để biểu thị danh sách rỗng. Các enum phải có phần tử đầu tiên có giá trị số 0, đây là giá trị mặc định<sup>6</sup>. Và các trường message mặc định là chưa được đặt<sup>7</sup>.

Việc bỏ qua các giá trị mặc định cho phép một số khả năng tương thích tiến: người dùng các phiên bản mới hơn của schema protobuf tạo ra cùng tuần tự hóa với người dùng các phiên bản cũ hơn miễn là các trường mới được thêm vào không được sử dụng (tức là được đặt thành giá trị mặc định của chúng).

### Triển Khai

Có ba chiến lược triển khai chính, được sắp xếp từ ít tùy chỉnh nhất đến nhiều nhất:

* **Sử dụng bộ tuần tự hóa protobuf tuân theo các quy tắc trên theo mặc định.** Vd: [gogoproto](https://pkg.go.dev/github.com/cosmos/gogoproto/gogoproto) được biết là tuân thủ trong hầu hết các trường hợp, nhưng không khi một số chú thích như `nullable = false` được sử dụng. Cũng có thể là một lựa chọn để cấu hình một bộ tuần tự hóa hiện có phù hợp.
* **Chuẩn hóa các giá trị mặc định trước khi mã hóa chúng.** Nếu bộ tuần tự hóa của bạn tuân theo quy tắc 1. và 2. và cho phép bạn rõ ràng hủy đặt các trường cho tuần tự hóa, bạn có thể chuẩn hóa các giá trị mặc định thành chưa đặt. Điều này có thể được thực hiện khi làm việc với [protobuf.js](https://www.npmjs.com/package/protobufjs):

  ```js
  const bytes = SignDoc.encode({
    bodyBytes: body.length > 0 ? body : null, // chuẩn hóa byte rỗng thành chưa đặt
    authInfoBytes: authInfo.length > 0 ? authInfo : null, // chuẩn hóa byte rỗng thành chưa đặt
    chainId: chainId || null, // chuẩn hóa "" thành chưa đặt
    accountNumber: accountNumber || null, // chuẩn hóa 0 thành chưa đặt
    accountSequence: accountSequence || null, // chuẩn hóa 0 thành chưa đặt
  }).finish();
  ```

* **Sử dụng bộ tuần tự hóa viết tay cho các kiểu bạn cần.** Nếu không có cách nào ở trên phù hợp với bạn, bạn có thể tự viết bộ tuần tự hóa. Đối với SignDoc điều này sẽ trông như thế này trong Go, xây dựng trên các tiện ích protobuf hiện có:

  ```go
  if !signDoc.body_bytes.empty() {
      buf.WriteUVarInt64(0xA) // wire type và field number cho body_bytes
      buf.WriteUVarInt64(signDoc.body_bytes.length())
      buf.WriteBytes(signDoc.body_bytes)
  }

  if !signDoc.auth_info.empty() {
      buf.WriteUVarInt64(0x12) // wire type và field number cho auth_info
      buf.WriteUVarInt64(signDoc.auth_info.length())
      buf.WriteBytes(signDoc.auth_info)
  }

  if !signDoc.chain_id.empty() {
      buf.WriteUVarInt64(0x1a) // wire type và field number cho chain_id
      buf.WriteUVarInt64(signDoc.chain_id.length())
      buf.WriteBytes(signDoc.chain_id)
  }

  if signDoc.account_number != 0 {
      buf.WriteUVarInt64(0x20) // wire type và field number cho account_number
      buf.WriteUVarInt(signDoc.account_number)
  }

  if signDoc.account_sequence != 0 {
      buf.WriteUVarInt64(0x28) // wire type và field number cho account_sequence
      buf.WriteUVarInt(signDoc.account_sequence)
  }
  ```

### Vector Kiểm Tra

Cho định nghĩa protobuf `Article.proto`

```protobuf
package blog;
syntax = "proto3";

enum Type {
  UNSPECIFIED = 0;
  IMAGES = 1;
  NEWS = 2;
};

enum Review {
  UNSPECIFIED = 0;
  ACCEPTED = 1;
  REJECTED = 2;
};

message Article {
  string title = 1;
  string description = 2;
  uint64 created = 3;
  uint64 updated = 4;
  bool public = 5;
  bool promoted = 6;
  Type type = 7;
  Review review = 8;
  repeated string comments = 9;
  repeated string backlinks = 10;
};
```

Tuần tự hóa các giá trị

```yaml
title: "The world needs change 🌳"
description: ""
created: 1596806111080
updated: 0
public: true
promoted: false
type: Type.NEWS
review: Review.UNSPECIFIED
comments: ["Nice one", "Thank you"]
backlinks: []
```

phải dẫn đến tuần tự hóa

```text
0a1b54686520776f726c64206e65656473206368616e676520f09f8cb318e8bebec8bc2e280138024a084e696365206f6e654a095468616e6b20796f75
```

Khi kiểm tra tài liệu được tuần tự hóa, bạn thấy rằng mỗi trường thứ hai bị bỏ qua:

```shell
$ echo 0a1b54686520776f726c64206e65656473206368616e676520f09f8cb318e8bebec8bc2e280138024a084e696365206f6e654a095468616e6b20796f75 | xxd -r -p | protoc --decode_raw
1: "The world needs change \360\237\214\263"
3: 1596806111080
5: 1
7: 2
9: "Nice one"
9: "Thank you"
```

## Hậu Quả

Việc có sẵn một mã hóa như vậy cho phép chúng ta có được tuần tự hóa xác định cho tất cả các tài liệu protobuf chúng ta cần trong bối cảnh ký Cosmos SDK.

### Tích Cực

* Các quy tắc được định nghĩa rõ ràng có thể được xác minh độc lập với triển khai tham chiếu
* Đủ đơn giản để duy trì rào cản thấp cho việc triển khai ký giao dịch
* Cho phép chúng ta tiếp tục sử dụng 0 và các giá trị rỗng khác trong SignDoc, tránh nhu cầu giải quyết các vấn đề với sequence 0

### Tiêu Cực

* Khi triển khai ký giao dịch, các quy tắc mã hóa trên phải được hiểu và triển khai.
* Nhu cầu về quy tắc số 3. thêm một số độ phức tạp vào các triển khai.
* Một số cấu trúc dữ liệu có thể yêu cầu code tùy chỉnh để tuần tự hóa. Do đó code không rất di động -- nó sẽ yêu cầu thêm công việc cho mỗi client triển khai tuần tự hóa để xử lý đúng các cấu trúc dữ liệu tùy chỉnh.

### Trung Lập

### Sử Dụng Trong Cosmos SDK

Vì các lý do được đề cập ở trên (phần "Tiêu Cực"), chúng ta ưu tiên giữ các cách giải quyết cho cấu trúc dữ liệu được chia sẻ. Ví dụ: `TxRaw` đã đề cập sử dụng byte thô như cách giải quyết. Điều này cho phép họ sử dụng bất kỳ thư viện Protobuf hợp lệ nào mà không cần triển khai bộ tuần tự hóa tùy chỉnh tuân theo tiêu chuẩn này (và các rủi ro liên quan về lỗi).

## Tham Khảo

* <sup>1</sup> _Khi một message được tuần tự hóa, không có thứ tự đảm bảo cho các trường đã biết hoặc chưa biết của nó sẽ được viết như thế nào. Thứ tự tuần tự hóa là chi tiết triển khai và các chi tiết của bất kỳ triển khai cụ thể nào có thể thay đổi trong tương lai. Do đó, các trình phân tích cú pháp protocol buffer phải có khả năng phân tích các trường theo bất kỳ thứ tự nào._ từ https://developers.google.com/protocol-buffers/docs/encoding#order
* <sup>2</sup> https://developers.google.com/protocol-buffers/docs/encoding#signed_integers
* <sup>3</sup> _Lưu ý rằng đối với các trường message vô hướng, một khi message được phân tích cú pháp không có cách nào biết liệu trường đó có được đặt rõ ràng thành giá trị mặc định hay không..._ từ https://developers.google.com/protocol-buffers/docs/proto3#default
* <sup>4</sup> _Khi một message được phân tích cú pháp, nếu message được mã hóa không chứa một phần tử số ít cụ thể, trường tương ứng trong đối tượng được phân tích được đặt thành giá trị mặc định cho trường đó._ từ https://developers.google.com/protocol-buffers/docs/proto3#default
* <sup>5</sup> _Cũng lưu ý rằng nếu trường message vô hướng được đặt thành giá trị mặc định của nó, giá trị sẽ không được tuần tự hóa trên wire._ từ https://developers.google.com/protocol-buffers/docs/proto3#default
* <sup>6</sup> _Đối với enum, giá trị mặc định là giá trị enum đầu tiên được định nghĩa, phải là 0._ từ https://developers.google.com/protocol-buffers/docs/proto3#default
* <sup>7</sup> _Đối với các trường message, trường không được đặt. Giá trị chính xác của nó phụ thuộc vào ngôn ngữ._ từ https://developers.google.com/protocol-buffers/docs/proto3#default
