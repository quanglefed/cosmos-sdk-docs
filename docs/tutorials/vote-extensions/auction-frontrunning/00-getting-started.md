# Bắt Đầu

## Mục Lục

- [Bắt Đầu](#tổng-quan-dự-án)
- [Hiểu về Front-Running](./01-understanding-frontrunning.md)
- [Giảm thiểu Front-running với Vote Extensions](./02-mitigating-front-running-with-vote-extesions.md)
- [Demo Giảm thiểu Front-Running](./03-demo-of-mitigating-front-running.md)

## Bắt Đầu

### Tổng Quan Dự Án

Tutorial này trình bày việc phát triển một module được thiết kế để giảm thiểu front-running trong các phiên đấu giá nameservice. Các hàm sau đây là trọng tâm của module này:

* `ExtendVote`: Thu thập các bid từ mempool và đưa chúng vào vote extension để đảm bảo quy trình đấu giá công bằng và minh bạch.
* `PrepareProposal`: Xử lý các vote extension từ block trước, tạo ra một giao dịch đặc biệt đóng gói các bid để đưa vào proposal hiện tại.
* `ProcessProposal`: Xác thực rằng giao dịch đầu tiên trong proposal là giao dịch đặc biệt chứa các vote extension và đảm bảo tính toàn vẹn của các bid.

Trong tutorial nâng cao này, chúng ta sẽ làm việc với một ứng dụng mẫu hỗ trợ đấu giá nameservice. Để hiểu frontrunning và nameservice là gì, xem [tại đây](./01-understanding-frontrunning.md). Ứng dụng này cung cấp một trường hợp sử dụng thực tế để khám phá việc ngăn chặn front-running đấu giá, còn được gọi là "bid sniping" — khi một validator lợi dụng việc thấy một bid trong mempool để đặt bid cao hơn của mình trước khi bid gốc được xử lý.

Tutorial sẽ hướng dẫn bạn sử dụng Cosmos SDK để giảm thiểu front-running bằng cách dùng vote extension. Module sẽ được xây dựng trên nền tảng blockchain cơ bản trong thư mục `tutorials/base` và sẽ sử dụng module `auction` làm nền. Sau khi hoàn thành tutorial này, bạn sẽ hiểu rõ hơn về cách ngăn chặn front-running trong các phiên đấu giá blockchain, cụ thể trong bối cảnh đấu giá nameservice.

## Vote Extension Là Gì?

Vote extension là thông tin tùy ý có thể được chèn vào một block. Tính năng này là một phần của ABCI 2.0, có sẵn trong SDK phiên bản 0.50 và là một phần của CometBFT phiên bản 0.38.

Thông tin thêm về vote extension có thể xem [tại đây](https://docs.cosmos.network/main/build/abci/vote-extensions).

## Yêu Cầu và Thiết Lập

Trước khi bắt đầu tutorial nâng cao về mô phỏng front-running đấu giá, hãy đảm bảo bạn đáp ứng các yêu cầu sau:

* Đã cài đặt [Golang >1.21.5](https://golang.org/doc/install)
* Quen thuộc với các khái niệm về front-running và MEV, như được trình bày chi tiết trong [Hiểu về Front-Running](./01-understanding-frontrunning.md)
* Hiểu về Vote Extension như được mô tả [tại đây](https://docs.cosmos.network/main/build/abci/vote-extensions)

Bạn cũng sẽ cần một blockchain nền tảng để xây dựng cùng với module của riêng bạn. Thư mục `tutorials/base` có code blockchain cần thiết để bắt đầu dự án tùy chỉnh với Cosmos SDK. Với module, bạn có thể sử dụng module `auction` trong thư mục `tutorials/auction/x/auction` làm tài liệu tham khảo, nhưng lưu ý rằng tất cả code cần thiết để triển khai vote extension đã được triển khai trong module này.

Điều này sẽ thiết lập nền tảng vững chắc cho blockchain của bạn, cho phép tích hợp các tính năng nâng cao như mô phỏng front-running đấu giá.
