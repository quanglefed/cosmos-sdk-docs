# ADR 006: Thay Thế Secret Store

## Changelog

* 29 tháng 7 năm 2019: Bản nháp đầu tiên
* 11 tháng 9 năm 2019: Công việc đã bắt đầu
* 4 tháng 11: Các thay đổi Cosmos SDK đã được merge
* 18 tháng 11: Các thay đổi Gaia đã được merge

## Bối Cảnh

Hiện tại, thư mục CLI của một ứng dụng Cosmos SDK lưu trữ tài liệu khóa và metadata trong một cơ sở dữ liệu văn bản thuần túy trong thư mục home của người dùng. Tài liệu khóa được mã hóa bằng passphrase, được bảo vệ bởi thuật toán hashing bcrypt. Metadata (ví dụ: địa chỉ, khóa công khai, chi tiết lưu trữ khóa) có sẵn ở dạng văn bản thuần túy.

Điều này không mong muốn vì một số lý do. Lý do lớn nhất có lẽ là bảo vệ bảo mật không đủ cho tài liệu khóa và metadata. Rò rỉ văn bản thuần túy cho phép kẻ tấn công giám sát các khóa mà một máy tính nhất định kiểm soát thông qua nhiều kỹ thuật khác nhau, như các dependency bị xâm phạm mà không cần quyền thực thi đặc quyền. Điều này có thể dẫn đến một cuộc tấn công mục tiêu hơn vào một người dùng/máy tính cụ thể.

Tất cả hệ điều hành máy tính để bàn hiện đại (Ubuntu, Debian, MacOS, Windows) đều cung cấp secret store tích hợp sẵn được thiết kế để cho phép các ứng dụng lưu trữ thông tin được cách ly với tất cả các ứng dụng khác và yêu cầu nhập passphrase để truy cập dữ liệu.

Chúng ta đang tìm kiếm giải pháp cung cấp lớp trừu tượng chung cho nhiều backend khác nhau và phương án dự phòng hợp lý cho các nền tảng tối giản không cung cấp secret store gốc.

## Quyết Định

Chúng tôi khuyến nghị thay thế backend Keybase hiện tại dựa trên LevelDB bằng [Keyring](https://github.com/99designs/keyring) của 99 designs. Ứng dụng này được thiết kế để cung cấp lớp trừu tượng chung và giao diện thống nhất giữa nhiều secret store và được sử dụng bởi ứng dụng AWS Vault của 99-designs.

Điều này có vẻ đáp ứng yêu cầu bảo vệ cả tài liệu khóa và metadata khỏi phần mềm độc hại trên máy của người dùng.

## Trạng Thái

Đã Chấp Nhận

## Hậu Quả

### Tích Cực

Tăng cường bảo mật cho người dùng.

### Tiêu Cực

Người dùng phải migrate thủ công.

Kiểm thử với tất cả các backend được hỗ trợ là khó khăn.

Chạy test cục bộ trên Mac yêu cầu nhiều lần nhập password lặp đi lặp lại.

### Trung Lập

Không có.

## Tài Liệu Tham Khảo

* #4754 Chuyển secret store sang keyring secret store (PR gốc bởi @poldsam) [__CLOSED__]
* #5029 Thêm hỗ trợ cho keybases được hỗ trợ bởi github.com/99designs/keyring [__MERGED__]
* #5097 Thêm lệnh migrate keys [__MERGED__]
* #5180 Bỏ on-disk keybase để dùng keyring [_PENDING_REVIEW_]
* cosmos/gaia#164 Bỏ on-disk keybase để dùng keyring (thay đổi của gaia) [_PENDING_REVIEW_]
