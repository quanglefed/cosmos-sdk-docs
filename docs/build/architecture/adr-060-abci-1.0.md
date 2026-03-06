# ADR 60: Tích hợp ABCI 1.0 (Giai đoạn I)

## Changelog

* 2022-08-10: Bản nháp ban đầu (@alexanderbez, @tac0turtle)
* Nov 12, 2022: Cập nhật ngữ nghĩa `PrepareProposal` và `ProcessProposal` theo
  triển khai ban đầu [PR](https://github.com/cosmos/cosmos-sdk/pull/13453) (@alexanderbez)

## Trạng thái

ĐƯỢC CHẤP NHẬN

## Tóm tắt

ADR này mô tả việc áp dụng ban đầu [ABCI 1.0](https://github.com/tendermint/tendermint/blob/master/spec/abci%2B%2B/README.md),
bước tiến hoá tiếp theo của ABCI, trong Cosmos SDK. ABCI 1.0 nhằm cung cấp cho
nhà phát triển ứng dụng nhiều tính linh hoạt và quyền kiểm soát hơn đối với ngữ
nghĩa của ứng dụng và đồng thuận, ví dụ: mempool trong ứng dụng (in-application),
oracle chạy cùng tiến trình (in-process), và các matching engine kiểu sổ lệnh (order-book).

## Bối cảnh

Tendermint sẽ phát hành ABCI 1.0. Đáng chú ý, tại thời điểm viết tài liệu này,
Tendermint đang phát hành v0.37.0, phiên bản sẽ bao gồm `PrepareProposal` và `ProcessProposal`.

Phương thức ABCI `PrepareProposal` liên quan đến việc proposer của khối yêu cầu
ứng dụng đánh giá một loạt giao dịch để đưa vào khối tiếp theo, được mô tả như
một lát (slice) các đối tượng `TxRecord`. Ứng dụng có thể chấp nhận, từ chối,
hoặc hoàn toàn bỏ qua một phần hoặc toàn bộ các giao dịch này. Đây là điểm cần
cân nhắc quan trọng bởi vì về bản chất ứng dụng có thể tự định nghĩa và kiểm
soát mempool của riêng mình: bằng cách hoàn toàn bỏ qua các `TxRecords` mà
Tendermint gửi, và ưu tiên các giao dịch do chính ứng dụng lựa chọn. Điều này
về cơ bản khiến mempool của Tendermint hoạt động giống một cấu trúc dữ liệu
gossip hơn.

Phương thức ABCI thứ hai, `ProcessProposal`, được dùng để xử lý đề xuất khối
của proposer như đã được định nghĩa bởi `PrepareProposal`. Cần lưu ý các điểm
sau đối với `ProcessProposal`:

* Việc thực thi `ProcessProposal` phải mang tính quyết định (deterministic).
* Phải có tính nhất quán (coherence) giữa `PrepareProposal` và `ProcessProposal`.
  Nói cách khác, với bất kỳ hai tiến trình đúng *p* và *q*, nếu Tendermint của *q*
  gọi `RequestProcessProposal` trên *u<sub>p</sub>*, thì Ứng dụng của *q* sẽ trả
  về ACCEPT trong `ResponseProcessProposal`.

Cần lưu ý rằng trong tích hợp ABCI 1.0, ứng dụng KHÔNG chịu trách nhiệm về ngữ
nghĩa khoá (locking) — Tendermint vẫn chịu trách nhiệm cho phần đó. Tuy nhiên,
trong tương lai, ứng dụng sẽ chịu trách nhiệm về locking, điều này mở ra khả
năng thực thi song song.

## Quyết định

Chúng ta sẽ tích hợp ABCI 1.0, sẽ được giới thiệu trong Tendermint v0.37.0, vào
lần phát hành major tiếp theo của Cosmos SDK. Chúng ta sẽ tích hợp các phương
thức ABCI 1.0 trên kiểu `BaseApp`. Bên dưới mô tả riêng lẻ phần triển khai của
hai phương thức.

Trước khi mô tả triển khai của hai phương thức mới, cần lưu ý rằng các phương
thức ABCI hiện có như `CheckTx`, `DeliverTx`, v.v... vẫn tồn tại và giữ nguyên
chức năng như hiện tại.

### `PrepareProposal`

Trước khi đánh giá quyết định về cách triển khai `PrepareProposal`, cần lưu ý
rằng `CheckTx` vẫn sẽ được thực thi và vẫn chịu trách nhiệm đánh giá tính hợp
lệ của giao dịch như hiện tại, nhưng có một điểm phân biệt *bổ sung* rất quan trọng.

Khi thực thi giao dịch trong `CheckTx`, ứng dụng giờ đây sẽ thêm các giao dịch
hợp lệ, tức là vượt qua AnteHandler, vào cấu trúc dữ liệu mempool của riêng nó.
Để cung cấp một cách tiếp cận linh hoạt đáp ứng các nhu cầu khác nhau của nhà
phát triển ứng dụng, chúng ta sẽ định nghĩa cả một interface mempool và một cấu
trúc dữ liệu sử dụng generics của Golang, cho phép nhà phát triển chỉ tập trung
vào việc sắp xếp (ordering) giao dịch. Các nhà phát triển cần quyền kiểm soát
tuyệt đối có thể tự cài đặt mempool tuỳ biến.

Chúng ta định nghĩa interface mempool tổng quát như sau (có thể thay đổi):

```go
type Mempool interface {
	// Insert attempts to insert a Tx into the app-side mempool returning
	// an error upon failure.
	Insert(sdk.Context, sdk.Tx) error

	// Select returns an Iterator over the app-side mempool. If txs are specified,
	// then they shall be incorporated into the Iterator. The Iterator must
	// be closed by the caller.
	Select(sdk.Context, [][]byte) Iterator

	// CountTx returns the number of transactions currently in the mempool.
	CountTx() int

	// Remove attempts to remove a transaction from the mempool, returning an error
	// upon failure.
	Remove(sdk.Tx) error
}

// Iterator defines an app-side mempool iterator interface that is as minimal as
// possible. The order of iteration is determined by the app-side mempool
// implementation.
type Iterator interface {
	// Next returns the next transaction from the mempool. If there are no more
	// transactions, it returns nil.
	Next() Iterator

	// Tx returns the transaction at the current position of the iterator.
	Tx() sdk.Tx
}
```

Chúng ta sẽ định nghĩa một triển khai của `Mempool`, được định nghĩa bởi
`nonceMempool`, để bao phủ phần lớn các use-case cơ bản. Cụ thể, nó sẽ ưu tiên
giao dịch theo người gửi, cho phép nhiều giao dịch từ cùng một người gửi.

Triển khai mempool phía ứng dụng mặc định, `nonceMempool`, sẽ vận hành trên một
cấu trúc dữ liệu skip list duy nhất. Cụ thể, các giao dịch có nonce nhỏ nhất
trên toàn cục sẽ được ưu tiên. Các giao dịch có cùng nonce được ưu tiên theo
địa chỉ người gửi.

```go
type nonceMempool struct {
	txQueue *huandu.SkipList
}
```

Các thảo luận trước đó<sup>1</sup> đã thống nhất rằng Tendermint sẽ gửi một yêu
cầu đến ứng dụng, thông qua `RequestPrepareProposal`, với một số lượng giao dịch
nhất định được reaping từ mempool cục bộ của Tendermint. Số lượng giao dịch được
reaping sẽ được xác định bởi cấu hình cục bộ của operator. Cách tiếp cận này
được gọi là “one-shot approach” trong các thảo luận.

Khi Tendermint reaps giao dịch từ mempool cục bộ và gửi chúng đến ứng dụng qua
`RequestPrepareProposal`, ứng dụng sẽ phải đánh giá các giao dịch. Cụ thể, ứng
dụng cần thông báo cho Tendermint giao dịch nào nên bị từ chối và/hoặc được đưa
vào. Lưu ý, ứng dụng thậm chí có thể *thay thế* hoàn toàn các giao dịch bằng các
giao dịch khác.

Khi đánh giá giao dịch từ `RequestPrepareProposal`, ứng dụng sẽ bỏ qua *TOÀN BỘ*
các giao dịch được gửi trong request và thay vào đó reaps tối đa
`RequestPrepareProposal.max_tx_bytes` từ mempool của chính mình.

Vì ứng dụng về mặt kỹ thuật có thể chèn (insert) hoặc tiêm (inject) giao dịch
trong `Insert` trong quá trình thực thi `CheckTx`, nên khuyến nghị các ứng dụng
đảm bảo tính hợp lệ của giao dịch khi reaping giao dịch trong `PrepareProposal`.
Tuy nhiên, “hợp lệ” nghĩa là gì sẽ hoàn toàn do ứng dụng quyết định.

Cosmos SDK sẽ cung cấp một triển khai `PrepareProposal` mặc định, đơn giản là
chọn tối đa `MaxBytes` giao dịch *hợp lệ*.

Tuy nhiên, các ứng dụng có thể override triển khai mặc định này bằng triển khai
của riêng mình và gán nó trên `BaseApp` thông qua `SetPrepareProposal`.

### `ProcessProposal`

Phương thức ABCI `ProcessProposal` tương đối đơn giản. Nó chịu trách nhiệm đảm
bảo tính hợp lệ của khối được đề xuất chứa các giao dịch đã được chọn từ bước
`PrepareProposal`. Tuy nhiên, cách một ứng dụng xác định tính hợp lệ của một khối
đề xuất phụ thuộc vào ứng dụng và các use-case khác nhau. Với phần lớn ứng dụng,
chỉ cần gọi chuỗi `AnteHandler` là đủ, nhưng hoàn toàn có thể có các ứng dụng
khác cần kiểm soát nhiều hơn đối với quá trình xác thực khối, chẳng hạn đảm bảo
các tx theo một thứ tự nhất định hoặc đảm bảo có chứa các giao dịch cụ thể.
Về lý thuyết có thể đạt được bằng một `AnteHandler` tuỳ biến, nhưng đó không phải
UX sạch nhất hoặc giải pháp hiệu quả nhất.

Thay vào đó, chúng ta sẽ định nghĩa một phương thức ABCI bổ sung trên interface
`Application` hiện có, tương tự như các phương thức ABCI hiện có như `BeginBlock`
hoặc `EndBlock`. Phương thức interface mới này được định nghĩa như sau:

```go
ProcessProposal(sdk.Context, abci.ProcessProposalRequest) error {}
```

Lưu ý, chúng ta phải gọi `ProcessProposal` với một state phân nhánh (branched)
nội bộ mới trên đối số `Context` bởi vì không thể đơn giản dùng `checkState` hiện
có, vì tại thời điểm này `BaseApp` đã có `checkState` bị chỉnh sửa. Vì vậy khi
thực thi `ProcessProposal`, chúng ta tạo một state phân nhánh tương tự,
`processProposalState`, từ `deliverState`. Lưu ý `processProposalState` không bao
giờ được commit và sẽ bị loại bỏ hoàn toàn sau khi `ProcessProposal` kết thúc.

Cosmos SDK sẽ cung cấp một triển khai mặc định của `ProcessProposal` trong đó
tất cả giao dịch được xác thực theo luồng CheckTx, tức là AnteHandler, và sẽ
luôn trả về ACCEPT trừ khi có giao dịch nào không thể decode.

### `DeliverTx`

Vì giao dịch không thực sự bị xoá khỏi mempool phía ứng dụng trong `PrepareProposal`,
do `ProcessProposal` có thể thất bại hoặc cần nhiều vòng (round), và chúng ta
không muốn mất giao dịch, nên cần cuối cùng xoá giao dịch khỏi mempool phía ứng
dụng trong `DeliverTx` vì trong giai đoạn này, các giao dịch đang được đưa vào
khối đề xuất.

Hoặc, chúng ta có thể coi giao dịch bị xoá thật sự ngay trong giai đoạn reaping
trong `PrepareProposal` và thêm chúng trở lại mempool phía ứng dụng trong trường
hợp `ProcessProposal` thất bại.

## Hệ quả

### Tương thích ngược

ABCI 1.0 một cách tự nhiên là không tương thích ngược với các phiên bản trước
của Cosmos SDK và Tendermint. Ví dụ, một ứng dụng thực hiện `RequestPrepareProposal`
đến một ứng dụng không “nói” ABCI 1.0 sẽ thất bại.

Tuy nhiên, trong giai đoạn tích hợp đầu tiên, các phương thức ABCI hiện có như
chúng ta biết ngày nay vẫn tồn tại và hoạt động như hiện tại.

### Tích cực

* Ứng dụng giờ đây có toàn quyền kiểm soát thứ tự và mức ưu tiên của giao dịch.
* Đặt nền tảng cho việc tích hợp đầy đủ ABCI 1.0, giúp mở khoá nhiều use-case
  phía ứng dụng quanh xây dựng khối và tích hợp với consensus engine của Tendermint.

### Tiêu cực

* Yêu cầu “mempool”, như một cấu trúc dữ liệu tổng quát để thu thập và lưu trữ
  các giao dịch chưa commit, bị nhân đôi giữa Tendermint và Cosmos SDK.
* Phát sinh thêm các request giữa Tendermint và Cosmos SDK trong bối cảnh thực thi khối.
  Tuy vậy, overhead dự kiến là không đáng kể.
* Không tương thích ngược với các phiên bản trước của Tendermint và Cosmos SDK.

## Thảo luận thêm

Có thể thiết kế triển khai phía ứng dụng của `Mempool[T MempoolTx]` theo nhiều
cách khác nhau bằng các cấu trúc dữ liệu và cài đặt khác nhau, mỗi cách có các
đánh đổi riêng. Giải pháp đề xuất giữ mọi thứ đơn giản và bao phủ các trường hợp
cần thiết cho hầu hết ứng dụng cơ bản. Có thể thực hiện các đánh đổi để cải thiện
hiệu năng của việc reaping và inserting vào triển khai mempool được cung cấp.

## Tham khảo

* https://github.com/tendermint/tendermint/blob/master/spec/abci%2B%2B/README.md
* [1] https://github.com/tendermint/tendermint/issues/7750#issuecomment-1076806155
* [2] https://github.com/tendermint/tendermint/issues/7750#issuecomment-1075717151

