# ADR 063: Core Module API

## Changelog

* 2022-08-18: Bản nháp đầu tiên
* 2022-12-08: Bản nháp đầu tiên
* 2023-01-24: Cập nhật

## Trạng Thái

ACCEPTED Đã Triển Khai Một Phần

## Tóm Tắt

Một core API mới được đề xuất như là cách phát triển ứng dụng cosmos-sdk sẽ cuối cùng thay thế các framework `AppModule` và `sdk.Context` hiện tại bằng một tập hợp các core service và extension interface. Core API này nhằm mục tiêu:

* Đơn giản hơn.
* Có khả năng mở rộng tốt hơn.
* Ổn định hơn so với framework hiện tại.
* Cho phép các sự kiện và truy vấn có tính xác định.
* Hỗ trợ event listener.
* Hỗ trợ client [ADR 033: Protobuf-based Inter-Module Communication](./adr-033-protobuf-inter-module-comm.md).

## Bối Cảnh

Về mặt lịch sử, các module đã phơi bày chức năng của chúng cho framework thông qua các interface `AppModule` và `AppModuleBasic` có những hạn chế sau:

* Cả `AppModule` và `AppModuleBasic` đều cần được định nghĩa và đăng ký, điều này phản trực giác.
* Các ứng dụng cần triển khai các interface đầy đủ, ngay cả các phần chúng không cần.
* Các phương thức interface phụ thuộc nhiều vào các phụ thuộc bên thứ ba không ổn định, đặc biệt là Comet.

Bằng cách cô lập tất cả chức năng state machine vào `sdk.Context`, tập hợp các chức năng có sẵn cho các module bị kết hợp chặt chẽ với kiểu này.

## Quyết Định

`core` API đề xuất một tập hợp core API mà các module có thể dựa vào để tương tác với state machine và phơi bày chức năng của chúng theo cách được thiết kế có nguyên tắc:

* Sự kết hợp chặt chẽ của các dependency và các chức năng không liên quan được giảm thiểu hoặc loại bỏ.
* API có thể có đảm bảo ổn định dài hạn.
* SDK framework có thể mở rộng theo cách an toàn và đơn giản.

Nguyên tắc thiết kế của core API:

* Mọi thứ mà một module muốn tương tác trong state machine đều là một service.
* Tất cả service phối hợp state thông qua `context.Context`.
* Tất cả service độc lập được cô lập trong các package độc lập với API tối thiểu và phụ thuộc tối thiểu.
* Core API phải tối giản và được thiết kế để hỗ trợ dài hạn (LTS).

### Core Service

#### Store Service

```go
type KVStoreService interface {
    OpenKVStore(context.Context) KVStore
}

type MemoryStoreService interface {
    OpenMemoryStore(context.Context) KVStore
}

type TransientStoreService interface {
    OpenTransientStore(context.Context) KVStore
}
```

Các module có thể sử dụng các service này như sau:

```go
func (k msgServer) Send(ctx context.Context, msg *types.MsgSend) (*types.MsgSendResponse, error) {
    store := k.kvStoreSvc.OpenKVStore(ctx)
}
```

#### Event Service

```go
package event

type Service interface {
  EmitProtoEvent(ctx context.Context, event protoiface.MessageV1) error
  EmitKVEvent(ctx context.Context, eventType string, attrs ...KVEventAttribute) error
  EmitProtoEventNonConsensus(ctx context.Context, event protoiface.MessageV1) error
}
```

Các sự kiện được emit bởi `EmitProto` được giả định là một phần của đồng thuận blockchain. Các sự kiện được emit bởi `EmitKVEvent` và `EmitProtoEventNonConsensus` không được coi là một phần của đồng thuận.

#### Logger

Logger (`cosmossdk.io/log`) phải được cung cấp sử dụng `depinject`:

```go
func NewKeeper(logger log.Logger) Keeper {
  return Keeper{
    logger: logger.With(log.ModuleKey, "x/"+types.ModuleName),
  }
}
```

### Core `AppModule` Extension Interface

