# Bech32 trên Cosmos

Mạng Cosmos ưu tiên sử dụng định dạng địa chỉ Bech32 bất cứ nơi nào người dùng phải xử lý dữ liệu nhị phân. Mã hóa Bech32 cung cấp kiểm tra tính toàn vẹn mạnh mẽ trên dữ liệu và phần có thể đọc được bởi con người (HRP) cung cấp các gợi ý ngữ cảnh có thể hỗ trợ các nhà phát triển UI cung cấp thông báo lỗi hữu ích.

Trong mạng Cosmos, khóa và địa chỉ có thể đề cập đến một số vai trò khác nhau trong mạng như tài khoản, validator, v.v.

## Bảng HRP

| HRP | Định nghĩa |
| ---------------- | ------------------------------------- |
| cosmos | Địa chỉ tài khoản Cosmos |
| cosmosvalcons | Địa chỉ consensus validator Cosmos |
| cosmosvaloper | Địa chỉ operator validator Cosmos |

## Mã hóa

Trong khi tất cả các giao diện hướng đến người dùng của phần mềm Cosmos nên hiển thị các giao diện Bech32, nhiều giao diện nội bộ mã hóa giá trị nhị phân ở dạng hex hoặc base64.

Để chuyển đổi giữa các biểu diễn nhị phân khác của địa chỉ và khóa, điều quan trọng là trước tiên áp dụng quy trình mã hóa Amino trước khi mã hóa Bech32.

Triển khai đầy đủ định dạng tuần tự hóa Amino là không cần thiết trong hầu hết các trường hợp. Chỉ cần thêm các byte từ [bảng](https://github.com/cometbft/cometbft/blob/main/spec/blockchain/encoding.md) này vào payload chuỗi byte trước khi mã hóa Bech32 sẽ đủ cho biểu diễn tương thích.
