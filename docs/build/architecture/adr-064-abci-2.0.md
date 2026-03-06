# ADR 64: Tích hợp ABCI 2.0 (Giai đoạn II)

## Changelog

* 2023-01-17: Bản nháp ban đầu (@alexanderbez)
* 2023-04-06: Bổ sung phần nâng cấp (@alexanderbez)
* 2023-04-10: Đơn giản hoá cơ chế lưu trữ trạng thái vote extension (@alexanderbez)
* 2023-07-07: Sửa đổi cơ chế lưu trữ trạng thái vote extension (@alexanderbez)
* 2023-08-24: Sửa đổi cách tính voting power của vote extension và interface staking (@davidterpay)

## Trạng thái

ĐƯỢC CHẤP NHẬN

## Tóm tắt

ADR này phác thảo việc tiếp tục nỗ lực triển khai ABCI++ trong Cosmos SDK như đã
được nêu trong [ADR 060: ABCI 1.0 (Giai đoạn I)](adr-060-abci-1.0.md).

Cụ thể, ADR này phác thảo thiết kế và triển khai ABCI 2.0, bao gồm `ExtendVote`,
`VerifyVoteExtension` và `FinalizeBlock`.

## Bối cảnh

ABCI 2.0 tiếp tục các cập nhật được hứa hẹn từ ABCI++, cụ thể là ba phương thức
ABCI bổ sung mà ứng dụng có thể triển khai để có thêm quyền kiểm soát, quan sát
và tuỳ biến đối với quá trình đồng thuận, mở khoá nhiều use-case mới trước đây
không thể thực hiện được. Chúng ta mô tả ba phương thức mới này bên dưới:

### `ExtendVote`

Phương thức này cho phép mỗi tiến trình validator mở rộng pha pre-commit của
quy trình đồng thuận CometBFT. Cụ thể, nó cho phép ứng dụng thực thi logic nghiệp
vụ tuỳ biến để mở rộng lá phiếu pre-commit và cung cấp thêm dữ liệu kèm theo
lá phiếu, dù phần dữ liệu này được ký riêng (separately) bởi cùng một khoá.

Dữ liệu này, gọi là vote extension, sẽ được phát tán và nhận cùng với lá phiếu
mà nó mở rộng, và sẽ được cung cấp cho ứng dụng ở chiều cao khối (height) tiếp
theo. Cụ thể, proposer của khối tiếp theo sẽ nhận được các vote extension trong
`RequestPrepareProposal.local_last_commit.votes`.

Nếu ứng dụng không có thông tin vote extension để cung cấp, nó trả về một mảng
byte có độ dài 0 làm vote extension.

**LƯU Ý**:

* Dù mỗi tiến trình validator gửi vote extension của riêng nó, CHỈ *proposer*
  của khối *kế tiếp* mới nhận được tất cả vote extension được bao gồm như một phần
  của pha pre-commit của khối trước. Điều này nghĩa là chỉ proposer sẽ mặc định
  có quyền truy cập tới tất cả vote extension, thông qua `RequestPrepareProposal`,
  và không phải tất cả vote extension đều nhất thiết được bao gồm, vì validator
  không cần chờ mọi pre-commit, chỉ cần 2/3.
* Lá phiếu pre-commit được ký độc lập so với vote extension.

### `VerifyVoteExtension`

Phương thức này cho phép các validator xác thực dữ liệu vote extension được gắn
kèm với mỗi thông điệp pre-commit mà nó nhận được. Nếu việc xác thực thất bại,
toàn bộ thông điệp pre-commit sẽ bị coi là không hợp lệ và bị CometBFT bỏ qua.

CometBFT sử dụng `VerifyVoteExtension` khi xác thực một lá phiếu pre-commit. Cụ thể,
đối với một pre-commit, CometBFT sẽ:

* Từ chối thông điệp nếu nó không chứa một lá phiếu đã ký VÀ một vote extension đã ký
* Từ chối thông điệp nếu chữ ký của lá phiếu HOẶC chữ ký của vote extension không xác thực được
* Từ chối thông điệp nếu `VerifyVoteExtension` bị ứng dụng từ chối

Ngược lại, CometBFT sẽ chấp nhận thông điệp pre-commit.

Lưu ý, điều này có hệ quả quan trọng đối với tính sống (liveness), ví dụ: nếu
vote extension liên tục không thể được xác thực bởi các validator đúng, CometBFT
có thể không thể chốt (finalize) một khối ngay cả khi đủ nhiều (+2/3) validator
gửi pre-commit cho khối đó. Vì vậy, `VerifyVoteExtension` cần được dùng hết sức thận trọng.

