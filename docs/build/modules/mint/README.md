---
sidebar_position: 1
---

# `x/mint`

Module `x/mint` xử lý việc mint đều đặn các token mới theo cách có thể cấu hình.

## Nội dung

* [State](#state)
  * [Minter](#minter)
  * [Params](#params)
* [Begin-Block](#begin-block)
  * [NextInflationRate](#nextinflationrate)
  * [NextAnnualProvisions](#nextannualprovisions)
  * [BlockProvision](#blockprovision)
* [Parameters](#parameters)
* [Events](#events)
  * [BeginBlocker](#beginblocker)
* [Client](#client)
  * [CLI](#cli)
  * [gRPC](#grpc)
  * [REST](#rest)

## Khái niệm

### Cơ chế mint

Cơ chế mint mặc định được thiết kế để:

* cho phép một mức lạm phát linh hoạt được quyết định bởi nhu cầu thị trường, nhắm tới một tỷ lệ bonded-stake nhất định
* tạo cân bằng giữa thanh khoản thị trường và lượng cung đang stake

Để xác định mức lạm phát phù hợp theo thị trường cho phần thưởng, một cơ chế “tốc
độ thay đổi trượt” (moving change rate) được dùng. Cơ chế này đảm bảo rằng nếu %
bonded thực tế lớn hơn hoặc nhỏ hơn mục tiêu %-bonded, thì tỷ lệ lạm phát sẽ điều
chỉnh để khuyến khích hoặc giảm khuyến khích việc bonded tương ứng. Đặt mục tiêu
%-bonded nhỏ hơn 100% khuyến khích mạng duy trì một phần token không stake, giúp
tạo thanh khoản.

Có thể tóm tắt như sau:

* Nếu % token bonded thực tế thấp hơn mục tiêu %-bonded, tỷ lệ lạm phát sẽ tăng cho tới khi chạm giá trị tối đa
* Nếu mục tiêu % bonded (67% trong Cosmos Hub) được duy trì, tỷ lệ lạm phát sẽ giữ nguyên
* Nếu % token bonded thực tế cao hơn mục tiêu %-bonded, tỷ lệ lạm phát sẽ giảm cho tới khi chạm giá trị tối thiểu

### Custom minter

Từ Cosmos SDK v0.53.0, developer có thể đặt một `MintFn` tuỳ biến cho module để
thực hiện logic mint token chuyên biệt.

Chữ ký hàm mà `MintFn` phải triển khai như sau:

```go
// MintFn defines the function that needs to be implemented in order to customize the minting process.
type MintFn func(ctx sdk.Context, k *Keeper) error
```

Có thể truyền vào `Keeper` khi khởi tạo bằng một `Option` bổ sung:

```go
app.MintKeeper = mintkeeper.NewKeeper(
		appCodec,
		runtime.NewKVStoreService(keys[minttypes.StoreKey]),
		app.StakingKeeper,
		app.AccountKeeper,
		app.BankKeeper,
		authtypes.FeeCollectorName,
		authtypes.NewModuleAddress(govtypes.ModuleName).String(),
		// mintkeeper.WithMintFn(CUSTOM_MINT_FN), // custom mintFn can be added here
	)
```

#### Ví dụ DI cho custom minter

Dưới đây là một cách đơn giản để tạo hàm mint tuỳ biến với thêm phụ thuộc (dependency)
trong cấu hình DI. Với ví dụ cơ bản này, ta sẽ làm cho minter chỉ đơn giản “nhân đôi”
total supply của coin `foo`.

Trước hết, ta định nghĩa một hàm nhận các dependency cần thiết và trả về một `MintFn`.

```go
// MyCustomMintFunction is a custom mint function that doubles the supply of `foo` coin.
func MyCustomMintFunction(bank bankkeeper.BaseKeeper) mintkeeper.MintFn {
	return func(ctx sdk.Context, k *mintkeeper.Keeper) error {
		supply := bank.GetSupply(ctx, "foo")
		err := k.MintCoins(ctx, sdk.NewCoins(supply.Add(supply)))
		if err != nil {
			return err
		}
		return nil
	}
}
```

Sau đó, truyền hàm đã định nghĩa ở trên vào `depinject.Supply` cùng các dependency cần thiết.

```go
// NewSimApp returns a reference to an initialized SimApp.
func NewSimApp(
    logger log.Logger,
    db dbm.DB,
    traceStore io.Writer,
    loadLatest bool,
    appOpts servertypes.AppOptions,
    baseAppOptions ...func(*baseapp.BaseApp),
) *SimApp {
    var (
        app        = &SimApp{}
        appBuilder *runtime.AppBuilder
        appConfig = depinject.Configs(
            AppConfig,
            depinject.Supply(
                appOpts,
                logger,
                // our custom mint function with the necessary dependency passed in.
                MyCustomMintFunction(app.BankKeeper),
            ),
        )
	)
	// ...
}
```

## State

### Minter

Minter là nơi lưu thông tin lạm phát hiện tại.

* Minter: `0x00 -> ProtocolBuffer(minter)`

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/mint/v1beta1/mint.proto#L10-L24
```

### Params

Module mint lưu params trong state với prefix `0x01`, có thể được cập nhật bởi
governance hoặc địa chỉ authority.

* Params: `mint/params -> legacy_amino(params)`

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/mint/v1beta1/mint.proto#L26-L59
```

## Begin-Block

Tham số mint được tính lại và lạm phát được chi trả ở đầu mỗi block.

### Tính toán tỷ lệ lạm phát

Tỷ lệ lạm phát được tính bằng một “hàm tính lạm phát” được truyền vào `NewAppModule`.
Nếu không truyền hàm, SDK sẽ dùng hàm mặc định (`NextInflationRate`). Nếu cần custom
logic tính lạm phát, có thể làm bằng cách định nghĩa và truyền một hàm khớp chữ ký
`InflationCalculationFn`.

```go
type InflationCalculationFn func(ctx sdk.Context, minter Minter, params Params, bondedRatio math.LegacyDec) math.LegacyDec
```

#### NextInflationRate

Mục tiêu tỷ lệ lạm phát theo năm được tính lại ở mỗi block.
Lạm phát cũng phụ thuộc vào mức thay đổi (dương hoặc âm) tuỳ theo khoảng cách tới
tỷ lệ mục tiêu (67%). Mức thay đổi tối đa được định nghĩa là 13% mỗi năm; tuy nhiên,
lạm phát theo năm bị giới hạn trong khoảng 7% đến 20%.

```go
NextInflationRate(params Params, bondedRatio math.LegacyDec) (inflation math.LegacyDec) {
	inflationRateChangePerYear = (1 - bondedRatio/params.GoalBonded) * params.InflationRateChange
	inflationRateChange = inflationRateChangePerYear/blocksPerYr

	// increase the new annual inflation for this next block
	inflation += inflationRateChange
	if inflation > params.InflationMax {
		inflation = params.InflationMax
	}
	if inflation < params.InflationMin {
		inflation = params.InflationMin
	}

	return inflation
}
```

### NextAnnualProvisions

Tính annual provisions dựa trên total supply hiện tại và tỷ lệ lạm phát.
Tham số này được tính một lần mỗi block.

```go
NextAnnualProvisions(params Params, totalSupply math.LegacyDec) (provisions math.LegacyDec) {
	return Inflation * totalSupply
```

### BlockProvision

Tính provisions được tạo ra cho mỗi block dựa trên annual provisions hiện tại.
Provisions sau đó được mint bởi `ModuleMinterAccount` của module `mint`, rồi được
chuyển sang `FeeCollector` `ModuleAccount` của `auth`.

```go
BlockProvision(params Params) sdk.Coin {
	provisionAmt = AnnualProvisions/ params.BlocksPerYear
	return sdk.NewCoin(params.MintDenom, provisionAmt.Truncate())
```

## Parameters

Module mint có các tham số sau:

| Key                 | Type            | Ví dụ                   |
|---------------------|-----------------|-------------------------|
| MintDenom           | string          | "uatom"                 |
| InflationRateChange | string (dec)    | "0.130000000000000000"  |
| InflationMax        | string (dec)    | "0.200000000000000000"  |
| InflationMin        | string (dec)    | "0.070000000000000000"  |
| GoalBonded          | string (dec)    | "0.670000000000000000"  |
| BlocksPerYear       | string (uint64) | "6311520"               |

## Events

Module mint phát ra các event sau:

### BeginBlocker

| Type | Attribute Key     | Attribute Value    |
|------|-------------------|--------------------|
| mint | bonded_ratio      | {bondedRatio}      |
| mint | inflation         | {inflation}        |
| mint | annual_provisions | {annualProvisions} |
| mint | amount            | {amount}           |

## Client

### CLI

Người dùng có thể truy vấn và tương tác với module `mint` qua CLI.

#### Query

Các lệnh `query` cho phép truy vấn state của `mint`.

```shell
simd query mint --help
```

##### annual-provisions

Lệnh `annual-provisions` cho phép truy vấn giá trị annual provisions hiện tại.

```shell
simd query mint annual-provisions [flags]
```

Ví dụ:

```shell
simd query mint annual-provisions
```

Ví dụ output:

```shell
22268504368893.612100895088410693
```

##### inflation

Lệnh `inflation` cho phép truy vấn giá trị lạm phát hiện tại.

```shell
simd query mint inflation [flags]
```

Ví dụ:

```shell
simd query mint inflation
```

Ví dụ output:

```shell
0.199200302563256955
```

##### params

Lệnh `params` cho phép truy vấn tham số mint hiện tại.

```shell
simd query mint params [flags]
```

Ví dụ:

```yml
blocks_per_year: "4360000"
goal_bonded: "0.670000000000000000"
inflation_max: "0.200000000000000000"
inflation_min: "0.070000000000000000"
inflation_rate_change: "0.130000000000000000"
mint_denom: stake
```

### gRPC

Người dùng có thể truy vấn module `mint` qua các endpoint gRPC.

#### AnnualProvisions

Endpoint `AnnualProvisions` cho phép truy vấn annual provisions hiện tại.

```shell
/cosmos.mint.v1beta1.Query/AnnualProvisions
```

Ví dụ:

```shell
grpcurl -plaintext localhost:9090 cosmos.mint.v1beta1.Query/AnnualProvisions
```

Ví dụ output:

```json
{
  "annualProvisions": "1432452520532626265712995618"
}
```

#### Inflation

Endpoint `Inflation` cho phép truy vấn giá trị lạm phát hiện tại.

```shell
/cosmos.mint.v1beta1.Query/Inflation
```

Ví dụ:

```shell
grpcurl -plaintext localhost:9090 cosmos.mint.v1beta1.Query/Inflation
```

Ví dụ output:

```json
{
  "inflation": "130197115720711261"
}
```

#### Params

Endpoint `Params` cho phép truy vấn tham số mint hiện tại.

```shell
/cosmos.mint.v1beta1.Query/Params
```

Ví dụ:

```shell
grpcurl -plaintext localhost:9090 cosmos.mint.v1beta1.Query/Params
```

Ví dụ output:

```json
{
  "params": {
    "mintDenom": "stake",
    "inflationRateChange": "130000000000000000",
    "inflationMax": "200000000000000000",
    "inflationMin": "70000000000000000",
    "goalBonded": "670000000000000000",
    "blocksPerYear": "6311520"
  }
}
```

### REST

Người dùng có thể truy vấn module `mint` qua các endpoint REST.

#### annual-provisions

```shell
/cosmos/mint/v1beta1/annual_provisions
```

Ví dụ:

```shell
curl "localhost:1317/cosmos/mint/v1beta1/annual_provisions"
```

Ví dụ output:

```json
{
  "annualProvisions": "1432452520532626265712995618"
}
```

#### inflation

```shell
/cosmos/mint/v1beta1/inflation
```

Ví dụ:

```shell
curl "localhost:1317/cosmos/mint/v1beta1/inflation"
```

Ví dụ output:

```json
{
  "inflation": "130197115720711261"
}
```

#### params

```shell
/cosmos/mint/v1beta1/params
```

Ví dụ:

```shell
curl "localhost:1317/cosmos/mint/v1beta1/params"
```

Ví dụ output:

```json
{
  "params": {
    "mintDenom": "stake",
    "inflationRateChange": "130000000000000000",
    "inflationMax": "200000000000000000",
    "inflationMin": "70000000000000000",
    "goalBonded": "670000000000000000",
    "blocksPerYear": "6311520"
  }
}
```

