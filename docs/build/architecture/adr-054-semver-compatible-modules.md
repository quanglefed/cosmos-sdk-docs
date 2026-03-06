# ADR 054: Các Module SDK Tương Thích Semver

## Changelog

* 2022-04-27: Bản nháp đầu tiên

## Trạng Thái

DRAFT

## Tóm Tắt

Để chuyển Cosmos SDK sang một hệ thống các module được phiên bản semantic độc lập có thể kết hợp theo các tổ hợp khác nhau (ví dụ: staking v3 với bank v1 và distribution v2), chúng ta cần xem xét lại cách tổ chức bề mặt API của các module để tránh các vấn đề với go semantic import versioning và circular dependencies.

## Bối Cảnh

Đã có [mong muốn đáng kể](https://github.com/cosmos/cosmos-sdk/discussions/10162) trong cộng đồng về semantic versioning trong SDK và đã có động thái đáng kể để chia các module SDK thành [go module độc lập](https://github.com/cosmos/cosmos-sdk/issues/11899).

Để đạt được điều này, chúng ta cần giải quyết các vấn đề sau:

1. Vì cách thức hoạt động của [go semantic import versioning](https://research.swtch.com/vgo-import) (SIV), việc chuyển sang SIV một cách ngây thơ sẽ thực sự khiến việc đạt được các mục tiêu này khó khăn hơn.
2. Các circular dependencies giữa các module cần được phá vỡ để thực sự phát hành nhiều module trong SDK một cách độc lập.
3. Các minor version incompatibility ngầm được giới thiệu thông qua việc [phát triển schema protobuf](https://developers.google.com/protocol-buffers/docs/proto3#updating) đúng cách mà không có [lọc unknown field](./adr-020-protobuf-transaction-encoding.md#unknown-field-filtering) đúng cách.

### Vấn Đề 1: Tương Thích Semantic Import Versioning

Hãy xem xét module `foo` định nghĩa `MsgDoSomething` và đã phát hành state machine trong go module `example.com/foo`. Khi chúng ta tạo revision với trường `condition` mới và phát hành `example.com/foo/v2`:

Nếu `bar` sử dụng `foo.MsgDoSomething` qua interface `FooKeeper`:
* **Kịch bản A (Foo mới hơn, Bar cũ hơn)**: Chain không thể nâng cấp lên `foo/v2` cho đến khi `bar` cập nhật tham chiếu của nó — ngay cả khi `bar` không sử dụng các tính năng mới.
* **Kịch bản B (Foo cũ hơn, Bar mới hơn)**: Chain không thể nâng cấp lên `bar/v2` mà không nâng cấp lên `foo/v2` ngay cả khi `bar/v2` không thực sự sử dụng tính năng mới của `foo/v2`.

### Vấn Đề 2: Circular Dependencies

Không có phương pháp nào ở trên cho phép `foo` và `bar` là các module riêng biệt nếu `foo` và `bar` phụ thuộc lẫn nhau theo các cách khác nhau.

### Vấn Đề 3: Xử Lý Minor Version Incompatibilities

Nếu `bar/v2` muốn sử dụng `MsgDoSomething.condition` chỉ có trong `foo/v2`, nhưng hoạt động với `foo` v1, thì `foo` sẽ âm thầm bỏ qua trường này dẫn đến lỗi logic nguy hiểm.

## Giải Pháp

### Phương Pháp A) Tách Module API và State Machine

Cô lập tất cả code được tạo từ protobuf vào một module riêng biệt khỏi module state machine. Điều này có nghĩa là chúng ta có thể có các go module state machine `foo` và `foo/v2` sử dụng một go module types hoặc API say `foo/api`. Module `foo/api` sẽ vĩnh viễn ở `v1.x` và chỉ chấp nhận các thay đổi không breaking.

Để giải quyết các vấn đề còn lại:
1. Xóa tất cả code vi phạm state machine khỏi module API (ví dụ `ValidateBasic` và các phương thức interface khác).
2. Nhúng các file descriptor đúng để lọc unknown field trong binary.

### Phương Pháp B) Thay Đổi Code Được Tạo Ra

Thay đổi cách code protobuf được tạo và chuyển các module chủ yếu sang giao tiếp inter-module như mô tả trong [ADR 033](./adr-033-protobuf-inter-module-comm.md). Trong mô hình này, một module có thể tạo tất cả các type mà nó cần nội bộ — bao gồm các API type của các module khác — và giao tiếp với các module khác qua ranh giới client-server.

### Phương Pháp C) Không Giải Quyết Các Vấn Đề Này

Nếu các giải pháp trên được coi là quá phức tạp, chúng ta cũng có thể quyết định không làm gì rõ ràng. Trong trường hợp đó, khi nhà phát triển đối mặt với các vấn đề được mô tả, họ có thể yêu cầu các dependency cập nhật đồng bộ (những gì chúng ta làm hiện nay) hoặc cố gắng một số giải pháp ad-hoc.

### Phương Pháp D) Tránh Code Được Tạo Từ Protobuf trong Public API

Tránh code được tạo từ protobuf trong các public module API, sử dụng các tham số vị trí thay thế.

## Quyết Định

Đề xuất **DRAFT** mới nhất là:

1. Chúng ta đồng thuận áp dụng [ADR 033](./adr-033-protobuf-inter-module-comm.md) không chỉ như một phần bổ sung cho framework mà như một sự thay thế core cho mô hình keeper.
2. Router inter-module ADR 033 sẽ phù hợp với bất kỳ biến thể nào của phương pháp (A) hoặc (B) với các quy tắc: a) nếu kiểu client giống kiểu server thì chuyển trực tiếp, b) nếu cả client và server dùng zero-copy generated code wrappers thì chuyển memory buffers, c) marshal/unmarshal types giữa client và server.

### Sửa Đổi API Minor

Để khai báo các minor API revision của proto files, chúng tôi đề xuất các hướng dẫn sau:

* Các proto package được sửa đổi từ phiên bản ban đầu nên bao gồm comment `Revision N` trong một số file .proto.
* Tất cả các trường, message, v.v. được thêm vào trong phiên bản ngoài revision ban đầu nên thêm comment `Since: Revision N`.

### Lọc Unknown Field

Để thực hiện đúng [lọc unknown field](./adr-020-protobuf-transaction-encoding.md#unknown-field-filtering), router inter-module có thể thực hiện một trong các cách sau:

* Sử dụng API `protoreflect` cho các message hỗ trợ điều đó.
* Đối với gogo proto messages, marshal và sử dụng code `codec/unknownproto` hiện có.

## Hậu Quả

### Tương Thích Ngược

Các module migrate hoàn toàn sang ADR 033 sẽ không tương thích với các module hiện có sử dụng mô hình keeper.

### Tích Cực

* Chúng ta sẽ có thể cung cấp các module có phiên bản semantic có thể tương tác, điều này sẽ tăng đáng kể khả năng của hệ sinh thái Cosmos SDK để lặp lại các tính năng mới.
* Sẽ có thể viết các module Cosmos SDK bằng các ngôn ngữ khác trong tương lai gần.

### Tiêu Cực

* Tất cả các module sẽ cần được tái cấu trúc khá đáng kể.

## Tài Liệu Tham Khảo

* https://github.com/cosmos/cosmos-sdk/discussions/10162
* https://github.com/cosmos/cosmos-sdk/discussions/10582
* [ADR 020](./adr-020-protobuf-transaction-encoding.md)
* [ADR 033](./adr-033-protobuf-inter-module-comm.md)