CometBFT khuyến nghị rằng một ứng dụng phát hiện vote extension không hợp lệ
NÊN chấp nhận nó trong `ResponseVerifyVoteExtension` và bỏ qua nó trong logic của
riêng ứng dụng.

### `FinalizeBlock`

Phương thức này chuyển giao một khối đã được quyết định (decided) tới ứng dụng.
Ứng dụng phải thực thi các giao dịch trong khối theo cách quyết định (deterministic)
và cập nhật state tương ứng. Các cam kết mật mã (cryptographic commitments) đối
với khối và kết quả giao dịch, được trả về thông qua các tham số tương ứng trong
`ResponseFinalizeBlock`, sẽ được đưa vào header của khối tiếp theo. CometBFT gọi
nó khi một khối mới được quyết định.

Nói cách khác, `FinalizeBlock` gói gọn luồng thực thi ABCI hiện tại gồm `BeginBlock`,
một hoặc nhiều `DeliverTx`, và `EndBlock` vào một phương thức ABCI duy nhất.
CometBFT sẽ không còn thực thi request cho các phương thức legacy này và thay vào
đó chỉ gọi `FinalizeBlock`.

## Quyết định

Chúng ta sẽ thảo luận các thay đổi đối với Cosmos SDK để triển khai ABCI 2.0 theo
hai giai đoạn riêng biệt: `VoteExtensions` và `FinalizeBlock`.

### `VoteExtensions`

Tương tự như `PrepareProposal` và `ProcessProposal`, chúng ta đề xuất đưa vào
hai handler mới mà ứng dụng có thể triển khai để cung cấp và xác minh vote extension.

Chúng ta đề xuất các handler mới sau để ứng dụng triển khai:

```go
type ExtendVoteHandler func(sdk.Context, abci.ExtendVoteRequest) abci.ExtendVoteResponse
type VerifyVoteExtensionHandler func(sdk.Context, abci.VerifyVoteExtensionRequest) abci.VerifyVoteExtensionResponse
```

Một context và state tạm thời (ephemeral) sẽ được cung cấp cho cả hai handler.
Context sẽ chứa metadata liên quan như block height và block hash. State sẽ là
một bản cached của committed state của ứng dụng và sẽ bị loại bỏ sau khi handler
chạy xong; điều này nghĩa là cả hai handler nhận được một góc nhìn state “mới”
và mọi thay đổi tạo ra sẽ không được ghi.

Nếu một ứng dụng quyết định triển khai `ExtendVoteHandler`, nó phải trả về một
`ResponseExtendVote.VoteExtension` khác-nil.

Nhắc lại, triển khai `ExtendVoteHandler` KHÔNG cần phải quyết định (deterministic),
tuy nhiên, với một tập vote extension cho trước, `VerifyVoteExtensionHandler` phải
quyết định, nếu không chain có thể gặp lỗi liveness. Ngoài ra, nhớ rằng CometBFT
tiến hành theo các vòng (round) cho mỗi height, vì vậy nếu không thể đưa ra quyết
định về một đề xuất khối ở một height nào đó, CometBFT sẽ chuyển sang vòng kế tiếp
và vì thế sẽ thực thi lại `ExtendVote` và `VerifyVoteExtension` cho vòng mới đối
với mỗi validator cho tới khi có thể lấy được 2/3 pre-commit hợp lệ.

Với phạm vi rộng của các triển khai và use-case tiềm năng của vote extension và
cách xác minh chúng, phần lớn ứng dụng nên chọn triển khai các handler thông qua
một kiểu handler duy nhất, kiểu này có thể inject bất kỳ số lượng phụ thuộc nào
như keepers. Ngoài ra, kiểu handler này có thể chứa một số khái niệm về quản lý
trạng thái vote extension biến động (volatile) để hỗ trợ việc xác minh. Quản lý
trạng thái này có thể là tạm thời hoặc là một dạng lưu trữ trên đĩa.

Ví dụ:

