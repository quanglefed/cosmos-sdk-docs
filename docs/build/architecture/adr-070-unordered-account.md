# ADR 070: Giao dịch không cần thứ tự (Unordered Transactions)

## Changelog

* Dec 4, 2023: Bản nháp ban đầu (@yihuang, @tac0turtle, @alexanderbez)
* Jan 30, 2024: Bổ sung mục về encoding giao dịch mang tính quyết định
* Mar 18, 2025: Sửa đổi triển khai để dùng Cosmos SDK KV Store và yêu cầu timeout duy nhất theo từng địa chỉ (@technicallyty)
* Apr 25, 2025: Thêm lưu ý về việc từ chối unordered tx có giá trị sequence.

## Trạng thái

ĐƯỢC CHẤP NHẬN (Chưa triển khai)

## Tóm tắt

Chúng ta đề xuất một cách để chống replay-attack mà không bắt buộc thứ tự giao
dịch và không yêu cầu dùng sequence tăng đơn điệu. Thay vào đó, chúng ta đề xuất
dùng một sequence tạm thời (ephemeral) dựa trên thời gian.

## Bối cảnh

Giá trị sequence của tài khoản phục vụ để ngăn replay attack và đảm bảo giao dịch
từ cùng một người gửi được đưa vào khối và thực thi theo thứ tự tuần tự. Thật
không may, điều này khiến việc gửi nhiều giao dịch đồng thời từ cùng một người
gửi trở nên khó tin cậy. Những đối tượng chịu ảnh hưởng điển hình gồm IBC relayer
và các sàn giao dịch crypto.

## Quyết định

Chúng ta đề xuất thêm một trường boolean `unordered` và một trường
google.protobuf.Timestamp `timeout_timestamp` vào phần body của giao dịch.

Giao dịch unordered sẽ bỏ qua các quy tắc sequence tài khoản truyền thống và tuân
theo các quy tắc mô tả bên dưới, mà không ảnh hưởng tới giao dịch ordered truyền
thống (vốn vẫn theo quy tắc sequence như trước).

Chúng ta sẽ đưa vào cơ chế lưu trữ mới cho các unordered sequence tạm thời dựa
trên thời gian, dùng thư viện KV Store hiện có của SDK. Cụ thể, chúng ta sẽ tận
dụng x/auth KV store hiện có để lưu unordered sequence.

Khi một giao dịch unordered được đưa vào một khối, một chuỗi ghép (concatenation)
của `timeout_timestamp` và bytes của địa chỉ người gửi sẽ được ghi vào state (tức
`542939323/<address_bytes>`). Trong trường hợp nhiều bên cùng ký, một entry cho
mỗi signer sẽ được ghi vào state.

Giao dịch mới sẽ được kiểm tra đối chiếu với state để ngăn gửi trùng. Để ngăn
state tăng vô hạn, chúng ta đề xuất:

* Định nghĩa một cận trên cho giá trị `timeout_timestamp` (ví dụ 10 phút).
* Thêm phương thức PreBlocker vào x/auth để xoá các entry state có `timeout_timestamp`
  sớm hơn thời gian của khối hiện tại.

### Định dạng giao dịch

```protobuf
message TxBody {
  ...
          
  bool unordered = 4;
  google.protobuf.Timestamp timeout_timestamp = 5;
}
```

### Bảo vệ chống replay

Chúng ta thực hiện bảo vệ chống replay bằng cách lưu unordered sequence trong
Cosmos SDK KV store. Khi giao dịch đi vào (ingress), chúng ta kiểm tra unordered
sequence của giao dịch có tồn tại trong state hay không, hoặc giá trị TTL đã “cũ”
(stale) hay không, tức là sớm hơn thời gian của khối hiện tại. Nếu có, ta từ chối.
Nếu không, ta thêm unordered sequence vào state. Phần state này thuộc module `x/auth`.

State sẽ được đánh giá trong `PreBlocker` của x/auth. Mọi giao dịch có unordered
sequence sớm hơn thời gian khối hiện tại sẽ bị xoá.

```go
func (am AppModule) PreBlock(ctx context.Context) (appmodule.ResponsePreBlock, error) {
	err := am.accountKeeper.RemoveExpired(sdk.UnwrapSDKContext(ctx))
	if err != nil {
		return nil, err
	}
	return &sdk.ResponsePreBlock{ConsensusParamsChanged: false}, nil
}
```

