# Hướng Dẫn Nâng Cấp

Tài liệu này cung cấp hướng dẫn đầy đủ để nâng cấp một Cosmos SDK chain từ `v0.50.x` lên `v0.53.x`.

Hướng dẫn này bao gồm một thay đổi **bắt buộc** và ba tính năng **tùy chọn**.

Sau khi hoàn thành hướng dẫn này, các ứng dụng sẽ có:

* Module `x/protocolpool`
* Module `x/epochs`
* Hỗ trợ Unordered Transaction

## Mục Lục

* [App Wiring Changes (BẮT BUỘC)](#app-wiring-changes-required)
* [Thêm Module ProtocolPool (TÙY CHỌN)](#adding-protocolpool-module-optional)
    * [ProtocolPool Manual Wiring](#protocolpool-manual-wiring)
    * [ProtocolPool DI Wiring](#protocolpool-di-wiring)
* [Thêm Module Epochs (TÙY CHỌN)](#adding-epochs-module-optional)
    * [Epochs Manual Wiring](#epochs-manual-wiring)
    * [Epochs DI Wiring](#epochs-di-wiring)
* [Bật Unordered Transactions (TÙY CHỌN)](#enable-unordered-transactions-optional)
* [Upgrade Handler](#upgrade-handler)

## App Wiring Changes **BẮT BUỘC**

Module `x/auth` hiện chứa một `PreBlocker` _phải_ được đặt trong phương thức `SetOrderPreBlockers` của module manager.

```go
app.ModuleManager.SetOrderPreBlockers(
    upgradetypes.ModuleName,
    authtypes.ModuleName, // MỚI
)
```

## Thêm Module ProtocolPool **TÙY CHỌN**

:::warning

Sử dụng community pool bên ngoài như `x/protocolpool` sẽ khiến các handler `x/distribution` sau trả về lỗi:

**QueryService**

* `CommunityPool`

**MsgService**

* `CommunityPoolSpend`
* `FundCommunityPool`

Nếu các dịch vụ của bạn phụ thuộc vào chức năng này từ `x/distribution`, vui lòng cập nhật chúng để sử dụng `x/protocolpool` hoặc các giải pháp thay thế community pool bên ngoài tùy chỉnh của bạn.

:::

### Manual Wiring

Import các phần sau:

```go
import (
    // ...
    "github.com/cosmos/cosmos-sdk/x/protocolpool"
    protocolpoolkeeper "github.com/cosmos/cosmos-sdk/x/protocolpool/keeper"
    protocolpooltypes "github.com/cosmos/cosmos-sdk/x/protocolpool/types"
)
```

Đặt quyền tài khoản module.

```go
maccPerms = map[string][]string{
    // ...
    protocolpooltypes.ModuleName:                nil,
    protocolpooltypes.ProtocolPoolEscrowAccount: nil,
}
```

Thêm protocol pool keeper vào struct ứng dụng của bạn.

```go
ProtocolPoolKeeper protocolpoolkeeper.Keeper
```

Thêm store key:

```go
keys := storetypes.NewKVStoreKeys(
    // ...
    protocolpooltypes.StoreKey,
)
```

Khởi tạo keeper.

Đảm bảo thực hiện điều này trước khi khởi tạo module distribution, vì bạn sẽ truyền keeper vào đó tiếp theo.

```go
app.ProtocolPoolKeeper = protocolpoolkeeper.NewKeeper(
    appCodec,
    runtime.NewKVStoreService(keys[protocolpooltypes.StoreKey]),
    app.AccountKeeper,
    app.BankKeeper,
    authtypes.NewModuleAddress(govtypes.ModuleName).String(),
)
```

Truyền protocolpool keeper vào distribution keeper:

```go
app.DistrKeeper = distrkeeper.NewKeeper(
    appCodec,
    runtime.NewKVStoreService(keys[distrtypes.StoreKey]),
    app.AccountKeeper,
    app.BankKeeper,
    app.StakingKeeper,
    authtypes.FeeCollectorName,
    authtypes.NewModuleAddress(govtypes.ModuleName).String(),
    distrkeeper.WithExternalCommunityPool(app.ProtocolPoolKeeper), // MỚI
)
```

Thêm module protocolpool vào module manager:

```go
app.ModuleManager = module.NewManager(
    // ...
    protocolpool.NewAppModule(appCodec, app.ProtocolPoolKeeper, app.AccountKeeper, app.BankKeeper),
)
```

Thêm entry cho SetOrderBeginBlockers, SetOrderEndBlockers, SetOrderInitGenesis và SetOrderExportGenesis.

```go
app.ModuleManager.SetOrderBeginBlockers(
    // phải đến SAU distribution.
    distrtypes.ModuleName,
    protocolpooltypes.ModuleName,
)
```

```go
app.ModuleManager.SetOrderEndBlockers(
    // thứ tự không quan trọng.
    protocolpooltypes.ModuleName,
)
```

```go
app.ModuleManager.SetOrderInitGenesis(
    // thứ tự không quan trọng.
    protocolpooltypes.ModuleName,   
)
```

```go
app.ModuleManager.SetOrderInitGenesis(
    protocolpooltypes.ModuleName, // phải được export trước bank.
    banktypes.ModuleName,
)
```

### DI Wiring

Lưu ý: _miễn là một external community pool keeper (ở đây là `x/protocolpool`) được kết nối trong DI configs, `x/distribution` sẽ tự động sử dụng nó cho external pool của nó._

Đầu tiên, thiết lập keeper cho ứng dụng.

Import protocolpool keeper:

```go
protocolpoolkeeper "github.com/cosmos/cosmos-sdk/x/protocolpool/keeper"
```

Thêm keeper vào struct ứng dụng của bạn:

```go
ProtocolPoolKeeper protocolpoolkeeper.Keeper
```

Thêm keeper vào hệ thống depinject:

```go
depinject.Inject(
    appConfig,
    &appBuilder,
    &app.appCodec,
    &app.legacyAmino,
    &app.txConfig,
    &app.interfaceRegistry,
    // ... other modules
    &app.ProtocolPoolKeeper, // MODULE MỚI!
)
```

Tiếp theo, thiết lập cấu hình cho module.

Import các phần sau:

```go
import (
    protocolpoolmodulev1 "cosmossdk.io/api/cosmos/protocolpool/module/v1"
    
    _ "github.com/cosmos/cosmos-sdk/x/protocolpool" // import for side-effects
    protocolpooltypes "github.com/cosmos/cosmos-sdk/x/protocolpool/types"
)
```

Module protocolpool có các tài khoản module xử lý quỹ. Thêm chúng vào cấu hình quyền tài khoản module:

```go
moduleAccPerms = []*authmodulev1.ModuleAccountPermission{
    // ...
    {Account: protocolpooltypes.ModuleName},
    {Account: protocolpooltypes.ProtocolPoolEscrowAccount},
}
```

Tiếp theo, thêm entry cho BeginBlockers, EndBlockers, InitGenesis và ExportGenesis.

```go
BeginBlockers: []string{
    // ...
    // phải đến SAU distribution.
    distrtypes.ModuleName,
    protocolpooltypes.ModuleName,
},
```

```go
EndBlockers: []string{
    // ...
    // thứ tự của protocolpool không quan trọng.
    protocolpooltypes.ModuleName,
},
```

```go
InitGenesis: []string{
    // ... phải đến SAU distribution.
    distrtypes.ModuleName,
    protocolpooltypes.ModuleName,
},
```

```go
ExportGenesis: []string{
    // ...
    // Phải được export trước x/bank.
    protocolpooltypes.ModuleName, 
    banktypes.ModuleName,
},
```

Cuối cùng, thêm entry cho protocolpool trong ModuleConfig.

```go
{
    Name:   protocolpooltypes.ModuleName,
    Config: appconfig.WrapAny(&protocolpoolmodulev1.Module{}),
},
```

## Thêm Module Epochs **TÙY CHỌN**

### Manual Wiring

Import các phần sau:

```go
import (
    // ...
    "github.com/cosmos/cosmos-sdk/x/epochs"
    epochskeeper "github.com/cosmos/cosmos-sdk/x/epochs/keeper"
    epochstypes "github.com/cosmos/cosmos-sdk/x/epochs/types"
)
```

Thêm epochs keeper vào struct ứng dụng của bạn:

```go
EpochsKeeper epochskeeper.Keeper
```

Thêm store key:

```go
keys := storetypes.NewKVStoreKeys(
    // ...
    epochstypes.StoreKey,
)
```

Khởi tạo keeper:

```go
app.EpochsKeeper = epochskeeper.NewKeeper(
    runtime.NewKVStoreService(keys[epochstypes.StoreKey]),
    appCodec,
)
```

Thiết lập hooks cho epochs keeper:

Để tìm hiểu cách viết hooks cho epoch keeper, xem [x/epoch README](https://github.com/cosmos/cosmos-sdk/blob/main/x/epochs/README.md)

```go
app.EpochsKeeper.SetHooks(
    epochstypes.NewMultiEpochHooks(
        // chèn epoch hooks receivers tại đây
        app.SomeOtherModule
    ),
)
```

Thêm module epochs vào module manager:

```go
app.ModuleManager = module.NewManager(
    // ...
    epochs.NewAppModule(appCodec, app.EpochsKeeper),
)
```

Thêm entries cho SetOrderBeginBlockers và SetOrderInitGenesis:

```go
app.ModuleManager.SetOrderBeginBlockers(
    // ...
    epochstypes.ModuleName,
)
```

```go
app.ModuleManager.SetOrderInitGenesis(
    // ...
    epochstypes.ModuleName,
)
```

### DI Wiring

Đầu tiên, thiết lập keeper cho ứng dụng.

Import epochs keeper:

```go
epochskeeper "github.com/cosmos/cosmos-sdk/x/epochs/keeper"
```

Thêm keeper vào struct ứng dụng của bạn:

```go
EpochsKeeper epochskeeper.Keeper
```

Thêm keeper vào hệ thống depinject:

```go
depinject.Inject(
    appConfig,
    &appBuilder,
    &app.appCodec,
    &app.legacyAmino,
    &app.txConfig,
    &app.interfaceRegistry,
    // ... other modules
    &app.EpochsKeeper, // MODULE MỚI!
)
```

Tiếp theo, thiết lập cấu hình cho module.

Import các phần sau:

```go
import (
    epochsmodulev1 "cosmossdk.io/api/cosmos/epochs/module/v1"
    
    _ "github.com/cosmos/cosmos-sdk/x/epochs" // import for side-effects
    epochstypes "github.com/cosmos/cosmos-sdk/x/epochs/types"
)
```

Thêm entry cho BeginBlockers và InitGenesis:

```go
BeginBlockers: []string{
    // ...
    epochstypes.ModuleName,
},
```

```go
InitGenesis: []string{
    // ...
    epochstypes.ModuleName,
},
```

Cuối cùng, thêm entry cho epochs trong ModuleConfig:

```go
{
    Name:   epochstypes.ModuleName,
    Config: appconfig.WrapAny(&epochsmodulev1.Module{}),
},
```

## Bật Unordered Transactions **TÙY CHỌN**

Để bật hỗ trợ unordered transaction trên ứng dụng, `x/auth` keeper phải được cung cấp tùy chọn `WithUnorderedTransactions`.

Lưu ý rằng unordered transactions yêu cầu giá trị sequence bằng không, và sẽ **THẤT BẠI** nếu giá trị sequence khác không được đặt.
Hãy đảm bảo không có giá trị sequence nào được đặt khi gửi unordered transaction.
Các dịch vụ phụ thuộc vào các giả định trước đây về giá trị sequence nên được cập nhật để xử lý unordered transactions.
Các dịch vụ cần biết rằng khi transaction là unordered, sequence của transaction sẽ luôn bằng không.

```go
	app.AccountKeeper = authkeeper.NewAccountKeeper(
		appCodec,
		runtime.NewKVStoreService(keys[authtypes.StoreKey]),
		authtypes.ProtoBaseAccount,
		maccPerms,
		authcodec.NewBech32Codec(sdk.Bech32MainPrefix),
		sdk.Bech32MainPrefix,
		authtypes.NewModuleAddress(govtypes.ModuleName).String(),
		authkeeper.WithUnorderedTransactions(true), // tùy chọn mới!
	)
```

Nếu sử dụng dependency injection, hãy cập nhật cấu hình module auth.

```go
		{
			Name: authtypes.ModuleName,
			Config: appconfig.WrapAny(&authmodulev1.Module{
				Bech32Prefix:             "cosmos",
				ModuleAccountPermissions: moduleAccPerms,
				EnableUnorderedTransactions: true, // xóa dòng này nếu bạn không muốn unordered transactions.
			}),
		},
```

Theo mặc định, unordered transactions sử dụng thời gian timeout của transaction là 10 phút và phí gas mặc định là 2240 gas units.
Để thay đổi các giá trị mặc định này, truyền các tùy chọn tương ứng vào trường `SigVerifyOptions` mới trong `HandlerOptions` của `x/auth`.

```go
options := ante.HandlerOptions{
    SigVerifyOptions: []ante.SigVerificationDecoratorOption{
        // thay đổi bên dưới theo nhu cầu.
        ante.WithUnorderedTxGasCost(ante.DefaultUnorderedTxGasCost),
        ante.WithMaxUnorderedTxTimeoutDuration(ante.DefaultMaxTimeoutDuration),
    },
}
```

```go
anteDecorators := []sdk.AnteDecorator{
	// ... other decorators ...
    ante.NewSigVerificationDecorator(options.AccountKeeper, options.SignModeHandler, options.SigVerifyOptions...), // cung cấp tùy chọn mới
}
```

## Upgrade Handler

Upgrade handler chỉ yêu cầu thêm các store upgrades cho các module được thêm ở trên.
Nếu ứng dụng của bạn không thêm `x/protocolpool` hoặc `x/epochs`, bạn không cần thêm store upgrade.

```go
// UpgradeName định nghĩa tên nâng cấp on-chain cho SimApp upgrade mẫu
// từ v050 lên v053.
//
// LƯU Ý: Upgrade này định nghĩa triển khai tham chiếu về những gì một upgrade
// có thể trông như thế nào khi ứng dụng đang di chuyển từ Cosmos SDK phiên bản
// v0.50.x lên v0.53.x.
const UpgradeName = "v050-to-v053"

func (app SimApp) RegisterUpgradeHandlers() {
    app.UpgradeKeeper.SetUpgradeHandler(
        UpgradeName,
        func(ctx context.Context, _ upgradetypes.Plan, fromVM module.VersionMap) (module.VersionMap, error) {
            return app.ModuleManager.RunMigrations(ctx, app.Configurator(), fromVM)
        },
    )

    upgradeInfo, err := app.UpgradeKeeper.ReadUpgradeInfoFromDisk()
    if err != nil {
        panic(err)
    }

    if upgradeInfo.Name == UpgradeName && !app.UpgradeKeeper.IsSkipHeight(upgradeInfo.Height) {
        storeUpgrades := storetypes.StoreUpgrades{
            Added: []string{
                epochstypes.ModuleName, // nếu không thêm x/epochs vào chain, xóa dòng này.
                protocolpooltypes.ModuleName, // nếu không thêm x/protocolpool vào chain, xóa dòng này.
            },
        }

        // cấu hình store loader kiểm tra nếu version == upgradeHeight và áp dụng store upgrades
        app.SetStoreLoader(upgradetypes.UpgradeStoreLoader(upgradeInfo.Height, &storeUpgrades))
    }
}
```
