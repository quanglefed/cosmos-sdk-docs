---
sidebar_position: 1
---

# Testnet Ứng Dụng

Xây dựng ứng dụng rất phức tạp và đòi hỏi nhiều kiểm thử. Cosmos SDK cung cấp một cách để kiểm thử ứng dụng của bạn trong môi trường thực tế: một testnet.

Chúng tôi cho phép các nhà phát triển lấy trạng thái từ mainnet của họ và chạy kiểm thử trên trạng thái đó. Điều này hữu ích để kiểm thử các migration nâng cấp, hoặc để kiểm thử ứng dụng trong môi trường thực tế.

## Thiết Lập Testnet

Chúng ta sẽ phân tích các bước để tạo một testnet từ trạng thái mainnet.

```go 
  // InitSimAppForTestnet được chia thành hai phần:
  // Thay đổi Bắt buộc: Các thay đổi mà nếu không thực hiện sẽ khiến testnet bị dừng hoặc panic
  // Thay đổi Tùy chọn: Các thay đổi để tùy chỉnh testnet theo ý muốn (giảm thời gian bỏ phiếu, nạp tiền tài khoản, v.v.)
  func InitSimAppForTestnet(app *SimApp, newValAddr bytes.HexBytes, newValPubKey crypto.PubKey, newOperatorAddress, upgradeToTrigger string) *SimApp {
  ...
  }
```

### Thay Đổi Bắt Buộc

#### Staking

Khi tạo testnet, phần quan trọng là di chuyển bộ validator từ nhiều validator sang một hoặc vài validator. Điều này cho phép các nhà phát triển khởi động chuỗi mà không cần phải thay thế các khóa validator.

```go
	ctx := app.BaseApp.NewUncachedContext(true, tmproto.Header{})
	pubkey := &ed25519.PubKey{Key: newValPubKey.Bytes()}
	pubkeyAny, err := types.NewAnyWithValue(pubkey)
	if err != nil {
		tmos.Exit(err.Error())
	}

	// STAKING
	//

	// Tạo cấu trúc Validator cho validator mới.
	_, bz, err := bech32.DecodeAndConvert(newOperatorAddress)
	if err != nil {
		tmos.Exit(err.Error())
	}
	bech32Addr, err := bech32.ConvertAndEncode("simvaloper", bz)
	if err != nil {
		tmos.Exit(err.Error())
	}
	newVal := stakingtypes.Validator{
		OperatorAddress: bech32Addr,
		ConsensusPubkey: pubkeyAny,
		Jailed:          false,
		Status:          stakingtypes.Bonded,
		Tokens:          sdk.NewInt(900000000000000),
		DelegatorShares: sdk.MustNewDecFromStr("10000000"),
		Description: stakingtypes.Description{
			Moniker: "Testnet Validator",
		},
		Commission: stakingtypes.Commission{
			CommissionRates: stakingtypes.CommissionRates{
				Rate:          sdk.MustNewDecFromStr("0.05"),
				MaxRate:       sdk.MustNewDecFromStr("0.1"),
				MaxChangeRate: sdk.MustNewDecFromStr("0.05"),
			},
		},
		MinSelfDelegation: sdk.OneInt(),
	}

	// Xóa tất cả validator khỏi power store
	stakingKey := app.GetKey(stakingtypes.ModuleName)
	stakingStore := ctx.KVStore(stakingKey)
	iterator := app.StakingKeeper.ValidatorsPowerStoreIterator(ctx)
	for ; iterator.Valid(); iterator.Next() {
		stakingStore.Delete(iterator.Key())
	}
	iterator.Close()

	// Xóa tất cả validator khỏi last validators store
	iterator = app.StakingKeeper.LastValidatorsIterator(ctx)
	for ; iterator.Valid(); iterator.Next() {
		app.StakingKeeper.LastValidatorPower.Delete(iterator.Key())
	}
	iterator.Close()

	// Thêm validator của chúng ta vào power store và last validators store
	app.StakingKeeper.SetValidator(ctx, newVal)
	err = app.StakingKeeper.SetValidatorByConsAddr(ctx, newVal)
	if err != nil {
		panic(err)
	}
	app.StakingKeeper.SetValidatorByPowerIndex(ctx, newVal)
	app.StakingKeeper.SetLastValidatorPower(ctx, newVal.GetOperator(), 0)
	if err := app.StakingKeeper.Hooks().AfterValidatorCreated(ctx, newVal.GetOperator()); err != nil {
		panic(err)
	}
```

