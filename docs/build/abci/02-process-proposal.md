# Process Proposal

`ProcessProposal` xử lý việc xác thực một đề xuất từ `PrepareProposal`, trong đó cũng bao gồm tiêu đề block (block header). Sau khi một block được đề xuất, các validator còn lại có quyền chấp nhận hoặc từ chối block đó. Validator trong triển khai mặc định của `PrepareProposal` thực hiện các kiểm tra hợp lệ cơ bản trên từng giao dịch.

Lưu ý, `ProcessProposal` PHẢI có tính xác định (deterministic). Các hành vi không xác định sẽ gây ra sự không khớp apphash. Điều này có nghĩa là nếu `ProcessProposal` bị panic hoặc thất bại và chúng ta từ chối, tất cả các tiến trình validator trung thực đều phải từ chối (tức là prevote nil). Nếu vậy, CometBFT sẽ bắt đầu một vòng mới với một đề xuất block mới và chu kỳ tương tự sẽ lặp lại với `PrepareProposal` và `ProcessProposal` cho đề xuất mới.

Dưới đây là triển khai mặc định:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/baseapp/abci_utils.go#L219-L226
```

Tương tự như `PrepareProposal`, triển khai này là mặc định và có thể được sửa đổi bởi nhà phát triển ứng dụng trong [`app_di.go`](../building-apps/01-app-go-di.md). Nếu bạn quyết định tự triển khai handler `ProcessProposal` của riêng mình, bạn phải đảm bảo rằng các giao dịch được cung cấp trong đề xuất KHÔNG vượt quá gas block tối đa và `maxtxbytes` (nếu được đặt).

```go
processOpt := func(app *baseapp.BaseApp) {
    abciPropHandler := baseapp.NewDefaultProposalHandler(mempool, app)
    app.SetProcessProposal(abciPropHandler.ProcessProposalHandler())
}

baseAppOptions = append(baseAppOptions, processOpt)
```