```go
// VoteExtensionHandler implements an Oracle vote extension handler.
type VoteExtensionHandler struct {
	cdc   Codec
	mk    MyKeeper
	state VoteExtState // This could be a map or a DB connection object
}

// ExtendVoteHandler can do something with h.mk and possibly h.state to create
// a vote extension, such as fetching a series of prices for supported assets.
func (h VoteExtensionHandler) ExtendVoteHandler(ctx sdk.Context, req abci.ExtendVoteRequest) abci.ExtendVoteResponse {
	prices := GetPrices(ctx, h.mk.Assets())
	bz, err := EncodePrices(h.cdc, prices)
	if err != nil {
		panic(fmt.Errorf("failed to encode prices for vote extension: %w", err))
	}

	// store our vote extension at the given height
	//
	// NOTE: Vote extensions can be overridden since we can timeout in a round.
	SetPrices(h.state, req, bz)

	return abci.ExtendVoteResponse{VoteExtension: bz}
}

// VerifyVoteExtensionHandler can do something with h.state and req to verify
// the req.VoteExtension field, such as ensuring the provided oracle prices are
// within some valid range of our prices.
func (h VoteExtensionHandler) VerifyVoteExtensionHandler(ctx sdk.Context, req abci.VerifyVoteExtensionRequest) abci.VerifyVoteExtensionResponse {
	prices, err := DecodePrices(h.cdc, req.VoteExtension)
	if err != nil {
		log("failed to decode vote extension", "err", err)
		return abci.VerifyVoteExtensionResponse{Status: REJECT}
	}

	if err := ValidatePrices(h.state, req, prices); err != nil {
		log("failed to validate vote extension", "prices", prices, "err", err)
		return abci.VerifyVoteExtensionResponse{Status: REJECT}
	}

	// store updated vote extensions at the given height
	//
	// NOTE: Vote extensions can be overridden since we can timeout in a round.
	SetPrices(h.state, req, req.VoteExtension)

	return abci.VerifyVoteExtensionResponse{Status: ACCEPT}
}
```

#### Lan truyền & xác minh vote extension

Như đã đề cập trước đó, vote extension cho height `H` chỉ được cung cấp cho proposer
ở height `H+1` trong `PrepareProposal`. Tuy nhiên, để vote extension hữu ích, tất
cả validator nên có quyền truy cập vào vote extension đã được thống nhất tại height `H`
trong height `H+1`.

Vì CometBFT bao gồm tất cả chữ ký vote extension trong `RequestPrepareProposal`,
chúng ta đề xuất validator đang đề xuất khối (proposing validator) tự tay “tiêm”
(inject) vote extension cùng với chữ ký tương ứng thông qua một giao dịch đặc
biệt, `VoteExtsTx`, vào đề xuất khối trong `PrepareProposal`. `VoteExtsTx` sẽ
được điền với một đối tượng `ExtendedCommitInfo` duy nhất, nhận trực tiếp từ
`RequestPrepareProposal`.

Theo quy ước, giao dịch `VoteExtsTx` nên là giao dịch đầu tiên trong đề xuất khối,
mặc dù các chain có thể triển khai sở thích riêng. Để an toàn, chúng ta cũng đề
xuất proposer tự xác minh tất cả chữ ký vote extension nó nhận được trong
`RequestPrepareProposal`.

Một validator, khi nhận `RequestProcessProposal`, sẽ nhận được `VoteExtsTx` đã
được tiêm vào, chứa vote extension kèm chữ ký. Nếu không có giao dịch như vậy,
validator BẮT BUỘC PHẢI TỪ CHỐI (REJECT) đề xuất.

Khi một validator kiểm tra `VoteExtsTx`, nó sẽ đánh giá từng `SignedVoteExtension`.
Với mỗi signed vote extension, validator sẽ tạo signed bytes và xác minh chữ ký.
Phải có ít nhất 2/3 chữ ký hợp lệ, tính theo voting power, để đề xuất khối hợp lệ;
nếu không validator BẮT BUỘC PHẢI TỪ CHỐI đề xuất.

Để có khả năng xác minh chữ ký, `BaseApp` phải có quyền truy cập tới module `x/staking`,
vì module này lưu index từ địa chỉ đồng thuận (consensus address) tới public key.
Tuy nhiên, chúng ta sẽ tránh phụ thuộc trực tiếp vào `x/staking` và thay vào đó
dựa vào một interface. Ngoài ra, Cosmos SDK sẽ cung cấp một phương thức xác minh
chữ ký mặc định mà ứng dụng có thể dùng:

