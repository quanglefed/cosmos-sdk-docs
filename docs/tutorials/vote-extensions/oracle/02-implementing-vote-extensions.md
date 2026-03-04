# Triển Khai Vote Extensions

## Triển Khai ExtendVote

Đầu tiên chúng ta sẽ tạo struct `OracleVoteExtension` — đây là đối tượng sẽ được marshal thành bytes và được validator ký.

Trong ví dụ này chúng ta sẽ dùng JSON để marshal vote extension cho đơn giản, nhưng chúng tôi khuyến nghị tìm một encoding tạo ra đầu ra nhỏ hơn, vì vote extension lớn có thể ảnh hưởng đến hiệu suất của CometBFT. Các encoding tùy chỉnh và byte nén có thể được sử dụng ngay từ đầu.

```go
// OracleVoteExtension định nghĩa cấu trúc vote extension chuẩn.
type OracleVoteExtension struct {
 Height int64
 Prices map[string]math.LegacyDec
}
```

Sau đó chúng ta sẽ tạo một struct `VoteExtensionsHandler` chứa mọi thứ cần thiết để truy vấn giá.

```go
type VoteExtHandler struct {
 logger          log.Logger
 currentBlock    int64                            // chiều cao block hiện tại
 lastPriceSyncTS time.Time                        // lần cuối chúng ta đồng bộ giá
 providerTimeout time.Duration                    // timeout để lấy giá từ các provider
 providers       map[string]Provider              // ánh xạ từ tên provider đến provider (vd: Binance -> BinanceProvider)
 providerPairs   map[string][]keeper.CurrencyPair // ánh xạ từ tên provider đến các cặp được hỗ trợ (vd: Binance -> [ATOM/USD])

 Keeper keeper.Keeper // keeper của module oracle
}
```

Cuối cùng, cũng cần một hàm trả về `sdk.ExtendVoteHandler`, và đây là nơi logic vote extension của chúng ta sẽ tồn tại.

```go
func (h *VoteExtHandler) ExtendVoteHandler() sdk.ExtendVoteHandler {
    return func(ctx sdk.Context, req *abci.RequestExtendVote) (*abci.ResponseExtendVote, error) {
        // ở đây chúng ta có một hàm helper lấy tất cả giá và tính trung bình có trọng số theo khối lượng của mỗi thị trường
        prices := h.getAllVolumeWeightedPrices()

        voteExt := OracleVoteExtension{
            Height: req.Height,
            Prices: prices,
        }
        
        bz, err := json.Marshal(voteExt)
        if err != nil {
            return nil, fmt.Errorf("failed to marshal vote extension: %w", err)
        }

        return &abci.ResponseExtendVote{VoteExtension: bz}, nil
    }
}
```

