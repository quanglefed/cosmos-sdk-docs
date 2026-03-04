---
sidebar_position: 1
---

# Kiến Trúc Blockchain

## State machine

Về cốt lõi, một blockchain là một [state machine xác định được sao chép](https://en.wikipedia.org/wiki/State_machine_replication).

State machine là một khái niệm khoa học máy tính trong đó một máy có thể có nhiều trạng thái, nhưng chỉ một trạng thái tại bất kỳ thời điểm nào. Có một `state` (trạng thái) mô tả trạng thái hiện tại của hệ thống, và `transactions` (giao dịch) kích hoạt các chuyển đổi trạng thái.

Cho trạng thái S và giao dịch T, state machine sẽ trả về trạng thái mới S'.

```text
+--------+                 +--------+
|        |                 |        |
|   S    +---------------->+   S'   |
|        |    apply(T)     |        |
+--------+                 +--------+
```

Trong thực tế, các giao dịch được đóng gói trong các block để làm cho quá trình hiệu quả hơn. Cho trạng thái S và một block giao dịch B, state machine sẽ trả về trạng thái mới S'.

```text
+--------+                              +--------+
|        |                              |        |
|   S    +----------------------------> |   S'   |
|        |   For each T in B: apply(T)  |        |
+--------+                              +--------+
```

Trong ngữ cảnh blockchain, state machine là xác định (deterministic). Điều này có nghĩa là nếu một node được khởi động ở một trạng thái nhất định và phát lại cùng một chuỗi giao dịch, nó sẽ luôn kết thúc với cùng trạng thái cuối cùng.

Cosmos SDK cho phép các nhà phát triển tối đa sự linh hoạt để định nghĩa trạng thái của ứng dụng, các loại giao dịch và các hàm chuyển đổi trạng thái. Quá trình xây dựng state-machine với Cosmos SDK sẽ được mô tả chi tiết hơn trong các phần tiếp theo. Nhưng trước tiên, hãy xem cách state-machine được sao chép bằng **CometBFT**.

## CometBFT

Nhờ Cosmos SDK, các nhà phát triển chỉ cần định nghĩa state machine, và [*CometBFT*](https://docs.cometbft.com/v0.37/introduction/what-is-cometbft) sẽ xử lý việc sao chép qua mạng lưới cho họ.

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

[CometBFT](https://docs.cometbft.com/v0.37/introduction/what-is-cometbft) là một engine không phụ thuộc vào ứng dụng, chịu trách nhiệm xử lý các lớp *networking* (mạng lưới) và *consensus* (đồng thuận) của một blockchain. Trong thực tế, điều này có nghĩa là CometBFT chịu trách nhiệm truyền bá và sắp xếp thứ tự các byte giao dịch. CometBFT dựa vào thuật toán Byzantine-Fault-Tolerant (BFT) cùng tên để đạt đồng thuận về thứ tự giao dịch.

[Thuật toán đồng thuận](https://docs.cometbft.com/v0.37/introduction/what-is-cometbft#consensus-overview) của CometBFT hoạt động với một tập hợp các node đặc biệt gọi là *Validator*. Validator có trách nhiệm thêm các block giao dịch vào blockchain. Tại bất kỳ block nào, có một tập hợp validator V. Một validator trong V được thuật toán chọn làm proposer (người đề xuất) của block tiếp theo. Block này được coi là hợp lệ nếu hơn hai phần ba V ký một `prevote` và một `precommit` trên nó, và nếu tất cả các giao dịch chứa trong đó đều hợp lệ. Tập hợp validator có thể được thay đổi bởi các quy tắc được viết trong state-machine.

## ABCI

CometBFT truyền các giao dịch đến ứng dụng thông qua một interface gọi là [ABCI](https://docs.cometbft.com/v0.37/spec/abci/), mà ứng dụng phải triển khai.

```text
              +---------------------+
              |                     |
              |     Application     |
              |                     |
              +--------+---+--------+
                       ^   |
                       |   | ABCI
                       |   v
              +--------+---+--------+
              |                     |
              |                     |
              |       CometBFT      |
              |                     |
              |                     |
              +---------------------+
```

Lưu ý rằng **CometBFT chỉ xử lý các byte giao dịch**. Nó không có kiến thức về ý nghĩa của các byte này. Tất cả những gì CometBFT làm là sắp xếp các byte giao dịch này theo thứ tự xác định. CometBFT truyền các byte đến ứng dụng qua ABCI và mong đợi một mã trả về để thông báo liệu các message chứa trong giao dịch có được xử lý thành công hay không.

Dưới đây là các message quan trọng nhất của ABCI:

* `CheckTx`: Khi một giao dịch được CometBFT nhận, nó được truyền đến ứng dụng để kiểm tra xem một số yêu cầu cơ bản có được đáp ứng không. `CheckTx` được dùng để bảo vệ mempool của các full-node chống lại các giao dịch spam. Một handler đặc biệt gọi là [`AnteHandler`](../beginner/04-gas-fees.md#antehandler) được dùng để thực thi một loạt các bước xác thực như kiểm tra phí đủ và xác minh chữ ký. Nếu các kiểm tra hợp lệ, giao dịch được thêm vào [mempool](https://docs.cometbft.com/v0.37/spec/p2p/legacy-docs/messages/mempool) và chuyển tiếp đến các peer node. Lưu ý rằng các giao dịch không được xử lý (tức là không có sửa đổi trạng thái nào xảy ra) với `CheckTx` vì chúng chưa được đưa vào một block.
* `DeliverTx`: Khi một [block hợp lệ](https://docs.cometbft.com/v0.37/spec/core/data_structures#block) được CometBFT nhận, mỗi giao dịch trong block được truyền đến ứng dụng qua `DeliverTx` để được xử lý. Chính trong giai đoạn này các chuyển đổi trạng thái xảy ra. `AnteHandler` thực thi lại, cùng với [`Msg` service](../../build/building-modules/03-msg-services.md) RPC thực tế cho mỗi message trong giao dịch.
* `BeginBlock`/`EndBlock`: Các message này được thực thi ở đầu và cuối mỗi block, dù block có chứa giao dịch hay không. Nó hữu ích để kích hoạt thực thi logic tự động. Tuy nhiên cần thận trọng, vì các vòng lặp tốn kém về mặt tính toán có thể làm chậm blockchain của bạn, hoặc thậm chí đóng băng nó nếu vòng lặp là vô hạn.

Xem chi tiết hơn về các phương thức ABCI từ [tài liệu CometBFT](https://docs.cometbft.com/v0.37/spec/abci/).

Bất kỳ ứng dụng nào được xây dựng trên CometBFT đều cần triển khai interface ABCI để giao tiếp với CometBFT engine cục bộ bên dưới. May mắn thay, bạn không cần phải tự triển khai interface ABCI. Cosmos SDK cung cấp một triển khai boilerplate của nó dưới dạng [baseapp](./03-sdk-design.md#baseapp).