```go
type ValidatorStore interface {
	GetPubKeyByConsAddr(context.Context, sdk.ConsAddress) (cmtprotocrypto.PublicKey, error)
}

// ValidateVoteExtensions is a function that an application can execute in
// ProcessProposal to verify vote extension signatures.
func (app *BaseApp) ValidateVoteExtensions(ctx sdk.Context, currentHeight int64, extCommit abci.ExtendedCommitInfo) error {
	votingPower := 0
	totalVotingPower := 0

	for _, vote := range extCommit.Votes {
		totalVotingPower += vote.Validator.Power

		if !vote.SignedLastBlock || len(vote.VoteExtension) == 0 {
			continue
		}

		valConsAddr := sdk.ConsAddress(vote.Validator.Address)
		pubKeyProto, err := valStore.GetPubKeyByConsAddr(ctx, valConsAddr)
		if err != nil {
			return fmt.Errorf("failed to get public key for validator %s: %w", valConsAddr, err)
		}

		if len(vote.ExtensionSignature) == 0 {
			return fmt.Errorf("received a non-empty vote extension with empty signature for validator %s", valConsAddr)
		}

		cmtPubKey, err := cryptoenc.PubKeyFromProto(pubKeyProto)
		if err != nil {
			return fmt.Errorf("failed to convert validator %X public key: %w", valConsAddr, err)
		}

		cve := cmtproto.CanonicalVoteExtension{
			Extension: vote.VoteExtension,
			Height:    currentHeight - 1, // the vote extension was signed in the previous height
			Round:     int64(extCommit.Round),
			ChainId:   app.GetChainID(),
		}

		extSignBytes, err := cosmosio.MarshalDelimited(&cve)
		if err != nil {
			return fmt.Errorf("failed to encode CanonicalVoteExtension: %w", err)
		}

		if !cmtPubKey.VerifySignature(extSignBytes, vote.ExtensionSignature) {
			return errors.New("received vote with invalid signature")
		}

		votingPower += vote.Validator.Power
	}

	if (votingPower / totalVotingPower) < threshold {
		return errors.New("not enough voting power for the vote extensions")
	}

	return nil
}
```

Khi đã nhận và xác minh ít nhất 2/3 chữ ký, tính theo voting power, validator có
thể dùng vote extension để suy ra dữ liệu bổ sung hoặc đưa ra quyết định dựa trên
vote extension.

> LƯU Ý: Điều rất quan trọng cần nêu rõ rằng: cả kỹ thuật lan truyền vote lẫn cơ
> chế xác minh vote extension mô tả ở trên đều KHÔNG bắt buộc để ứng dụng phải
> triển khai. Nói cách khác, proposer không bắt buộc phải xác minh và lan truyền
> vote extension cùng chữ ký, và các proposer cũng không bắt buộc phải xác minh
> các chữ ký đó. Ứng dụng có thể triển khai cơ chế PKI riêng và dùng cơ chế đó để
> ký và xác minh vote extension.

#### Lưu trữ vote extension

Trong một số bối cảnh, có thể hữu ích hoặc cần thiết để ứng dụng lưu trữ dữ liệu
được suy ra từ vote extension. Để hỗ trợ use-case này, chúng ta đề xuất cho phép
nhà phát triển ứng dụng định nghĩa một hook pre-Blocker sẽ được gọi ở ngay đầu
`FinalizeBlock`, tức là trước `BeginBlock` (xem bên dưới).

Lưu ý, chúng ta không thể cho phép ứng dụng ghi trực tiếp vào application state
trong `ProcessProposal` vì trong replay, CometBFT sẽ KHÔNG gọi `ProcessProposal`,
điều này sẽ dẫn tới một góc nhìn state không đầy đủ.

```go
func (a MyApp) PreBlocker(ctx sdk.Context, req *abci.FinalizeBlockRequest) error {
	voteExts := GetVoteExtensions(ctx, req.Txs)
	
	// Process and perform some compute on vote extensions, storing any resulting
	// state.
	if err a.processVoteExtensions(ctx, voteExts); if err != nil {
		return err
	}
}
```

### `FinalizeBlock`

Các phương thức ABCI hiện có `BeginBlock`, `DeliverTx`, và `EndBlock` đã tồn tại
từ những ngày đầu của các ứng dụng dựa trên ABCI. Vì vậy, ứng dụng, tooling, và
nhà phát triển đã quen thuộc với các phương thức này và use-case của chúng. Cụ thể,
`BeginBlock` và `EndBlock` đã trở nên khá “cốt lõi” và mạnh mẽ trong các ứng dụng
dựa trên ABCI. Ví dụ: một ứng dụng có thể muốn chạy các thao tác liên quan tới
phân phối (distribution) và lạm phát (inflation) trước khi thực thi giao dịch,
và sau đó thực hiện các thay đổi liên quan tới staking sau khi thực thi hết giao dịch.