```golang
package keeper

import (
	sdk "github.com/cosmos/cosmos-sdk/types"

	"cosmossdk.io/collections"
	"cosmossdk.io/core/store"
)

var (
	// just arbitrarily picking some upper bound number.
	unorderedSequencePrefix = collections.NewPrefix(90)
)

type AccountKeeper struct {
	// ...
	unorderedSequences collections.KeySet[collections.Pair[uint64, []byte]]
}

func (m *AccountKeeper) Contains(ctx sdk.Context, sender []byte, timestamp uint64) (bool, error) {
	return m.unorderedSequences.Has(ctx, collections.Join(timestamp, sender))
}

func (m *AccountKeeper) Add(ctx sdk.Context, sender []byte, timestamp uint64) error {
	return m.unorderedSequences.Set(ctx, collections.Join(timestamp, sender))
}

func (m *AccountKeeper) RemoveExpired(ctx sdk.Context) error {
	blkTime := ctx.BlockTime().UnixNano()
	it, err := m.unorderedSequences.Iterate(ctx, collections.NewPrefixUntilPairRange[uint64, []byte](uint64(blkTime)))
	if err != nil {
		return err
	}
	defer it.Close()

	keys, err := it.Keys()
	if err != nil {
		return err
	}

	for _, key := range keys {
		if err := m.unorderedSequences.Remove(ctx, key); err != nil {
			return err
		}
	}

	return nil
}

```

### AnteHandler Decorator

Để bỏ qua việc xác minh nonce, chúng ta phải sửa decorator AnteHandler hiện có
`IncrementSequenceDecorator` để bỏ qua xác minh nonce khi giao dịch được đánh dấu
là unordered.

```golang
func (isd IncrementSequenceDecorator) AnteHandle(ctx sdk.Context, tx sdk.Tx, simulate bool, next sdk.AnteHandler) (sdk.Context, error) {
  if tx.UnOrdered() {
    return next(ctx, tx, simulate)
  }

  // ...
}
```

Chúng ta cũng giới thiệu một decorator mới để thực hiện xác minh giao dịch unordered.

```golang
package ante

import (
	"slices"
	"strings"
	"time"

	sdk "github.com/cosmos/cosmos-sdk/types"
	sdkerrors "github.com/cosmos/cosmos-sdk/types/errors"
	authkeeper "github.com/cosmos/cosmos-sdk/x/auth/keeper"
	authsigning "github.com/cosmos/cosmos-sdk/x/auth/signing"

	errorsmod "cosmossdk.io/errors"
)

var _ sdk.AnteDecorator = (*UnorderedTxDecorator)(nil)

// UnorderedTxDecorator defines an AnteHandler decorator that is responsible for
// checking if a transaction is intended to be unordered and, if so, evaluates
// the transaction accordingly. An unordered transaction will bypass having its
// nonce incremented, which allows fire-and-forget transaction broadcasting,
// removing the necessity of ordering on the sender-side.
//
// The transaction sender must ensure that unordered=true and a timeout_height
// is appropriately set. The AnteHandler will check that the transaction is not
// a duplicate and will evict it from state when the timeout is reached.
//
// The UnorderedTxDecorator should be placed as early as possible in the AnteHandler
// chain to ensure that during DeliverTx, the transaction is added to the unordered sequence state.
type UnorderedTxDecorator struct {
	// maxUnOrderedTTL defines the maximum TTL a transaction can define.
	maxTimeoutDuration time.Duration
	txManager          authkeeper.UnorderedTxManager
}

func NewUnorderedTxDecorator(
	utxm authkeeper.UnorderedTxManager,
) *UnorderedTxDecorator {
	return &UnorderedTxDecorator{
		maxTimeoutDuration: 10 * time.Minute,
		txManager:          utxm,
	}
}

func (d *UnorderedTxDecorator) AnteHandle(
	ctx sdk.Context,
	tx sdk.Tx,
	_ bool,
	next sdk.AnteHandler,
) (sdk.Context, error) {
	if err := d.ValidateTx(ctx, tx); err != nil {
		return ctx, err
	}
	return next(ctx, tx, false)
}

func (d *UnorderedTxDecorator) ValidateTx(ctx sdk.Context, tx sdk.Tx) error {
	unorderedTx, ok := tx.(sdk.TxWithUnordered)
	if !ok || !unorderedTx.GetUnordered() {
		// If the transaction does not implement unordered capabilities or has the
		// unordered value as false, we bypass.
		return nil
	}

	blockTime := ctx.BlockTime()
	timeoutTimestamp := unorderedTx.GetTimeoutTimeStamp()
	if timeoutTimestamp.IsZero() || timeoutTimestamp.Unix() == 0 {
		return errorsmod.Wrap(
			sdkerrors.ErrInvalidRequest,
			"unordered transaction must have timeout_timestamp set",
		)
	}
	if timeoutTimestamp.Before(blockTime) {
		return errorsmod.Wrap(
			sdkerrors.ErrInvalidRequest,
			"unordered transaction has a timeout_timestamp that has already passed",
		)
	}
	if timeoutTimestamp.After(blockTime.Add(d.maxTimeoutDuration)) {
		return errorsmod.Wrapf(
			sdkerrors.ErrInvalidRequest,
			"unordered tx ttl exceeds %s",
			d.maxTimeoutDuration.String(),
		)
	}

	execMode := ctx.ExecMode()
	if execMode == sdk.ExecModeSimulate {
		return nil
	}

	signerAddrs, err := getSigners(tx)
	if err != nil {
		return err
	}
	
	for _, signer := range signerAddrs {
		contains, err := d.txManager.Contains(ctx, signer, uint64(unorderedTx.GetTimeoutTimeStamp().Unix()))
		if err != nil {
			return errorsmod.Wrap(
				sdkerrors.ErrIO,
				"failed to check contains",
			)
		}
		if contains {
			return errorsmod.Wrapf(
				sdkerrors.ErrInvalidRequest,
				"tx is duplicated for signer %x", signer,
			)
		}

		if err := d.txManager.Add(ctx, signer, uint64(unorderedTx.GetTimeoutTimeStamp().Unix())); err != nil {
			return errorsmod.Wrap(
				sdkerrors.ErrIO,
				"failed to add unordered sequence to state",
			)
		}
    }
	
	
	return nil
}

func getSigners(tx sdk.Tx) ([][]byte, error) {
	sigTx, ok := tx.(authsigning.SigVerifiableTx)
	if !ok {
		return nil, errorsmod.Wrap(sdkerrors.ErrTxDecode, "invalid tx type")
	}
	return sigTx.GetSigners()
}

```

