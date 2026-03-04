# Giới Thiệu

## ABCI là gì?

ABCI (Application Blockchain Interface - Giao diện Blockchain Ứng dụng) là giao diện giữa CometBFT và ứng dụng. Thông tin chi tiết về ABCI có thể xem tại [đây](https://docs.cometbft.com/v0.38/spec/abci/). CometBFT phiên bản 0.38 đã giới thiệu một phiên bản mới của ABCI (gọi là ABCI 2.0), bổ sung thêm một số phương thức mới.

5 phương thức được giới thiệu trong ABCI 2.0 là:

* `PrepareProposal`
* `ProcessProposal`
* `ExtendVote`
* `VerifyVoteExtension`
* `FinalizeBlock`


## Luồng Xử Lý

## PrepareProposal

Dựa trên quyền bỏ phiếu của validator, CometBFT chọn ra một proposer (người đề xuất) cho block và gọi `PrepareProposal` trên ứng dụng (Cosmos SDK) của proposer đó. Proposer được chọn có trách nhiệm thu thập các giao dịch đang chờ xử lý từ mempool, tuân thủ theo các đặc tả của ứng dụng. Ứng dụng có thể áp dụng thứ tự giao dịch tùy chỉnh và đưa thêm các giao dịch bổ sung vào, có thể được tạo ra từ các vote extension ở block trước.

Để thực hiện thao tác này phía ứng dụng, cần phải triển khai một handler tùy chỉnh. Theo mặc định, Cosmos SDK cung cấp `PrepareProposalHandler`, được sử dụng kết hợp với mempool do ứng dụng tự quản lý. Nhà phát triển ứng dụng có thể viết handler tùy chỉnh; nếu cung cấp một noop handler, tất cả các giao dịch đều được coi là hợp lệ.

Lưu ý rằng vote extension chỉ khả dụng ở chiều cao block tiếp theo sau khi vote extension được bật. Thông tin chi tiết về vote extension có thể xem tại [đây](https://docs.cosmos.network/main/build/abci/vote-extensions).

Sau khi tạo xong đề xuất, proposer trả kết quả về cho CometBFT.

PrepareProposal CÓ THỂ không xác định (non-deterministic).

## ProcessProposal

Phương thức này cho phép các validator thực hiện các kiểm tra đặc thù của ứng dụng đối với đề xuất block và được gọi trên tất cả các validator. Đây là bước quan trọng trong quá trình đồng thuận, vì nó đảm bảo rằng block hợp lệ và đáp ứng các yêu cầu của ứng dụng. Ví dụ, các validator có thể kiểm tra xem block có chứa tất cả các giao dịch cần thiết hay không, hoặc block có tạo ra bất kỳ trạng thái chuyển đổi không hợp lệ nào không.

Việc triển khai `ProcessProposal` PHẢI có tính xác định (deterministic).

## ExtendVote và VerifyVoteExtensions

Các phương thức này cho phép ứng dụng mở rộng quá trình bỏ phiếu bằng cách yêu cầu các validator thực hiện các hành động bổ sung ngoài việc đơn thuần xác thực block.

Nếu vote extension được bật, `ExtendVote` sẽ được gọi trên mọi validator và mỗi validator sẽ trả về vote extension của mình, về bản chất là một chuỗi byte. Như đã đề cập, dữ liệu này (vote extension) chỉ có thể được truy xuất ở chiều cao block tiếp theo trong quá trình `PrepareProposal`. Ngoài ra, dữ liệu này có thể tùy ý, nhưng trong các hướng dẫn được cung cấp, nó đóng vai trò như một oracle hoặc bằng chứng về các giao dịch trong mempool. Về cơ bản, vote extension được xử lý và đưa vào như các giao dịch. Các ví dụ về trường hợp sử dụng vote extension bao gồm giá cho một oracle giá hoặc các phần chia sẻ mã hóa cho một mempool giao dịch được mã hóa. `ExtendVote` CÓ THỂ không xác định (non-deterministic).

`VerifyVoteExtensions` được thực thi trên mọi validator nhiều lần để xác minh vote extension của các validator khác. Quá trình kiểm tra này nhằm xác nhận tính toàn vẹn và hợp lệ của các vote extension, ngăn chặn các vote extension độc hại hoặc không hợp lệ.

Ngoài ra, ứng dụng phải giữ dữ liệu vote extension gọn nhẹ vì nó có thể làm giảm hiệu suất của chuỗi; xem kết quả kiểm thử tại [đây](https://docs.cometbft.com/v0.38/qa/cometbft-qa-38#vote-extensions-testbed).

`VerifyVoteExtensions` PHẢI có tính xác định (deterministic).


## FinalizeBlock

`FinalizeBlock` sau đó được gọi và chịu trách nhiệm cập nhật trạng thái của blockchain, đồng thời làm cho block khả dụng cho người dùng.