Chúng ta đề xuất giữ `BeginBlock` và `EndBlock` chỉ trong các core module interface
của SDK để nhà phát triển ứng dụng có thể tiếp tục xây dựng dựa trên các luồng
thực thi hiện có. Tuy nhiên, chúng ta sẽ loại bỏ `BeginBlock`, `DeliverTx` và
`EndBlock` khỏi triển khai `BaseApp` của SDK và do đó khỏi bề mặt ABCI.

Khi đó sẽ chỉ còn một luồng thực thi `FinalizeBlock`. Cụ thể, trong `FinalizeBlock`
chúng ta sẽ thực thi `BeginBlock` của ứng dụng, tiếp theo là thực thi tất cả
giao dịch, và cuối cùng là thực thi `EndBlock` của ứng dụng.

Lưu ý, chúng ta vẫn giữ cơ chế thực thi giao dịch hiện có trong `BaseApp`, nhưng
mọi khái niệm về `DeliverTx` sẽ bị loại bỏ, tức là `deliverState` sẽ được thay
bằng `finalizeState`, và state này sẽ được commit trong `Commit`.

Tuy nhiên, hiện có các tham số và field đang tồn tại trong các kiểu ABCI
`BeginBlock` và `EndBlock` như votes dùng trong distribution và byzantine validators
dùng trong xử lý evidence. Các tham số này tồn tại trong request `FinalizeBlock`,
và sẽ cần được chuyển vào các triển khai `BeginBlock` và `EndBlock` của ứng dụng.

Điều này nghĩa là các core module interface của Cosmos SDK sẽ cần được cập nhật
để phản ánh các tham số này. Cách đơn giản và thẳng nhất để đạt được điều này là
chỉ truyền `RequestFinalizeBlock` vào `BeginBlock` và `EndBlock`. Hoặc, chúng ta
có thể tạo các kiểu proxy riêng trong SDK phản ánh các kiểu ABCI legacy, ví dụ:
`LegacyBeginBlockRequest` và `LegacyEndBlockRequest`. Hoặc, chúng ta có thể đặt
tên và kiểu mới hoàn toàn.

```go
func (app *BaseApp) FinalizeBlock(req abci.FinalizeBlockRequest) (*abci.FinalizeBlockResponse, error) {
	ctx := ...

	if app.preBlocker != nil {
		ctx := app.finalizeBlockState.ctx
		rsp, err := app.preBlocker(ctx, req)
		if err != nil {
			return nil, err
		}
		if rsp.ConsensusParamsChanged {
			app.finalizeBlockState.ctx = ctx.WithConsensusParams(app.GetConsensusParams(ctx))
		}
	}
	beginBlockResp, err := app.beginBlock(req)
	appendBlockEventAttr(beginBlockResp.Events, "begin_block")

	txExecResults := make([]abci.ExecTxResult, 0, len(req.Txs))
	for _, tx := range req.Txs {
		result := app.runTx(runTxModeFinalize, tx)
		txExecResults = append(txExecResults, result)
	}

	endBlockResp, err := app.endBlock(app.finalizeBlockState.ctx)
	appendBlockEventAttr(beginBlockResp.Events, "end_block")

	return abci.FinalizeBlockResponse{
		TxResults:             txExecResults,
		Events:                joinEvents(beginBlockResp.Events, endBlockResp.Events),
		ValidatorUpdates:      endBlockResp.ValidatorUpdates,
		ConsensusParamUpdates: endBlockResp.ConsensusParamUpdates,
		AppHash:               nil,
	}
}
```

#### Sự kiện (Events)

Nhiều công cụ, indexer và thư viện trong hệ sinh thái dựa vào sự tồn tại của các
sự kiện `BeginBlock` và `EndBlock`. Vì CometBFT giờ chỉ expose `FinalizeBlockEvents`,
chúng ta nhận thấy vẫn hữu ích cho các client và công cụ này tiếp tục truy vấn và
dựa vào các event hiện có, đặc biệt vì ứng dụng vẫn sẽ định nghĩa các triển khai
`BeginBlock` và `EndBlock`.

Để hỗ trợ chức năng event hiện có, chúng ta đề xuất mọi event `BeginBlock` và
`EndBlock` có một `EventAttribute` chuyên biệt với `key=block` và
`value=begin_block|end_block`. `EventAttribute` này sẽ được append vào mỗi event
trong cả `BeginBlock` và `EndBlock`.

