---
sidebar_position: 1
---

# Store (Kho lưu trữ)

:::note Tóm tắt
Store là một cấu trúc dữ liệu lưu giữ trạng thái của ứng dụng.
:::

:::note Tài liệu cần đọc trước

* [Cấu trúc của một ứng dụng Cosmos SDK](../beginner/00-app-anatomy.md)

:::

## Giới thiệu về Cosmos SDK Stores

Cosmos SDK đi kèm với một tập hợp lớn các store để lưu trữ trạng thái của ứng dụng. Theo mặc định, store chính của các ứng dụng Cosmos SDK là một `multistore`, tức là một store của các store. Các nhà phát triển có thể thêm bất kỳ số lượng key-value store nào vào multistore, tùy thuộc vào nhu cầu của ứng dụng. Multistore tồn tại để hỗ trợ tính module hóa của Cosmos SDK, vì nó cho phép mỗi module khai báo và quản lý tập hợp trạng thái riêng của mình. Các key-value store trong multistore chỉ có thể được truy cập bằng một `key` capability cụ thể, thường được giữ trong [`keeper`](../../build/building-modules/06-keeper.md) của module đã khai báo store.

```text
+-----------------------------------------------------+
|                                                     |
|    +--------------------------------------------+   |
|    |                                            |   |
|    |  KVStore 1 - Quản lý bởi keeper Module 1   |   |
|    |                                            |   |
|    +--------------------------------------------+   |
|                                                     |
|    +--------------------------------------------+   |
|    |                                            |   |
|    |  KVStore 2 - Quản lý bởi keeper Module 2   |   |
|    |                                            |   |
|    +--------------------------------------------+   |
|                                                     |
|    +--------------------------------------------+   |
|    |                                            |   |
|    |  KVStore 3 - Quản lý bởi keeper Module 2   |   |
|    |                                            |   |
|    +--------------------------------------------+   |
|                                                     |
|    +--------------------------------------------+   |
|    |                                            |   |
|    |  KVStore 4 - Quản lý bởi keeper Module 3   |   |
|    |                                            |   |
|    +--------------------------------------------+   |
|                                                     |
|    +--------------------------------------------+   |
|    |                                            |   |
|    |  KVStore 5 - Quản lý bởi keeper Module 4   |   |
|    |                                            |   |
|    +--------------------------------------------+   |
|                                                     |
|                    Main Multistore                  |
|                                                     |
+-----------------------------------------------------+

                   Trạng thái ứng dụng
```

### Interface Store

Ở cấp độ cơ bản nhất, một `store` trong Cosmos SDK là một đối tượng chứa `CacheWrapper` và có phương thức `GetStoreType()`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/store/types/store.go#L17-L20
```

`GetStoreType` là phương thức đơn giản trả về kiểu của store, trong khi `CacheWrapper` là một interface đơn giản triển khai cache đọc store và write branching thông qua phương thức `Write`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/store/types/store.go#L285-L317
```

