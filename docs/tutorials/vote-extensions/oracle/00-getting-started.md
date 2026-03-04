# Bắt Đầu

## Mục Lục

* [Oracle Là Gì?](./01-what-is-an-oracle.md)
* [Triển Khai Vote Extensions](./02-implementing-vote-extensions.md)
* [Kiểm Thử Module Oracle](./03-testing-oracle.md)

## Điều Kiện Tiên Quyết

Trước khi bắt đầu tutorial này, hãy đảm bảo bạn có:

* Một dự án chain đang hoạt động. Tutorial này sẽ không đề cập đến các bước tạo chain/module mới.
* Quen thuộc với Cosmos SDK. Nếu chưa, chúng tôi đề nghị bạn bắt đầu với [Cosmos SDK Tutorials](https://tutorials.cosmos.network), vì ABCI++ được coi là chủ đề nâng cao.
* Đã đọc và hiểu [Oracle Là Gì?](01-what-is-an-oracle.md). Bài này cung cấp thông tin nền cần thiết để hiểu module Oracle.
* Hiểu biết cơ bản về ngôn ngữ lập trình Go.

## Vote Extension Là Gì?

Vote extension là thông tin tùy ý có thể được chèn vào một block. Tính năng này là một phần của ABCI 2.0, có sẵn trong SDK phiên bản 0.50 và là một phần của CometBFT phiên bản 0.38.

Thông tin thêm về vote extension có thể xem [tại đây](https://docs.cosmos.network/main/build/abci/vote-extensions).

## Tổng Quan Dự Án

Chúng ta sẽ đi qua việc tạo một module price oracle đơn giản tập trung vào việc triển khai vote extension, bỏ qua các chi tiết bên trong bản thân price oracle.

Chúng ta sẽ đi qua việc triển khai:

* `ExtendVote` để lấy thông tin từ các API giá bên ngoài.
* `VerifyVoteExtension` để kiểm tra rằng định dạng của các vote được cung cấp là đúng.
* `PrepareProposal` để xử lý các vote extension từ block trước và đưa chúng vào proposal như một giao dịch.
* `ProcessProposal` để kiểm tra rằng giao dịch đầu tiên trong proposal thực sự là một "giao dịch đặc biệt" chứa thông tin giá.
* `PreBlocker` để làm cho thông tin giá có sẵn trong FinalizeBlock.

Nếu bạn muốn xem module oracle hoàn chỉnh đang hoạt động, vui lòng xem [tại đây](https://github.com/cosmos/sdk-tutorials/blob/master/tutorials/oracle/base/x/oracle)
