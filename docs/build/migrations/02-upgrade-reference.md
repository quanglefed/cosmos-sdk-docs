# Tài Liệu Tham Khảo Nâng Cấp

Tài liệu này cung cấp tài liệu tham khảo nhanh cho việc nâng cấp từ `v0.53.x` lên `v0.54.x` của Cosmos SDK.

Lưu ý, luôn đọc phần **App Wiring Changes** để biết thêm thông tin về các cập nhật kết nối ứng dụng.

🚨Nâng cấp lên v0.54.x sẽ yêu cầu nâng cấp chain **có điều phối**.🚨

### TLDR (Tóm Tắt)

**Tính năng lớn duy nhất trong Cosmos SDK v0.54.x là việc nâng cấp từ CometBFT v0.x.x lên CometBFT v2.**

Để xem danh sách đầy đủ các thay đổi, xem [Changelog](https://github.com/cosmos/cosmos-sdk/blob/release/v0.54.x/CHANGELOG.md).

#### Ngừng Sử Dụng `TimeoutCommit`

CometBFT v2 đã ngừng sử dụng `TimeoutCommit` cho một trường mới, `NextBlockDelay`, là một phần của
thông điệp ABCI `FinalizeBlockResponse` được trả về cho CometBFT thông qua SDK baseapp. Thêm thông tin từ
repo CometBFT có thể tìm thấy [tại đây](https://github.com/cometbft/cometbft/blob/88ef3d267de491db98a654be0af6d791e8724ed0/spec/abci/abci%2B%2B_methods.md?plain=1#L689).

Đối với nhà phát triển ứng dụng SDK và người vận hành node, điều này có nghĩa là giá trị `timeout_commit` trong file `config.toml`
vẫn được sử dụng nếu `NextBlockDelay` bằng 0 (giá trị mặc định của nó). Điều này có nghĩa là khi nâng cấp lên Cosmos SDK v0.54.x,
các giá trị `timeout_commit` hiện tại mà các validator đang sử dụng sẽ được duy trì và có hành vi tương tự.

Để đặt trường trong ứng dụng của bạn, có một tùy chọn `baseapp` mới, `SetNextBlockDelay`, có thể được truyền cho ứng dụng khi
khởi tạo trong `app.go`. Đặt giá trị này thành bất kỳ giá trị nào khác không sẽ ghi đè bất cứ điều gì được đặt trong `config.toml` của validator.
