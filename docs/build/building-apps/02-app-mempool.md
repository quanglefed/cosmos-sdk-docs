---
sidebar_position: 1
---

# Application Mempool

:::note Tóm tắt
Phần này mô tả cách mempool phía ứng dụng có thể được sử dụng và thay thế.
:::

Kể từ `v0.47`, ứng dụng có mempool riêng để cho phép xây dựng block chi tiết hơn nhiều so với các phiên bản trước. Thay đổi này được kích hoạt bởi [ABCI 1.0](https://github.com/cometbft/cometbft/blob/v0.37.0/spec/abci). Đáng chú ý là nó giới thiệu các bước `PrepareProposal` và `ProcessProposal` của ABCI++.

:::note Đọc Trước

* [BaseApp](../../learn/advanced/00-baseapp.md)
* [ABCI](../abci/00-introduction.md)

:::

## Mempool

Có vô số thiết kế mà nhà phát triển ứng dụng có thể viết cho mempool, SDK chỉ cung cấp các triển khai mempool đơn giản. Cụ thể, SDK cung cấp các mempool sau:

* [No-op Mempool](#no-op-mempool)
* [Sender Nonce Mempool](#sender-nonce-mempool)
* [Priority Nonce Mempool](#priority-nonce-mempool)

Theo mặc định, SDK sử dụng [No-op Mempool](#no-op-mempool), nhưng có thể được thay thế bởi nhà phát triển ứng dụng trong [`app.go`](./01-app-go-di.md):

```go
nonceMempool := mempool.NewSenderNonceMempool()
mempoolOpt   := baseapp.SetMempool(nonceMempool)
baseAppOptions = append(baseAppOptions, mempoolOpt)
```

### No-op Mempool

No-op mempool là mempool mà các giao dịch bị loại bỏ và bỏ qua hoàn toàn khi BaseApp tương tác với mempool. Khi mempool này được sử dụng, người ta giả định rằng ứng dụng sẽ dựa vào thứ tự giao dịch của CometBFT được định nghĩa trong `RequestPrepareProposal`, vốn được sắp xếp theo FIFO theo mặc định.

> Lưu ý: Nếu sử dụng NoOp mempool, cả PrepareProposal và ProcessProposal đều nên nhận thức được điều này vì PrepareProposal có thể bao gồm các giao dịch có thể thất bại khi xác minh trong ProcessProposal.

### Sender Nonce Mempool

Nonce mempool là mempool giữ các giao dịch từ một người gửi được sắp xếp theo nonce để tránh các vấn đề với nonce. Nó hoạt động bằng cách lưu trữ giao dịch trong một danh sách được sắp xếp theo nonce của giao dịch. Khi proposer yêu cầu các giao dịch để đưa vào block, nó ngẫu nhiên chọn một người gửi và lấy giao dịch đầu tiên trong danh sách. Nó lặp lại điều này cho đến khi mempool rỗng hoặc block đầy.

Có thể cấu hình với các tham số sau:

#### MaxTxs

Là giá trị nguyên đặt mempool ở một trong ba chế độ: *bounded* (giới hạn), *unbounded* (không giới hạn) hoặc *disabled* (vô hiệu hóa).

* **âm**: Vô hiệu hóa, mempool không chèn giao dịch mới và trả về sớm.
* **không**: Mempool không giới hạn: không có giới hạn số giao dịch và sẽ không bao giờ thất bại với `ErrMempoolTxMaxCapacity`.
* **dương**: Giới hạn, nó thất bại với `ErrMempoolTxMaxCapacity` khi giá trị `maxTx` bằng `CountTx()`.

#### Seed

Đặt seed cho bộ tạo số ngẫu nhiên được dùng để chọn giao dịch từ mempool.

### Priority Nonce Mempool

[Priority nonce mempool](https://github.com/cosmos/cosmos-sdk/blob/main/types/mempool/priority_nonce_spec.md) là triển khai mempool lưu trữ các tx trong một tập được sắp xếp một phần theo 2 chiều:

* priority (ưu tiên)
* sender-nonce (sequence number)

Nội bộ nó sử dụng một [skip list](https://pkg.go.dev/github.com/huandu/skiplist) được sắp xếp theo priority và một skip list cho mỗi người gửi được sắp xếp theo sender-nonce (sequence number). Khi có nhiều tx từ cùng một người gửi, chúng không phải lúc nào cũng có thể so sánh theo priority với các tx của người gửi khác và phải được sắp xếp một phần theo cả sender-nonce và priority.

Có thể cấu hình với các tham số sau:

#### MaxTxs

Là giá trị nguyên đặt mempool ở một trong ba chế độ: *bounded* (giới hạn), *unbounded* (không giới hạn) hoặc *disabled* (vô hiệu hóa).

* **âm**: Vô hiệu hóa, mempool không chèn giao dịch mới và trả về sớm.
* **không**: Mempool không giới hạn: không có giới hạn số giao dịch và sẽ không bao giờ thất bại với `ErrMempoolTxMaxCapacity`.
* **dương**: Giới hạn, nó thất bại với `ErrMempoolTxMaxCapacity` khi giá trị `maxTx` bằng `CountTx()`.

#### Callback

Priority nonce mempool cung cấp các tùy chọn mempool cho phép ứng dụng đặt callback.

* **OnRead**: Đặt callback được gọi khi một giao dịch được đọc từ mempool.
* **TxReplacement**: Đặt callback được gọi khi phát hiện nonce giao dịch trùng lặp trong quá trình chèn vào mempool. Ứng dụng có thể định nghĩa quy tắc thay thế giao dịch dựa trên priority của tx hoặc một số trường giao dịch nhất định.

Thông tin thêm về triển khai mempool của SDK có thể tìm thấy trong [godocs](https://pkg.go.dev/github.com/cosmos/cosmos-sdk/types/mempool).
