# Store

Gói store định nghĩa các interface, kiểu và abstraction cho các module Cosmos SDK để đọc và ghi vào trạng thái Merkleized trong ứng dụng Cosmos SDK. Gói store cung cấp nhiều primitive cho các nhà phát triển sử dụng để làm việc với cả lưu trữ trạng thái và cam kết trạng thái. Dưới đây chúng ta mô tả các abstraction khác nhau.

## Các kiểu

### `Store`

Phần lớn các interface store được định nghĩa [tại đây](https://github.com/cosmos/cosmos-sdk/blob/main/store/types/store.go), nơi interface primitive cơ sở, mà các interface khác xây dựng dựa trên, là kiểu `Store`. Interface `Store` định nghĩa khả năng cho biết loại store triển khai và khả năng cache wrap thông qua interface `CacheWrapper`.

### `CacheWrapper` & `CacheWrap`

Một trong những tính năng quan trọng nhất mà store có khả năng thực hiện là khả năng cache wrap. Cache wrapping về cơ bản là store cơ sở bọc chính nó trong một kiểu store khác thực hiện caching cho cả đọc và ghi với khả năng flush ghi qua `Write()`.

### `KVStore` & `CacheKVStore`

Một trong những interface quan trọng nhất mà cả nhà phát triển và module tương tác, cũng cung cấp cơ sở cho hầu hết các thao tác lưu trữ trạng thái và cam kết, là `KVStore`. Interface `KVStore` cung cấp khả năng CRUD cơ bản và lặp dựa trên prefix, bao gồm lặp ngược.

Thông thường, mỗi module có instance `KVStore` riêng, mà nó có thể truy cập qua `sdk.Context` và việc sử dụng khóa có tên dựa trên con trỏ -- `KVStoreKey`. `KVStoreKey` cung cấp pseudo-OCAP. Cách chính xác `KVStoreKey` ánh xạ tới `KVStore` sẽ được minh họa bên dưới thông qua `CommitMultiStore`.

Lưu ý, `KVStore` không thể trực tiếp commit trạng thái. Thay vào đó, `KVStore` có thể được bọc bởi `CacheKVStore` mở rộng `KVStore` và cung cấp khả năng cho người gọi thực thi `Write()` để commit trạng thái vào lưu trữ trạng thái cơ sở. Lưu ý, điều này không thực sự flush ghi vào đĩa vì các ghi được giữ trong bộ nhớ cho đến khi `Commit()` được gọi trên `CommitMultiStore`.

### `CommitMultiStore`

Interface `CommitMultiStore` hiển thị interface cấp cao nhất được sử dụng để quản lý cam kết và lưu trữ trạng thái bởi ứng dụng SDK và abstract khái niệm nhiều `KVStore` được sử dụng bởi nhiều module. Cụ thể, nó hỗ trợ các primitive cấp cao sau:

* Cho phép người gọi truy xuất `KVStore` bằng cách cung cấp `KVStoreKey`.
* Hiển thị cơ chế pruning để loại bỏ trạng thái được ghim với height/version cụ thể trong quá khứ.
* Cho phép tải lưu trữ trạng thái tại height/version cụ thể trong quá khứ để cung cấp truy vấn head hiện tại và lịch sử.
* Cung cấp khả năng rollback trạng thái về height/version trước đó.
* Cung cấp khả năng tải lưu trữ trạng thái tại height/version cụ thể đồng thời thực hiện nâng cấp store, được sử dụng trong quá trình di chuyển trạng thái ứng dụng hard-fork trực tiếp.
* Cung cấp khả năng commit tất cả trạng thái tích lũy hiện tại vào đĩa và thực hiện cam kết Merkle.

## Chi tiết triển khai

Trong khi có nhiều interface mà gói `store` cung cấp, thông thường có một triển khai cốt lõi cho mỗi interface chính mà các module và nhà phát triển tương tác được định nghĩa trong Cosmos SDK.

### `iavl.Store`

`iavl.Store` cung cấp triển khai cốt lõi cho lưu trữ và cam kết trạng thái bằng cách triển khai các interface sau:

* `KVStore`
* `CommitStore`
* `CommitKVStore`
* `Queryable`
* `StoreWithInitialVersion`

Nó cho phép tất cả các thao tác CRUD được thực hiện cùng với cho phép truy vấn trạng thái hiện tại và lịch sử, lặp prefix, và cam kết trạng thái cùng với các thao tác Merkle proof. `iavl.Store` cũng cung cấp khả năng loại bỏ trạng thái lịch sử khỏi lớp cam kết trạng thái.

Tổng quan về triển khai IAVL có thể tìm thấy [tại đây](https://github.com/cosmos/iavl/blob/master/docs/overview.md). Điều quan trọng cần lưu ý là IAVL store cung cấp cả cam kết trạng thái và các thao tác lưu trữ logic, điều này đi kèm với nhược điểm vì có nhiều tác động hiệu năng, một số rất rõ rệt, khi nói đến các thao tác đã đề cập ở trên.

Khi xử lý quản lý trạng thái trong các module và client, Cosmos SDK cung cấp các lớp abstraction khác nhau hoặc "store wrapping", trong đó `iavl.Store` là lớp dưới cùng nhất. Khi yêu cầu store thực hiện đọc hoặc ghi trong module, thứ tự lớp abstraction điển hình được định nghĩa như sau:

```text
iavl.Store <- cachekv.Store <- gaskv.Store <- cachemulti.Store <- rootmulti.Store
```

### Sử dụng đồng thời IAVL store

Cây dưới `iavl.Store` không an toàn cho việc sử dụng đồng thời. Trách nhiệm của người gọi là đảm bảo rằng truy cập đồng thời vào store không được thực hiện.

Vấn đề chính với việc sử dụng đồng thời là khi dữ liệu được ghi đồng thời với việc lặp qua nó. Làm như vậy sẽ gây ra lỗi nghiêm trọng không thể khôi phục do đọc và ghi đồng thời vào map nội bộ.

Mặc dù không được khuyến nghị, bạn có thể lặp qua các giá trị trong khi ghi vào nó bằng cách vô hiệu hóa "FastNode" **mà không đảm bảo rằng các giá trị đang được ghi sẽ được trả về trong quá trình lặp** (nếu bạn cần điều này, bạn có thể muốn xem xét lại thiết kế ứng dụng của mình). Điều này được thực hiện bằng cách đặt `iavl-disable-fastnode` thành `true` trong tệp cấu hình TOML.

### `cachekv.Store`

Store `cachekv.Store` bọc `KVStore` cơ sở, thường là `iavl.Store` và chứa cache trong bộ nhớ để lưu trữ các ghi đang chờ vào `KVStore` cơ sở. Các lệnh gọi `Set` và `Delete` được thực thi trên cache trong bộ nhớ, trong khi các lệnh gọi `Has` được proxy tới `KVStore` cơ sở.

Một trong những lệnh gọi quan trọng nhất tới `cachekv.Store` là `Write()`, đảm bảo rằng các cặp key-value được ghi vào `KVStore` cơ sở theo cách xác định và có thứ tự bằng cách sắp xếp các khóa trước. Store theo dõi các khóa "dirty" và sử dụng chúng để xác định khóa nào cần sắp xếp. Ngoài ra, nó cũng theo dõi các khóa đã xóa và đảm bảo chúng cũng được loại bỏ khỏi `KVStore` cơ sở.

`cachekv.Store` cũng cung cấp khả năng thực hiện lặp và lặp ngược. Lặp được thực hiện thông qua kiểu `cacheMergeIterator` và sử dụng cả cache dirty và `KVStore` cơ sở để lặp qua các cặp key-value.

Lưu ý, tất cả các lệnh gọi tới các thao tác CRUD và lặp trên `cachekv.Store` đều an toàn luồng.

### `gaskv.Store`

Store `gaskv.Store` cung cấp triển khai đơn giản của `KVStore`. Cụ thể, nó chỉ bọc `KVStore` hiện có, chẳng hạn như `iavl.Store` được cache wrap, và phát sinh chi phí gas có thể cấu hình cho các thao tác CRUD qua các lệnh gọi `ConsumeGas()` được định nghĩa trên `GasMeter` tồn tại trong `sdk.Context` và sau đó proxy lệnh gọi CRUD cơ sở tới store cơ sở. Lưu ý, `GasMeter` được reset mỗi block.

### `cachemulti.Store` & `rootmulti.Store`

`rootmulti.Store` hoạt động như abstraction xung quanh một chuỗi các store. Cụ thể, nó triển khai interface `CommitMultiStore` và `Queryable`. Thông qua `rootmulti.Store`, module SDK có thể yêu cầu truy cập `KVStore` để thực hiện các thao tác CRUD trạng thái và truy vấn bằng cách giữ quyền truy cập vào `KVStoreKey` duy nhất.

`rootmulti.Store` đảm bảo các truy vấn và thao tác trạng thái này được thực hiện thông qua các instance được cache wrap của `cachekv.Store` được mô tả ở trên. Triển khai `rootmulti.Store` cũng chịu trách nhiệm commit tất cả trạng thái tích lũy từ mỗi `KVStore` vào đĩa và trả về Merkle root trạng thái ứng dụng.

Các truy vấn có thể được thực hiện để trả về dữ liệu trạng thái cùng với các Merkle proof cam kết trạng thái liên quan cho cả height/version trước đó và trạng thái root hiện tại. Các truy vấn được định tuyến dựa trên tên store, tức là module, cùng với các tham số khác được định nghĩa trong `abci.QueryRequest`.

`rootmulti.Store` cũng cung cấp primitive cho việc pruning dữ liệu tại height/version nhất định khỏi lưu trữ trạng thái. Khi một height được commit, `rootmulti.Store` sẽ xác định xem các height trước đó khác có nên được xem xét để loại bỏ dựa trên cài đặt pruning của operator được định nghĩa bởi `PruningOptions`, định nghĩa bao nhiêu phiên bản gần đây cần giữ trên đĩa và khoảng thời gian để loại bỏ các height đã prune "staged" khỏi đĩa. Trong mỗi khoảng thời gian, các height staged được loại bỏ khỏi mỗi `KVStore`. Lưu ý, tùy thuộc vào triển khai `KVStore` cơ sở để xác định cách pruning thực sự được thực hiện. `PruningOptions` được định nghĩa như sau:

```go
type PruningOptions struct {
	// KeepRecent defines how many recent heights to keep on disk.
	KeepRecent uint64

	// Interval defines when the pruned heights are removed from disk.
	Interval uint64

	// Strategy defines the kind of pruning strategy. See below for more information on each.
	Strategy PruningStrategy
}
```

Cosmos SDK định nghĩa một số lượng preset các "chiến lược" pruning: `default`, `everything`, `nothing`, và `custom`.

Điều quan trọng cần lưu ý là `rootmulti.Store` coi mỗi `KVStore` là store logic riêng biệt. Nói cách khác, chúng không chia sẻ Merkle tree hoặc cấu trúc dữ liệu tương đương. Điều này có nghĩa là khi trạng thái được commit qua `rootmulti.Store`, mỗi store được commit tuần tự và do đó không phải là atomic.

Về mặt xây dựng và kết nối store, mỗi ứng dụng Cosmos SDK chứa instance `BaseApp` có tham chiếu nội bộ tới `CommitMultiStore` được triển khai bởi `rootmulti.Store`. Ứng dụng sau đó đăng ký một hoặc nhiều `KVStoreKey` liên quan đến module duy nhất và do đó là `KVStore`. Thông qua việc sử dụng `sdk.Context` và `KVStoreKey`, mỗi module có thể truy cập trực tiếp instance `KVStore` tương ứng của nó.

Ví dụ:

```go
func NewApp(...) Application {
  // ...
  
  bApp := baseapp.NewBaseApp(appName, logger, db, txConfig.TxDecoder(), baseAppOptions...)
  bApp.SetCommitMultiStoreTracer(traceStore)
  bApp.SetVersion(version.Version)
  bApp.SetInterfaceRegistry(interfaceRegistry)

	// ...

  keys := sdk.NewKVStoreKeys(...)
	transientKeys := sdk.NewTransientStoreKeys(...)
	memKeys := sdk.NewMemoryStoreKeys(...)

	// ...

	// initialize stores
	app.MountKVStores(keys)
	app.MountTransientStores(transientKeys)
	app.MountMemoryStores(memKeys)

	// ...
}
```

Bản thân `rootmulti.Store` có thể được cache wrap trả về instance của `cachemulti.Store`. Cho mỗi block, `BaseApp` đảm bảo các abstraction phù hợp được tạo trên `CommitMultiStore`, tức là đảm bảo `rootmulti.Store` được cache wrap và sử dụng `cachemulti.Store` kết quả để đặt trên `sdk.Context` sau đó được sử dụng cho việc thực thi block và giao dịch. Kết quả là, tất cả các thay đổi trạng thái do thực thi block và giao dịch thực sự được giữ tạm thời cho đến khi `Commit()` được gọi bởi ABCI client. Khái niệm này được mở rộng thêm khi AnteHandler được thực thi mỗi giao dịch để đảm bảo trạng thái không được commit cho các giao dịch thất bại CheckTx.
