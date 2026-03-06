# Inter-block Cache

* [Inter-block Cache](#inter-block-cache)
 * [Tóm tắt](#synopsis)
 * [Tổng quan và khái niệm cơ bản](#overview-and-basic-concepts)
 * [Động lực](#motivation)
 * [Định nghĩa](#definitions)
 * [Mô hình hệ thống và thuộc tính](#system-model-and-properties)
 * [Giả định](#assumptions)
 * [Thuộc tính](#properties)
 * [An toàn luồng](#thread-safety)
 * [Khôi phục sau sự cố](#crash-recovery)
 * [Lặp](#iteration)
 * [Đặc tả kỹ thuật](#technical-specification)
 * [Thiết kế chung](#general-design)
 * [API](#api)
 * [CommitKVCacheManager](#commitkvcachemanager)
 * [CommitKVStoreCache](#commitkvstorecache)
 * [Chi tiết triển khai](#implementation-details)
 * [Lịch sử](#history)
 * [Bản quyền](#copyright)

## Tóm tắt {#synopsis}

Inter-block cache là cache trong bộ nhớ lưu trữ (trong hầu hết các trường hợp) trạng thái bất biến mà các module cần đọc giữa các block. Khi được bật, tất cả các sub-store của multi store, ví dụ `rootmulti`, được bọc.

## Tổng quan và khái niệm cơ bản {#overview-and-basic-concepts}

### Động lực {#motivation}

Mục tiêu của inter-block cache là cho phép các module SDK có truy cập nhanh vào dữ liệu thường được truy vấn trong quá trình thực thi mỗi block. Đây là dữ liệu không thay đổi thường xuyên, ví dụ tham số module. Inter-block cache bọc mỗi `CommitKVStore` của multi store như `rootmulti` bằng cache write-through kích thước cố định. Các cache không được xóa sau khi block được commit, khác với các lớp cache khác như `cachekv`.

### Định nghĩa {#definitions}

* `Store key` xác định duy nhất một store.
* `KVCache` là `CommitKVStore` được bọc với cache.
* `Cache manager` là thành phần chính của inter-block cache chịu trách nhiệm duy trì map từ `store keys` tới `KVCaches`.

## Mô hình hệ thống và thuộc tính {#system-model-and-properties}

### Giả định {#assumptions}

Đặc tả này giả định rằng tồn tại triển khai cache có thể truy cập bởi tính năng inter-block cache.

> Triển khai sử dụng adaptive replacement cache (ARC), cải tiến so với cache last-recently-used (LRU) chuẩn ở chỗ theo dõi cả tần suất và độ mới sử dụng.

Inter-block cache yêu cầu triển khai cache cung cấp phương thức để tạo cache, thêm cặp key/value, xóa cặp key/value và truy xuất giá trị liên kết với khóa. Trong đặc tả này, chúng ta giả định tính năng `Cache` cung cấp chức năng này thông qua các phương thức sau:

* `NewCache(size int)` tạo cache mới với dung lượng `size` và trả về nó.
* `Get(key string)` cố gắng truy xuất cặp key/value từ `Cache`. Trả về `(value []byte, success bool)`. Nếu `Cache` chứa khóa, `value` chứa giá trị liên kết và `success=true`. Nếu không, `success=false` và `value` nên bị bỏ qua.
* `Add(key string, value []byte)` chèn cặp key/value vào `Cache`.
* `Remove(key string)` xóa cặp key/value được xác định bởi `key` khỏi `Cache`.

Đặc tả cũng giả định `CommitKVStore` cung cấp API sau:

* `Get(key string)` cố gắng truy xuất cặp key/value từ `CommitKVStore`.
* `Set(key, string, value []byte)` chèn cặp key/value vào `CommitKVStore`.
* `Delete(key string)` xóa cặp key/value được xác định bởi `key` khỏi `CommitKVStore`.

> Lý tưởng nhất, cả `Cache` và `CommitKVStore` nên được đặc tả trong tài liệu khác và được tham chiếu ở đây.

### Thuộc tính {#properties}

#### An toàn luồng {#thread-safety}

Truy cập `cache manager` hoặc `KVCache` không an toàn luồng: không có phương thức nào được bảo vệ bằng khóa.
Lưu ý rằng điều này đúng ngay cả khi triển khai cache an toàn luồng.

> Ví dụ, giả sử hai thao tác `Set` được thực thi đồng thời trên cùng một khóa, mỗi thao tác ghi giá trị khác nhau. Sau khi cả hai được thực thi, cache và store cơ sở có thể không nhất quán, mỗi cái lưu trữ giá trị khác nhau dưới cùng một khóa.

#### Khôi phục sau sự cố {#crash-recovery}

Inter-block cache minh bạch ủy quyền `Commit()` cho `CommitKVStore` tổng hợp của nó. Nếu `CommitKVStore` tổng hợp hỗ trợ ghi atomic và sử dụng chúng để đảm bảo trạng thái store luôn nhất quán trên đĩa, inter-block cache có thể được chuyển minh bạch sang trạng thái nhất quán khi xảy ra sự cố.

> Lưu ý rằng đây là trường hợp của `IAVLStore`, `CommitKVStore` được ưu tiên. Khi commit, nó gọi `SaveVersion()` trên `MutableTree` cơ sở. `SaveVersion` ghi vào đĩa là atomic qua batching. Điều này có nghĩa là chỉ các phiên bản nhất quán của store (cây) được ghi vào đĩa. Do đó, trong trường hợp sự cố trong lệnh gọi `SaveVersion`, khi khôi phục từ đĩa, phiên bản của store sẽ nhất quán.

#### Lặp {#iteration}

Lặp qua mỗi store được bọc được hỗ trợ thông qua interface `CommitKVStore` nhúng.

## Đặc tả kỹ thuật {#technical-specification}

### Thiết kế chung {#general-design}

Tính năng inter-block cache được cấu thành bởi hai thành phần: `CommitKVCacheManager` và `CommitKVCache`.

`CommitKVCacheManager` triển khai cache manager. Nó duy trì ánh xạ từ store key tới `KVStore`.

```go
type CommitKVStoreCacheManager interface{
    cacheSize uint
    caches map[string]CommitKVStore
}
```

`CommitKVStoreCache` triển khai `KVStore`: cache write-through bọc `CommitKVStore`. Điều này có nghĩa là xóa và ghi luôn xảy ra với cả cache và `CommitKVStore` cơ sở. Đọc mặt khác trước tiên truy cập cache nội bộ. Trong cache miss, đọc được ủy quyền cho `CommitKVStore` cơ sở và được cache.

```go
type CommitKVStoreCache interface{
    store CommitKVStore
    cache Cache
}
```

Để bật inter-block cache trên `rootmulti`, người ta cần khởi tạo `CommitKVCacheManager` và đặt nó bằng cách gọi `SetInterBlockCache()` trước khi gọi một trong `LoadLatestVersion()`, `LoadLatestVersionAndUpgrade(...)`, `LoadVersionAndUpgrade(...)` và `LoadVersion(version)`.

### API {#api}

#### CommitKVCacheManager {#commitkvcachemanager}

Phương thức `NewCommitKVStoreCacheManager` tạo cache manager mới và trả về nó.

| Tên | Kiểu | Mô tả |
| ------------- | ---------|------- |
| size | integer | Xác định dung lượng của mỗi KVCache được duy trì bởi manager |

```go
func NewCommitKVStoreCacheManager(size uint) CommitKVStoreCacheManager {
    manager = CommitKVStoreCacheManager{size, make(map[string]CommitKVStore)}
    return manager
}
```

`GetStoreCache` trả về cache từ CommitStoreCacheManager cho store key đã cho. Nếu không tồn tại cache cho store key, thì một cache được tạo và đặt.

| Tên | Kiểu | Mô tả |
| ------------- | ---------|------- |
| manager | `CommitKVStoreCacheManager` | Cache manager |
| storeKey | string | Store key của store đang được truy xuất |
| store | `CommitKVStore` | Store được cache trong trường hợp manager không có bất kỳ trong map caches của nó |

```go
func GetStoreCache(
    manager CommitKVStoreCacheManager,
    storeKey string,
    store CommitKVStore) CommitKVStore {

    if manager.caches.has(storeKey) {
        return manager.caches.get(storeKey)
    } else {
        cache = CommitKVStoreCacheManager{store, manager.cacheSize}
        manager.set(storeKey, cache)
        return cache
    }
}
```

`Unwrap` trả về CommitKVStore cơ sở cho store key đã cho.

| Tên | Kiểu | Mô tả |
| ------------- | ---------|------- |
| manager | `CommitKVStoreCacheManager` | Cache manager |
| storeKey | string | Store key của store đang được unwrap |

```go
func Unwrap(
    manager CommitKVStoreCacheManager,
    storeKey string) CommitKVStore {

    if manager.caches.has(storeKey) {
        cache = manager.caches.get(storeKey)
        return cache.store
    } else {
        return nil
    }
}
```

`Reset` đặt lại map caches của manager.

| Tên | Kiểu | Mô tả |
| ------------- | ---------|------- |
| manager | `CommitKVStoreCacheManager` | Cache manager |

```go
function Reset(manager CommitKVStoreCacheManager) {

    for (let storeKey of manager.caches.keys()) {
        manager.caches.delete(storeKey)
    }
}
```

#### CommitKVStoreCache {#commitkvstorecache}

`NewCommitKVStoreCache` tạo `CommitKVStoreCache` mới và trả về nó.

| Tên | Kiểu | Mô tả |
| ------------- | ---------|------- |
| store | CommitKVStore | Store cần được cache |
| size | string | Xác định dung lượng của cache đang được tạo |

```go
func NewCommitKVStoreCache(
    store CommitKVStore,
    size uint) CommitKVStoreCache {
    KVCache = CommitKVStoreCache{store, NewCache(size)}
    return KVCache
}
```

`Get` truy xuất giá trị theo khóa. Trước tiên truy cập cache. Nếu khóa không có trong cache, truy vấn được ủy quyền cho `CommitKVStore` cơ sở. Trong trường hợp sau, cặp key/value được cache. Phương thức trả về giá trị.

| Tên | Kiểu | Mô tả |
| ------------- | ---------|------- |
| KVCache | `CommitKVStoreCache` | `CommitKVStoreCache` mà cặp key/value được truy xuất |
| key | string | Khóa của cặp key/value đang được truy xuất |

```go
func Get(
    KVCache CommitKVStoreCache,
    key string) []byte {
    valueCache, success := KVCache.cache.Get(key)
    if success {
        // cache hit
        return valueCache
    } else {
        // cache miss
        valueStore = KVCache.store.Get(key)
        KVCache.cache.Add(key, valueStore)
        return valueStore
    }
}
```

`Set` chèn cặp key/value vào cả cache write-through và `CommitKVStore` cơ sở.

| Tên | Kiểu | Mô tả |
| ------------- | ---------|------- |
| KVCache | `CommitKVStoreCache` | `CommitKVStoreCache` mà cặp key/value được chèn vào |
| key | string | Khóa của cặp key/value đang được chèn |
| value | []byte | Giá trị của cặp key/value đang được chèn |

```go
func Set(
    KVCache CommitKVStoreCache,
    key string,
    value []byte) {

    KVCache.cache.Add(key, value)
    KVCache.store.Set(key, value)
}
```

`Delete` xóa cặp key/value khỏi cả cache write-through và `CommitKVStore` cơ sở.

| Tên | Kiểu | Mô tả |
| ------------- | ---------|------- |
| KVCache | `CommitKVStoreCache` | `CommitKVStoreCache` mà cặp key/value được xóa khỏi |
| key | string | Khóa của cặp key/value đang được xóa |

```go
func Delete(
    KVCache CommitKVStoreCache,
    key string) {

    KVCache.cache.Remove(key)
    KVCache.store.Delete(key)
}
```

`CacheWrap` bọc `CommitKVStoreCache` với lớp cache khác (`CacheKV`).

> Chưa rõ liệu có trường hợp sử dụng cho `CacheWrap` hay không.

| Tên | Kiểu | Mô tả |
| ------------- | ---------|------- |
| KVCache | `CommitKVStoreCache` | `CommitKVStoreCache` đang được bọc |

```go
func CacheWrap(
    KVCache CommitKVStoreCache) {
     
    return CacheKV.NewStore(KVCache)
}
```

### Chi tiết triển khai {#implementation-details}

Triển khai inter-block cache sử dụng adaptive replacement cache (ARC) kích thước cố định làm cache. [Triển khai ARC](https://github.com/hashicorp/golang-lru/blob/main/arc/arc.go) an toàn luồng. ARC là cải tiến so với cache LRU chuẩn ở chỗ theo dõi cả tần suất và độ mới sử dụng. Điều này tránh việc burst truy cập vào các mục mới đẩy các mục cũ thường dùng ra. Nó thêm một số chi phí theo dõi bổ sung vào cache LRU chuẩn, về mặt tính toán chi phí khoảng `2x`, và chi phí bộ nhớ bổ sung tuyến tính với kích thước cache. Kích thước cache mặc định là `1000`.

## Lịch sử {#history}

20 tháng 12, 2022 - Bản nháp ban đầu hoàn thành và gửi dưới dạng PR

## Bản quyền {#copyright}

Tất cả nội dung ở đây được cấp phép theo [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0).