#### Distribution (Phân Phối)

Vì bộ validator đã thay đổi, chúng ta cần cập nhật các bản ghi phân phối cho validator mới.

```go
	// Khởi tạo bản ghi cho validator này trên tất cả các distribution store
	app.DistrKeeper.ValidatorHistoricalRewards.Set(ctx, newVal.GetOperator(), 0, distrtypes.NewValidatorHistoricalRewards(sdk.DecCoins{}, 1))
	app.DistrKeeper.ValidatorCurrentRewards.Set(ctx, newVal.GetOperator(), distrtypes.NewValidatorCurrentRewards(sdk.DecCoins{}, 1))
	app.DistrKeeper.ValidatorAccumulatedCommission.Set(ctx, newVal.GetOperator(), distrtypes.InitialValidatorAccumulatedCommission())
	app.DistrKeeper.ValidatorOutstandingRewards.Set(ctx, newVal.GetOperator(), distrtypes.ValidatorOutstandingRewards{Rewards: sdk.DecCoins{}})
```

#### Slashing

Chúng ta cũng cần đặt thông tin ký (signing info) của validator cho validator mới.

```go
  // SLASHING
	//

	// Đặt thông tin ký của validator cho validator mới.
	newConsAddr := sdk.ConsAddress(newValAddr.Bytes())
	newValidatorSigningInfo := slashingtypes.ValidatorSigningInfo{
		Address:     newConsAddr.String(),
		StartHeight: app.LastBlockHeight() - 1,
		Tombstoned:  false,
	}
	app.SlashingKeeper.ValidatorSigningInfo.Set(ctx, newConsAddr, newValidatorSigningInfo)
```

#### Bank

Việc tạo các tài khoản mới cho mục đích kiểm thử rất hữu ích. Điều này giúp tránh cần phải có cùng khóa như trên mainnet.

```go
  // BANK
	//

	defaultCoins := sdk.NewCoins(sdk.NewInt64Coin("ustake", 1000000000000))

	localSimAppAccounts := []sdk.AccAddress{
		sdk.MustAccAddressFromBech32("cosmos12smx2wdlyttvyzvzg54y2vnqwq2qjateuf7thj"),
		sdk.MustAccAddressFromBech32("cosmos1cyyzpxplxdzkeea7kwsydadg87357qnahakaks"),
		sdk.MustAccAddressFromBech32("cosmos18s5lynnmx37hq4wlrw9gdn68sg2uxp5rgk26vv"),
		sdk.MustAccAddressFromBech32("cosmos1qwexv7c6sm95lwhzn9027vyu2ccneaqad4w8ka"),
		sdk.MustAccAddressFromBech32("cosmos14hcxlnwlqtq75ttaxf674vk6mafspg8xwgnn53"),
		sdk.MustAccAddressFromBech32("cosmos12rr534cer5c0vj53eq4y32lcwguyy7nndt0u2t"),
		sdk.MustAccAddressFromBech32("cosmos1nt33cjd5auzh36syym6azgc8tve0jlvklnq7jq"),
		sdk.MustAccAddressFromBech32("cosmos10qfrpash5g2vk3hppvu45x0g860czur8ff5yx0"),
		sdk.MustAccAddressFromBech32("cosmos1f4tvsdukfwh6s9swrc24gkuz23tp8pd3e9r5fa"),
		sdk.MustAccAddressFromBech32("cosmos1myv43sqgnj5sm4zl98ftl45af9cfzk7nhjxjqh"),
		sdk.MustAccAddressFromBech32("cosmos14gs9zqh8m49yy9kscjqu9h72exyf295afg6kgk"),
		sdk.MustAccAddressFromBech32("cosmos1jllfytsz4dryxhz5tl7u73v29exsf80vz52ucc")}

  // Nạp tiền cho các tài khoản localSimApp
	for _, account := range localSimAppAccounts {
		err := app.BankKeeper.MintCoins(ctx, minttypes.ModuleName, defaultCoins)
		if err != nil {
			tmos.Exit(err.Error())
		}
		err = app.BankKeeper.SendCoinsFromModuleToAccount(ctx, minttypes.ModuleName, account, defaultCoins)
		if err != nil {
			tmos.Exit(err.Error())
		}
	}
```

