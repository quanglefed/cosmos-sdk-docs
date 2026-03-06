# ADR 041: Migration Store “Tại chỗ” (In-Place)

## Changelog

* 17.02.2021: Bản nháp ban đầu

## Trạng thái

Được chấp nhận

## Tóm tắt

ADR này giới thiệu một cơ chế để thực hiện migration “tại chỗ” (in-place) trên
state store trong quá trình nâng cấp phần mềm của chain.

## Bối cảnh

Khi một nâng cấp chain đưa vào các thay đổi “phá vỡ state” (state-breaking) bên
trong các module, quy trình hiện tại gồm: export toàn bộ state ra một file JSON
(thông qua lệnh `simd export`), chạy script migration trên file JSON (lệnh
`simd genesis migrate`), xoá sạch store (lệnh `simd unsafe-reset-all`), và khởi
động một chain mới với file JSON đã migrate làm genesis mới (tuỳ chọn đặt custom
initial block height). Một ví dụ về quy trình như vậy có thể xem trong
[hướng dẫn migration Cosmos Hub 3->4](https://github.com/cosmos/gaia/blob/v4.0.3/docs/migration/cosmoshub-3.md#upgrade-procedure).

Quy trình này rườm rà vì nhiều lý do:

* Quy trình tốn thời gian. Có thể mất hàng giờ để chạy lệnh `export`, cộng thêm
  vài giờ nữa để chạy `InitChain` trên chain mới với file JSON đã migrate.
* File JSON export có thể rất nặng (~100MB-1GB), khiến việc xem, sửa, và chuyển
  file khó khăn; kéo theo đó là công việc bổ sung để xử lý các vấn đề này (ví dụ
  [streaming genesis](https://github.com/cosmos/cosmos-sdk/issues/6936)).

## Quyết định

Chúng ta đề xuất một quy trình migration dựa trên việc sửa KV store “tại chỗ”
không cần tới luồng export JSON -> migrate -> import như mô tả ở trên.

### `ConsensusVersion` của module

Chúng ta giới thiệu một phương thức mới trên interface `AppModule`:

```go
type AppModule interface {
    // --snip--
    ConsensusVersion() uint64
}
```

Phương thức này trả về một `uint64`, đóng vai trò là phiên bản “phá vỡ đồng thuận”
(consensus-breaking) của module. Nó BẮT BUỘC phải được tăng lên mỗi khi module
đưa vào một thay đổi phá vỡ đồng thuận. Để tránh lỗi tiềm ẩn do giá trị mặc định,
phiên bản ban đầu của một module BẮT BUỘC phải đặt là 1. Trong Cosmos SDK, phiên
bản 1 tương ứng với các module thuộc dòng v0.41.

### Hàm migration theo module

Với mỗi thay đổi phá vỡ đồng thuận mà module đưa vào, một script migration từ
ConsensusVersion `N` sang phiên bản `N+1` BẮT BUỘC phải được đăng ký vào
`Configurator` bằng phương thức `RegisterMigration` mới được thêm vào. Tất cả
module nhận một tham chiếu đến configurator trong phương thức `RegisterServices`
trên `AppModule`, và đây là nơi nên đăng ký các hàm migration. Các hàm migration
nên được đăng ký theo thứ tự tăng dần.

```go
func (am AppModule) RegisterServices(cfg module.Configurator) {
    // --snip--
    cfg.RegisterMigration(types.ModuleName, 1, func(ctx sdk.Context) error {
        // Perform in-place store migrations from ConsensusVersion 1 to 2.
    })
     cfg.RegisterMigration(types.ModuleName, 2, func(ctx sdk.Context) error {
        // Perform in-place store migrations from ConsensusVersion 2 to 3.
    })
    // etc.
}
```

Ví dụ, nếu ConsensusVersion mới của một module là `N`, thì phải đăng ký `N-1`
hàm migration vào configurator.

Trong Cosmos SDK, các hàm migration được xử lý bởi keeper của từng module, vì
keeper nắm `sdk.StoreKey` dùng để thực hiện migration “tại chỗ”. Để tránh làm
keeper trở nên quá tải, mỗi module dùng một wrapper `Migrator` để xử lý các hàm
migration:

```go
// Migrator is a struct for handling in-place store migrations.
type Migrator struct {
  BaseKeeper
}
```

Các hàm migration nên nằm trong thư mục `migrations/` của mỗi module, và được
gọi bởi các phương thức của Migrator. Chúng ta đề xuất định dạng tên phương thức
`Migrate{M}to{N}`.

```go
// Migrate1to2 migrates from version 1 to 2.
func (m Migrator) Migrate1to2(ctx sdk.Context) error {
	return v2bank.MigrateStore(ctx, m.keeper.storeKey) // v043bank is package `x/bank/migrations/v2`.
}
```

Các hàm migration của mỗi module phụ thuộc vào quá trình tiến hoá store của riêng
module đó và không được mô tả trong ADR này. Một ví dụ về migration store key của
x/bank sau khi đưa vào ADR-028 địa chỉ có prefix độ dài có thể xem trong
[mã `store.go` này](https://github.com/cosmos/cosmos-sdk/blob/36f68eb9e041e20a5bb47e216ac5eb8b91f95471/x/bank/legacy/v043/store.go#L41-L62).

### Theo dõi phiên bản module trong `x/upgrade`

Chúng ta giới thiệu một prefix store mới trong store của `x/upgrade`. Store này
sẽ theo dõi phiên bản hiện tại của mỗi module; có thể mô hình hoá nó như một
`map[string]uint64` ánh xạ từ tên module sang ConsensusVersion của module, và
nó sẽ được dùng khi chạy migrations (xem phần sau). Key prefix dùng là `0x1`,
và định dạng key/value là:

```text
0x2 | {bytes(module_name)} => BigEndian(module_consensus_version)
```

State ban đầu của store được thiết lập từ phương thức `InitChainer` trong `app.go`.

Chữ ký (signature) của UpgradeHandler cần được cập nhật để nhận `VersionMap`,
đồng thời trả về `VersionMap` đã nâng cấp và một error:

```diff
- type UpgradeHandler func(ctx sdk.Context, plan Plan)
+ type UpgradeHandler func(ctx sdk.Context, plan Plan, versionMap VersionMap) (VersionMap, error)
```

Để áp dụng một nâng cấp, chúng ta truy vấn `VersionMap` từ store `x/upgrade` và
truyền nó vào handler. Handler chạy các hàm migration thực tế (xem phần kế tiếp),
và nếu thành công, trả về một `VersionMap` đã cập nhật để được lưu vào state.

```diff
func (k UpgradeKeeper) ApplyUpgrade(ctx sdk.Context, plan types.Plan) {
    // --snip--
-   handler(ctx, plan)
+   updatedVM, err := handler(ctx, plan, k.GetModuleVersionMap(ctx)) // k.GetModuleVersionMap() fetches the VersionMap stored in state.
+   if err != nil {
+       return err
+   }
+
+   // Set the updated consensus versions to state
+   k.SetModuleVersionMap(ctx, updatedVM)
}
```

Một endpoint truy vấn gRPC để truy vấn `VersionMap` được lưu trong state của
`x/upgrade` cũng sẽ được thêm vào, để developer ứng dụng có thể kiểm tra lại
`VersionMap` trước khi upgrade handler chạy.

### Chạy migrations

Sau khi tất cả migration handler đã được đăng ký trong configurator (xảy ra khi
khởi động), việc chạy migrations có thể thực hiện bằng cách gọi `RunMigrations`
trên `module.Manager`. Hàm này sẽ lặp qua tất cả module, và với mỗi module:

* Lấy ConsensusVersion cũ của module từ đối số `VersionMap` (gọi là `M`).
* Lấy ConsensusVersion mới của module từ phương thức `ConsensusVersion()` trên
  `AppModule` (gọi là `N`).
* Nếu `N>M`, chạy tuần tự tất cả migration đã đăng ký cho module theo chuỗi
  `M -> M+1 -> M+2...` cho tới `N`.
  * Có một trường hợp đặc biệt khi không có ConsensusVersion cho module: điều này
    nghĩa là module mới được thêm trong đợt nâng cấp. Trong trường hợp này, không
    chạy hàm migration nào, và ConsensusVersion hiện tại của module sẽ được lưu vào
    store `x/upgrade`.

Nếu thiếu một migration bắt buộc (ví dụ: chưa được đăng ký trong `Configurator`),
thì `RunMigrations` sẽ lỗi.

Trong thực tế, `RunMigrations` nên được gọi bên trong một `UpgradeHandler`.

```go
app.UpgradeKeeper.SetUpgradeHandler("my-plan", func(ctx sdk.Context, plan upgradetypes.Plan, vm module.VersionMap)  (module.VersionMap, error) {
    return app.mm.RunMigrations(ctx, vm)
})
```

Giả sử chain nâng cấp tại block `n`, quy trình chạy nên như sau:

* binary cũ sẽ dừng (halt) trong `BeginBlock` khi bắt đầu block `N`. Trong store
  của nó, các ConsensusVersion của module của binary cũ đã được lưu.
* binary mới sẽ khởi động tại block `N`. UpgradeHandler được thiết lập trong
  binary mới, vì vậy sẽ chạy ở `BeginBlock` của binary mới. Bên trong `ApplyUpgrade`
  của `x/upgrade`, `VersionMap` sẽ được lấy từ store (của binary cũ) và truyền vào
  `RunMigrations`, thực hiện migrate “tại chỗ” tất cả module store trước khi các
  `BeginBlock` của chính module chạy.

## Hệ quả

### Tương thích ngược

ADR này giới thiệu một phương thức mới `ConsensusVersion()` trên `AppModule`, mà
tất cả module cần triển khai. Nó cũng thay đổi chữ ký hàm UpgradeHandler. Do đó,
nó không tương thích ngược.

Mặc dù module BẮT BUỘC phải đăng ký các hàm migration khi tăng ConsensusVersion,
việc chạy các script đó bằng upgrade handler là tuỳ chọn. Một ứng dụng hoàn toàn
có thể quyết định không gọi `RunMigrations` trong upgrade handler, và tiếp tục
đi theo đường migration JSON legacy.

### Tích cực

* Thực hiện nâng cấp chain mà không cần thao tác các file JSON.
* Dù chưa có benchmark, nhiều khả năng migration store “tại chỗ” sẽ tốn ít thời
  gian hơn migration JSON. Lý do chính là sẽ bỏ qua cả lệnh `simd export` trên
  binary cũ và hàm `InitChain` trên binary mới.

### Tiêu cực

* Developer module BẮT BUỘC phải theo dõi chính xác các thay đổi phá vỡ đồng thuận
  trong module. Nếu một thay đổi phá vỡ đồng thuận được đưa vào một module mà không
  tăng `ConsensusVersion()`, thì `RunMigrations` sẽ không phát hiện migration, và
  nâng cấp chain có thể thất bại. Tài liệu nên phản ánh rõ điều này.

### Trung tính

* Cosmos SDK sẽ tiếp tục hỗ trợ migration JSON thông qua các lệnh hiện có
  `simd export` và `simd genesis migrate`.
* ADR hiện tại không cho phép tạo, đổi tên, hay xoá store; chỉ cho phép sửa key
  và value của store hiện có. Cosmos SDK đã có `StoreLoader` cho các thao tác đó.

## Thảo luận thêm

## Tham khảo

* Thảo luận ban đầu: https://github.com/cosmos/cosmos-sdk/discussions/8429
* Triển khai `ConsensusVersion` và `RunMigrations`: https://github.com/cosmos/cosmos-sdk/pull/8485
* Issue thảo luận thiết kế `x/upgrade`: https://github.com/cosmos/cosmos-sdk/issues/8514