### Unordered sequence

Unordered sequence cung cấp cơ chế đơn giản, thẳng thắn để bảo vệ chống cả transaction
malleability lẫn trùng lặp giao dịch. Cần lưu ý rằng unordered sequence vẫn phải
duy nhất. Tuy nhiên, giá trị không cần tăng строго (strictly increasing) như
sequence thông thường, và thứ tự node nhận giao dịch không còn quan trọng. Client
có thể xây dựng giao dịch unordered tương tự như đoạn code bên dưới:

```go
for _, tx := range txs {
	tx.SetUnordered(true)
	tx.SetTimeoutTimestamp(time.Now() + 1 * time.Nanosecond)
}
```

Chúng ta sẽ từ chối các giao dịch có cả sequence và unordered timeouts được thiết lập.
Chúng ta làm vậy để tránh phải suy đoán ý định của người dùng.

### Quản lý state

Việc lưu trữ unordered sequence sẽ được thực hiện bằng Cosmos SDK KV Store service.

## Ghi chú về vòng thiết kế trước

Phiên bản trước của giao dịch unordered hoạt động bằng một hệ thống quản lý state
tự phát (ad-hoc) và tạo ra rủi ro nghiêm trọng cũng như một vector để xử lý tx
trùng lặp. Nó phụ thuộc vào việc ứng dụng đóng một cách “êm” (graceful) để flush
trạng thái hiện tại của mapping unordered sequence. Nếu 2/3 mạng bị crash và quá
trình graceful closure không kích hoạt, hệ thống sẽ mất dấu toàn bộ sequence trong
mapping, cho phép các giao dịch đó bị replay. Triển khai được đề xuất trong phiên
bản cập nhật ADR này giải quyết điều đó bằng cách ghi trực tiếp vào Cosmos KV Store.
Dù cách này kém hiệu năng hơn, trong triển khai ban đầu chúng ta chọn con đường
an toàn hơn và hoãn tối ưu hiệu năng cho tới khi có thêm dữ liệu về tác động thực
tế và có một cách tiếp cận tối ưu “thực chiến” hơn.

Ngoài ra, phiên bản trước dựa vào việc dùng hash để tạo cái mà ta gọi là “unordered
sequence”. Có các vấn đề đã biết về transaction malleability trong các chế độ ký
(signing mode) của Cosmos SDK. ADR này tránh vấn đề đó bằng cách ép buộc nonce unordered
dùng một lần (single-use), thay vì suy dẫn nonce từ bytes trong giao dịch.

## Hệ quả

### Tích cực

* Hỗ trợ việc đưa giao dịch unordered vào khối, cho phép “fire and forget” nhiều
  giao dịch cùng lúc.

### Tiêu cực

* Yêu cầu overhead lưu trữ bổ sung.
* Yêu cầu timestamp duy nhất cho mỗi giao dịch tạo thêm một chút overhead cho client.
  Client phải đảm bảo timeout timestamp của mỗi giao dịch khác nhau. Tuy nhiên, chỉ
  cần chênh lệch nano giây là đủ.
* Việc dùng Cosmos SDK KV store chậm hơn so với dùng một store không merkle hoá hoặc
  các phương pháp ad-hoc, và block time có thể chậm lại theo đó.

## Tham khảo

* https://github.com/cosmos/cosmos-sdk/issues/13009

