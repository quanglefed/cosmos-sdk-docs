# Nâng Cấp Cosmos SDK

Hướng dẫn này cung cấp các hướng dẫn để nâng cấp lên các phiên bản cụ thể của Cosmos SDK.
Lưu ý, luôn đọc phần **SimApp** để biết thêm thông tin về các cập nhật kết nối ứng dụng.

## [v0.50.x](https://github.com/cosmos/cosmos-sdk/releases/tag/v0.50.0)

### Migration sang CometBFT (Phần 2)

Cosmos SDK đã di chuyển trong các phiên bản trước của nó sang CometBFT.
Một số hàm đã được đổi tên để phản ánh sự thay đổi tên.

Danh sách đầy đủ:

* `client.TendermintRPC` -> `client.CometRPC`
* `clitestutil.MockTendermintRPC` -> `clitestutil.MockCometRPC`
* `clitestutilgenutil.CreateDefaultTendermintConfig` -> `clitestutilgenutil.CreateDefaultCometConfig`
* Package `client/grpc/tmservice` -> `client/grpc/cmtservice`

Ngoài ra, các lệnh và flag đề cập đến `tendermint` đã được đổi tên thành `comet`.
Các lệnh và flag này vẫn được hỗ trợ cho khả năng tương thích ngược.

Để tương thích ngược, các gRPC service `**/tendermint/**` vẫn được hỗ trợ.

Ngoài ra, SDK bắt đầu trừu tượng hóa khỏi các kiểu Go của CometBFT trong codebase:

