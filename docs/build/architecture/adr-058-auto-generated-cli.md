# ADR 058: CLI Tự Động Tạo

## Changelog

* 2022-05-04: Bản nháp đầu tiên

## Trạng Thái

ACCEPTED Đã Triển Khai Một Phần

## Tóm Tắt

Để giúp nhà phát triển viết các module Cosmos SDK dễ dàng hơn, chúng tôi cung cấp cơ sở hạ tầng tự động tạo các lệnh CLI dựa trên các định nghĩa protobuf.

## Bối Cảnh

Các module Cosmos SDK hiện tại thường triển khai một lệnh CLI cho mỗi giao dịch và mỗi truy vấn được module hỗ trợ. Các lệnh này được viết tay cho mỗi lệnh và về cơ bản cung cấp một số CLI flag hoặc đối số vị trí cho các trường cụ thể trong các message protobuf.

Để đảm bảo các lệnh CLI được triển khai đúng cách cũng như đảm bảo ứng dụng hoạt động trong các kịch bản end-to-end, chúng ta thực hiện các integration test sử dụng các lệnh CLI. Mặc dù các test này có giá trị ở một mức độ nào đó, chúng có thể khó viết và bảo trì, và chạy chậm.

## Quyết Định

Để đơn giản hóa việc phát triển module, chúng tôi cung cấp cơ sở hạ tầng trong go module [`client/v2`](https://github.com/cosmos/cosmos-sdk/tree/main/client/v2) mới để tự động tạo các lệnh CLI dựa trên định nghĩa protobuf.

Thiết kế cơ bản để tự động tạo lệnh CLI là:

* Tạo một lệnh CLI cho mỗi phương thức `rpc` trong dịch vụ `Query` hoặc `Msg` protobuf.
* Tạo một CLI flag cho mỗi trường trong kiểu yêu cầu `rpc`.
* Đối với các lệnh `query`, gọi gRPC và in phản hồi dưới dạng protobuf JSON hoặc YAML.
* Đối với các lệnh `tx`, tạo giao dịch và áp dụng các flag giao dịch chung.

Để làm cho CLI tự động tạo dễ sử dụng, chúng ta cần xử lý tùy chỉnh các kiểu trường protobuf cụ thể:

* `Coin`, `Coins`, `DecCoin`, và `DecCoins` nên được nhập sử dụng định dạng hiện có (tức là `1000uatom`).
* Có thể chỉ định địa chỉ sử dụng chuỗi địa chỉ bech32 hoặc khóa được đặt tên trong keyring.
* `Timestamp` và `Duration` nên chấp nhận các chuỗi như `2001-01-01T00:00:00Z` và `1h3m`.
* Phân trang được xử lý với các flag như `--page-limit`, `--page-offset`, v.v.

## Hậu Quả

### Tương Thích Ngược

Các module hiện có có thể kết hợp các lệnh CLI tự động tạo và viết tay.

### Tích Cực

* Nhà phát triển module sẽ không cần viết lệnh CLI.
* Nhà phát triển module sẽ không cần test lệnh CLI.

## Thảo Luận Thêm

Chúng tôi muốn có thể tùy chỉnh:

* Chuỗi sử dụng ngắn và dài cho lệnh.
* Alias cho flag (ví dụ `-a` cho `--amount`).
* Các trường nào là tham số vị trí thay vì flag.

Đây là [thảo luận mở](https://github.com/cosmos/cosmos-sdk/pull/11725#issuecomment-1108676129) về việc các tùy chọn tùy chỉnh này nên nằm ở đâu: trong file .proto, file cấu hình riêng biệt, hoặc trực tiếp trong code.

## Tài Liệu Tham Khảo

* https://github.com/cosmos/cosmos-sdk/tree/main/client/v2
* https://github.com/regen-network/regen-ledger/issues/1041
