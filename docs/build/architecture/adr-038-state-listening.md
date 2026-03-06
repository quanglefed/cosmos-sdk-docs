# ADR 038: Lắng Nghe Trạng Thái (State Listening)

## Nhật Ký Thay Đổi

* 2020-09-23: Bản nháp đầu tiên

## Trạng Thái

Đề Xuất

## Tóm Tắt

ADR này định nghĩa một tập hợp các thay đổi cho `MultiStore` để cho phép lắng nghe ghi KV store và cung cấp dữ liệu cho các hệ thống bên ngoài.

## Bối Cảnh

Hiện tại không có cơ chế hệ thống để xử lý các thay đổi trạng thái KV store mà không truy vấn trực tiếp node. Điều này tạo ra sự phụ thuộc vào việc thăm dò (polling) từ các dịch vụ bên ngoài. ADR này đề xuất giải quyết vấn đề này bằng cách cung cấp một hệ thống lắng nghe.

## Quyết Định

### Định Nghĩa WriteListener

Chúng ta định nghĩa interface `WriteListener` để cho phép các hệ thống bên ngoài đăng ký nghe các thay đổi KV store:

```go
// WriteListener interface được sử dụng để lắng nghe các thay đổi KV store
type WriteListener interface {
    // Lắng nghe một thao tác KV store đang diễn ra
    // delete=true nghĩa là đây là một thao tác xóa
    // delete=false và value=nil nghĩa là đây là một thao tác set với giá trị nil
    OnWrite(storeKey StoreKey, key []byte, value []byte, delete bool) error
}
```

### Kho Lưu Trữ Được Trang Bị Listener

Chúng ta định nghĩa một kho lưu trữ được trang bị listener bao bọc một KVStore và phát ra các sự kiện ra cho các listeners của nó:

```go
// ListenKVStore bao bọc một KVStore và phát ra các sự kiện ghi cho listeners của nó
type ListenKVStore struct {
    parent    types.KVStore
    listeners []types.WriteListener
}

func (lkv *ListenKVStore) Set(key, value []byte) {
    lkv.parent.Set(key, value)
    for _, l := range lkv.listeners {
        if err := l.OnWrite(lkv.storeKey, key, value, false); err != nil {
            panic(err)
        }
    }
}

func (lkv *ListenKVStore) Delete(key []byte) {
    lkv.parent.Delete(key)
    for _, l := range lkv.listeners {
        if err := l.OnWrite(lkv.storeKey, key, nil, true); err != nil {
            panic(err)
        }
    }
}
```

### Tích Hợp MultiStore

`MultiStore` sẽ được cập nhật để hỗ trợ việc thêm listeners cho các store key cụ thể:

```go
type MultiStore interface {
    // ... phương thức hiện có ...
    
    // AddListeners thêm WriteListeners cho StoreKey đã cho
    AddListeners(key StoreKey, listeners []WriteListener)
    
    // ListeningEnabled trả về true nếu listeners được kích hoạt cho StoreKey đã cho
    ListeningEnabled(key StoreKey) bool
}
```

### Streaming Service

Chúng ta định nghĩa `ABCIListener` để tích hợp với vòng lặp ABCI:

```go
// ABCIListener là interface được triển khai bởi các dịch vụ muốn lắng nghe
// các sự kiện ABCI phát sinh từ quá trình xử lý giao dịch
type ABCIListener interface {
    // ListenBeginBlock cập nhật dịch vụ bằng dữ liệu BeginBlock
    ListenBeginBlock(ctx context.Context, req abci.RequestBeginBlock, res abci.ResponseBeginBlock) error
    // ListenEndBlock cập nhật dịch vụ bằng dữ liệu EndBlock
    ListenEndBlock(ctx context.Context, req abci.RequestEndBlock, res abci.ResponseEndBlock) error
    // ListenDeliverTx cập nhật dịch vụ bằng dữ liệu DeliverTx
    ListenDeliverTx(ctx context.Context, req abci.RequestDeliverTx, res abci.ResponseDeliverTx) error
}
```

### Cấu Hình BaseApp

`BaseApp` sẽ được cập nhật để hỗ trợ đăng ký các streaming service:

```go
// SetStreamingService đặt streaming service cho BaseApp
func (app *BaseApp) SetStreamingService(s StreamingService) {
    app.streamingServices = append(app.streamingServices, s)
}
```

### Cấu Hình Từ app.toml

Streaming service có thể được cấu hình trong `app.toml`:

```toml
[store]
  [store.streamers]
    [store.streamers.file]
      keys = ["list", "of", "store", "keys"]
      write_dir = "path/to/write/dir"
      prefix = ""
```

## Hậu Quả

### Tích Cực

* Cung cấp cho các hệ thống bên ngoài khả năng lắng nghe các thay đổi trạng thái trong thời gian thực
* Không cần thiết phải thăm dò node để phát hiện thay đổi trạng thái
* Cho phép xây dựng các công cụ lập chỉ mục, phân tích và giám sát hiệu quả hơn

### Tiêu Cực

* Tăng thêm độ phức tạp cho `MultiStore` và `BaseApp`
* Listeners bị lỗi có thể gây ra sự cố cho node nếu không được xử lý đúng cách

### Trung Lập

* Tính năng là tùy chọn và không ảnh hưởng đến các node không sử dụng nó

## Tham Khảo

* [PR ban đầu cho lắng nghe trạng thái](https://github.com/cosmos/cosmos-sdk/pull/7951)