### Nâng cấp

CometBFT định nghĩa một tham số đồng thuận,
[`VoteExtensionsEnableHeight`](https://github.com/cometbft/cometbft/blob/v0.38.0-alpha.1/spec/abci/abci%2B%2B_app_requirements.md#abciparamsvoteextensionsenableheight),
tham số này chỉ định height mà tại đó vote extension được bật và **bắt buộc**.
Nếu giá trị đặt là 0, là giá trị mặc định, thì vote extension bị tắt và ứng dụng
không bắt buộc phải triển khai và dùng vote extension.

Tuy nhiên, nếu giá trị `H` là số dương, thì tại mọi height lớn hơn height cấu hình
`H`, vote extension phải hiện diện (kể cả rỗng). Khi đạt đến height cấu hình `H`,
`PrepareProposal` sẽ chưa bao gồm vote extension, nhưng `ExtendVote` và
`VerifyVoteExtension` sẽ được gọi. Sau đó, khi đạt height `H+1`, `PrepareProposal`
sẽ bao gồm vote extension từ height `H`.

Cần đặc biệt lưu ý rằng, với mọi height sau H:

* Vote extension KHÔNG THỂ bị tắt
* Chúng là bắt buộc, tức là mọi thông điệp pre-commit được gửi PHẢI có extension
  đính kèm (kể cả rỗng)

Khi một ứng dụng cập nhật lên phiên bản Cosmos SDK có hỗ trợ CometBFT v0.38, trong
upgrade handler nó phải đảm bảo thiết lập tham số đồng thuận
`VoteExtensionsEnableHeight` đúng giá trị. Ví dụ: nếu một ứng dụng dự kiến nâng cấp
tại height `H`, thì giá trị của `VoteExtensionsEnableHeight` nên được đặt thành
bất kỳ giá trị nào `>=H+1`. Điều này nghĩa là tại height nâng cấp `H`, vote extension
chưa được bật, nhưng tại height `H+1` chúng sẽ được bật.

## Hệ quả

### Tương thích ngược

ABCI 2.0 một cách tự nhiên là không tương thích ngược với các phiên bản trước của
Cosmos SDK và CometBFT. Ví dụ, một ứng dụng thực hiện `RequestFinalizeBlock` đến
một ứng dụng không “nói” ABCI 2.0 sẽ thất bại.

Ngoài ra, `BeginBlock`, `DeliverTx` và `EndBlock` sẽ bị loại bỏ khỏi các interface
ABCI của ứng dụng, cùng với việc input và output bị chỉnh sửa trong các module
interface.

### Tích cực

* Ngữ nghĩa `BeginBlock` và `EndBlock` vẫn được giữ, vì vậy gánh nặng cho nhà phát
  triển ứng dụng sẽ hạn chế.
* Ít overhead giao tiếp hơn vì nhiều request ABCI được gom lại thành một request duy nhất.
* Đặt nền tảng cho optimistic execution.
* Vote extension cho phép phát triển một tập primitive ứng dụng hoàn toàn mới,
  như oracle giá chạy cùng tiến trình và mempool được mã hoá.

### Tiêu cực

* Một số core API hiện có của Cosmos SDK có thể cần được sửa và do đó bị phá vỡ.
* Xác minh chữ ký trong `ProcessProposal` của 100+ chữ ký vote extension sẽ thêm
  overhead hiệu năng đáng kể cho `ProcessProposal`. Dù vậy, quá trình xác minh
  chữ ký có thể diễn ra đồng thời bằng error group với `GOMAXPROCS` goroutine.

### Trung tính

* Việc phải tự tay “tiêm” vote extension vào đề xuất khối trong `PrepareProposal`
  là cách tiếp cận vụng và chiếm không gian khối không cần thiết.
* Yêu cầu `ResetProcessProposalState` có thể tạo một “footgun” cho nhà phát triển
  ứng dụng nếu không cẩn thận, nhưng điều này là cần thiết để ứng dụng có thể
  commit state từ việc tính toán vote extension.

## Thảo luận thêm

Các thảo luận tương lai bao gồm thiết kế và triển khai ABCI 3.0, là sự tiếp nối
của ABCI++ và thảo luận chung về optimistic execution.

## Tham khảo

* [ADR 060: ABCI 1.0 (Giai đoạn I)](adr-060-abci-1.0.md)