Branching và cache được sử dụng phổ biến trong Cosmos SDK và bắt buộc phải được triển khai trên mọi kiểu store. Một storage branch tạo ra một nhánh riêng biệt, tạm thời (ephemeral) của store, có thể được truyền xung quanh và cập nhật mà không ảnh hưởng đến store chính bên dưới. Điều này được dùng để kích hoạt các chuyển đổi trạng thái tạm thời có thể bị hoàn nguyên sau này nếu xảy ra lỗi. Đọc thêm trong [context](./02-context.md#Store-branching).

### Commit Store

Commit store là store có khả năng commit các thay đổi được thực hiện lên cây hoặc db bên dưới. Cosmos SDK phân biệt store đơn giản với commit store bằng cách mở rộng các interface store cơ bản với một `Committer`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/store/types/store.go#L34-L38
```

`Committer` là một interface định nghĩa các phương thức để lưu trữ các thay đổi lên đĩa:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/store/types/store.go#L22-L32
```

`CommitID` là một commit xác định (deterministic) của cây trạng thái. Hash của nó được trả về cho consensus engine bên dưới và được lưu trong block header. Lưu ý rằng các interface commit store tồn tại cho nhiều mục đích, một trong số đó là để đảm bảo không phải mọi đối tượng đều có thể commit store. Là một phần của [mô hình object-capabilities](./10-ocap.md) trong Cosmos SDK, chỉ `baseapp` mới có quyền commit store. Ví dụ, đây là lý do tại sao phương thức `ctx.KVStore()` mà các module thường dùng để truy cập store trả về `KVStore` chứ không phải `CommitKVStore`.

Cosmos SDK đi kèm nhiều kiểu store, phổ biến nhất là [`CommitMultiStore`](#multistore), [`KVStore`](#kvstore) và [`GasKv` store](#gaskv-store). [Các kiểu store khác](#other-stores) bao gồm `Transient` và `TraceKV` store.

## Multistore

### Interface Multistore

Mỗi ứng dụng Cosmos SDK lưu giữ một multistore ở cấp gốc (root) để lưu trữ trạng thái. Multistore là một store của các `KVStore` tuân theo interface `Multistore`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/store/types/store.go#L115-L147
```

Nếu tracing được bật, việc branch multistore sẽ trước tiên bọc tất cả `KVStore` bên dưới trong [`TraceKv.Store`](#tracekv-store).

### CommitMultiStore

Kiểu `Multistore` chính được dùng trong Cosmos SDK là `CommitMultiStore`, đây là phần mở rộng của interface `Multistore`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/store/types/store.go#L155-L225
```

Về triển khai cụ thể, [`rootMulti.Store`] là triển khai ưu tiên của interface `CommitMultiStore`.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/store/rootmulti/store.go#L56-L82
```

`rootMulti.Store` là một multistore lớp nền được xây dựng xung quanh một `db`, trên đó nhiều `KVStore` có thể được mount, và là multistore store mặc định được sử dụng trong [`baseapp`](./00-baseapp.md).

### CacheMultiStore

Bất cứ khi nào `rootMulti.Store` cần được branch, một [`cachemulti.Store`](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/store/cachemulti/store.go) được sử dụng.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/store/cachemulti/store.go#L20-L34
```

`cachemulti.Store` branch tất cả các substore (tạo một virtual store cho mỗi substore) trong constructor của nó và giữ chúng trong `Store.stores`. Hơn nữa, nó cache tất cả các query đọc. `Store.GetKVStore()` trả về store từ `Store.stores`, và `Store.Write()` gọi đệ quy `CacheWrap.Write()` trên tất cả các substore.

## KVStore Lớp Nền

### Interface `KVStore` và `CommitKVStore`

`KVStore` là một key-value store đơn giản dùng để lưu trữ và truy xuất dữ liệu. `CommitKVStore` là một `KVStore` cũng triển khai `Committer`. Theo mặc định, các store được mount trong `CommitMultiStore` chính của `baseapp` là `CommitKVStore`. Interface `KVStore` chủ yếu được dùng để hạn chế các module không truy cập được committer.

Các `KVStore` riêng lẻ được các module sử dụng để quản lý một tập hợp con của trạng thái toàn cục. `KVStore` có thể được truy cập bởi các đối tượng nắm giữ một `key` cụ thể. `key` này chỉ nên được tiết lộ cho [`keeper`](../../build/building-modules/06-keeper.md) của module định nghĩa store.

`CommitKVStore` được khai báo thông qua `key` tương ứng của chúng và được mount trên [multistore](#multistore) của ứng dụng trong [file ứng dụng chính](../beginner/00-app-anatomy.md#core-application-file). Trong cùng file, `key` cũng được truyền đến `keeper` của module chịu trách nhiệm quản lý store.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/store/types/store.go#L227-L264
```

Ngoài các phương thức `Get` và `Set` truyền thống mà `KVStore` phải triển khai thông qua interface `BasicKVStore`, một `KVStore` phải cung cấp phương thức `Iterator(start, end)` trả về một đối tượng `Iterator`. Nó được dùng để duyệt qua một dải các key, thường là các key có chung tiền tố. Dưới đây là ví dụ từ keeper của module bank, dùng để duyệt qua tất cả số dư tài khoản:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/x/bank/keeper/view.go#L121-L137
```

### `IAVL` Store

Triển khai mặc định của `KVStore` và `CommitKVStore` được dùng trong `baseapp` là `iavl.Store`.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/store/iavl/store.go#L36-L41
```

`iavl` store dựa trên [IAVL Tree](https://github.com/cosmos/iavl), một cây nhị phân tự cân bằng (self-balancing binary tree) đảm bảo rằng:

* Các thao tác `Get` và `Set` là O(log n), trong đó n là số phần tử trong cây.
* Iteration trả về các phần tử đã được sắp xếp trong khoảng một cách hiệu quả.
* Mỗi phiên bản cây là bất biến và có thể được truy xuất ngay cả sau khi commit (tùy thuộc vào cài đặt pruning).

Tài liệu về IAVL Tree nằm [ở đây](https://github.com/cosmos/iavl/blob/master/docs/overview.md).

### `DbAdapter` Store

`dbadapter.Store` là một adapter cho `dbm.DB` giúp nó thỏa mãn interface `KVStore`.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/store/dbadapter/store.go#L13-L16
```

`dbadapter.Store` nhúng `dbm.DB`, nghĩa là hầu hết các hàm interface `KVStore` được triển khai sẵn. Các hàm còn lại (chủ yếu là các hàm linh tinh) được triển khai thủ công. Store này chủ yếu được dùng trong [Transient Stores](#transient-store).

### `Transient` Store

`Transient.Store` là một `KVStore` lớp nền tự động bị hủy vào cuối block.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/store/transient/store.go#L16-L19
```

`Transient.Store` là một `dbadapter.Store` với `dbm.NewMemDB()`. Tất cả các phương thức `KVStore` được tái sử dụng. Khi `Store.Commit()` được gọi, một `dbadapter.Store` mới được gán, loại bỏ tham chiếu trước đó và làm cho nó bị garbage collected.

Kiểu store này hữu ích để lưu giữ thông tin chỉ liên quan trong phạm vi một block. Một ví dụ là lưu các thay đổi tham số (ví dụ: bool được đặt thành `true` nếu một tham số thay đổi trong block).

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/x/params/types/subspace.go#L22-L32
```

Transient store thường được truy cập thông qua [`context`](./02-context.md) bằng phương thức `TransientStore()`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/types/context.go#L347-L350
```

## KVStore Wrappers

### CacheKVStore

`cachekv.Store` là một `KVStore` wrapper cung cấp chức năng ghi có buffer / đọc có cache trên `KVStore` bên dưới.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/store/cachekv/store.go#L26-L36
```

Đây là kiểu được dùng bất cứ khi nào một IAVL Store cần được branch để tạo store riêng biệt (thường khi cần thay đổi trạng thái có thể bị hoàn nguyên sau này).

#### `Get`

`Store.Get()` trước tiên kiểm tra xem `Store.cache` có giá trị liên kết với key không. Nếu giá trị tồn tại, hàm trả về nó. Nếu không, hàm gọi `Store.parent.Get()`, cache kết quả trong `Store.cache`, và trả về nó.

#### `Set`

`Store.Set()` đặt cặp key-value vào `Store.cache`. `cValue` có trường `dirty bool` cho biết giá trị được cache có khác với giá trị bên dưới không. Khi `Store.Set()` cache một cặp mới, `cValue.dirty` được đặt thành `true` để khi `Store.Write()` được gọi, nó có thể được ghi vào store bên dưới.

#### `Iterator`

`Store.Iterator()` phải duyệt qua cả các mục đã được cache và các mục gốc. Trong `Store.iterator()`, hai iterator được tạo cho mỗi cái, và được hợp nhất. `memIterator` về cơ bản là một slice của `KVPairs`, dùng cho các mục đã cache. `mergeIterator` là sự kết hợp của hai iterator, nơi việc duyệt diễn ra theo thứ tự trên cả hai iterator.

### `GasKv` Store

Các ứng dụng Cosmos SDK sử dụng [`gas`](../beginner/04-gas-fees.md) để theo dõi mức sử dụng tài nguyên và ngăn chặn spam. [`GasKv.Store`](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/store/gaskv/store.go) là một `KVStore` wrapper cho phép tự động tiêu thụ gas mỗi khi có thao tác đọc hoặc ghi vào store. Đây là giải pháp ưu tiên để theo dõi mức sử dụng bộ nhớ trong các ứng dụng Cosmos SDK.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/store/gaskv/store.go#L11-L17
```

Khi các phương thức của `KVStore` cha được gọi, `GasKv.Store` tự động tiêu thụ lượng gas phù hợp tùy thuộc vào `Store.gasConfig`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/store/types/gas.go#L219-L228
```

Theo mặc định, tất cả `KVStore` đều được bọc trong `GasKv.Stores` khi được truy xuất. Điều này được thực hiện trong phương thức `KVStore()` của [`context`](./02-context.md):

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/types/context.go#L342-L345
```

Trong trường hợp này, cấu hình gas được đặt trong `context` được sử dụng. Cấu hình gas có thể được đặt bằng phương thức `WithKVGasConfig` của `context`. Nếu không, nó sẽ sử dụng giá trị mặc định sau:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/store/types/gas.go#L230-L241
```

### `TraceKv` Store

`tracekv.Store` là một `KVStore` wrapper cung cấp chức năng tracing thao tác trên `KVStore` bên dưới. Nó được Cosmos SDK tự động áp dụng trên tất cả `KVStore` nếu tracing được bật trên `MultiStore` cha.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/store/tracekv/store.go#L20-L43
```

Khi mỗi phương thức `KVStore` được gọi, `tracekv.Store` tự động ghi `traceOperation` vào `Store.writer`. `traceOperation.Metadata` được điền với `Store.context` khi nó không nil. `TraceContext` là một `map[string]interface{}`.

### `Prefix` Store

`prefix.Store` là một `KVStore` wrapper cung cấp chức năng tự động thêm tiền tố key trên `KVStore` bên dưới.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/store/prefix/store.go#L15-L21
```

Khi `Store.{Get, Set}()` được gọi, store chuyển tiếp lời gọi đến cha của nó, với key được thêm tiền tố `Store.prefix`.

Khi `Store.Iterator()` được gọi, nó không đơn giản thêm tiền tố `Store.prefix`, vì điều đó không hoạt động như mong đợi. Trong trường hợp đó, một số phần tử được duyệt qua ngay cả khi chúng không bắt đầu bằng tiền tố.

### `ListenKv` Store

`listenkv.Store` là một `KVStore` wrapper cung cấp khả năng lắng nghe trạng thái (state listening) trên `KVStore` bên dưới. Nó được Cosmos SDK tự động áp dụng trên bất kỳ `KVStore` nào có `StoreKey` được chỉ định trong cấu hình state streaming. Thông tin bổ sung về cấu hình state streaming có thể tìm thấy trong [store/streaming/README.md](https://github.com/cosmos/cosmos-sdk/tree/v0.53.0/store/streaming).

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/store/listenkv/store.go#L11-L18
```

Khi các phương thức `KVStore.Set` hoặc `KVStore.Delete` được gọi, `listenkv.Store` tự động ghi các thao tác vào tập hợp `Store.listeners`.

## Interface `BasicKVStore`

Một interface chỉ cung cấp các chức năng CRUD cơ bản (các phương thức `Get`, `Set`, `Has` và `Delete`), không có iteration hay caching. Interface này được dùng để tiết lộ một phần các thành phần của một store lớn hơn.
