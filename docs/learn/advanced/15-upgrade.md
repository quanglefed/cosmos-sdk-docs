---
sidebar_position: 1
---

# Di Chuyển Store In-Place

:::warning
Hãy đọc và hiểu toàn bộ tài liệu di chuyển store in-place trước khi chạy migration trên chuỗi đang hoạt động.
:::

:::note Tóm tắt
Nâng cấp các module ứng dụng một cách trơn tru với logic di chuyển store in-place tùy chỉnh.
:::

Cosmos SDK sử dụng hai phương pháp để thực hiện nâng cấp:

* Xuất toàn bộ trạng thái ứng dụng sang file JSON bằng lệnh CLI `export`, thực hiện thay đổi, rồi khởi động binary mới với file JSON đã thay đổi làm genesis file.

* Thực hiện nâng cấp in-place, giúp giảm đáng kể thời gian nâng cấp cho các chuỗi có trạng thái lớn. Sử dụng [Hướng dẫn Nâng cấp Module](../../build/building-modules/13-upgrade.md) để thiết lập các module ứng dụng tận dụng nâng cấp in-place.

Tài liệu này cung cấp các bước để sử dụng phương pháp nâng cấp Di Chuyển Store In-Place.

## Theo dõi phiên bản module

Mỗi module được gán một consensus version bởi nhà phát triển module. Consensus version đóng vai trò là phiên bản thay đổi phá vỡ tương thích của module. Cosmos SDK theo dõi tất cả consensus version của module trong store `VersionMap` của x/upgrade. Trong quá trình nâng cấp, sự khác biệt giữa `VersionMap` cũ được lưu trong trạng thái và `VersionMap` mới được tính toán bởi Cosmos SDK. Với mỗi sự khác biệt được xác định, các migration dành riêng cho module sẽ được chạy và consensus version tương ứng của mỗi module được nâng cấp sẽ tăng lên.

### Consensus Version

Consensus version được định nghĩa trên mỗi app module bởi nhà phát triển module và đóng vai trò là phiên bản thay đổi phá vỡ tương thích của module. Consensus version thông báo cho Cosmos SDK biết module nào cần được nâng cấp. Ví dụ, nếu module bank là phiên bản 2 và một lần nâng cấp đưa ra module bank phiên bản 3, Cosmos SDK sẽ nâng cấp module bank và chạy script migration "từ phiên bản 2 lên 3".

### Version Map

Version map là ánh xạ từ tên module đến consensus version. Map này được lưu vào trạng thái của x/upgrade để sử dụng trong quá trình di chuyển in-place. Khi migration hoàn tất, version map đã cập nhật được lưu trong trạng thái.

## Upgrade Handlers

Các lần nâng cấp sử dụng `UpgradeHandler` để tạo điều kiện cho migration. Các hàm `UpgradeHandler` được nhà phát triển ứng dụng triển khai phải tuân theo chữ ký hàm sau đây. Các hàm này lấy `VersionMap` từ trạng thái của x/upgrade và trả về `VersionMap` mới để lưu vào x/upgrade sau khi nâng cấp. Sự chênh lệch giữa hai `VersionMap` xác định module nào cần nâng cấp.

```go
type UpgradeHandler func(ctx sdk.Context, plan Plan, fromVM VersionMap) (VersionMap, error)
```

Bên trong các hàm này, bạn phải thực hiện bất kỳ logic nâng cấp nào để đưa vào `plan` được cung cấp. Tất cả các hàm upgrade handler phải kết thúc bằng dòng code sau:

```go
  return app.mm.RunMigrations(ctx, cfg, fromVM)
```

## Chạy Migrations

Migrations được chạy bên trong một `UpgradeHandler` bằng cách dùng `app.mm.RunMigrations(ctx, cfg, vm)`. Các hàm `UpgradeHandler` mô tả chức năng sẽ xảy ra trong quá trình nâng cấp. Hàm `RunMigration` lặp qua tham số `VersionMap` và chạy các script migration cho tất cả các phiên bản nhỏ hơn phiên bản của module binary ứng dụng mới. Sau khi migration hoàn tất, một `VersionMap` mới được trả về để lưu các phiên bản module đã nâng cấp vào trạng thái.

```go
cfg := module.NewConfigurator(...)
app.UpgradeKeeper.SetUpgradeHandler("my-plan", func(ctx sdk.Context, plan upgradetypes.Plan, fromVM module.VersionMap) (module.VersionMap, error) {

    // ...
    // logic nâng cấp bổ sung
    // ...

    // trả về VersionMap với ConsensusVersion của các module đã được cập nhật
    return app.mm.RunMigrations(ctx, fromVM)
})
```

Để tìm hiểu thêm về cách cấu hình script migration cho các module của bạn, xem [Hướng dẫn Nâng cấp Module](../../build/building-modules/13-upgrade.md).

### Thứ tự Migrations

