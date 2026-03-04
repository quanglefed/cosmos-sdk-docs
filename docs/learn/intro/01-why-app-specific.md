---
sidebar_position: 1
---

# Blockchain Dành Riêng Cho Ứng Dụng

:::note Tóm tắt
Tài liệu này giải thích blockchain dành riêng cho ứng dụng là gì, và tại sao các nhà phát triển lại muốn xây dựng một cái thay vì viết Smart Contract.
:::

## Blockchain Dành Riêng Cho Ứng Dụng Là Gì

Blockchain dành riêng cho ứng dụng là các blockchain được tùy chỉnh để vận hành một ứng dụng duy nhất. Thay vì xây dựng một ứng dụng phi tập trung trên một blockchain nền như Ethereum, các nhà phát triển xây dựng blockchain của riêng họ từ đầu. Điều này có nghĩa là xây dựng một full-node client, một light-client, và tất cả các giao diện cần thiết (CLI, REST, ...) để tương tác với các node.

```text
                ^  +-------------------------------+  ^
                |  |                               |  |   Xây dựng với Cosmos SDK
                |  |  State-machine = Application  |  |
                |  |                               |  v
                |  +-------------------------------+
                |  |                               |  ^
Blockchain node |  |           Consensus           |  |
                |  |                               |  |
                |  +-------------------------------+  |   CometBFT
                |  |                               |  |
                |  |           Networking          |  |
                |  |                               |  |
                v  +-------------------------------+  v
```

## Những Hạn Chế Của Smart Contract Là Gì

Các blockchain máy ảo như Ethereum đã đáp ứng nhu cầu về khả năng lập trình cao hơn vào năm 2014. Lúc đó, các lựa chọn để xây dựng ứng dụng phi tập trung khá hạn chế. Hầu hết các nhà phát triển sẽ xây dựng trên ngôn ngữ scripting Bitcoin phức tạp và hạn chế, hoặc fork codebase Bitcoin vốn khó làm việc và tùy chỉnh.

Blockchain máy ảo xuất hiện với một đề xuất giá trị mới. State-machine của chúng tích hợp một máy ảo có khả năng thông dịch các chương trình Turing-complete gọi là Smart Contract. Các Smart Contract này rất tốt cho các trường hợp sử dụng như các sự kiện một lần (ví dụ: ICO), nhưng chúng có thể không đủ để xây dựng các nền tảng phi tập trung phức tạp. Lý do như sau:

* Smart Contract thường được phát triển bằng các ngôn ngữ lập trình cụ thể có thể được máy ảo bên dưới thông dịch. Các ngôn ngữ lập trình này thường chưa trưởng thành và vốn dĩ bị hạn chế bởi các ràng buộc của chính máy ảo. Ví dụ, Ethereum Virtual Machine không cho phép các nhà phát triển triển khai thực thi code tự động. Các nhà phát triển cũng bị giới hạn với hệ thống dựa trên tài khoản của EVM, và họ chỉ có thể chọn từ một tập hợp hàm hạn chế cho các thao tác mật mã của mình. Đây là những ví dụ, nhưng chúng gợi lên sự thiếu hụt về **tính linh hoạt** mà môi trường smart contract thường mang lại.
* Smart Contract đều chạy trên cùng một máy ảo. Điều này có nghĩa là chúng cạnh tranh tài nguyên với nhau, có thể làm giảm đáng kể **hiệu suất**. Và ngay cả khi state-machine được chia thành nhiều tập con (ví dụ: qua sharding), Smart Contract vẫn cần được thông dịch bởi máy ảo, điều này sẽ hạn chế hiệu suất so với một ứng dụng gốc được triển khai ở cấp độ state-machine (các benchmark của chúng tôi cho thấy cải thiện khoảng 10 lần về hiệu suất khi loại bỏ máy ảo).
* Một vấn đề khác với việc Smart Contract chia sẻ cùng một môi trường bên dưới là hạn chế về **chủ quyền**. Một ứng dụng phi tập trung là một hệ sinh thái liên quan đến nhiều bên. Nếu ứng dụng được xây dựng trên một blockchain máy ảo đa năng, các bên liên quan có chủ quyền rất hạn chế đối với ứng dụng của họ và cuối cùng bị chi phối bởi quản trị của blockchain bên dưới. Nếu có lỗi trong ứng dụng, rất ít có thể làm được.

