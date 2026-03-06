# ADR 049: Hook State Sync

## Changelog

* 19 tháng 1 năm 2022: Bản nháp đầu tiên
* 29 tháng 4 năm 2022: Interface extension snapshotter an toàn hơn

## Trạng Thái

Đã Triển Khai

## Tóm Tắt

ADR này phác thảo một cơ chế dựa trên hook cho các module ứng dụng để cung cấp trạng thái bổ sung (ngoài cây IAVL) được sử dụng trong quá trình state sync.

## Bối Cảnh

Các client mới sử dụng state-sync để tải xuống các snapshot của trạng thái module từ các peer. Hiện tại, snapshot bao gồm một luồng các `SnapshotStoreItem` và `SnapshotIAVLItem`, có nghĩa là các module ứng dụng định nghĩa trạng thái của họ bên ngoài cây IAVL không thể đưa trạng thái của họ vào quá trình state-sync.

## Quyết Định

Chúng ta thêm hai loại message mới: `SnapshotExtensionMeta` và `SnapshotExtensionPayload`, và chúng được thêm vào luồng multi-store hiện có với `SnapshotExtensionMeta` hoạt động như một dấu phân cách giữa các extension.

```protobuf
message SnapshotItem {
  oneof item {
    SnapshotStoreItem        store             = 1;
    SnapshotIAVLItem         iavl              = 2 [(gogoproto.customname) = "IAVL"];
    SnapshotExtensionMeta    extension         = 3;
    SnapshotExtensionPayload extension_payload = 4;
  }
}

message SnapshotExtensionMeta {
  string name   = 1;
  uint32 format = 2;
}

message SnapshotExtensionPayload {
  bytes payload = 1;
}
```

Khi tạo luồng snapshot, snapshot `multistore` luôn được đặt ở đầu luồng nhị phân, và các snapshot extension khác được sắp xếp theo thứ tự alphabet theo tên của `ExtensionSnapshotter` tương ứng.

Chúng ta cung cấp interface `Snapshotter` và `ExtensionSnapshotter` cho các module để triển khai các snapshotter:

```go
type ExtensionSnapshotter interface {
    SnapshotName() string
    SnapshotFormat() uint32
    SupportedFormats() []uint32
    SnapshotExtension(height uint64, payloadWriter ExtensionPayloadWriter) error
    RestoreExtension(height uint64, format uint32, payloadReader ExtensionPayloadReader) error
}
```

Khi thiết lập ứng dụng, snapshot `Manager` nên gọi `RegisterExtensions([]ExtensionSnapshotter…)` để đăng ký tất cả extension snapshotter.

## Hậu Quả

Kết quả của triển khai này, chúng ta có thể tạo các snapshot của luồng chunk nhị phân cho trạng thái mà chúng ta duy trì bên ngoài cây IAVL, ví dụ CosmWasm blobs. Và các client mới có thể lấy các snapshot trạng thái cho tất cả module đã triển khai interface tương ứng từ các peer node.

### Tương Thích Ngược

ADR này giới thiệu các kiểu proto mới, thêm trường `extensions` trong snapshot `Manager`, và thêm interface `ExtensionSnapshotter` mới, vì vậy không tương thích ngược nếu có extension.

Nhưng đối với các ứng dụng không có dữ liệu trạng thái bên ngoài cây IAVL cho bất kỳ module nào, luồng snapshot tương thích ngược.

### Tích Cực

* Trạng thái được duy trì bên ngoài cây IAVL như CosmWasm blobs có thể tạo snapshot.

### Trung Lập

* Tất cả module duy trì trạng thái bên ngoài cây IAVL cần triển khai `ExtensionSnapshotter`.

## Tài Liệu Tham Khảo

* https://github.com/cosmos/cosmos-sdk/pull/10961
* https://github.com/cosmos/cosmos-sdk/issues/7340