Theo mặc định, tất cả migrations được chạy theo thứ tự bảng chữ cái tăng dần của tên module, ngoại trừ `x/auth` chạy cuối cùng. Lý do là sự phụ thuộc trạng thái giữa x/auth và các module khác (bạn có thể đọc thêm trong [issue #10606](https://github.com/cosmos/cosmos-sdk/issues/10606)).

Nếu bạn muốn thay đổi thứ tự migration, hãy gọi `app.mm.SetOrderMigrations(module1, module2, ...)` trong file app.go. Hàm này sẽ panic nếu bạn quên đưa một module vào danh sách tham số.

## Thêm module mới trong quá trình nâng cấp

Bạn có thể đưa các module hoàn toàn mới vào ứng dụng trong quá trình nâng cấp. Các module mới được nhận diện vì chúng chưa được đăng ký trong store `VersionMap` của x/upgrade. Trong trường hợp đó, `RunMigrations` gọi hàm `InitGenesis` từ module tương ứng để thiết lập trạng thái ban đầu của nó.

### Thêm StoreUpgrades cho module mới

Tất cả các chuỗi chuẩn bị chạy di chuyển store in-place sẽ cần thêm store upgrades thủ công cho các module mới và sau đó cấu hình store loader để áp dụng các nâng cấp đó. Điều này đảm bảo rằng store của module mới được thêm vào multistore trước khi migrations bắt đầu.

```go
upgradeInfo, err := app.UpgradeKeeper.ReadUpgradeInfoFromDisk()
if err != nil {
	panic(err)
}

if upgradeInfo.Name == "my-plan" && !app.UpgradeKeeper.IsSkipHeight(upgradeInfo.Height) {
	storeUpgrades := storetypes.StoreUpgrades{
		// thêm store upgrades cho các module mới
		// Ví dụ:
		//    Added: []string{"foo", "bar"},
		// ...
	}

	// cấu hình store loader kiểm tra nếu version == upgradeHeight và áp dụng store upgrades
	app.SetStoreLoader(upgradetypes.UpgradeStoreLoader(upgradeInfo.Height, &storeUpgrades))
}
```

## Trạng thái Genesis

Khi khởi động một chuỗi mới, consensus version của mỗi module PHẢI được lưu vào trạng thái trong quá trình genesis của ứng dụng. Để lưu consensus version, thêm dòng sau vào phương thức `InitChainer` trong `app.go`:

```diff
func (app *MyApp) InitChainer(ctx sdk.Context, req abci.InitChainRequest) abci.InitChainResponse {
  ...
+ app.UpgradeKeeper.SetModuleVersionMap(ctx, app.mm.GetVersionMap())
  ...
}
```

Thông tin này được Cosmos SDK sử dụng để phát hiện khi các module có phiên bản mới hơn được đưa vào ứng dụng.

Đối với module `foo` mới, `InitGenesis` được gọi bởi `RunMigration` chỉ khi `foo` được đăng ký trong module manager nhưng không được thiết lập trong `fromVM`. Do đó, nếu bạn muốn bỏ qua `InitGenesis` khi một module mới được thêm vào ứng dụng, hãy đặt phiên bản module của nó trong `fromVM` bằng consensus version của module:

```go
app.UpgradeKeeper.SetUpgradeHandler("my-plan", func(ctx sdk.Context, plan upgradetypes.Plan, fromVM module.VersionMap) (module.VersionMap, error) {
    // ...

    // Đặt phiên bản của foo thành ConsensusVersion mới nhất trong VersionMap.
    // Điều này sẽ bỏ qua việc chạy InitGenesis trên Foo
    fromVM[foo.ModuleName] = foo.AppModule{}.ConsensusVersion()

    return app.mm.RunMigrations(ctx, fromVM)
})
```

### Ghi đè hàm Genesis

Cosmos SDK cung cấp các module mà nhà phát triển ứng dụng có thể import vào ứng dụng của họ. Các module này thường đã có hàm `InitGenesis` được định nghĩa sẵn.

Bạn có thể viết hàm `InitGenesis` tùy chỉnh cho một module được import. Để làm điều này, hãy kích hoạt thủ công hàm genesis tùy chỉnh trong upgrade handler.

:::warning
Bạn PHẢI đặt thủ công consensus version trong version map được truyền đến hàm `UpgradeHandler`. Nếu không, SDK sẽ chạy code `InitGenesis` hiện có của module dù bạn đã kích hoạt hàm tùy chỉnh trong `UpgradeHandler`.
:::

```go
import foo "github.com/my/module/foo"

app.UpgradeKeeper.SetUpgradeHandler("my-plan", func(ctx sdk.Context, plan upgradetypes.Plan, fromVM module.VersionMap)  (module.VersionMap, error) {

    // Đăng ký consensus version trong version map
    // để tránh SDK kích hoạt hàm
    // InitGenesis mặc định.
    fromVM["foo"] = foo.AppModule{}.ConsensusVersion()

    // Chạy InitGenesis tùy chỉnh cho foo
    app.mm["foo"].InitGenesis(ctx, app.appCodec, myCustomGenesisState)

    return app.mm.RunMigrations(ctx, cfg, fromVM)
})
```

## Đồng bộ Full Node với Blockchain đã nâng cấp

Bạn có thể đồng bộ một full node với một blockchain hiện có đã được nâng cấp bằng cách sử dụng Cosmovisor.

Để đồng bộ thành công, bạn phải bắt đầu với binary ban đầu mà blockchain đã khởi động tại genesis. Nếu tất cả Software Upgrade Plans chứa thông tin binary, bạn có thể chạy Cosmovisor với tùy chọn auto-download để tự động xử lý việc tải xuống và chuyển sang binary liên quan đến mỗi lần nâng cấp tuần tự. Nếu không, bạn cần cung cấp tất cả binary cho Cosmovisor theo cách thủ công.

Để tìm hiểu thêm về Cosmovisor, xem [Hướng dẫn nhanh Cosmovisor](../../../../tools/cosmovisor/README.md).
