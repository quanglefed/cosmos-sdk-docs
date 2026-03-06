# ADR 17: Module Header Lịch Sử

## Changelog

* 26 tháng 11 năm 2019: Bắt đầu phiên bản đầu tiên
* 2 tháng 12 năm 2019: Bản nháp cuối cùng của phiên bản đầu tiên

## Bối Cảnh

Để Cosmos SDK triển khai [đặc tả IBC](https://github.com/cosmos/ibc), các module trong Cosmos SDK phải có khả năng introspect các consensus state gần đây (validator set và commitment root) vì bằng chứng của các giá trị này trên các chain khác phải được kiểm tra trong quá trình handshake.

## Quyết Định

Ứng dụng PHẢI lưu trữ `n` header gần nhất trong một persistent store. Lúc đầu, store này CÓ THỂ là Merklised store hiện tại. Một non-Merklised store CÓ THỂ được sử dụng sau này vì không cần bằng chứng.

Ứng dụng PHẢI lưu trữ thông tin này bằng cách lưu trữ header mới ngay lập tức khi xử lý `abci.RequestBeginBlock`:

```go
func BeginBlock(ctx sdk.Context, keeper HistoricalHeaderKeeper, req abci.RequestBeginBlock) abci.ResponseBeginBlock {
  info := HistoricalInfo{
    Header: ctx.BlockHeader(),
    ValSet: keeper.StakingKeeper.GetAllValidators(ctx), // lưu ý rằng điều này phải được lưu trữ theo thứ tự canonical
  }
  keeper.SetHistoricalInfo(ctx, ctx.BlockHeight(), info)
  n := keeper.GetParamRecentHeadersToStore()
  keeper.PruneHistoricalInfo(ctx, ctx.BlockHeight() - n)
  // tiếp tục xử lý request
}
```

Ngoài ra, ứng dụng CÓ THỂ chỉ lưu trữ hash của validator set.

Ứng dụng PHẢI làm cho `n` header đã cam kết trong quá khứ này có thể truy vấn được bởi các module Cosmos SDK thông qua hàm `GetHistoricalInfo` của `Keeper`. Điều này CÓ THỂ được triển khai trong một module mới, hoặc cũng CÓ THỂ được tích hợp vào một module hiện có (có thể là `x/staking` hoặc `x/ibc`).

`n` CÓ THỂ được cấu hình như một tham số trong param store, trong trường hợp đó nó có thể được thay đổi bởi `ParameterChangeProposal`, mặc dù sẽ mất một số block để thông tin được lưu trữ bắt kịp nếu `n` được tăng lên.

## Trạng Thái

Đề Xuất.

## Hậu Quả

Triển khai ADR này sẽ yêu cầu thay đổi Cosmos SDK. Nó sẽ không yêu cầu thay đổi Tendermint.

### Tích Cực

* Dễ dàng truy xuất header và state root cho các chiều cao trong quá khứ gần đây bởi các module bất kỳ trong Cosmos SDK.
* Không cần RPC call đến Tendermint.
* Không cần thay đổi ABCI.

### Tiêu Cực

* Sao chép dữ liệu `n` header trong Tendermint và ứng dụng (sử dụng đĩa bổ sung) — về lâu dài, một cách tiếp cận như [thế này](https://github.com/tendermint/tendermint/issues/4210) có thể được ưu tiên hơn.

### Trung Lập

(không có gì đã biết)

## Tài Liệu Tham Khảo

* [ICS 2: "Introspection trạng thái consensus"](https://github.com/cosmos/ibc/tree/master/spec/core/ics-002-client-semantics#consensus-state-introspection)
