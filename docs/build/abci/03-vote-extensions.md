# Vote Extensions

:::note Tóm tắt
Phần này mô tả cách ứng dụng có thể định nghĩa và sử dụng vote extension được định nghĩa trong ABCI++.
:::

## Mở Rộng Phiếu Bầu (Extend Vote)

ABCI 2.0 (thường được gọi là ABCI++) cho phép ứng dụng mở rộng một pre-commit vote với dữ liệu tùy ý. Quá trình này KHÔNG cần phải xác định, và dữ liệu được trả về có thể là duy nhất đối với tiến trình validator. Cosmos SDK định nghĩa [`baseapp.ExtendVoteHandler`](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/types/abci.go#L32):

```go
type ExtendVoteHandler func(Context, *abci.ExtendVoteRequest) (*abci.ExtendVoteResponse, error)
```

Ứng dụng có thể đặt handler này trong `app.go` qua hàm tùy chọn `baseapp.SetExtendVoteHandler` của `BaseApp`. `sdk.ExtendVoteHandler`, nếu được định nghĩa, được gọi trong phương thức ABCI `ExtendVote`. Lưu ý, nếu ứng dụng quyết định triển khai `baseapp.ExtendVoteHandler`, nó PHẢI trả về một `VoteExtension` không nil. Tuy nhiên, vote extension có thể rỗng. Xem [tại đây](https://github.com/cometbft/cometbft/blob/v0.38.0-rc1/spec/abci/abci++_methods.md#extendvote) để biết thêm chi tiết.

Có nhiều use-case phi tập trung chống kiểm duyệt cho vote extension. Ví dụ, một validator có thể muốn gửi giá cho một price oracle hoặc các encryption share cho một encrypted transaction mempool. Lưu ý, ứng dụng nên cẩn thận xem xét kích thước của vote extension vì chúng có thể tăng độ trễ trong việc tạo block. Xem [tại đây](https://github.com/cometbft/cometbft/blob/v0.38.0-rc1/docs/qa/CometBFT-QA-38.md#vote-extensions-testbed) để biết thêm chi tiết.

Nhấp [tại đây](https://docs.cosmos.network/main/build/abci/vote-extensions) nếu bạn muốn xem hướng dẫn cách triển khai vote extension.

## Xác Minh Vote Extension (Verify Vote Extension)

Tương tự như việc mở rộng phiếu bầu, ứng dụng cũng có thể xác minh vote extension từ các validator khác khi xác thực pre-commit của họ. Đối với một vote extension nhất định, quá trình này PHẢI là xác định. Cosmos SDK định nghĩa [`sdk.VerifyVoteExtensionHandler`](https://github.com/cosmos/cosmos-sdk/blob/v0.50.1/types/abci.go#L29-L31):

```go
type VerifyVoteExtensionHandler func(Context, *abci.VerifyVoteExtensionRequest) (*abci.VerifyVoteExtensionResponse, error)
```

Ứng dụng có thể đặt handler này trong `app.go` qua hàm tùy chọn `baseapp.SetVerifyVoteExtensionHandler` của `BaseApp`. `sdk.VerifyVoteExtensionHandler`, nếu được định nghĩa, được gọi trong phương thức ABCI `VerifyVoteExtension`. Nếu ứng dụng định nghĩa một vote extension handler, nó cũng nên định nghĩa một verification handler. Lưu ý, không phải tất cả validator sẽ có cùng góc nhìn về những vote extension nào họ xác minh tùy thuộc vào cách các phiếu bầu được truyền bá. Xem [tại đây](https://github.com/cometbft/cometbft/blob/v0.38.0-rc1/spec/abci/abci++_methods.md#verifyvoteextension) để biết thêm chi tiết.

Ngoài ra, lưu ý rằng hiệu suất có thể bị ảnh hưởng nếu vote extension quá lớn (https://docs.cometbft.com/v0.38/qa/cometbft-qa-38#vote-extensions-testbed), vì vậy chúng tôi khuyến nghị mạnh mẽ việc xác thực kích thước trong `VerifyVoteExtensions`.

## Truyền Bá Vote Extension

Các vote extension đã được đồng thuận ở chiều cao `H` được cung cấp cho validator đề xuất ở chiều cao `H+1` trong `PrepareProposal`. Kết quả là, vote extension không được cung cấp hoặc expose tự nhiên cho các validator còn lại trong `ProcessProposal`. Do đó, nếu ứng dụng yêu cầu rằng các vote extension đã được đồng thuận từ chiều cao `H` có sẵn cho tất cả validator ở `H+1`, ứng dụng phải truyền bá các vote extension này thủ công trong chính block proposal. Điều này có thể được thực hiện bằng cách "tiêm" chúng vào block proposal, vì trường `Txs` trong `PrepareProposal` chỉ là một slice của các byte slice.

`FinalizeBlock` sẽ bỏ qua bất kỳ byte slice nào không triển khai `sdk.Tx`, vì vậy bất kỳ vote extension được tiêm nào sẽ được bỏ qua an toàn trong `FinalizeBlock`. Để biết thêm chi tiết về truyền bá, xem [ABCI++ 2.0 ADR](https://github.com/cosmos/cosmos-sdk/blob/main/docs/architecture/adr-064-abci-2.0.md#vote-extension-propagation--verification).

### Khôi Phục Vote Extension Được Tiêm

Như đã nêu trước đây, vote extension có thể được tiêm vào một block proposal (cùng với các giao dịch khác trong trường `Txs`). Cosmos SDK cung cấp một hook pre-FinalizeBlock để cho phép ứng dụng khôi phục vote extension, thực hiện bất kỳ tính toán cần thiết nào trên chúng, và sau đó lưu trữ kết quả trong cached store. Các kết quả này sẽ có sẵn cho ứng dụng trong lần gọi `FinalizeBlock` tiếp theo.

Một ví dụ về cách một hook pre-FinalizeBlock có thể trông như thế này được hiển thị bên dưới:

```go
app.SetPreBlocker(func(ctx sdk.Context, req *abci.FinalizeBlockRequest) error {
    allVEs := []VE{} // lưu trữ tất cả vote extension đã phân tích ở đây
    for _, tx := range req.Txs {
        // định nghĩa một hàm tùy chỉnh cố gắng phân tích tx như một vote extension
        ve, ok := parseVoteExtension(tx)
        if !ok {
            continue
        }

        allVEs = append(allVEs, ve)
    }

    // thực hiện bất kỳ tính toán cần thiết nào trên vote extension và lưu trữ kết quả
    // trong cached store
    result := compute(allVEs)
    err := storeVEResult(ctx, result)
    if err != nil {
        return err
    }

    return nil
})
```

Sau đó, trong một module của ứng dụng, ứng dụng có thể lấy kết quả của việc tính toán vote extension từ cached store:

```go
func (k Keeper) BeginBlocker(ctx context.Context) error {
    // lấy kết quả của việc tính toán vote extension từ cached store
    result, err := k.GetVEResult(ctx)
    if err != nil {
        return err
    }

    // sử dụng kết quả của việc tính toán vote extension
    k.setSomething(result)

    return nil
}
```