#### Upgrade (Nâng Cấp)

Nếu bạn muốn lên lịch một lần nâng cấp, có thể sử dụng đoạn code dưới đây.

```go
	// UPGRADE
	//

	if upgradeToTrigger != "" {
		upgradePlan := upgradetypes.Plan{
			Name:   upgradeToTrigger,
			Height: app.LastBlockHeight(),
		}
		err = app.UpgradeKeeper.ScheduleUpgrade(ctx, upgradePlan)
		if err != nil {
			panic(err)
		}
	}
```

### Thay Đổi Tùy Chọn

Nếu bạn có các module tùy chỉnh phụ thuộc vào trạng thái cụ thể từ các module trên và/hoặc bạn muốn kiểm thử module tùy chỉnh của mình, bạn sẽ cần cập nhật trạng thái của module tùy chỉnh đó để phản ánh nhu cầu của bạn.

## Chạy Testnet

Trước khi có thể chạy testnet, chúng ta phải kết nối mọi thứ lại với nhau.

Trong `root.go`, trong hàm `initRootCmd`, thêm:

```diff
  server.AddCommands(rootCmd, simapp.DefaultNodeHome, newApp, createSimAppAndExport, addModuleInitFlags)
	++ server.AddTestnetCreatorCommand(rootCmd, simapp.DefaultNodeHome, newTestnetApp, addModuleInitFlags)
```

Tiếp theo, chúng ta sẽ thêm hàm helper `newTestnetApp`:

```diff
// newTestnetApp bắt đầu bằng cách chạy phương thức newApp thông thường. Từ đó, interface ứng dụng
// được trả về sẽ được sửa đổi để tạo một testnet từ ứng dụng đã cung cấp.
func newTestnetApp(logger log.Logger, db cometbftdb.DB, traceStore io.Writer, appOpts servertypes.AppOptions) servertypes.Application {
	// Tạo một app và ép kiểu thành SimApp
	app := newApp(logger, db, traceStore, appOpts)
	simApp, ok := app.(*simapp.SimApp)
	if !ok {
		panic("app created from newApp is not of type simApp")
	}

	newValAddr, ok := appOpts.Get(server.KeyNewValAddr).(bytes.HexBytes)
	if !ok {
		panic("newValAddr is not of type bytes.HexBytes")
	}
	newValPubKey, ok := appOpts.Get(server.KeyUserPubKey).(crypto.PubKey)
	if !ok {
		panic("newValPubKey is not of type crypto.PubKey")
	}
	newOperatorAddress, ok := appOpts.Get(server.KeyNewOpAddr).(string)
	if !ok {
		panic("newOperatorAddress is not of type string")
	}
	upgradeToTrigger, ok := appOpts.Get(server.KeyTriggerTestnetUpgrade).(string)
	if !ok {
		panic("upgradeToTrigger is not of type string")
	}

	// Thực hiện các sửa đổi cần thiết cho SimApp thông thường để chạy mạng cục bộ
	return simapp.InitSimAppForTestnet(simApp, newValAddr, newValPubKey, newOperatorAddress, upgradeToTrigger)
}
```
