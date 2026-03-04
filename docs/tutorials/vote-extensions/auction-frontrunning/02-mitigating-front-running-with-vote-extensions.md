# Giảm Thiểu Front-running Với Vote Extensions

## Mục Lục

* [Điều kiện tiên quyết](#điều-kiện-tiên-quyết)
* [Triển khai Struct cho Vote Extensions](#triển-khai-struct-cho-vote-extensions)
* [Triển khai và Cấu hình Handlers](#triển-khai-và-cấu-hình-handlers)

## Điều Kiện Tiên Quyết

Trước khi triển khai vote extension để giảm thiểu front-running, hãy đảm bảo bạn đã có một module sẵn sàng để triển khai vote extension. Nếu bạn cần tạo hoặc tham khảo một module tương tự, xem `x/auction` để được hướng dẫn.

Trong phần này, chúng ta sẽ thảo luận các bước để giảm thiểu front-running bằng cách sử dụng vote extension. Chúng ta sẽ giới thiệu các kiểu mới trong file `abci/types.go`. Các kiểu này sẽ được dùng để xử lý quá trình chuẩn bị proposal, xử lý proposal và xử lý vote extension.

### Triển Khai Struct Cho Vote Extensions

Đầu tiên, sao chép các struct sau vào `abci/types.go`. Mỗi struct phục vụ một mục đích cụ thể trong quá trình giảm thiểu front-running bằng vote extension:

```go
package abci

import (
 //import các file cần thiết
)

type PrepareProposalHandler struct {
 logger      log.Logger
 txConfig    client.TxConfig
 cdc         codec.Codec
 mempool     *mempool.ThresholdMempool
 txProvider  provider.TxProvider
 keyname     string
 runProvider bool
}
```

Struct `PrepareProposalHandler` được dùng để xử lý việc chuẩn bị một proposal trong quá trình đồng thuận. Nó chứa nhiều trường: `logger` để ghi log thông tin và lỗi, `txConfig` để cấu hình giao dịch, `cdc` (Codec) để mã hóa và giải mã giao dịch, `mempool` để tham chiếu đến tập hợp các giao dịch chưa được xác nhận, `txProvider` để xây dựng proposal với các giao dịch, `keyname` là tên của khóa dùng để ký giao dịch, và `runProvider` — một flag boolean cho biết liệu provider có nên được chạy để xây dựng proposal hay không.

```go
type ProcessProposalHandler struct {
 TxConfig client.TxConfig
 Codec    codec.Codec
 Logger   log.Logger
}
```

Sau khi proposal đã được chuẩn bị và vote extension đã được đưa vào, `ProcessProposalHandler` được dùng để xử lý proposal. Điều này bao gồm xác thực proposal và các vote extension được đưa vào. `ProcessProposalHandler` cho phép bạn truy cập cấu hình giao dịch và codec, cần thiết để xử lý vote extension.

```go
type VoteExtHandler struct {
 logger       log.Logger
 currentBlock int64
 mempool      *mempool.ThresholdMempool
 cdc          codec.Codec
}
```

Struct này được dùng để xử lý vote extension. Nó chứa logger để ghi log sự kiện, số block hiện tại, mempool để lưu trữ giao dịch, và codec để mã hóa và giải mã. Vote extension là phần quan trọng trong quá trình giảm thiểu front-running, vì chúng cho phép thông tin bổ sung được đưa vào cùng với mỗi phiếu bầu.

```go
type InjectedVoteExt struct {
 VoteExtSigner []byte
 Bids          [][]byte
}

type InjectedVotes struct {
 Votes []InjectedVoteExt
}
```

Các struct này được dùng để xử lý injected vote extension. Chúng bao gồm người ký vote extension và các bid liên quan đến vote extension. Mỗi mảng byte trong `Bids` là dạng tuần tự hóa của một giao dịch bid. Injected vote extension được dùng để thêm thông tin bổ sung vào một phiếu bầu sau khi nó đã được tạo. Các giao dịch bid được tuần tự hóa cung cấp cách để đưa dữ liệu giao dịch phức tạp vào định dạng nhỏ gọn, hiệu quả.

```go
type AppVoteExtension struct {
 Height int64
 Bids   [][]byte
}
```

Struct này được dùng cho application vote extension. Nó bao gồm chiều cao của block và các bid liên quan đến vote extension. Application vote extension được dùng để thêm thông tin bổ sung vào một phiếu bầu ở cấp độ ứng dụng, hữu ích để thêm ngữ cảnh hoặc dữ liệu bổ sung cụ thể cho ứng dụng.

```go
type SpecialTransaction struct {
 Height int
 Bids   [][]byte
}
```

Struct này được dùng cho các giao dịch đặc biệt. Nó bao gồm chiều cao của block và các bid liên quan đến giao dịch. Giao dịch đặc biệt được dùng cho các giao dịch cần được xử lý khác với giao dịch thông thường, chẳng hạn như các giao dịch là một phần của quá trình giảm thiểu front-running.

### Triển Khai và Cấu Hình Handlers

Để thiết lập `VoteExtensionHandler`, thực hiện các bước sau:

1. Điều hướng đến file `abci/proposal.go`. Đây là nơi chúng ta sẽ triển khai `VoteExtensionHandler`.

2. Triển khai hàm `NewVoteExtensionHandler`. Hàm này là constructor cho struct `VoteExtHandler`. Nó nhận một logger, một mempool và một codec làm tham số và trả về một instance mới của `VoteExtHandler`.

```go
func NewVoteExtensionHandler(lg log.Logger, mp *mempool.ThresholdMempool, cdc codec.Codec) *VoteExtHandler {  
   return &VoteExtHandler{  
      logger:  lg,  
      mempool: mp,  
      cdc:     cdc,  
   }  
}
```

3. Triển khai phương thức `ExtendVoteHandler()`. Phương thức này nên xử lý logic mở rộng phiếu bầu, bao gồm kiểm tra mempool và gửi danh sách tất cả các bid đang chờ xử lý. Điều này sẽ cho phép bạn truy cập danh sách các giao dịch chưa được xác nhận trong `abci.RequestPrepareProposal` trong block tiếp theo.

```go
func (h *VoteExtHandler) ExtendVoteHandler() sdk.ExtendVoteHandler {
 return func(ctx sdk.Context, req *abci.RequestExtendVote) (*abci.ResponseExtendVote, error) {
      h.logger.Info(fmt.Sprintf("Extending votes at block height : %v", req.Height))

 voteExtBids := [][]byte{}

 // Lấy các tx từ mempool
 itr := h.mempool.SelectPending(context.Background(), nil)
 for itr != nil {
  tmptx := itr.Tx()
  sdkMsgs := tmptx.GetMsgs()

  // Duyệt qua các msg, kiểm tra bid nào
  for _, msg := range sdkMsgs {
   switch msg := msg.(type) {
   case *nstypes.MsgBid:
   // Marshal sdk bids thành []byte
    bz, err := h.cdc.Marshal(msg)
    if err != nil {
     h.logger.Error(fmt.Sprintf("Error marshalling VE Bid : %v", err))
     break
    }
    voteExtBids = append(voteExtBids, bz)
   default:
   }
  }

  // Chuyển tx sang ready pool
  err := h.mempool.Update(context.Background(), tmptx)
  
  // Xóa tx khỏi app side mempool
  if err != nil {
   h.logger.Info(fmt.Sprintf("Unable to update mempool tx: %v", err))
  }
  
  itr = itr.Next()
 }

 // Tạo vote extension
 voteExt := AppVoteExtension{
 Height: req.Height,
 Bids: voteExtBids,
 }

 // Mã hóa Vote Extension
 bz, err := json.Marshal(voteExt)
  if err != nil {
  return nil, fmt.Errorf("Error marshalling VE: %w", err)
 }

 return &abci.ResponseExtendVote{VoteExtension: bz}, nil
}
```

4. Cấu hình handler trong `app/app.go` như sau

```go
bApp := baseapp.NewBaseApp(AppName, logger, db, txConfig.TxDecoder(), baseAppOptions...)
voteExtHandler := abci2.NewVoteExtensionHandler(logger, mempool, appCodec)
bApp.SetExtendVoteHandler(voteExtHandler.ExtendVoteHandler())
```

Để cung cấp thêm ngữ cảnh về những gì đang xảy ra ở trên: đầu tiên chúng ta tạo một instance mới của `VoteExtensionHandler` với các dependency cần thiết (logger, mempool và codec). Sau đó, chúng ta đặt handler này làm `ExtendVoteHandler` cho ứng dụng. Điều này có nghĩa là bất cứ khi nào một phiếu bầu cần được mở rộng, phương thức `ExtendVoteHandler()` tùy chỉnh của chúng ta sẽ được gọi.

Để kiểm tra xem vote extension đã được truyền bá hay chưa, thêm phần sau vào `PrepareProposalHandler`:

```go
if req.Height > 2 {  
   voteExt := req.GetLocalLastCommit()  
   h.logger.Info(fmt.Sprintf("🛠️ :: Get vote extensions: %v", voteExt))  
}
```

Đây là hình dạng toàn bộ hàm sẽ trông như thế nào:

```go
func (h *PrepareProposalHandler) PrepareProposalHandler() sdk.PrepareProposalHandler {
 return func(ctx sdk.Context, req *abci.RequestPrepareProposal) (*abci.ResponsePrepareProposal, error) {
  h.logger.Info(fmt.Sprintf("🛠️ :: Prepare Proposal"))
  var proposalTxs [][]byte

  var txs []sdk.Tx

  // Lấy Vote Extensions
  if req.Height > 2 {
   voteExt := req.GetLocalLastCommit()
   h.logger.Info(fmt.Sprintf("🛠️ :: Get vote extensions: %v", voteExt))
  }

  itr := h.mempool.Select(context.Background(), nil)
  for itr != nil {
   tmptx := itr.Tx()

   txs = append(txs, tmptx)
   itr = itr.Next()
  }
  h.logger.Info(fmt.Sprintf("🛠️ :: Number of Transactions available from mempool: %v", len(txs)))

  if h.runProvider {
   tmpMsgs, err := h.txProvider.BuildProposal(ctx, txs)
   if err != nil {
    h.logger.Error(fmt.Sprintf("❌️ :: Error Building Custom Proposal: %v", err))
   }
   txs = tmpMsgs
  }

  for _, sdkTxs := range txs {
   txBytes, err := h.txConfig.TxEncoder()(sdkTxs)
   if err != nil {
    h.logger.Info(fmt.Sprintf("❌~Error encoding transaction: %v", err.Error()))
   }
   proposalTxs = append(proposalTxs, txBytes)
  }

  h.logger.Info(fmt.Sprintf("🛠️ :: Number of Transactions in proposal: %v", len(proposalTxs)))

  return &abci.ResponsePrepareProposal{Txs: proposalTxs}, nil
 }
}
```

Như đã đề cập ở trên, chúng ta kiểm tra xem vote extension đã được truyền bá hay chưa bằng cách kiểm tra log xem có các message liên quan như `🛠️ :: Get vote extensions:` không. Nếu log không cung cấp đủ thông tin, bạn cũng có thể khởi tạo lại môi trường test cục bộ bằng cách chạy lại script `./scripts/single_node/setup.sh`.

5. Triển khai `ProcessProposalHandler()`. Hàm này chịu trách nhiệm xử lý proposal. Nó nên xử lý logic xử lý vote extension, bao gồm kiểm tra proposal và xác thực các bid.

```go
func (h *ProcessProposalHandler) ProcessProposalHandler() sdk.ProcessProposalHandler {
 return func(ctx sdk.Context, req *abci.RequestProcessProposal) (resp *abci.ResponseProcessProposal, err error) {
  h.Logger.Info(fmt.Sprintf("⚙️ :: Process Proposal"))

  // Giao dịch đầu tiên luôn là Giao Dịch Đặc Biệt
  numTxs := len(req.Txs)

  h.Logger.Info(fmt.Sprintf("⚙️:: Number of transactions :: %v", numTxs))

  if numTxs >= 1 {
   var st SpecialTransaction
   err = json.Unmarshal(req.Txs[0], &st)
   if err != nil {
    h.Logger.Error(fmt.Sprintf("❌️:: Error unmarshalling special Tx in Process Proposal :: %v", err))
   }
   if len(st.Bids) > 0 {
    h.Logger.Info(fmt.Sprintf("⚙️:: There are bids in the Special Transaction"))
    var bids []nstypes.MsgBid
    for i, b := range st.Bids {
     var bid nstypes.MsgBid
     h.Codec.Unmarshal(b, &bid)
     h.Logger.Info(fmt.Sprintf("⚙️:: Special Transaction Bid No %v :: %v", i, bid))
     bids = append(bids, bid)
    }
    // Xác thực Bids trong Tx
    txs := req.Txs[1:]
    ok, err := ValidateBids(h.TxConfig, bids, txs, h.Logger)
    if err != nil {
     h.Logger.Error(fmt.Sprintf("❌️:: Error validating bids in Process Proposal :: %v", err))
     return &abci.ResponseProcessProposal{Status: abci.ResponseProcessProposal_REJECT}, nil
    }
    if !ok {
     h.Logger.Error(fmt.Sprintf("❌️:: Unable to validate bids in Process Proposal :: %v", err))
     return &abci.ResponseProcessProposal{Status: abci.ResponseProcessProposal_REJECT}, nil
    }
    h.Logger.Info("⚙️:: Successfully validated bids in Process Proposal")
   }
  }

  return &abci.ResponseProcessProposal{Status: abci.ResponseProcessProposal_ACCEPT}, nil
 }
}
```

6. Triển khai hàm `ProcessVoteExtensions()`. Hàm này nên xử lý logic xử lý vote extension, bao gồm xác thực các bid.

```go
func processVoteExtensions(req *abci.RequestPrepareProposal, log log.Logger) (SpecialTransaction, error) {
 log.Info(fmt.Sprintf("🛠️ :: Process Vote Extensions"))

 // Tạo phản hồi rỗng
 st := SpecialTransaction{
  0,
  [][]byte{},
 }

 // Lấy Vote Ext cho H-1 từ Req
 voteExt := req.GetLocalLastCommit()
 votes := voteExt.Votes

 // Duyệt qua các vote
 var ve AppVoteExtension
 for _, vote := range votes {
  // Unmarshal thành AppExt
  err := json.Unmarshal(vote.VoteExtension, &ve)
  if err != nil {
   log.Error(fmt.Sprintf("❌ :: Error unmarshalling Vote Extension"))
  }

  st.Height = int(ve.Height)

  // Nếu có Bids trong VE, thêm vào Special Transaction
  if len(ve.Bids) > 0 {
   log.Info("🛠️ :: Bids in VE")
   for _, b := range ve.Bids {
    st.Bids = append(st.Bids, b)
   }
  }
 }

 return st, nil
}
```

7. Cấu hình `ProcessProposalHandler()` trong app/app.go:

```go
processPropHandler := abci2.ProcessProposalHandler{app.txConfig, appCodec, logger}
bApp.SetProcessProposal(processPropHandler.ProcessProposalHandler())
```

Điều này đặt `ProcessProposalHandler()` cho ứng dụng của chúng ta. Điều này có nghĩa là bất cứ khi nào một proposal cần được xử lý, phương thức `ProcessProposalHandler()` tùy chỉnh của chúng ta sẽ được gọi.

Để kiểm tra xem việc xử lý proposal và vote extension có hoạt động đúng không, bạn có thể kiểm tra log để tìm các message liên quan. Nếu log không cung cấp đủ thông tin, bạn cũng có thể khởi tạo lại môi trường test cục bộ bằng cách chạy script `./scripts/single_node/setup.sh`.