Blockchain Dành Riêng Cho Ứng Dụng được thiết kế để giải quyết những hạn chế này.

## Lợi Ích Của Blockchain Dành Riêng Cho Ứng Dụng

### Tính Linh Hoạt

Blockchain dành riêng cho ứng dụng mang lại tối đa tính linh hoạt cho các nhà phát triển:

* Trong các blockchain Cosmos, state-machine thường được kết nối với consensus engine bên dưới qua một interface gọi là [ABCI](https://docs.cometbft.com/v0.37/spec/abci/). Interface này có thể được bọc trong bất kỳ ngôn ngữ lập trình nào, nghĩa là các nhà phát triển có thể xây dựng state-machine bằng ngôn ngữ lập trình họ chọn.

* Các nhà phát triển có thể chọn trong số nhiều framework để xây dựng state-machine. Phổ biến nhất hiện nay là Cosmos SDK, nhưng cũng có các lựa chọn khác (ví dụ: [Lotion](https://github.com/nomic-io/lotion), [Weave](https://github.com/iov-one/weave), ...). Thông thường, lựa chọn sẽ dựa trên ngôn ngữ lập trình họ muốn sử dụng (Cosmos SDK và Weave viết bằng Golang, Lotion bằng Javascript, ...).
* ABCI cũng cho phép các nhà phát triển hoán đổi consensus engine của blockchain dành riêng cho ứng dụng của họ. Hiện tại, chỉ CometBFT đã sẵn sàng cho production, nhưng trong tương lai các consensus engine khác dự kiến sẽ xuất hiện.
* Ngay cả khi đã chọn framework và consensus engine, các nhà phát triển vẫn có thể tự do tinh chỉnh chúng nếu chúng không hoàn toàn phù hợp với yêu cầu ở dạng gốc.
* Các nhà phát triển được tự do khám phá toàn bộ phổ đánh đổi (ví dụ: số lượng validator so với throughput giao dịch, an toàn so với khả dụng trong tình huống không đồng bộ, ...) và các lựa chọn thiết kế (DB hoặc cây IAVL để lưu trữ, UTXO hoặc mô hình tài khoản, ...).
* Các nhà phát triển có thể triển khai thực thi code tự động. Trong Cosmos SDK, logic có thể được kích hoạt tự động ở đầu và cuối mỗi block. Họ cũng được tự do chọn thư viện mật mã được sử dụng trong ứng dụng của mình, trái ngược với việc bị ràng buộc bởi những gì được cung cấp bởi môi trường bên dưới trong trường hợp blockchain máy ảo.

Danh sách trên chứa một vài ví dụ cho thấy blockchain dành riêng cho ứng dụng mang lại bao nhiêu tính linh hoạt cho các nhà phát triển. Mục tiêu của Cosmos và Cosmos SDK là làm cho các công cụ phát triển càng chung chung và có thể kết hợp càng tốt, để mỗi phần của stack có thể được fork, tinh chỉnh và cải tiến mà không mất tính tương thích. Khi cộng đồng phát triển, nhiều lựa chọn thay thế cho mỗi thành phần cốt lõi sẽ xuất hiện, mang lại nhiều tùy chọn hơn cho các nhà phát triển.

### Hiệu Suất

Các ứng dụng phi tập trung được xây dựng bằng Smart Contract vốn bị giới hạn hiệu suất bởi môi trường bên dưới. Để một ứng dụng phi tập trung tối ưu hóa hiệu suất, nó cần được xây dựng như một blockchain dành riêng cho ứng dụng. Dưới đây là một số lợi ích về hiệu suất mà blockchain dành riêng cho ứng dụng mang lại:

* Các nhà phát triển blockchain dành riêng cho ứng dụng có thể chọn vận hành với một consensus engine mới như CometBFT. So với Proof-of-Work (được hầu hết các blockchain máy ảo ngày nay sử dụng), nó mang lại lợi ích đáng kể về throughput.
* Một blockchain dành riêng cho ứng dụng chỉ vận hành một ứng dụng duy nhất, nên ứng dụng không cạnh tranh với các ứng dụng khác về tính toán và lưu trữ. Đây là điều ngược lại với hầu hết các blockchain máy ảo không phân mảnh ngày nay, nơi các smart contract đều cạnh tranh nhau về tính toán và lưu trữ.
* Ngay cả khi một blockchain máy ảo cung cấp sharding theo ứng dụng kết hợp với thuật toán đồng thuận hiệu quả, hiệu suất vẫn bị hạn chế bởi chính máy ảo. Nút thắt cổ chai throughput thực sự là state-machine, và yêu cầu giao dịch được thông dịch bởi máy ảo làm tăng đáng kể độ phức tạp tính toán của việc xử lý chúng.

### Bảo Mật

Bảo mật khó định lượng và biến đổi rất nhiều giữa các nền tảng. Tuy vậy, đây là một số lợi ích quan trọng mà blockchain dành riêng cho ứng dụng có thể mang lại về bảo mật:

* Các nhà phát triển có thể chọn các ngôn ngữ lập trình đã được chứng minh như Go khi xây dựng blockchain dành riêng cho ứng dụng của mình, trái ngược với các ngôn ngữ lập trình smart contract thường chưa trưởng thành hơn.
* Các nhà phát triển không bị ràng buộc bởi các hàm mật mã được cung cấp bởi các máy ảo bên dưới. Họ có thể sử dụng mật mã tùy chỉnh của riêng mình và dựa vào các thư viện crypto đã được kiểm tra kỹ lưỡng.
* Các nhà phát triển không cần lo lắng về các lỗi tiềm ẩn hoặc cơ chế có thể khai thác trong máy ảo bên dưới, giúp dễ dàng hơn trong việc suy luận về bảo mật của ứng dụng.

### Chủ Quyền

Một trong những lợi ích lớn nhất của blockchain dành riêng cho ứng dụng là chủ quyền. Một ứng dụng phi tập trung là một hệ sinh thái liên quan đến nhiều tác nhân: người dùng, nhà phát triển, dịch vụ bên thứ ba, và nhiều hơn nữa. Khi các nhà phát triển xây dựng trên một blockchain máy ảo nơi nhiều ứng dụng phi tập trung cùng tồn tại, cộng đồng của ứng dụng khác với cộng đồng của blockchain bên dưới, và cộng đồng sau chi phối cộng đồng trước trong quá trình quản trị. Nếu có lỗi hoặc cần một tính năng mới, các bên liên quan của ứng dụng có rất ít khả năng nâng cấp code. Nếu cộng đồng của blockchain bên dưới từ chối hành động, không có gì có thể xảy ra.

Vấn đề cơ bản ở đây là quản trị của ứng dụng và quản trị của mạng lưới không nhất quán với nhau. Vấn đề này được giải quyết bởi blockchain dành riêng cho ứng dụng. Vì blockchain dành riêng cho ứng dụng chuyên biệt để vận hành một ứng dụng duy nhất, các bên liên quan của ứng dụng có toàn quyền kiểm soát toàn bộ chuỗi. Điều này đảm bảo rằng cộng đồng sẽ không bị mắc kẹt nếu phát hiện lỗi, và có quyền tự do lựa chọn cách phát triển.
