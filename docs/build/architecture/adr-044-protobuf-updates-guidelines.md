# ADR 044: Hướng Dẫn Cập Nhật Định Nghĩa Protobuf

## Changelog

* 28.06.2021: Bản nháp đầu tiên
* 02.12.2021: Thêm comment `Since:` cho các trường mới
* 21.07.2022: Xóa quy tắc không có `Msg` mới trong cùng phiên bản proto.

## Trạng Thái

Bản Nháp

## Tóm Tắt

ADR này cung cấp các hướng dẫn và thực hành khuyến nghị khi cập nhật các định nghĩa Protobuf. Các hướng dẫn này nhắm đến nhà phát triển module.

## Bối Cảnh

Cosmos SDK duy trì một tập hợp [định nghĩa Protobuf](https://github.com/cosmos/cosmos-sdk/tree/main/proto/cosmos). Điều quan trọng là phải thiết kế đúng các định nghĩa Protobuf để tránh bất kỳ thay đổi breaking nào trong cùng một phiên bản. Lý do là không phá vỡ công cụ (bao gồm các indexer và explorer), ví và các tích hợp bên thứ ba khác.

Khi thực hiện các thay đổi đối với các định nghĩa Protobuf này, Cosmos SDK hiện chỉ tuân theo các khuyến nghị của [Buf](https://docs.buf.build/). Tuy nhiên chúng tôi nhận thấy rằng các khuyến nghị của Buf vẫn có thể dẫn đến các thay đổi breaking trong SDK trong một số trường hợp. Ví dụ:

* Thêm trường vào `Msg`. Thêm trường không phải là thao tác vi phạm đặc tả Protobuf. Tuy nhiên, khi thêm trường mới vào `Msg`, tính năng từ chối unknown field sẽ báo lỗi khi gửi `Msg` mới đến node cũ hơn.
* Đánh dấu trường là `reserved`. Protobuf đề xuất từ khóa `reserved` để xóa trường mà không cần tăng phiên bản package. Tuy nhiên, bằng cách làm vậy, tính tương thích ngược phía client bị phá vỡ vì Protobuf không tạo ra bất cứ thứ gì cho các trường `reserved`. Xem [#9446](https://github.com/cosmos/cosmos-sdk/issues/9446) để biết thêm chi tiết về vấn đề này.

Hơn nữa, các nhà phát triển module thường phải đối mặt với các câu hỏi khác xung quanh định nghĩa Protobuf như "Tôi có thể đổi tên trường không?" hoặc "Tôi có thể deprecated trường không?" ADR này nhằm trả lời tất cả các câu hỏi này bằng cách cung cấp các hướng dẫn rõ ràng về các cập nhật được phép cho định nghĩa Protobuf.

## Quyết Định

Chúng tôi quyết định giữ các khuyến nghị của [Buf](https://docs.buf.build/) với các ngoại lệ sau:

* `UNARY_RPC`: Cosmos SDK hiện không hỗ trợ streaming RPC.
* `COMMENT_FIELD`: Cosmos SDK cho phép các trường không có comment.
* `SERVICE_SUFFIX`: chúng ta sử dụng quy ước đặt tên dịch vụ `Query` và `Msg`, không sử dụng hậu tố `-Service`.
* `PACKAGE_VERSION_SUFFIX`: một số package, chẳng hạn như `cosmos.crypto.ed25519`, không sử dụng hậu tố phiên bản.
* `RPC_REQUEST_STANDARD_NAME`: Các yêu cầu cho dịch vụ `Msg` không có hậu tố `-Request` để duy trì khả năng tương thích ngược.

Trên các khuyến nghị của Buf, chúng tôi thêm các hướng dẫn sau dành riêng cho Cosmos SDK.

### Cập Nhật Định Nghĩa Protobuf Mà Không Tăng Phiên Bản

#### 1. Nhà phát triển module CÓ THỂ thêm các định nghĩa Protobuf mới

Nhà phát triển module CÓ THỂ thêm các `message` mới, `Service` mới, endpoint `rpc` mới, và các trường mới vào các message hiện có. Khuyến nghị này tuân theo đặc tả Protobuf, nhưng được thêm vào tài liệu này để rõ ràng hơn, vì SDK yêu cầu thêm một thay đổi nữa.

SDK yêu cầu comment Protobuf của phần bổ sung mới phải chứa một dòng với định dạng sau:

```protobuf
// Since: cosmos-sdk <version>{, <version>...}
```

Trong đó mỗi `version` biểu thị phiên bản minor ("0.45") hoặc patch ("0.44.5") kể từ đó trường mới có sẵn. Điều này sẽ giúp ích rất lớn cho các thư viện client, những thư viện có thể tùy chọn sử dụng reflection hoặc code generation tùy chỉnh để hiển thị/ẩn các trường này tùy thuộc vào phiên bản node được nhắm đến.

Ví dụ, các comment sau là hợp lệ:

```protobuf
// Since: cosmos-sdk 0.44

// Since: cosmos-sdk 0.42.11, 0.44.5
```

và các comment sau KHÔNG hợp lệ:

```protobuf
// Since cosmos-sdk v0.44

// since: cosmos-sdk 0.44

// Since: cosmos-sdk 0.42.11 0.44.5

// Since: Cosmos SDK 0.42.11, 0.44.5
```

#### 2. Các trường CÓ THỂ được đánh dấu là `deprecated`, và các node CÓ THỂ triển khai thay đổi vi phạm giao thức để xử lý các trường này

Protobuf hỗ trợ [tùy chọn trường `deprecated`](https://developers.google.com/protocol-buffers/docs/proto#options), và tùy chọn này CÓ THỂ được sử dụng trên bất kỳ trường nào, bao gồm cả các trường `Msg`. Nếu một node xử lý một message Protobuf với trường deprecated không trống, node CÓ THỂ thay đổi hành vi của mình khi xử lý, ngay cả theo cách vi phạm giao thức. Khi có thể, node PHẢI xử lý khả năng tương thích ngược mà không vi phạm đồng thuận (trừ khi chúng ta tăng phiên bản proto).

Ví dụ, bản cập nhật Cosmos SDK v0.42 lên v0.43 chứa hai thay đổi vi phạm Protobuf, được liệt kê bên dưới. Thay vì tăng phiên bản package từ `v1beta1` lên `v1`, nhóm SDK quyết định tuân theo hướng dẫn này, bằng cách hoàn nguyên các thay đổi breaking, đánh dấu những thay đổi đó là deprecated, và sửa đổi triển khai node khi xử lý message với các trường deprecated.

* Cosmos SDK gần đây đã xóa hỗ trợ cho [nâng cấp phần mềm dựa trên thời gian](https://github.com/cosmos/cosmos-sdk/pull/8849). Vì vậy, trường `time` đã được đánh dấu là deprecated trong `cosmos.upgrade.v1beta1.Plan`. Hơn nữa, node sẽ từ chối bất kỳ đề xuất nào chứa upgrade Plan có trường `time` không trống.
* Cosmos SDK hiện hỗ trợ [phiếu bầu phân chia governance](./adr-037-gov-split-vote.md). Khi truy vấn phiếu bầu, message `cosmos.gov.v1beta1.Vote` được trả về có trường `option` (được dùng cho 1 tùy chọn bỏ phiếu) deprecated để ủng hộ trường `options` (cho phép nhiều tùy chọn bỏ phiếu). Khi có thể, SDK vẫn điền vào trường deprecated `option`, tức là khi và chỉ khi `len(options) == 1` và `options[0].Weight == 1.0`.

#### 3. Các trường KHÔNG ĐƯỢC đổi tên

Trong khi các khuyến nghị Protobuf chính thức không cấm đổi tên trường, vì nó không vi phạm biểu diễn nhị phân Protobuf, SDK cấm rõ ràng việc đổi tên trường trong các struct Protobuf. Lý do chính cho lựa chọn này là để tránh đưa ra các thay đổi breaking cho client, thường phụ thuộc vào các trường hard-coded từ các kiểu được tạo. Hơn nữa, đổi tên trường sẽ dẫn đến các biểu diễn JSON vi phạm client của các định nghĩa Protobuf, được sử dụng trong các REST endpoint và trong CLI.

### Tăng Phiên Bản Package Protobuf

TODO, cần kiến trúc review. Một số chủ đề:

* Tần suất tăng phiên bản
* Khi tăng phiên bản, Cosmos SDK có nên hỗ trợ cả hai phiên bản không?
    * tức là v1beta1 -> v1, chúng ta có nên có hai thư mục trong Cosmos SDK và handler cho cả hai phiên bản không?
* đề cập ADR-023 đặt tên Protobuf

## Hậu Quả

### Tương Thích Ngược

### Tích Cực

* ít đau đớn hơn cho nhà phát triển công cụ
* khả năng tương thích tốt hơn trong hệ sinh thái

### Tiêu Cực

{hậu quả tiêu cực}

### Trung Lập

* xem xét Protobuf nghiêm ngặt hơn

## Thảo Luận Thêm

ADR này vẫn đang ở giai đoạn DRAFT, và "Tăng Phiên Bản Package Protobuf" sẽ được điền khi chúng ta đưa ra quyết định về cách thực hiện đúng.

## Tài Liệu Tham Khảo

* [#9445](https://github.com/cosmos/cosmos-sdk/issues/9445) Phát hành định nghĩa proto v1
* [#9446](https://github.com/cosmos/cosmos-sdk/issues/9446) Giải quyết các thay đổi vi phạm proto v1beta1
