# Prepare Proposal

`PrepareProposal` xử lý việc xây dựng block, có nghĩa là khi một proposer chuẩn bị đề xuất một block, nó yêu cầu ứng dụng đánh giá một `RequestPrepareProposal`, trong đó chứa một loạt giao dịch từ mempool của CometBFT. Tại thời điểm này, ứng dụng có toàn quyền kiểm soát đề xuất. Ứng dụng có thể sửa đổi, xóa và đưa thêm các giao dịch từ mempool phía ứng dụng của mình vào đề xuất, hoặc thậm chí bỏ qua hoàn toàn tất cả các giao dịch. Những gì ứng dụng làm với các giao dịch được `RequestPrepareProposal` cung cấp không ảnh hưởng đến mempool của CometBFT.

Lưu ý rằng ứng dụng định nghĩa ngữ nghĩa của `PrepareProposal` và nó CÓ THỂ không xác định (non-deterministic), chỉ được thực thi bởi proposer của block hiện tại.

Đọc lại câu trên thấy đề cập đến mempool hai lần, hãy làm rõ hơn. CometBFT có một mempool để gossip (lan truyền) các giao dịch đến các node khác trong mạng. Thứ tự của các giao dịch này được quyết định bởi mempool của CometBFT, sử dụng FIFO (vào trước ra trước) làm cơ chế sắp xếp duy nhất. Đáng chú ý là priority mempool trong Comet đã bị gỡ bỏ hoặc không còn được hỗ trợ. Tuy nhiên, vì ứng dụng có khả năng kiểm tra toàn bộ giao dịch, nó có thể cung cấp khả năng kiểm soát tốt hơn về thứ tự giao dịch. Việc cho phép ứng dụng xử lý việc sắp xếp thứ tự giúp ứng dụng tự định nghĩa cách nó muốn block được xây dựng.

Cosmos SDK định nghĩa kiểu `DefaultProposalHandler`, cung cấp cho ứng dụng các handler `PrepareProposal` và `ProcessProposal`. Nếu bạn quyết định tự triển khai handler `PrepareProposal` của riêng mình, bạn phải đảm bảo rằng các giao dịch được chọn KHÔNG vượt quá gas block tối đa (nếu được đặt) và số byte tối đa được cung cấp bởi `req.MaxBytes`.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/baseapp/abci_utils.go
```

Triển khai mặc định này có thể được ghi đè bởi nhà phát triển ứng dụng trong [`app_di.go`](../building-apps/01-app-go-di.md):

```go
prepareOpt := func(app *baseapp.BaseApp) {
    abciPropHandler := baseapp.NewDefaultProposalHandler(mempool, app)
    app.SetPrepareProposal(abciPropHandler.PrepareProposalHandler())
}

baseAppOptions = append(baseAppOptions, prepareOpt)
```
