---
sidebar_position: 1
---

# Cosmos SDK là gì

[Cosmos SDK](https://github.com/cosmos/cosmos-sdk) là một bộ công cụ mã nguồn mở để xây dựng các blockchain Proof-of-Stake (PoS) đa tài sản công khai, như Cosmos Hub, cũng như các blockchain Proof-of-Authority (PoA) có cấp phép. Các blockchain được xây dựng với Cosmos SDK thường được gọi là **blockchain dành riêng cho ứng dụng** (application-specific blockchains).

Mục tiêu của Cosmos SDK là cho phép các nhà phát triển dễ dàng tạo ra các blockchain tùy chỉnh từ đầu, có khả năng tương tác gốc với các blockchain khác. Chúng tôi thúc đẩy thêm cách tiếp cận theo module này bằng cách cho phép nhà phát triển cắm vào và hoán đổi các consensus engine khác nhau — có thể là [CometBFT](https://github.com/cometbft/cometbft) hoặc [Rollkit](https://rollkit.dev/).

Các blockchain dựa trên SDK có thể chọn sử dụng các module được định nghĩa sẵn hoặc xây dựng module của riêng mình. Điều này có nghĩa là các nhà phát triển có thể xây dựng một blockchain phù hợp với trường hợp sử dụng cụ thể của họ, mà không phải lo lắng về các chi tiết cấp thấp của việc xây dựng blockchain từ đầu. Các module được định nghĩa sẵn bao gồm staking, governance, và phát hành token, trong số nhiều module khác.

Hơn nữa, Cosmos SDK là một hệ thống dựa trên capabilities (năng lực), cho phép các nhà phát triển suy luận tốt hơn về bảo mật của các tương tác giữa các module. Để tìm hiểu sâu hơn về capabilities, hãy xem [Mô hình Object-Capability](../advanced/10-ocap.md).

Hãy hình dung SDK giống như một bộ lego. Bạn có thể chọn xây ngôi nhà cơ bản theo hướng dẫn, hoặc bạn có thể tùy chỉnh ngôi nhà của mình và thêm nhiều tầng, nhiều cửa, nhiều cửa sổ hơn. Lựa chọn là của bạn.

## Blockchain Dành Riêng Cho Ứng Dụng Là Gì

Một mô hình phát triển trong thế giới blockchain ngày nay là các blockchain máy ảo như Ethereum, nơi việc phát triển thường xoay quanh việc xây dựng các ứng dụng phi tập trung trên một blockchain hiện có dưới dạng tập hợp smart contract. Mặc dù smart contract rất tốt cho một số trường hợp sử dụng như các ứng dụng dùng một lần (ví dụ: ICO), chúng thường không đủ để xây dựng các nền tảng phi tập trung phức tạp. Nói chung, smart contract có thể bị hạn chế về tính linh hoạt, chủ quyền và hiệu suất.

Blockchain dành riêng cho ứng dụng cung cấp một mô hình phát triển hoàn toàn khác so với blockchain máy ảo. Blockchain dành riêng cho ứng dụng là một blockchain được tùy chỉnh để vận hành một ứng dụng duy nhất: các nhà phát triển có toàn quyền tự do thực hiện các quyết định thiết kế cần thiết để ứng dụng hoạt động tối ưu. Chúng cũng có thể cung cấp chủ quyền, bảo mật và hiệu suất tốt hơn.

Tìm hiểu thêm về [blockchain dành riêng cho ứng dụng](./01-why-app-specific.md).

## Tính Module Là Gì

Ngày nay có nhiều cuộc thảo luận về tính module và các cuộc tranh luận giữa kiến trúc monolithic và modular. Ban đầu, Cosmos SDK được xây dựng với tầm nhìn về tính module. Tính module xuất phát từ việc tách blockchain thành các lớp có thể tùy chỉnh bao gồm thực thi (execution), đồng thuận (consensus), thanh toán (settlement) và khả dụng dữ liệu (data availability) — đây chính là những gì Cosmos SDK cho phép. Điều này có nghĩa là các nhà phát triển có thể cắm vào và hoán đổi, làm cho blockchain của họ có thể tùy chỉnh bằng cách sử dụng phần mềm khác nhau cho các lớp khác nhau. Ví dụ, bạn có thể chọn xây dựng một chuỗi cơ bản và sử dụng Cosmos SDK với CometBFT. CometBFT sẽ là lớp đồng thuận và bản thân chuỗi sẽ là lớp thanh toán và thực thi. Một hướng khác có thể là sử dụng SDK với Rollkit và Celestia làm lớp đồng thuận và khả dụng dữ liệu. Lợi ích của tính module là bạn có thể tùy chỉnh chuỗi theo trường hợp sử dụng cụ thể của mình.

## Tại Sao Chọn Cosmos SDK

Cosmos SDK là framework tiên tiến nhất để xây dựng các blockchain dành riêng cho ứng dụng theo dạng module tùy chỉnh hiện nay. Dưới đây là một vài lý do tại sao bạn có thể muốn xem xét xây dựng ứng dụng phi tập trung với Cosmos SDK:

* Nó cho phép bạn cắm vào và hoán đổi, tùy chỉnh lớp đồng thuận. Như đã đề cập ở trên, bạn có thể sử dụng Rollkit và Celestia làm lớp đồng thuận và khả dụng dữ liệu. Điều này mang lại rất nhiều sự linh hoạt và khả năng tùy chỉnh.
* Trước đây, consensus engine mặc định trong Cosmos SDK là [CometBFT](https://github.com/cometbft/cometbft). CometBFT là consensus engine BFT trưởng thành nhất hiện tại. Nó được sử dụng rộng rãi trong ngành và được coi là tiêu chuẩn vàng cho việc xây dựng các hệ thống Proof-of-Stake.
* Cosmos SDK là mã nguồn mở và được thiết kế để dễ dàng xây dựng blockchain từ các [module](../../build/modules) có thể kết hợp. Khi hệ sinh thái các module Cosmos SDK mã nguồn mở phát triển, việc xây dựng các nền tảng phi tập trung phức tạp sẽ ngày càng dễ dàng hơn.
* Cosmos SDK được lấy cảm hứng từ bảo mật dựa trên capabilities, và được định hướng bởi nhiều năm làm việc với các state-machine blockchain. Điều này làm cho Cosmos SDK trở thành môi trường rất an toàn để xây dựng blockchain.
* Quan trọng nhất, Cosmos SDK đã được sử dụng để xây dựng nhiều blockchain dành riêng cho ứng dụng đang hoạt động trong thực tế. Trong số đó, có thể kể đến [Cosmos Hub](https://hub.cosmos.network), [IRIS Hub](https://irisnet.org), [Binance Chain](https://docs.binance.org/), [Terra](https://terra.money/) hoặc [Kava](https://www.kava.io/). [Nhiều blockchain khác](https://cosmos.network/ecosystem) đang được xây dựng trên Cosmos SDK.

## Bắt Đầu Với Cosmos SDK

* Tìm hiểu thêm về [kiến trúc của một ứng dụng Cosmos SDK](./02-sdk-app-architecture.md)
* Tìm hiểu cách xây dựng blockchain dành riêng cho ứng dụng từ đầu với [Cosmos SDK Tutorial](https://cosmos.network/docs/tutorial)
