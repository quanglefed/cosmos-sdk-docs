# ADR 050: SIGN_MODE_TEXTUAL: Phụ Lục 2 – Hướng Dẫn Hiển Thị Thiết Bị

## Changelog

* 03 tháng 10 năm 2022: Bản nháp đầu tiên

## Trạng Thái

DRAFT

## Tóm Tắt

Phụ lục này cung cấp hướng dẫn chuẩn tắc về cách các thiết bị nên render một tài liệu `SIGN_MODE_TEXTUAL`.

## Bối Cảnh

`SIGN_MODE_TEXTUAL` cho phép phiên bản dễ đọc của giao dịch được ký trên thiết bị bảo mật phần cứng, chẳng hạn như Ledger. Các phiên bản đầu của thiết kế render trực tiếp giao dịch thành các dòng văn bản ASCII, nhưng điều này tỏ ra bất tiện với tín hiệu in-band và việc cần hiển thị văn bản Unicode trong giao dịch.

## Quyết Định

`SIGN_MODE_TEXTUAL` render thành biểu diễn trừu tượng, để phần mềm dành riêng cho thiết bị quyết định cách trình bày biểu diễn này tùy theo khả năng, hạn chế và quy ước của thiết bị.

Chúng tôi đưa ra các hướng dẫn chuẩn tắc sau:

1. Bản trình bày phải càng dễ đọc càng tốt đối với người dùng, tùy theo khả năng của thiết bị.

2. Bản trình bày phải có thể đảo ngược nếu có thể mà không hy sinh đáng kể tính dễ đọc. Bất kỳ thay đổi nào đối với dữ liệu được render phải dẫn đến thay đổi hiển thị cho bản trình bày.

3. Bản trình bày phải tuân theo các quy ước thông thường của thiết bị, mà không hy sinh tính dễ đọc hoặc khả năng đảo ngược.

Như minh họa về các nguyên tắc này, đây là một ví dụ thuật toán để trình bày trên thiết bị có thể hiển thị một dòng ASCII 80 ký tự:

* Bản trình bày được chia thành các dòng, và mỗi dòng được trình bày theo thứ tự, với điều khiển người dùng để đi tiến hoặc lùi một dòng.
* Các màn hình chế độ expert chỉ được trình bày nếu thiết bị ở chế độ expert.
* Mỗi dòng màn hình bắt đầu bằng số ký tự `>` bằng mức thụt lề của màn hình, theo sau bởi ký tự `+` nếu đây không phải dòng đầu tiên của màn hình.
* Nếu dòng kết thúc bằng khoảng trắng hoặc ký tự `@`, một ký tự `@` bổ sung được thêm vào cuối dòng.
* Các ký tự điều khiển ASCII và backslash được chuyển đổi theo cách của string literals trong nhiều ngôn ngữ (ví dụ: `\n`, `\t`, `\\`).
* Tất cả các ký tự điều khiển ASCII khác và các điểm code Unicode không phải ASCII được hiển thị là `\u` theo sau bởi 4 ký tự hex.

Ví dụ đầu ra:

```
An introductory line.
key1: 123456
key2: a string that ends in whitespace   @
introducing an aggregate
> key5: false
> key6: a very long line of text, please co\u00F6perate and break into
>+  multiple lines.
>> You bet we can!
```

Ánh xạ ngược cho chúng ta đầu vào duy nhất có thể đã tạo ra đầu ra này:

```
Indent  Text
------  ----
0       "An introductory line."
0       "key1: 123456"
0       "key2: a string that ends in whitespace   "
0       "introducing an aggregate"
1       "key5: false"
1       "key6: a very long line of text, please coöperate and break into multiple lines."
2       "You bet we can!"
```