* Việc sử dụng CometBFT logger đã được thay thế bằng interface logger của Cosmos SDK (`cosmossdk.io/log.Logger`).
* Việc sử dụng `github.com/cometbft/cometbft/libs/bytes.HexByte` đã được thay thế bằng `[]byte`.
* Sử dụng application genesis (xem [genutil](#xgenutil)).

#### Kích Hoạt Vote Extensions

:::tip
Đây là tính năng tùy chọn và bị tắt theo mặc định.
:::

Khi tất cả các thay đổi mã cần thiết để triển khai Vote Extensions đã được thực hiện,
chúng có thể được kích hoạt bằng cách đặt consensus param `Abci.VoteExtensionsEnableHeight`
thành giá trị lớn hơn không.

Trên chain mới, điều này có thể được thực hiện trong file `genesis.json`.

Đối với các chain hiện có, có thể thực hiện theo hai cách:

* Trong quá trình upgrade, giá trị được đặt trong upgrade handler.
* Một governance proposal thay đổi consensus param **sau khi đã hoàn thành một coordinated upgrade**.

### BaseApp

Tất cả các phương thức ABCI hiện chấp nhận con trỏ đến các kiểu request và response được định nghĩa
bởi CometBFT. Ngoài ra, chúng cũng trả về các lỗi. Một phương thức ABCI chỉ nên
trả về lỗi trong trường hợp xảy ra lỗi thảm khốc và ứng dụng
cần dừng lại. Tuy nhiên, điều này được trừu tượng hóa khỏi nhà phát triển ứng dụng. Bất kỳ
handler nào mà ứng dụng có thể định nghĩa hoặc đặt trả về lỗi, sẽ được xử lý một cách
graceful bởi `BaseApp` thay mặt ứng dụng.

Các lời gọi `BeginBlock` và `Endblock` của BaseApp hiện là private nhưng vẫn được
hiển thị cho ứng dụng để định nghĩa thông qua kiểu `Manager`. `FinalizeBlock` là public
và nên được sử dụng để test và chạy các operations. Điều này có nghĩa là mặc dù
`BeginBlock` và `Endblock` không còn tồn tại trong ABCI interface, chúng được tự động
gọi bởi `BaseApp` trong `FinalizeBlock`. Cụ thể, thứ tự các operations
là `BeginBlock` -> `DeliverTx` (cho tất cả các tx) -> `EndBlock`.

ABCI++ 2.0 cũng mang lại các phương thức ABCI `ExtendVote` và `VerifyVoteExtension`. Các
phương thức này cho phép các ứng dụng mở rộng và xác minh các phiếu pre-commit. Cosmos SDK
cho phép ứng dụng định nghĩa handler cho các phương thức này thông qua `ExtendVoteHandler`
và `VerifyVoteExtensionHandler` tương ứng. Vui lòng xem [tại đây](https://docs.cosmos.network/v0.50/build/building-apps/vote-extensions)
để biết thêm thông tin.

#### Đặt PreBlocker

Phương thức `SetPreBlocker` đã được thêm vào BaseApp. Điều này rất cần thiết để BaseApp chạy `PreBlock`,
cái chạy trước begin blocker của các module khác, và cho phép sửa đổi consensus parameter, và các thay đổi
này hiển thị với các logic state machine tiếp theo.
Đọc thêm về các trường hợp sử dụng khác [tại đây](https://github.com/cosmos/cosmos-sdk/blob/main/docs/architecture/adr-068-preblock.md).

Người dùng `depinject` / app di cần thêm `x/upgrade` vào `app_config.go` / `app.yml` của họ:

```diff
+ PreBlockers: []string{
+	upgradetypes.ModuleName,
+ },
BeginBlockers: []string{
-	upgradetypes.ModuleName,
	minttypes.ModuleName,
}
```

Khi sử dụng kết nối ứng dụng (legacy), cần thêm phần sau vào `app.go`:

```diff
+app.ModuleManager.SetOrderPreBlockers(
+	upgradetypes.ModuleName,
+)

app.ModuleManager.SetOrderBeginBlockers(
-	upgradetypes.ModuleName,
)

+ app.SetPreBlocker(app.PreBlocker)

// ... //

+func (app *SimApp) PreBlocker(ctx sdk.Context, req *abci.RequestFinalizeBlock) (*sdk.ResponsePreBlock, error) {
+	return app.ModuleManager.PreBlock(ctx, req)
+}
```

#### Events

Phần log của `abci.TxResult` không được điền trong trường hợp thực thi
msg(s) thành công. Thay vào đó, một thuộc tính mới được thêm vào tất cả các message chỉ ra
`msg_index` giúp xác định sự kiện và thuộc tính nào liên quan đến cùng một
giao dịch.

Các sự kiện `BeginBlock` và `EndBlock` hiện được phát ra thông qua `FinalizeBlock` nhưng có
thêm một thuộc tính, `mode=BeginBlock|EndBlock`, để xác định sự kiện thuộc về
`BeginBlock` hay `EndBlock`.

### Config files (File Cấu Hình)

Confix là công cụ SDK mới để sửa đổi và di chuyển cấu hình của SDK.
Đây là sự thay thế cho lệnh `config.Cmd` từ package `client/config`.

Sử dụng lệnh sau để di chuyển cấu hình của bạn:

```bash
simd config migrate v0.50
```

Nếu bạn đang sử dụng `<appd> config [key]` hoặc `<appd> config [key] [value]` để đặt và lấy giá trị từ `client.toml`, hãy thay thế bằng `<appd> config get client [key]` và `<appd> config set client [key] [value]`. Độ verbose thêm là do các chức năng bổ sung được thêm vào config.

Thông tin thêm về [confix](https://docs.cosmos.network/main/tooling/confix) và cách thêm nó vào binary ứng dụng trong [tài liệu](https://docs.cosmos.network/main/tooling/confix).

#### gRPC-Web

gRPC-Web hiện đang lắng nghe cùng địa chỉ và port với máy chủ API gRPC Gateway (mặc định: `localhost:1317`).
Khả năng lắng nghe trên địa chỉ khác đã bị xóa, cùng với các cài đặt của nó.
Sử dụng `confix` để dọn dẹp `app.toml` của bạn. Có thể đặt nginx (hoặc tương tự) reverse-proxy để giữ hành vi trước đây.

#### Hỗ Trợ Database

ClevelDB, BoltDB và BadgerDB không còn được hỗ trợ nữa. Để migrate từ database không được hỗ trợ sang database được hỗ trợ, vui lòng sử dụng công cụ migration database.

### Protobuf

Với việc ngừng sử dụng Amino JSON codec được định nghĩa trong [cosmos/gogoproto](https://github.com/cosmos/gogoproto) thay bằng codec x/tx/aminojson chạy trên protoreflect, nhà phát triển module được khuyến khích xác minh rằng các message của họ có các annotation protobuf đúng để tạo ra đầu ra giống hệt nhau từ cả hai codec một cách xác định.

Đối với các kiểu SDK core, tính tương đương được khẳng định thông qua generative testing của [SignableTypes](https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-beta.0/tests/integration/rapidgen/rapidgen.go#L102) trong [TestAminoJSON_Equivalence](https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-beta.0/tests/integration/tx/aminojson/aminojson_test.go#L94).

**TODO: tóm tắt các yêu cầu annotation proto.**

#### Stringer

Annotation `gogoproto.goproto_stringer = false` đã bị xóa khỏi hầu hết các file proto. Điều này có nghĩa là phương thức `String()` đang được tạo cho các kiểu trước đây có annotation này. Phương thức `String()` được tạo sử dụng `proto.CompactTextString` để _stringify_ các struct.
[Xác minh](https://github.com/cosmos/cosmos-sdk/pull/13850#issuecomment-1328889651) việc sử dụng các phương thức `String()` đã sửa đổi và kiểm tra kỹ rằng chúng không được sử dụng trong mã state-machine.

### SimApp

Trong phần này chúng tôi mô tả các thay đổi được thực hiện trong SimApp của Cosmos SDK.
**Các thay đổi này có thể áp dụng trực tiếp cho kết nối ứng dụng của bạn.**

#### Module Assertions

Trước đây, tất cả các module đều phải được đặt trong `OrderBeginBlockers`, `OrderEndBlockers` và `OrderInitGenesis / OrderExportGenesis` trong `app.go` / `app_config.go`. Điều này không còn là trường hợp nữa, điều kiện đã được nới lỏng để chỉ yêu cầu các module triển khai tương ứng các interface `appmodule.HasBeginBlocker`, `appmodule.HasEndBlocker` và `appmodule.HasGenesis` / `module.HasGenesis`.

#### Module wiring (Kết Nối Module)

Hàm `NewKeeper` của các module sau hiện nhận `KVStoreService` thay vì `StoreKey`:

* `x/auth`
* `x/authz`
* `x/bank`
* `x/consensus`
* `x/crisis`
* `x/distribution`
* `x/evidence`
* `x/feegrant`
* `x/gov`
* `x/mint`
* `x/nft`
* `x/slashing`
* `x/upgrade`

**Người dùng dùng `depinject` / app di không cần thay đổi gì, điều này được trừu tượng hóa cho họ.**

Người dùng kết nối chain thủ công cần sử dụng phương thức `runtime.NewKVStoreService` để tạo `KVStoreService` từ `StoreKey`:

```diff
app.ConsensusParamsKeeper = consensusparamkeeper.NewKeeper(
  appCodec,
- keys[consensusparamtypes.StoreKey]
+ runtime.NewKVStoreService(keys[consensusparamtypes.StoreKey]),
  authtypes.NewModuleAddress(govtypes.ModuleName).String(),
)
```

#### Logger

Thay thế tất cả các import CometBFT logger của bạn bằng `cosmossdk.io/log`.

Ngoài ra, người dùng `depinject` / app di bây giờ phải cung cấp logger thông qua hàm `depinject.Supply` chính thay vì truyền nó cho `appBuilder.Build`.

```diff
appConfig = depinject.Configs(
	AppConfig,
	depinject.Supply(
		// supply the application options
		appOpts,
+		logger,
	...
```

```diff
- app.App = appBuilder.Build(logger, db, traceStore, baseAppOptions...)
+ app.App = appBuilder.Build(db, traceStore, baseAppOptions...)
```

Người dùng kết nối chain thủ công cần thêm đối số logger khi tạo keeper `x/bank`.

#### Module Basics

Trước đây, `ModuleBasics` là một biến toàn cục được sử dụng để đăng ký triển khai `AppModuleBasic` của tất cả các module.
Biến toàn cục đã bị xóa và basic module manager giờ có thể được tạo từ module manager.

Điều này được thực hiện tự động cho người dùng `depinject` / app di, tuy nhiên để cung cấp các triển khai module app khác nhau, truyền chúng qua `depinject.Supply` trong `AppConfig` chính (`app_config.go`):

```go
depinject.Supply(
			// supply custom module basics
			map[string]module.AppModuleBasic{
				genutiltypes.ModuleName: genutil.NewAppModuleBasic(genutiltypes.DefaultMessageValidator),
				govtypes.ModuleName: gov.NewAppModuleBasic(
					[]govclient.ProposalHandler{
						paramsclient.ProposalHandler,
					},
				),
			},
		)
```

Người dùng kết nối chain thủ công cần sử dụng hàm `module.NewBasicManagerFromManager` mới, sau khi tạo module manager, và truyền `map[string]module.AppModuleBasic` làm đối số để tùy chọn ghi đè `AppModuleBasic` của một số module.

#### AutoCLI

[`AutoCLI`](https://docs.cosmos.network/main/core/autocli) đã được SDK triển khai cho tất cả các CLI query của module. Điều này có nghĩa là các chain phải thêm phần sau vào `root.go` để bật `AutoCLI` trong ứng dụng của họ:

```go
if err := autoCliOpts.EnhanceRootCommand(rootCmd); err != nil {
	panic(err)
}
```

Trong đó `autoCliOpts` là các tùy chọn autocli của app, chứa tất cả các module và codec.
Giá trị đó có thể được inject bởi depinject ([xem root_v2.go](https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-beta.0/simapp/simd/cmd/root_v2.go#L49-L67)) hoặc được cung cấp thủ công bởi app ([xem legacy app.go](https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-beta.0/simapp/app.go#L636-L655)).

:::warning
Không làm điều này sẽ dẫn đến tất cả các query của module SDK core không được đưa vào binary.
:::

Ngoài ra, `AutoCLI` tự động thêm các lệnh module tùy chỉnh vào lệnh gốc cho tất cả các module triển khai interface [`appmodule.AppModule`](https://pkg.go.dev/cosmossdk.io/core/appmodule#AppModule).
Điều này có nghĩa là sau khi đảm bảo tất cả các module được sử dụng triển khai interface này, có thể xóa phần sau khỏi `root.go`:

```diff
func txCommand() *cobra.Command {
	....
- appd.ModuleBasics.AddTxCommands(cmd)
}
```

```diff
func queryCommand() *cobra.Command {
	....
- appd.ModuleBasics.AddQueryCommands(cmd)
}
```

### Packages (Gói)

#### Math

Các tham chiếu đến `types/math.go` chứa các alias cho các kiểu math aliasing package `cosmossdk.io/math` đã bị xóa.
Import trực tiếp package `cosmossdk.io/math` thay thế.

#### Store

Các tham chiếu đến `types/store.go` chứa các alias cho các kiểu store đã được ánh xạ lại để trỏ đến `store/types` thích hợp, do đó file `types/store.go` không còn cần thiết và đã bị xóa.

##### Tách Store thành module độc lập

Module `store` được tách ra để có file go.mod riêng cho phép nó là một module độc lập.
Tất cả các import store hiện được đổi tên để sử dụng `cosmossdk.io/store` thay vì `github.com/cosmos/cosmos-sdk/store` trong SDK.

##### Streaming

[ADR-38](https://docs.cosmos.network/main/architecture/adr-038-state-listening) đã được triển khai trong SDK.

Để tiếp tục sử dụng state streaming, thay thế `streaming.LoadStreamingServices` bằng phần sau trong `app.go`:

```go
if err := app.RegisterStreamingServices(appOpts, app.kvStoreKeys()); err != nil {
	panic(err)
}
```

#### Client

Kiểu trả về của phương thức interface `TxConfig.SignModeHandler()` đã được thay đổi từ `x/auth/signing.SignModeHandler` thành `x/tx/signing.HandlerMap`. Thay đổi này trong suốt với hầu hết người dùng vì interface `TxConfig` thường được triển khai bởi struct `x/auth/tx.config` private (như được trả về bởi `auth.NewTxConfig`) đã được cập nhật để trả về kiểu mới. Nếu người dùng đã triển khai interface `TxConfig` của riêng họ, họ cần cập nhật triển khai để trả về kiểu mới.

##### Textual sign mode (Chế Độ Ký Dạng Văn Bản)

Một sign mode mới có sẵn trong SDK tạo ra đầu ra dễ đọc hơn cho con người, hiện chỉ khả dụng trên
thiết bị Ledger nhưng sắp được triển khai trong các UI khác.

:::tip
Sign mode này không cho phép ký ngoại tuyến
:::

Khi sử dụng kết nối ứng dụng (legacy), cần thêm phần sau vào `app.go` sau khi đặt bank keeper của app:

```go
	enabledSignModes := append(tx.DefaultSignModes, sigtypes.SignMode_SIGN_MODE_TEXTUAL)
	txConfigOpts := tx.ConfigOptions{
		EnabledSignModes:           enabledSignModes,
		TextualCoinMetadataQueryFn: txmodule.NewBankKeeperCoinMetadataQueryFn(app.BankKeeper),
	}
	txConfig, err := tx.NewTxConfigWithOptions(
		appCodec,
		txConfigOpts,
	)
	if err != nil {
		log.Fatalf("Failed to create new TxConfig with options: %v", err)
	}
	app.txConfig = txConfig
```

Khi sử dụng `depinject` / `app di`, **nó được bật theo mặc định** nếu có bank keeper.

Và trong application client (thường là `root.go`):

```go
	if !clientCtx.Offline {
		txConfigOpts.EnabledSignModes = append(txConfigOpts.EnabledSignModes, signing.SignMode_SIGN_MODE_TEXTUAL)
		txConfigOpts.TextualCoinMetadataQueryFn = txmodule.NewGRPCCoinMetadataQueryFn(clientCtx)
		txConfigWithTextual, err := tx.NewTxConfigWithOptions(
			codec.NewProtoCodec(clientCtx.InterfaceRegistry),
			txConfigOpts,
		)
		if err != nil {
			return err
		}
		clientCtx = clientCtx.WithTxConfig(txConfigWithTextual)
	}
```

Khi sử dụng `depinject` / `app di`, tx config nên được tạo lại từ `txConfigOpts` để sử dụng `NewGRPCCoinMetadataQueryFn` thay vì phụ thuộc vào bank keeper (được sử dụng trong server).

Để tìm hiểu thêm xem [docs](https://docs.cosmos.network/main/learn/advanced/transactions#sign_mode_textual) và [ADR-050](https://docs.cosmos.network/main/build/architecture/adr-050-sign-mode-textual).

### Modules

#### `**all**` (Tất Cả Module)

* [RFC 001](https://docs.cosmos.network/main/rfc/rfc-001-tx-validation) đã định nghĩa sự đơn giản hóa quy trình xác thực message cho các module.
  Interface `sdk.Msg` đã được cập nhật để không yêu cầu triển khai phương thức `ValidateBasic`.
  Hiện khuyến nghị xác thực message trực tiếp trong message server. Khi việc xác thực được thực hiện trong message server, phương thức `ValidateBasic` trên message không còn cần thiết và có thể bị xóa.

* Các message không còn cần triển khai interface `LegacyMsg` và các triển khai của `GetSignBytes` có thể bị xóa. Do sự thay đổi này, các định nghĩa amino codec kế thừa toàn cục và việc đăng ký chúng trong `init()` cũng có thể được xóa an toàn.

* Interface `AppModuleBasic` đã được đơn giản hóa. Không còn cần thiết phải định nghĩa `GetTxCmd() *cobra.Command` và `GetQueryCmd() *cobra.Command`. Module manager phát hiện khi các lệnh module được định nghĩa. Nếu AutoCLI được bật, `EnhanceRootCommand()` sẽ thêm các lệnh được tạo tự động vào lệnh gốc, trừ khi một lệnh module tùy chỉnh được định nghĩa và đăng ký thay thế.

* Các phương thức `Keeper` của các module sau hiện nhận `context.Context` thay vì `sdk.Context`. Bất kỳ module nào có interface cho chúng (như "expected keepers") sẽ cần cập nhật và tạo lại mock nếu cần:

    * `x/authz`
    * `x/bank`
    * `x/mint`
    * `x/crisis`
    * `x/distribution`
    * `x/evidence`
    * `x/gov`
    * `x/slashing`
    * `x/upgrade`

* `BeginBlock` và `EndBlock` đã thay đổi chữ ký, vì vậy điều quan trọng là bất kỳ module nào triển khai chúng đều được cập nhật tương ứng.

```diff
- BeginBlock(sdk.Context, abci.RequestBeginBlock)
+ BeginBlock(context.Context) error
```

```diff
- EndBlock(sdk.Context, abci.RequestEndBlock) []abci.ValidatorUpdate
+ EndBlock(context.Context) error
```

Trong trường hợp module cần trả về `abci.ValidatorUpdate` từ `EndBlock`, có thể sử dụng interface `HasABCIEndBlock` thay thế.

```diff
- EndBlock(sdk.Context, abci.RequestEndBlock) []abci.ValidatorUpdate
+ EndBlock(context.Context) ([]abci.ValidatorUpdate, error)
```

:::tip
Có thể đảm bảo rằng một module triển khai đúng các interface bằng cách sử dụng compiler assertions trong `x/{moduleName}/module.go`:

```go
var (
	_ module.AppModuleBasic      = (*AppModule)(nil)
	_ module.AppModuleSimulation = (*AppModule)(nil)
	_ module.HasGenesis          = (*AppModule)(nil)

	_ appmodule.AppModule        = (*AppModule)(nil)
	_ appmodule.HasBeginBlocker  = (*AppModule)(nil)
	_ appmodule.HasEndBlocker    = (*AppModule)(nil)
	...
)
```

Đọc thêm về các interface đó [tại đây](https://docs.cosmos.network/v0.50/building-modules/module-manager#application-module-interfaces).

:::

* `GetSigners()` không còn cần được triển khai trên các kiểu `Msg`. SDK sẽ tự động suy ra các signer từ trường `Signer` trên message. Trường signer là bắt buộc trên tất cả các message trừ khi sử dụng hàm signer tùy chỉnh.

Để tìm hiểu thêm, đọc tài liệu về [trường signer](../../build/building-modules/05-protobuf-annotations.md#signer) và [tại đây](https://github.com/cosmos/cosmos-sdk/blob/7352d0bce8e72121e824297df453eb1059c28da8/docs/docs/build/building-modules/02-messages-and-queries.md#L40).

#### `x/auth`

Đối với việc xây dựng ante handler qua `ante.NewAnteHandler`, trường `ante.HandlerOptions.SignModeHandler` đã được cập nhật thành `x/tx/signing/HandlerMap` từ `x/auth/signing/SignModeHandler`. Người gọi thường lấy giá trị này từ `client.TxConfig.SignModeHandler()` (cũng đã thay đổi) nên thay đổi này nên trong suốt với hầu hết người dùng.

#### `x/capability`

Module capability đã được chuyển sang [cosmos/ibc-go](https://github.com/cosmos/ibc-go). IBC v8 sẽ chứa các thay đổi cần thiết để tích hợp vị trí module mới. Trong `app.go` của bạn, cần import module capability từ vị trí mới:

```diff
+	"github.com/cosmos/ibc-go/modules/capability"
+	capabilitykeeper "github.com/cosmos/ibc-go/modules/capability/keeper"
+	capabilitytypes "github.com/cosmos/ibc-go/modules/capability/types"
-	"github.com/cosmos/cosmos-sdk/x/capability/types"
-	capabilitykeeper "github.com/cosmos/cosmos-sdk/x/capability/keeper"
-	capabilitytypes "github.com/cosmos/cosmos-sdk/x/capability/types"
```

Tương tự như các phiên bản trước, module manager của bạn phải bao gồm module capability.

```go
app.ModuleManager = module.NewManager(
	capability.NewAppModule(encodingConfig.Codec, *app.CapabilityKeeper, true),
	// remaining modules
)
```

#### `x/genutil`

Cosmos SDK đã di chuyển từ genesis CometBFT sang file genesis do ứng dụng quản lý.
Genesis hiện được xử lý hoàn toàn bởi `x/genutil`. Điều này không có hậu quả gì cho các chain đang chạy:

* Import genesis CometBFT vẫn được hỗ trợ.
* Export genesis hiện xuất genesis dưới dạng application genesis.

Khi cần đọc application genesis, sử dụng các helper sau từ package `x/genutil/types`:

```go
// AppGenesisFromReader đọc AppGenesis từ reader.
func AppGenesisFromReader(reader io.Reader) (*AppGenesis, error)

// AppGenesisFromFile đọc AppGenesis từ file được cung cấp.
func AppGenesisFromFile(genFile string) (*AppGenesis, error)
```

#### `x/gov`

##### Expedited Proposals (Đề Xuất Nhanh)

Module `gov` v1 hiện hỗ trợ expedited governance proposals. Khi một proposal được expedited, thời gian bỏ phiếu sẽ được rút ngắn xuống tham số `ExpeditedVotingPeriod`. Một expedited proposal phải có ngưỡng bỏ phiếu cao hơn proposal thông thường, ngưỡng đó được định nghĩa bởi tham số `ExpeditedThreshold`.

##### Cancelling Proposals (Hủy Đề Xuất)

Module `gov` hiện hỗ trợ hủy governance proposals. Khi một proposal bị hủy, tất cả các khoản deposit của proposal đó hoặc bị đốt hoặc được gửi đến địa chỉ `ProposalCancelDest`. Tỷ lệ đốt deposit sẽ được xác định bởi tham số mới gọi là `ProposalCancelRatio`.

```text
1. deposits * proposal_cancel_ratio sẽ bị đốt hoặc gửi đến địa chỉ `ProposalCancelDest`, nếu `ProposalCancelDest` trống thì deposits sẽ bị đốt.
2. deposits * (1 - proposal_cancel_ratio) sẽ được gửi cho người deposit.
```

Theo mặc định, tham số `ProposalCancelRatio` mới được đặt thành `0.5` trong quá trình migration và `ProposalCancelDest` được đặt thành chuỗi rỗng (tức là bị đốt).

#### `x/evidence`

##### Tách evidence thành module độc lập

Module `x/evidence` được tách ra để có file go.mod riêng cho phép nó là một module độc lập.
Tất cả các import evidence hiện được đổi tên để sử dụng `cosmossdk.io/x/evidence` thay vì `github.com/cosmos/cosmos-sdk/x/evidence` trong SDK.

#### `x/nft`

##### Tách nft thành module độc lập

Module `x/nft` được tách ra để có file go.mod riêng cho phép nó là một module độc lập.
Tất cả các import evidence hiện được đổi tên để sử dụng `cosmossdk.io/x/nft` thay vì `github.com/cosmos/cosmos-sdk/x/nft` trong SDK.

#### x/feegrant

##### Tách feegrant thành module độc lập

Module `x/feegrant` được tách ra để có file go.mod riêng cho phép nó là một module độc lập.
Tất cả các import feegrant hiện được đổi tên để sử dụng `cosmossdk.io/x/feegrant` thay vì `github.com/cosmos/cosmos-sdk/x/feegrant` trong SDK.

#### `x/upgrade`

##### Tách upgrade thành module độc lập

Module `x/upgrade` được tách ra để có file go.mod riêng cho phép nó là một module độc lập.
Tất cả các import upgrade hiện được đổi tên để sử dụng `cosmossdk.io/x/upgrade` thay vì `github.com/cosmos/cosmos-sdk/x/upgrade` trong SDK.

### Tooling (Công Cụ)

#### Rosetta

Rosetta đã chuyển sang [repo riêng](https://github.com/cosmos/rosetta) và không được SimApp của Cosmos SDK import theo mặc định nữa.
Bất kỳ người dùng nào quan tâm đến việc sử dụng công cụ này có thể kết nối nó độc lập với bất kỳ node nào mà không cần thêm nó như một phần của node binary.

Công cụ rosetta cũng cho phép kết nối multi chain.
