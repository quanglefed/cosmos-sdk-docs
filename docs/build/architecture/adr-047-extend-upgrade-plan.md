# ADR 047: Mở Rộng Upgrade Plan

## Changelog

* Tháng 11 năm 2021: Bản nháp đầu tiên
* Tháng 5 năm 2023: Đề xuất BỊ TỪ BỎ. `pre_run` và `post_run` không còn cần thiết và việc thêm `artifacts` mang lại ít lợi ích.

## Trạng Thái

BỊ TỪ BỎ

## Tóm Tắt

ADR này mở rộng message proto `Plan` hiện có của x/upgrade để bao gồm các trường mới cho việc định nghĩa quy trình pre-run và post-run trong công cụ upgrade. Nó cũng định nghĩa cấu trúc để cung cấp các artifact có thể tải xuống liên quan đến upgrade.

## Bối Cảnh

Module `upgrade` kết hợp với Cosmovisor được thiết kế để tạo điều kiện và tự động hóa quá trình chuyển đổi của blockchain từ phiên bản này sang phiên bản khác.

`Plan` hiện tại chứa các trường sau:

* `name`: Chuỗi ngắn định danh phiên bản mới.
* `height`: Chiều cao chain mà nâng cấp sẽ được thực hiện.
* `info`: Chuỗi chứa thông tin về nâng cấp.

## Quyết Định

### Cập Nhật Protobuf

Chúng ta sẽ cập nhật message `x/upgrade.Plan` để cung cấp hướng dẫn nâng cấp. Hướng dẫn nâng cấp sẽ chứa danh sách artifact có sẵn cho mỗi nền tảng, và cho phép định nghĩa các lệnh pre-run và post-run.

```protobuf
message Plan {
  // ... (các trường hiện có)
  UpgradeInstructions instructions = 6;
}

message UpgradeInstructions {
  string pre_run              = 1;
  string post_run             = 2;
  repeated Artifact artifacts = 3;
  string description          = 4;
}

message Artifact {
  string platform      = 1;
  string url           = 2;
  string checksum      = 3;
  string checksum_algo = 4;
}
```

### Cập Nhật Module Upgrade

Nếu một `Plan` upgrade không sử dụng trường `UpgradeInstructions` mới, chức năng hiện có sẽ được duy trì. Việc phân tích trường `info` như URL hoặc JSON `binaries` sẽ bị deprecated.

### Cập Nhật Cosmovisor

Nếu file `upgrade-info.json` không chứa bất kỳ `UpgradeInstructions` nào, chức năng hiện có sẽ được duy trì.

Timeline nâng cấp mới tương tự như hiện tại. Các thay đổi được in đậm:

1. Một đề xuất governance upgrade được gửi và phê duyệt.
2. Chiều cao nâng cấp được đạt đến.
3. Module `x/upgrade` ghi file `upgrade_info.json` **(nay có thể có `UpgradeInstructions`)**.
4. Chain dừng lại.
5. Cosmovisor backup thư mục data (nếu được cài đặt).
6. Cosmovisor tải xuống file thực thi mới (nếu chưa có).
7. Cosmovisor thực thi **lệnh `pre_run` nếu được cung cấp**, hoặc lệnh `${DAEMON_NAME} pre-upgrade`.
8. Cosmovisor khởi động lại ứng dụng bằng phiên bản mới.
9. **Cosmovisor ngay lập tức chạy lệnh `post_run` trong một tiến trình tách biệt.**

## Hậu Quả

### Tương Thích Ngược

Vì thay đổi duy nhất đối với các định nghĩa hiện có là việc thêm trường `instructions` vào message `Plan`, và trường đó là tùy chọn, không có sự không tương thích ngược.

### Tích Cực

* Cấu trúc để định nghĩa artifact rõ ràng hơn vì nó được định nghĩa trong proto.
* Sẵn có lệnh pre-run trở nên rõ ràng hơn.
* Lệnh post-run trở nên khả thi.

### Tiêu Cực

* Message `Plan` trở nên lớn hơn.
* Không có tùy chọn để cung cấp URL sẽ trả về `UpgradeInstructions`.

## Tài Liệu Tham Khảo

* [upgrade.proto hiện tại](https://github.com/cosmos/cosmos-sdk/blob/v0.44.5/proto/cosmos/upgrade/v1beta1/upgrade.proto)
* [Cosmovisor README](https://github.com/cosmos/cosmos-sdk/blob/cosmovisor/v1.0.0/cosmovisor/README.md)
