# ADR 053: Tái Cấu Trúc Go Module

## Changelog

* 2022-04-27: Bản nháp đầu tiên

## Trạng Thái

ĐỀ XUẤT

## Tóm Tắt

SDK hiện tại được xây dựng như một go module nguyên khối duy nhất. ADR này mô tả cách chúng ta tái cấu trúc SDK thành các go module nhỏ hơn được phiên bản độc lập để dễ bảo trì hơn.

## Bối Cảnh

Go module áp đặt các yêu cầu nhất định đối với các dự án phần mềm liên quan đến số phiên bản ổn định (bất cứ thứ gì trên 0.x) ở chỗ [bất kỳ thay đổi API breaking nào đều đòi hỏi tăng phiên bản major](https://go.dev/doc/modules/release-workflow#breaking), điều này về mặt kỹ thuật tạo ra một go module mới (với hậu tố v2, v3, v.v.).

Cosmos SDK là một dự án khá lớn xuất phát trước khi go module ra đời và luôn ở phiên bản v0.x mặc dù đã được sử dụng trong production trong nhiều năm, không phải vì nó không phải phần mềm chất lượng production, mà vì các đảm bảo tương thích API theo yêu cầu của go module khá phức tạp để tuân thủ với một dự án lớn như vậy.

Tuy nhiên, mong muốn về semantic versioning đã [mạnh mẽ trong cộng đồng](https://github.com/cosmos/cosmos-sdk/discussions/10162) và quy trình phát hành go module đơn lẻ đã khiến rất khó phát hành các thay đổi nhỏ cho các tính năng riêng lẻ kịp thời. Các chu kỳ phát hành thường vượt quá sáu tháng, có nghĩa là các cải tiến nhỏ thực hiện trong một hoặc hai ngày bị tắc nghẽn bởi mọi thứ khác trong chu kỳ phát hành nguyên khối.

## Quyết Định

Để cải thiện tình trạng hiện tại, SDK đang được tái cấu trúc thành nhiều go module trong kho lưu trữ hiện tại. Phương pháp được áp dụng là:

* Một go module nên nói chung được xác định phạm vi đến một tập hợp chức năng cụ thể nhất quán (chẳng hạn như math, errors, store, v.v.)
* Khi code được xóa khỏi core SDK và chuyển sang đường dẫn module mới, mọi nỗ lực nên được thực hiện để tránh thay đổi API breaking trong code hiện có bằng cách sử dụng alias và wrapper types.
* Các go module mới nên được chuyển sang domain độc lập (`cosmossdk.io`) trước khi được gắn tag là `v1.0.0`.
* Tất cả go module nên tuân theo hướng dẫn trong https://go.dev/blog/module-compatibility trước khi `v1.0.0` được gắn tag và nên sử dụng package `internal` để giới hạn bề mặt API được tiếp xúc.
* API của go module mới có thể khác với code hiện có ở những chỗ có sự cải thiện rõ ràng hoặc để xóa các phụ thuộc legacy (ví dụ về amino hoặc gogo proto), miễn là package cũ cố gắng tránh API breakage với alias và wrapper.

## Hậu Quả

### Tương Thích Ngược

Nếu các hướng dẫn trên được tuân theo để sử dụng alias hoặc wrapper types trỏ lại các go module mới, không nên có hoặc rất ít thay đổi breaking cho API hiện có.

### Tích Cực

* Các phần mềm độc lập sẽ đạt `v1.0.0` sớm hơn.
* Các tính năng mới cho chức năng cụ thể sẽ được phát hành sớm hơn.

### Tiêu Cực

* Sẽ có nhiều phiên bản go module hơn để cập nhật trong SDK và mỗi dự án, mặc dù hầu hết trong số này sẽ là gián tiếp.

## Thảo Luận Thêm

Các thảo luận tiếp theo đang diễn ra chủ yếu tại https://github.com/cosmos/cosmos-sdk/discussions/10582 và trong Cosmos SDK Framework Working Group.

## Tài Liệu Tham Khảo

* https://go.dev/doc/modules/release-workflow
* https://go.dev/blog/module-compatibility
* https://github.com/cosmos/cosmos-sdk/discussions/10162
* https://github.com/cosmos/cosmos-sdk/pull/10779