Các module sẽ cung cấp core service của chúng cho runtime module thông qua các extension interface được xây dựng trên tag interface `cosmossdk.io/core/appmodule.AppModule`:

```go
type AppModule interface {
  depinject.OnePerModuleType
  IsAppModule()
}
```

#### Đăng Ký `MsgServer` và `QueryServer`

```go
type HasServices interface {
    AppModule
    RegisterServices(grpc.ServiceRegistrar)
}
```

#### Genesis

```go
type HasGenesis interface {
    AppModule
    DefaultGenesis(GenesisTarget) error
    ValidateGenesis(GenesisSource) error
    InitGenesis(context.Context, GenesisSource) error
    ExportGenesis(context.Context, GenesisTarget) error
}
```

#### Pre Blocker, Begin và End Blocker

```go
type HasPreBlocker interface {
  AppModule
  PreBlock(context.Context) error
}

type HasBeginBlocker interface {
  AppModule
  BeginBlock(context.Context) error
}

type HasEndBlocker interface {
  AppModule
  EndBlock(context.Context) error
}
```

### Runtime Compatibility Version

Module `core` sẽ định nghĩa một biến số nguyên tĩnh `cosmossdk.io/core.RuntimeCompatibilityVersion`, là chỉ số phiên bản minor của module core có thể truy cập tại runtime. Các triển khai runtime module đúng đắn nên kiểm tra phiên bản tương thích này.

### Runtime Module

Module `runtime` ban đầu sẽ được tạo trong go module `github.com/cosmos/cosmos-sdk` hiện có. Để chuyển sang semantic versioning cũng như tính mô-đun của runtime, các runtime module được hỗ trợ chính thức mới sẽ được tạo dưới tiền tố `cosmossdk.io/runtime`:

* `cosmossdk.io/runtime/comet`
* `cosmossdk.io/runtime/comet/v2`
* `cosmossdk.io/runtime/rollkit`

Các runtime module này nên cố gắng được phiên bản semantic ngay cả khi consensus engine bên dưới không phải vậy.

### Ví Dụ Sử Dụng

Đây là ví dụ thiết lập module `foo` v2 giả thuyết sử dụng [ORM](./adr-055-orm.md):

```go
type Keeper struct {
    db orm.ModuleDB
    evtSrv event.Service
}

func (k Keeper) RegisterServices(r grpc.ServiceRegistrar) {
  foov1.RegisterMsgServer(r, k)
  foov1.RegisterQueryServer(r, k)
}

func (k Keeper) BeginBlock(context.Context) error {
    return nil
}

func ProvideApp(config *foomodulev2.Module, evtSvc event.EventService, db orm.ModuleDB) (Keeper, appmodule.AppModule){
    k := &Keeper{db: db, evtSvc: evtSvc}
    return k, k
}
```

## Hậu Quả

### Tương Thích Ngược

Các phiên bản đầu của runtime module nên hỗ trợ càng nhiều module được xây dựng với framework `AppModule`/`sdk.Context` hiện có càng tốt.

Module core bản thân nên cố gắng duy trì ở phiên bản go semantic `v1` càng lâu càng tốt.

### Tích Cực

* Đóng gói API tốt hơn và phân tách mối quan tâm.
* API ổn định hơn.
* Tính mở rộng framework tốt hơn.
* Sự kiện và truy vấn có tính xác định.
* Event listener.
* Hỗ trợ thực thi msg và query inter-module.
* Hỗ trợ rõ ràng hơn cho việc fork và merge của phiên bản module.

### Tiêu Cực

### Trung Lập

* Các module sẽ cần được tái cấu trúc để sử dụng API này.
* Một số thay thế cho chức năng `AppModule` vẫn cần được định nghĩa trong các followup.

## Tài Liệu Tham Khảo

* [ADR 033: Giao Tiếp Inter-Module dựa trên Protobuf](./adr-033-protobuf-inter-module-comm.md)
* [ADR 057: App Wiring](./adr-057-app-wiring.md)
* [ADR 055: ORM](./adr-055-orm.md)
