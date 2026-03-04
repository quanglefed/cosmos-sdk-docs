---
sidebar_position: 1
---

# Vote Extensions (Mở Rộng Bỏ Phiếu)

:::note Tóm tắt
Phần này mô tả cách ứng dụng có thể định nghĩa và sử dụng vote extension được xác định trong ABCI++.
:::

## Extend Vote (Mở Rộng Phiếu Bầu)

ABCI++ cho phép ứng dụng mở rộng một phiếu pre-commit với dữ liệu tùy ý. Quá trình này KHÔNG cần phải có tính xác định (deterministic), và dữ liệu trả về có thể là duy nhất đối với từng tiến trình validator. Cosmos SDK định nghĩa `baseapp.ExtendVoteHandler`:

```go
type ExtendVoteHandler func(Context, *abci.ExtendVoteRequest) (*abci.ExtendVoteResponse, error)
```

Ứng dụng có thể đặt handler này trong `app.go` thông qua hàm tùy chọn `baseapp.SetExtendVoteHandler` của `BaseApp`. `sdk.ExtendVoteHandler`, nếu được định nghĩa, sẽ được gọi trong quá trình thực thi phương thức ABCI `ExtendVote`. Lưu ý, nếu ứng dụng quyết định triển khai `baseapp.ExtendVoteHandler`, nó PHẢI trả về một `VoteExtension` khác null. Tuy nhiên, vote extension có thể rỗng. Xem [tại đây](https://github.com/cometbft/cometbft/blob/v0.38.0-rc1/spec/abci/abci++_methods.md#extendvote) để biết thêm chi tiết.

Có nhiều trường hợp sử dụng phi tập trung chống kiểm duyệt cho vote extension. Ví dụ, một validator có thể muốn gửi giá cho một oracle giá hoặc các phần chia sẻ mã hóa cho một mempool giao dịch được mã hóa. Lưu ý, ứng dụng nên cẩn thận xem xét kích thước của vote extension vì chúng có thể làm tăng độ trễ trong sản xuất block. Xem [tại đây](https://github.com/cometbft/cometbft/blob/v0.38.0-rc1/docs/qa/CometBFT-QA-38.md#vote-extensions-testbed) để biết thêm chi tiết.

## Verify Vote Extension (Xác Minh Mở Rộng Phiếu Bầu)

Tương tự như việc mở rộng phiếu bầu, ứng dụng cũng có thể xác minh vote extension từ các validator khác khi xác thực pre-commit của họ. Đối với một vote extension nhất định, quá trình này PHẢI có tính xác định (deterministic). Cosmos SDK định nghĩa `sdk.VerifyVoteExtensionHandler`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/types/abci.go#L26-L27
```

Ứng dụng có thể đặt handler này trong `app.go` thông qua hàm tùy chọn `baseapp.SetVerifyVoteExtensionHandler` của `BaseApp`. `sdk.VerifyVoteExtensionHandler`, nếu được định nghĩa, sẽ được gọi trong quá trình thực thi phương thức ABCI `VerifyVoteExtension`. Nếu ứng dụng định nghĩa một handler vote extension, nó cũng nên định nghĩa một handler xác minh. Lưu ý, không phải tất cả validator đều có cùng quan điểm về những vote extension mà họ xác minh, tùy thuộc vào cách các phiếu bầu được lan truyền. Xem [tại đây](https://github.com/cometbft/cometbft/blob/v0.38.0-rc1/spec/abci/abci++_methods.md#verifyvoteextension) để biết thêm chi tiết.

## Lan Truyền Vote Extension

Các vote extension đã được thống nhất tại chiều cao `H` được cung cấp cho validator đề xuất tại chiều cao `H+1` trong quá trình `PrepareProposal`. Do đó, các vote extension không được cung cấp hoặc hiển thị gốc cho các validator còn lại trong quá trình `ProcessProposal`. Vì vậy, nếu ứng dụng yêu cầu các vote extension đã được thống nhất từ chiều cao `H` phải khả dụng cho tất cả validator tại `H+1`, ứng dụng phải lan truyền các vote extension này thủ công trong chính đề xuất block. Điều này có thể được thực hiện bằng cách "đưa vào" (inject) chúng vào đề xuất block, vì trường `Txs` trong `PrepareProposal` chỉ là một slice của các byte slice.

`FinalizeBlock` sẽ bỏ qua bất kỳ byte slice nào không triển khai `sdk.Tx`, vì vậy bất kỳ vote extension nào được đưa vào sẽ được bỏ qua một cách an toàn trong `FinalizeBlock`. Để biết thêm chi tiết về việc lan truyền, hãy xem [ABCI++ 2.0 ADR](https://github.com/cosmos/cosmos-sdk/blob/main/docs/architecture/adr-064-abci-2.0.md#vote-extension-propagation--verification).

### Khôi Phục Vote Extension Được Đưa Vào

Như đã đề cập trước đó, các vote extension có thể được đưa vào một đề xuất block (cùng với các giao dịch khác trong trường `Txs`). Cosmos SDK cung cấp một hook pre-FinalizeBlock để cho phép ứng dụng khôi phục vote extension, thực hiện bất kỳ phép tính cần thiết nào trên chúng, và sau đó lưu kết quả vào cached store. Những kết quả này sẽ khả dụng cho ứng dụng trong lần gọi `FinalizeBlock` tiếp theo.

Một ví dụ về cách một hook pre-FinalizeBlock có thể trông như thế nào được hiển thị dưới đây:

```go
app.SetPreBlocker(func(ctx sdk.Context, req *abci.FinalizeBlockRequest) error {
    allVEs := []VE{} // lưu tất cả vote extension đã phân tích ở đây
    for _, tx := range req.Txs {
        // định nghĩa một hàm tùy chỉnh cố gắng phân tích tx như một vote extension
        ve, ok := parseVoteExtension(tx)
        if !ok {
            continue
        }

        allVEs = append(allVEs, ve)
    }

    // thực hiện bất kỳ phép tính cần thiết nào trên vote extension và lưu kết quả
    // vào cached store
    result := compute(allVEs)
    err := storeVEResult(ctx, result)
    if err != nil {
        return err
    }

    return nil
})

```

Sau đó, trong một module của ứng dụng, ứng dụng có thể lấy kết quả của phép tính trên vote extension từ cached store:

```go
func (k Keeper) BeginBlocker(ctx context.Context) error {
    // lấy kết quả của phép tính trên vote extension từ cached store
    result, err := k.GetVEResult(ctx)
    if err != nil {
        return err
    }

    // sử dụng kết quả của phép tính trên vote extension
    k.setSomething(result)

    return nil
}
```
