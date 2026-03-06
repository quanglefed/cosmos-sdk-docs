# ADR 035: Hỗ Trợ Rosetta API

## Tác Giả

* Jonathan Gimeno (@jgimeno)
* David Grierson (@senormonito)
* Alessio Treglia (@alessio)
* Frojdy Dymylja (@fdymylja)

## Nhật Ký Thay Đổi

* 12-05-2021: thư viện bên ngoài [cosmos-rosetta-gateway](https://github.com/tendermint/cosmos-rosetta-gateway) đã được chuyển vào Cosmos SDK.

## Bối Cảnh

[Rosetta API](https://www.rosetta-api.org/) là một đặc tả mã nguồn mở và bộ công cụ được phát triển bởi Coinbase để tiêu chuẩn hóa các tương tác blockchain.

Thông qua việc sử dụng API tiêu chuẩn để tích hợp các ứng dụng blockchain, nó sẽ:

* Giúp người dùng dễ dàng tương tác với blockchain đã cho hơn
* Cho phép các sàn giao dịch tích hợp các blockchain mới nhanh chóng và dễ dàng
* Cho phép các nhà phát triển ứng dụng xây dựng các ứng dụng cross-blockchain như trình khám phá block, ví và dApp với chi phí và công sức thấp hơn đáng kể.

## Quyết Định

Rõ ràng rằng việc thêm hỗ trợ Rosetta API vào Cosmos SDK sẽ mang lại giá trị cho tất cả các nhà phát triển và các chain dựa trên Cosmos SDK trong hệ sinh thái. Cách triển khai là yếu tố then chốt.

Các nguyên tắc chính của thiết kế được đề xuất là:

1. **Khả năng mở rộng:** việc thiết lập các cấu hình mạng để lộ các dịch vụ tuân thủ Rosetta API phải ít rủi ro và ít phức tạp nhất có thể cho các nhà phát triển ứng dụng.
2. **Hỗ trợ dài hạn:** Đề xuất này hướng tới cung cấp hỗ trợ cho tất cả các chuỗi phát hành Cosmos SDK.
3. **Hiệu quả chi phí:** Việc backport các thay đổi theo đặc tả Rosetta API từ `master` tới các nhánh ổn định khác nhau của Cosmos SDK là một chi phí cần được giảm thiểu.

Chúng ta sẽ đạt được điều này bằng cách cung cấp những nguyên tắc sau:

1. Sẽ có một gói `rosetta/lib` để triển khai các tính năng Rosetta API cốt lõi, cụ thể là:
   a. Các kiểu và interface (`Client`, `OfflineClient`...), điều này tách biệt thiết kế khỏi chi tiết triển khai.
   b. Chức năng `Server` vì nó độc lập với phiên bản Cosmos SDK.
   c. `Online/OfflineNetwork`, không được export, và triển khai API rosetta sử dụng interface `Client` để truy vấn node, xây dựng tx và các thao tác khác.
   d. Gói `errors` để mở rộng lỗi rosetta.
2. Do sự khác biệt giữa các chuỗi phát hành Cosmos, mỗi chuỗi sẽ có triển khai riêng của interface `Client`.
3. Sẽ có hai tùy chọn để khởi động dịch vụ API trong các ứng dụng:
   a. API chia sẻ tiến trình với ứng dụng
   b. Tiến trình riêng dành cho API.

## Kiến Trúc

### Repo Bên Ngoài

Phần này sẽ mô tả thư viện bên ngoài được đề xuất, bao gồm triển khai service, cộng với các kiểu và interface được định nghĩa.

#### Server

`Server` là một `struct` đơn giản được khởi động và lắng nghe trên cổng được chỉ định trong các thiết lập. Điều này được thiết kế để sử dụng trên tất cả các phiên bản Cosmos SDK đang được hỗ trợ tích cực.

Constructor như sau:

`func NewServer(settings Settings) (Server, error)`

`Settings`, được sử dụng để xây dựng server mới, như sau:

```go
// Settings định nghĩa các thiết lập server rosetta
type Settings struct {
	// Network chứa thông tin về mạng
	Network *types.NetworkIdentifier
	// Client là handler API trực tuyến
	Client crgtypes.Client
	// Listen là địa chỉ mà handler sẽ lắng nghe
	Listen string
	// Offline xác định nếu service rosetta nên được lộ ở chế độ ngoại tuyến
	Offline bool
	// Retries là số lần kiểm tra sẵn sàng sẽ được thực hiện khi khởi tạo handler
	// chỉ hợp lệ cho API trực tuyến
	Retries int
	// RetryWait là thời gian sẽ chờ giữa các lần thử
	RetryWait time.Duration
}
```

#### Các Kiểu

Gói types sử dụng hỗn hợp các kiểu rosetta và các wrapper kiểu tùy chỉnh được định nghĩa, mà client phải phân tích và trả về trong khi thực thi các hoạt động.

##### Interface

Mỗi phiên bản SDK sử dụng định dạng khác nhau để kết nối (rpc, gRPC, v.v.), truy vấn và xây dựng giao dịch, chúng ta đã trừu tượng hóa điều này trong interface `Client`. Client sử dụng các kiểu rosetta, trong khi `Online/OfflineNetwork` đảm nhận việc trả về các phản hồi và lỗi rosetta được phân tích đúng cách.

Mỗi chuỗi phát hành Cosmos SDK sẽ có triển khai `Client` riêng của họ. Các nhà phát triển có thể triển khai các `Client` tùy chỉnh của riêng họ khi cần.

```go
// Client định nghĩa API mà triển khai client nên cung cấp.
type Client interface {
	// Cần thiết nếu client cần thực hiện một số hành động trước khi kết nối.
	Bootstrap() error
	// Ready kiểm tra xem các ràng buộc servicer cho các truy vấn có được thỏa mãn không
	Ready() error

	// Data API
	Balances(ctx context.Context, addr string, height *int64) ([]*types.Amount, error)
	BlockByHash(ctx context.Context, hash string) (BlockResponse, error)
	BlockByHeight(ctx context.Context, height *int64) (BlockResponse, error)
	BlockTransactionsByHash(ctx context.Context, hash string) (BlockTransactionsResponse, error)
	BlockTransactionsByHeight(ctx context.Context, height *int64) (BlockTransactionsResponse, error)
	GetTx(ctx context.Context, hash string) (*types.Transaction, error)
	GetUnconfirmedTx(ctx context.Context, hash string) (*types.Transaction, error)
	Mempool(ctx context.Context) ([]*types.TransactionIdentifier, error)
	Peers(ctx context.Context) ([]*types.Peer, error)
	Status(ctx context.Context) (*types.SyncStatus, error)

	// Construction API
	PostTx(txBytes []byte) (res *types.TransactionIdentifier, meta map[string]interface{}, err error)
	ConstructionMetadataFromOptions(ctx context.Context, options map[string]interface{}) (meta map[string]interface{}, err error)
	OfflineClient
}

// OfflineClient định nghĩa các chức năng được hỗ trợ mà không có quyền truy cập vào node
type OfflineClient interface {
	NetworkInformationProvider
	SignedTx(ctx context.Context, txBytes []byte, sigs []*types.Signature) (signedTxBytes []byte, err error)
	TxOperationsAndSignersAccountIdentifiers(signed bool, hexBytes []byte) (ops []*types.Operation, signers []*types.AccountIdentifier, err error)
	ConstructionPayload(ctx context.Context, req *types.ConstructionPayloadsRequest) (resp *types.ConstructionPayloadsResponse, err error)
	PreprocessOperationsToOptions(ctx context.Context, req *types.ConstructionPreprocessRequest) (options map[string]interface{}, err error)
	AccountIdentifierFromPublicKey(pubKey *types.PublicKey) (*types.AccountIdentifier, error)
}
```

### 2. Triển Khai Cosmos SDK

Triển khai Cosmos SDK, dựa theo phiên bản, đảm nhận việc thỏa mãn interface `Client`. Trong Stargate, Launchpad và 0.37, chúng ta đã giới thiệu khái niệm rosetta.Msg, message này không nằm trong repo được chia sẻ vì kiểu sdk.Msg khác nhau giữa các phiên bản Cosmos SDK.

Interface rosetta.Msg như sau:

```go
// Msg đại diện cho một cosmos-sdk message có thể được chuyển đổi từ và sang một rosetta operation.
type Msg interface {
	sdk.Msg
	ToOperations(withStatus, hasError bool) []*types.Operation
	FromOperations(ops []*types.Operation) (sdk.Msg, error)
}
```

Do đó, các nhà phát triển muốn mở rộng tập hợp các rosetta operation được hỗ trợ chỉ cần mở rộng các sdk.Msg của module của họ với các phương thức `ToOperations` và `FromOperations`.

### 3. Kích Hoạt Dịch Vụ API

Như đã nêu ở đầu, các nhà phát triển ứng dụng sẽ có hai phương thức để kích hoạt dịch vụ Rosetta API:

1. Tiến trình được chia sẻ cho cả ứng dụng và API
2. Dịch vụ API độc lập

#### Tiến Trình Được Chia Sẻ (Chỉ Stargate)

Dịch vụ Rosetta API có thể chạy trong cùng tiến trình thực thi với ứng dụng. Điều này sẽ được kích hoạt thông qua thiết lập app.toml, và nếu gRPC không được bật, thể hiện rosetta sẽ được khởi động ở chế độ ngoại tuyến (chỉ có khả năng xây dựng tx).

#### Dịch Vụ API Riêng Biệt

Các nhà phát triển ứng dụng client cũng có thể viết một lệnh mới để khởi động một Rosetta API server như một tiến trình riêng biệt, sử dụng lệnh rosetta có trong gói `/server/rosetta`. Việc xây dựng lệnh phụ thuộc vào phiên bản Cosmos SDK. Các ví dụ có thể được tìm thấy trong `simd` cho stargate, và `contrib/rosetta/simapp` cho các chuỗi phát hành khác.

## Trạng Thái

Đề Xuất

## Hậu Quả

### Tích Cực

* Hỗ trợ Rosetta API tích hợp sẵn trong Cosmos SDK.
* Tiêu chuẩn hóa giao diện blockchain

## Tham Khảo

* https://www.rosetta-api.org/