Như bạn thấy ở trên, việc tạo vote extension khá đơn giản và chúng ta chỉ cần trả về bytes. CometBFT sẽ xử lý việc ký các byte này cho chúng ta. Chúng ta đã bỏ qua quá trình lấy giá nhưng bạn có thể xem ví dụ đầy đủ hơn [tại đây](https://github.com/cosmos/sdk-tutorials/blob/master/tutorials/oracle/base/x/oracle/abci/vote_extensions.go).

Ở đây chúng ta sẽ thực hiện một số kiểm tra đơn giản như:

* Vote extension có được unmarshal đúng không?
* Vote extension có đúng chiều cao không?
* Một số xác thực khác, ví dụ: giá từ extension này có quá lệch so với giá của tôi không? Hoặc có thể các kiểm tra phát hiện hành vi độc hại.

```go
func (h *VoteExtHandler) VerifyVoteExtensionHandler() sdk.VerifyVoteExtensionHandler {
    return func(ctx sdk.Context, req *abci.RequestVerifyVoteExtension) (*abci.ResponseVerifyVoteExtension, error) {
        var voteExt OracleVoteExtension
        err := json.Unmarshal(req.VoteExtension, &voteExt)
        if err != nil {
            return nil, fmt.Errorf("failed to unmarshal vote extension: %w", err)
        }
        
        if voteExt.Height != req.Height {
            return nil, fmt.Errorf("vote extension height does not match request height; expected: %d, got: %d", req.Height, voteExt.Height)
        }

        // Xác minh giá đến từ một validator là hợp lệ. Lưu ý, việc xác minh trong
        // VerifyVoteExtensionHandler PHẢI là xác định. Để ngắn gọn và mục đích demo,
        // chúng ta bỏ qua việc triển khai.
        if err := h.verifyOraclePrices(ctx, voteExt.Prices); err != nil {
            return nil, fmt.Errorf("failed to verify oracle prices from validator %X: %w", req.ValidatorAddress, err)
        }

        return &abci.ResponseVerifyVoteExtension{Status: abci.ResponseVerifyVoteExtension_ACCEPT}, nil
    }
}
```

## Triển Khai PrepareProposal

```go
type ProposalHandler struct {
    logger   log.Logger
    keeper   keeper.Keeper // keeper module oracle của chúng ta
    valStore baseapp.ValidatorStore // để lấy public key của các validator hiện tại
}
```

Và chúng ta tạo struct cho "giao dịch đặc biệt" sẽ chứa các giá và các phiếu bầu để sau này các validator có thể kiểm tra lại trong ProcessProposal rằng họ nhận được cùng kết quả với proposer của block. Với điều này chúng ta cũng có thể kiểm tra xem tất cả phiếu bầu đã được sử dụng hay chưa bằng cách so sánh các phiếu bầu nhận được trong ProcessProposal.

```go
type StakeWeightedPrices struct {
    StakeWeightedPrices map[string]math.LegacyDec
    ExtendedCommitInfo  abci.ExtendedCommitInfo
}
```

Bây giờ chúng ta tạo `PrepareProposalHandler`. Trong bước này, đầu tiên chúng ta sẽ kiểm tra xem chữ ký của các vote extension có đúng không bằng cách sử dụng hàm helper `ValidateVoteExtensions` từ package baseapp.

```go
func (h *ProposalHandler) PrepareProposal() sdk.PrepareProposalHandler {
    return func(ctx sdk.Context, req *abci.RequestPrepareProposal) (*abci.ResponsePrepareProposal, error) {
        err := baseapp.ValidateVoteExtensions(ctx, h.valStore, req.Height, ctx.ChainID(), req.LocalLastCommit)
        if err != nil {
            return nil, err
        }
...
```

Sau đó chúng ta tiến hành thực hiện tính toán chỉ khi chiều cao hiện tại cao hơn chiều cao mà vote extension đã được bật. Nhớ rằng vote extension được cung cấp cho proposer block ở block tiếp theo mà chúng được tạo ra/bật.

```go
...
        proposalTxs := req.Txs

        if req.Height > ctx.ConsensusParams().Abci.VoteExtensionsEnableHeight {
            stakeWeightedPrices, err := h.computeStakeWeightedOraclePrices(ctx, req.LocalLastCommit)
            if err != nil {
                return nil, errors.New("failed to compute stake-weighted oracle prices")
            }

            injectedVoteExtTx := StakeWeightedPrices{
                StakeWeightedPrices: stakeWeightedPrices,
                ExtendedCommitInfo:  req.LocalLastCommit,
            }
...
```

Cuối cùng chúng ta tiêm kết quả như một giao dịch ở một vị trí cụ thể, thường ở đầu block.

## Triển Khai ProcessProposal

Bây giờ chúng ta có thể triển khai phương thức mà tất cả validator sẽ thực thi để đảm bảo proposer đang làm đúng công việc của mình.

Ở đây, nếu vote extension được bật, chúng ta sẽ kiểm tra xem tx ở index 0 có phải là injected vote extension không.

```go
func (h *ProposalHandler) ProcessProposal() sdk.ProcessProposalHandler {
    return func(ctx sdk.Context, req *abci.RequestProcessProposal) (*abci.ResponseProcessProposal, error) {
        if req.Height > ctx.ConsensusParams().Abci.VoteExtensionsEnableHeight {
            var injectedVoteExtTx StakeWeightedPrices
            if err := json.Unmarshal(req.Txs[0], &injectedVoteExtTx); err != nil {
                h.logger.Error("failed to decode injected vote extension tx", "err", err)
                return &abci.ResponseProcessProposal{Status: abci.ResponseProcessProposal_REJECT}, nil
            }
...
```

Sau đó chúng ta xác thực lại chữ ký vote extension bằng `baseapp.ValidateVoteExtensions`, tính toán lại kết quả (giống như trong PrepareProposal) và so sánh với kết quả chúng ta nhận được từ injected tx.

```go
            err := baseapp.ValidateVoteExtensions(ctx, h.valStore, req.Height, ctx.ChainID(), injectedVoteExtTx.ExtendedCommitInfo)
            if err != nil {
                return nil, err
            }

            // Xác minh giá oracle có trọng số theo stake của proposer bằng cách tính
            // cùng phép tính và so sánh kết quả. Chúng ta bỏ qua xác minh để ngắn gọn
            // và mục đích demo.
            stakeWeightedPrices, err := h.computeStakeWeightedOraclePrices(ctx, injectedVoteExtTx.ExtendedCommitInfo)
            if err != nil {
                return &abci.ResponseProcessProposal{Status: abci.ResponseProcessProposal_REJECT}, nil
            }
            
            if err := compareOraclePrices(injectedVoteExtTx.StakeWeightedPrices, stakeWeightedPrices); err != nil {
                return &abci.ResponseProcessProposal{Status: abci.ResponseProcessProposal_REJECT}, nil
            }
        }

        return &abci.ResponseProcessProposal{Status: abci.ResponseProcessProposal_ACCEPT}, nil
    }
}
```

Quan trọng: Trong ví dụ này chúng ta đã tránh sử dụng mempool và các khái niệm cơ bản khác, vui lòng tham khảo `DefaultProposalHandler` để có triển khai đầy đủ: [https://github.com/cosmos/cosmos-sdk/blob/v0.50.1/baseapp/abci_utils.go](https://github.com/cosmos/cosmos-sdk/blob/v0.50.1/baseapp/abci_utils.go)

## Triển Khai PreBlocker

Bây giờ các validator đang mở rộng phiếu bầu của họ, xác minh các phiếu bầu khác và đưa kết quả vào block. Nhưng làm thế nào chúng ta thực sự sử dụng kết quả này? Điều này được thực hiện trong PreBlocker — code chạy trước bất kỳ code nào khác trong FinalizeBlock để chúng ta đảm bảo cung cấp thông tin này cho chain và các module trong suốt quá trình thực thi block (từ BeginBlock).

Ở bước này chúng ta biết rằng injected tx có định dạng tốt và đã được xác minh bởi các validator tham gia đồng thuận, vì vậy việc sử dụng nó rất đơn giản. Chỉ cần kiểm tra xem vote extension có được bật không, lấy giao dịch đầu tiên và sử dụng một phương thức trong keeper của module để đặt kết quả.

```go
func (h *ProposalHandler) PreBlocker(ctx sdk.Context, req *abci.RequestFinalizeBlock) (*sdk.ResponsePreBlock, error) {
    res := &sdk.ResponsePreBlock{}
    if len(req.Txs) == 0 {
        return res, nil
    }

    if req.Height > ctx.ConsensusParams().Abci.VoteExtensionsEnableHeight {
        var injectedVoteExtTx StakeWeightedPrices
        if err := json.Unmarshal(req.Txs[0], &injectedVoteExtTx); err != nil {
            h.logger.Error("failed to decode injected vote extension tx", "err", err)
            return nil, err
        }

        // đặt oracle prices sử dụng context được truyền vào, điều này sẽ làm cho các giá này có sẵn trong block hiện tại
        if err := h.keeper.SetOraclePrices(ctx, injectedVoteExtTx.StakeWeightedPrices); err != nil {
            return nil, err
        }
    }
    return res, nil
}

```

## Kết Luận

Trong tutorial này, chúng ta đã tạo một module price oracle đơn giản tích hợp vote extension. Chúng ta đã thấy cách triển khai `ExtendVote`, `VerifyVoteExtension`, `PrepareProposal`, `ProcessProposal` và `PreBlocker` để xử lý quá trình bỏ phiếu và xác minh vote extension, cũng như cách sử dụng kết quả trong quá trình thực thi block.
