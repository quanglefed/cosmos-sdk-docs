---
sidebar_position: 1
---

# `x/bank`

## Tóm tắt

Tài liệu này mô tả module bank của Cosmos SDK.

Module bank chịu trách nhiệm xử lý chuyển coin đa tài sản giữa các tài khoản và theo dõi các pseudo-transfer đặc biệt phải hoạt động khác với một số loại tài khoản cụ thể (đáng chú ý là delegate/undelegate cho tài khoản vesting). Module này cung cấp nhiều interface với các khả năng khác nhau để tương tác an toàn với các module khác cần thay đổi số dư người dùng.

Ngoài ra, module bank theo dõi và cung cấp hỗ trợ truy vấn cho tổng nguồn cung của tất cả tài sản được sử dụng trong ứng dụng.

Module này được sử dụng trong Cosmos Hub.

## Nội dung

* [Supply](#supply)
    * [Total Supply](#total-supply)
* [Module Accounts](#module-accounts)
    * [Permissions](#permissions)
* [State](#state)
* [Params](#params)
* [Keepers](#keepers)
* [Messages](#messages)
* [Events](#events)
    * [Message Events](#message-events)
    * [Keeper Events](#keeper-events)
* [Parameters](#parameters)
    * [SendEnabled](#sendenabled)
    * [DefaultSendEnabled](#defaultsendenabled)
* [Client](#client)
    * [CLI](#cli)
    * [Query](#query)
    * [Transactions](#transactions)
* [gRPC](#grpc)

## Supply

Chức năng `supply`:

* theo dõi thụ động tổng nguồn cung coin trong một chain,
* cung cấp mẫu cho các module để giữ/tương tác với `Coins`, và
* giới thiệu kiểm tra invariant để xác minh tổng nguồn cung của chain.

### Total Supply

Tổng `Supply` của mạng bằng tổng tất cả coin từ tài khoản. Tổng nguồn cung được cập nhật mỗi khi một `Coin` được mint (ví dụ: như một phần của cơ chế lạm phát) hoặc burn (ví dụ: do slashing hoặc nếu một đề xuất governance bị phủ quyết).

## Module Accounts

Chức năng supply giới thiệu một loại tài khoản `auth.Account` mới mà các module có thể sử dụng để phân bổ token và trong các trường hợp đặc biệt mint hoặc burn token. Ở cấp độ cơ bản, các module account này có khả năng gửi/nhận token đến và từ các `auth.Account` và module account khác. Thiết kế này thay thế các thiết kế thay thế trước đây, nơi để giữ token, các module sẽ burn token đến từ tài khoản người gửi, sau đó theo dõi các token đó nội bộ. Sau đó, để gửi token, module sẽ cần mint token hiệu quả trong tài khoản đích. Thiết kế mới loại bỏ logic trùng lặp giữa các module để thực hiện kế toán này.

Interface `ModuleAccount` được định nghĩa như sau:

```go
type ModuleAccount interface {
  auth.Account               // same methods as the Account interface

  GetName() string           // name of the module; used to obtain the address
  GetPermissions() []string  // permissions of module account
  HasPermission(string) bool
}
```

> **CẢNH BÁO!**
> Bất kỳ module hoặc message handler nào cho phép gửi trực tiếp hoặc gián tiếp quỹ phải đảm bảo rõ ràng rằng các quỹ đó không thể được gửi đến module accounts (trừ khi được phép).

Supply `Keeper` cũng giới thiệu các hàm wrapper mới cho auth `Keeper` và bank `Keeper` liên quan đến `ModuleAccount`s để có thể:

* Lấy và đặt `ModuleAccount`s bằng cách cung cấp `Name`.
* Gửi coin từ và đến các `ModuleAccount` khác hoặc `Account` chuẩn (`BaseAccount` hoặc `VestingAccount`) bằng cách chỉ truyền `Name`.
* `Mint` hoặc `Burn` coin cho một `ModuleAccount` (bị giới hạn theo quyền của nó).

### Permissions

Mỗi `ModuleAccount` có một tập quyền khác nhau cung cấp các object capability khác nhau để thực hiện các hành động nhất định. Các quyền cần được đăng ký khi tạo supply `Keeper` để mỗi khi `ModuleAccount` gọi các hàm được phép, `Keeper` có thể tra cứu quyền cho tài khoản cụ thể đó và thực hiện hoặc không thực hiện hành động.

Các quyền có sẵn:

* `Minter`: cho phép module mint một lượng coin cụ thể.
* `Burner`: cho phép module burn một lượng coin cụ thể.
* `Staking`: cho phép module delegate và undelegate một lượng coin cụ thể.

## State

Module `x/bank` lưu trữ state của các đối tượng chính sau:

1. Số dư tài khoản
2. Metadata denomination
3. Tổng nguồn cung của tất cả số dư
4. Thông tin về các denomination nào được phép gửi.

Ngoài ra, module `x/bank` duy trì các index sau để quản lý state nói trên:

* Supply Index: `0x0 | byte(denom) -> byte(amount)`
* Denom Metadata Index: `0x1 | byte(denom) -> ProtocolBuffer(Metadata)`
* Balances Index: `0x2 | byte(address length) | []byte(address) | []byte(balance.Denom) -> ProtocolBuffer(balance)`
* Reverse Denomination to Address Index: `0x03 | byte(denom) | 0x00 | []byte(address) -> 0`

## Params

Module bank lưu params trong state với prefix `0x05`, có thể được cập nhật thông qua governance hoặc địa chỉ có quyền.

* Params: `0x05 | ProtocolBuffer(Params)`

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/bank/v1beta1/bank.proto#L12-L23
```

## Keepers

Module bank cung cấp các keeper interface đã export sau có thể được truyền cho các module khác đọc hoặc cập nhật số dư tài khoản. Các module nên sử dụng interface có quyền ít nhất cung cấp chức năng cần thiết.

Các best practice đòi hỏi xem xét kỹ mã module `bank` để đảm bảo rằng quyền được giới hạn theo cách bạn mong đợi.

### Denied Addresses

Module `x/bank` chấp nhận một map các địa chỉ được coi là blocklisted khỏi việc nhận quỹ trực tiếp và rõ ràng thông qua các phương thức như `MsgSend` và `MsgMultiSend` và các API call trực tiếp như `SendCoinsFromModuleToAccount`.

Thông thường, các địa chỉ này là module accounts. Nếu các địa chỉ này nhận quỹ ngoài các quy tắc đã định của state machine, các invariant có thể bị vi phạm và có thể dẫn đến mạng bị dừng.

Bằng cách cung cấp cho module `x/bank` một tập địa chỉ blocklisted, một lỗi sẽ xảy ra cho thao tác nếu người dùng hoặc client cố gắng gửi quỹ trực tiếp hoặc gián tiếp đến tài khoản blocklisted, ví dụ bằng cách sử dụng [IBC](https://ibc.cosmos.network).

### Common Types

#### Input

Đầu vào của chuyển đa bên

```protobuf
// Input models transaction input.
message Input {
  string   address                        = 1;
  repeated cosmos.base.v1beta1.Coin coins = 2;
}
```

#### Output

Đầu ra của chuyển đa bên.

```protobuf
// Output models transaction outputs.
message Output {
  string   address                        = 1;
  repeated cosmos.base.v1beta1.Coin coins = 2;
}
```

### BaseKeeper

Base keeper cung cấp quyền truy cập đầy đủ: khả năng sửa đổi tùy ý số dư bất kỳ tài khoản nào và mint hoặc burn coin.

Quyền mint bị giới hạn theo từng module có thể đạt được bằng cách sử dụng baseKeeper với `WithMintCoinsRestriction` để đặt các giới hạn cụ thể cho mint (ví dụ: chỉ mint một số denom nhất định).

```go
// Keeper defines a module interface that facilitates the transfer of coins
// between accounts.
type Keeper interface {
    SendKeeper
    WithMintCoinsRestriction(MintingRestrictionFn) BaseKeeper

    InitGenesis(context.Context, *types.GenesisState)
    ExportGenesis(context.Context) *types.GenesisState

    GetSupply(ctx context.Context, denom string) sdk.Coin
    HasSupply(ctx context.Context, denom string) bool
    GetPaginatedTotalSupply(ctx context.Context, pagination *query.PageRequest) (sdk.Coins, *query.PageResponse, error)
    IterateTotalSupply(ctx context.Context, cb func(sdk.Coin) bool)
    GetDenomMetaData(ctx context.Context, denom string) (types.Metadata, bool)
    HasDenomMetaData(ctx context.Context, denom string) bool
    SetDenomMetaData(ctx context.Context, denomMetaData types.Metadata)
    IterateAllDenomMetaData(ctx context.Context, cb func(types.Metadata) bool)

    SendCoinsFromModuleToAccount(ctx context.Context, senderModule string, recipientAddr sdk.AccAddress, amt sdk.Coins) error
    SendCoinsFromModuleToModule(ctx context.Context, senderModule, recipientModule string, amt sdk.Coins) error
    SendCoinsFromAccountToModule(ctx context.Context, senderAddr sdk.AccAddress, recipientModule string, amt sdk.Coins) error
    DelegateCoinsFromAccountToModule(ctx context.Context, senderAddr sdk.AccAddress, recipientModule string, amt sdk.Coins) error
    UndelegateCoinsFromModuleToAccount(ctx context.Context, senderModule string, recipientAddr sdk.AccAddress, amt sdk.Coins) error
    MintCoins(ctx context.Context, moduleName string, amt sdk.Coins) error
    BurnCoins(ctx context.Context, moduleName string, amt sdk.Coins) error

    DelegateCoins(ctx context.Context, delegatorAddr, moduleAccAddr sdk.AccAddress, amt sdk.Coins) error
    UndelegateCoins(ctx context.Context, moduleAccAddr, delegatorAddr sdk.AccAddress, amt sdk.Coins) error

    // GetAuthority gets the address capable of executing governance proposal messages. Usually the gov module account.
    GetAuthority() string

    types.QueryServer
}
```

### SendKeeper

Send keeper cung cấp quyền truy cập số dư tài khoản và khả năng chuyển coin giữa các tài khoản. Send keeper không thay đổi tổng nguồn cung (mint hoặc burn coin).

```go
// SendKeeper defines a module interface that facilitates the transfer of coins
// between accounts without the possibility of creating coins.
type SendKeeper interface {
    ViewKeeper

    AppendSendRestriction(restriction SendRestrictionFn)
    PrependSendRestriction(restriction SendRestrictionFn)
    ClearSendRestriction()

    InputOutputCoins(ctx context.Context, input types.Input, outputs []types.Output) error
    SendCoins(ctx context.Context, fromAddr, toAddr sdk.AccAddress, amt sdk.Coins) error

    GetParams(ctx context.Context) types.Params
    SetParams(ctx context.Context, params types.Params) error

    IsSendEnabledDenom(ctx context.Context, denom string) bool
    SetSendEnabled(ctx context.Context, denom string, value bool)
    SetAllSendEnabled(ctx context.Context, sendEnableds []*types.SendEnabled)
    DeleteSendEnabled(ctx context.Context, denom string)
    IterateSendEnabledEntries(ctx context.Context, cb func(denom string, sendEnabled bool) (stop bool))
    GetAllSendEnabledEntries(ctx context.Context) []types.SendEnabled

    IsSendEnabledCoin(ctx context.Context, coin sdk.Coin) bool
    IsSendEnabledCoins(ctx context.Context, coins ...sdk.Coin) error

    BlockedAddr(addr sdk.AccAddress) bool
}
```

#### Send Restrictions

`SendKeeper` áp dụng `SendRestrictionFn` trước mỗi lần chuyển quỹ.

```golang
// A SendRestrictionFn can restrict sends and/or provide a new receiver address.
type SendRestrictionFn func(ctx context.Context, fromAddr, toAddr sdk.AccAddress, amt sdk.Coins) (newToAddr sdk.AccAddress, err error)
```

Sau khi `SendKeeper` (hoặc `BaseKeeper`) được tạo, các send restriction có thể được thêm vào bằng các hàm `AppendSendRestriction` hoặc `PrependSendRestriction`. Cả hai hàm đều kết hợp restriction được cung cấp với bất kỳ restriction đã cung cấp trước đó. `AppendSendRestriction` thêm restriction được cung cấp để chạy sau bất kỳ send restriction đã cung cấp trước đó. `PrependSendRestriction` thêm restriction để chạy trước bất kỳ send restriction đã cung cấp trước đó. Việc kết hợp sẽ short-circuit khi gặp lỗi. Tức là nếu cái đầu tiên trả về lỗi, cái thứ hai sẽ không được chạy.

Trong `SendCoins`, send restriction được áp dụng trước khi coin được loại bỏ khỏi địa chỉ from và thêm vào địa chỉ to. Trong `InputOutputCoins`, send restriction được áp dụng sau khi input coins được loại bỏ và một lần cho mỗi output trước khi quỹ được thêm vào.

Một hàm send restriction nên sử dụng một giá trị tùy chỉnh trong context để cho phép bỏ qua restriction cụ thể đó.

Send Restrictions không được đặt trên các chuyển `ModuleToAccount` hoặc `ModuleToModule`. Điều này được thực hiện do các module cần di chuyển quỹ đến tài khoản người dùng và các module account khác. Đây là quyết định thiết kế để cho phép linh hoạt hơn trong state machine. State machine phải có khả năng di chuyển quỹ giữa module accounts và tài khoản người dùng mà không có restriction.

Thứ hai, giới hạn này sẽ hạn chế việc sử dụng state machine ngay cả đối với chính nó. Người dùng sẽ không thể nhận phần thưởng, không thể di chuyển quỹ giữa các module accounts. Trong trường hợp người dùng gửi quỹ từ tài khoản người dùng đến community pool và sau đó một đề xuất governance được sử dụng để đưa các token đó vào tài khoản người dùng, điều này sẽ thuộc quyền quyết định của nhà phát triển app chain về những gì họ muốn làm ở đây. Chúng ta không thể đưa ra giả định mạnh ở đây. Thứ ba, vấn đề này có thể dẫn đến chain halt nếu một token bị vô hiệu hóa và token được di chuyển trong begin/endblock. Đây là lý do cuối cùng chúng ta thấy thay đổi hiện tại và gây hại nhiều hơn có lợi cho người dùng.

Ví dụ, trong package keeper của module, bạn định nghĩa hàm send restriction:

```golang
var _ banktypes.SendRestrictionFn = Keeper{}.SendRestrictionFn

func (k Keeper) SendRestrictionFn(ctx context.Context, fromAddr, toAddr sdk.AccAddress, amt sdk.Coins) (sdk.AccAddress, error) {
	// Bypass if the context says to.
	if mymodule.HasBypass(ctx) {
		return toAddr, nil
	}

	// Your custom send restriction logic goes here.
	return nil, errors.New("not implemented")
}
```

Bank keeper nên được cung cấp cho constructor của keeper để send restriction có thể được thêm vào:

```golang
func NewKeeper(cdc codec.BinaryCodec, storeKey storetypes.StoreKey, bankKeeper mymodule.BankKeeper) Keeper {
	rv := Keeper{/*...*/}
	bankKeeper.AppendSendRestriction(rv.SendRestrictionFn)
	return rv
}
```

Sau đó, trong package `mymodule`, định nghĩa các context helper:

```golang
const bypassKey = "bypass-mymodule-restriction"

// WithBypass returns a new context that will cause the mymodule bank send restriction to be skipped.
func WithBypass(ctx context.Context) context.Context {
	return sdk.UnwrapSDKContext(ctx).WithValue(bypassKey, true)
}

// WithoutBypass returns a new context that will cause the mymodule bank send restriction to not be skipped.
func WithoutBypass(ctx context.Context) context.Context {
	return sdk.UnwrapSDKContext(ctx).WithValue(bypassKey, false)
}

// HasBypass checks the context to see if the mymodule bank send restriction should be skipped.
func HasBypass(ctx context.Context) bool {
	bypassValue := ctx.Value(bypassKey)
	if bypassValue == nil {
		return false
	}
	bypass, isBool := bypassValue.(bool)
	return isBool && bypass
}
```

Bây giờ, bất cứ nơi nào bạn muốn sử dụng `SendCoins` hoặc `InputOutputCoins`, nhưng bạn không muốn send restriction của bạn được áp dụng:

```golang
func (k Keeper) DoThing(ctx context.Context, fromAddr, toAddr sdk.AccAddress, amt sdk.Coins) error {
	return k.bankKeeper.SendCoins(mymodule.WithBypass(ctx), fromAddr, toAddr, amt)
}
```

### ViewKeeper

View keeper cung cấp quyền truy cập chỉ đọc vào số dư tài khoản. View keeper không có chức năng thay đổi số dư. Tất cả tra cứu số dư đều là `O(1)`.

```go
// ViewKeeper defines a module interface that facilitates read only access to
// account balances.
type ViewKeeper interface {
    ValidateBalance(ctx context.Context, addr sdk.AccAddress) error
    HasBalance(ctx context.Context, addr sdk.AccAddress, amt sdk.Coin) bool

    GetAllBalances(ctx context.Context, addr sdk.AccAddress) sdk.Coins
    GetAccountsBalances(ctx context.Context) []types.Balance
    GetBalance(ctx context.Context, addr sdk.AccAddress, denom string) sdk.Coin
    LockedCoins(ctx context.Context, addr sdk.AccAddress) sdk.Coins
    SpendableCoins(ctx context.Context, addr sdk.AccAddress) sdk.Coins
    SpendableCoin(ctx context.Context, addr sdk.AccAddress, denom string) sdk.Coin

    IterateAccountBalances(ctx context.Context, addr sdk.AccAddress, cb func(coin sdk.Coin) (stop bool))
    IterateAllBalances(ctx context.Context, cb func(address sdk.AccAddress, coin sdk.Coin) (stop bool))
}
```

## Messages

### MsgSend

Gửi coin từ một địa chỉ đến địa chỉ khác.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/bank/v1beta1/tx.proto#L38-L53
```

Message sẽ thất bại trong các điều kiện sau:

* Chức năng gửi (send) của coin chưa được bật
* Địa chỉ `to` bị hạn chế

### MsgMultiSend

Gửi coin từ một người gửi đến một loạt địa chỉ khác nhau. Nếu bất kỳ địa chỉ nhận nào không tương ứng với tài khoản hiện có, một tài khoản mới sẽ được tạo.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/bank/v1beta1/tx.proto#L58-L69
```

Message sẽ thất bại trong các điều kiện sau:

* Bất kỳ coin nào chưa được bật chức năng gửi (send)
* Bất kỳ địa chỉ `to` nào bị hạn chế
* Bất kỳ coin nào bị khóa
* Input và output không tương ứng chính xác với nhau

### MsgUpdateParams

Params của module `bank` có thể được cập nhật thông qua `MsgUpdateParams`, có thể thực hiện bằng đề xuất governance. Người ký sẽ luôn là địa chỉ module account `gov`.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/bank/v1beta1/tx.proto#L74-L88
```

Xử lý message có thể thất bại nếu:

* người ký không phải là địa chỉ module account gov.

### MsgSetSendEnabled

Sử dụng với module x/gov để tạo/chỉnh sửa các mục SendEnabled.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/bank/v1beta1/tx.proto#L96-L117
```

Message sẽ thất bại trong các điều kiện sau:

* Authority không phải là địa chỉ bech32.
* Authority không phải là địa chỉ module x/gov.
* Có nhiều mục SendEnabled với cùng Denom.
* Một hoặc nhiều mục SendEnabled có Denom không hợp lệ.

## Events

Module bank phát ra các event sau:

### Message Events

#### MsgSend

| Type     | Attribute Key | Attribute Value    |
| -------- | ------------- | ------------------ |
| transfer | recipient     | {recipientAddress} |
| transfer | amount        | {amount}           |
| message  | module        | bank               |
| message  | action        | send               |
| message  | sender        | {senderAddress}    |

#### MsgMultiSend

| Type     | Attribute Key | Attribute Value    |
| -------- | ------------- | ------------------ |
| transfer | recipient     | {recipientAddress} |
| transfer | amount        | {amount}           |
| message  | module        | bank               |
| message  | action        | multisend          |
| message  | sender        | {senderAddress}    |

### Keeper Events

Ngoài message events, bank keeper sẽ tạo events khi các phương thức sau được gọi (hoặc bất kỳ phương thức nào cuối cùng gọi chúng)

#### MintCoins

```json
{
  "type": "coinbase",
  "attributes": [
    {
      "key": "minter",
      "value": "{{sdk.AccAddress of the module minting coins}}",
      "index": true
    },
    {
      "key": "amount",
      "value": "{{sdk.Coins being minted}}",
      "index": true
    }
  ]
}
```

```json
{
  "type": "coin_received",
  "attributes": [
    {
      "key": "receiver",
      "value": "{{sdk.AccAddress of the module minting coins}}",
      "index": true
    },
    {
      "key": "amount",
      "value": "{{sdk.Coins being received}}",
      "index": true
    }
  ]
}
```

#### BurnCoins

```json
{
  "type": "burn",
  "attributes": [
    {
      "key": "burner",
      "value": "{{sdk.AccAddress of the module burning coins}}",
      "index": true
    },
    {
      "key": "amount",
      "value": "{{sdk.Coins being burned}}",
      "index": true
    }
  ]
}
```

```json
{
  "type": "coin_spent",
  "attributes": [
    {
      "key": "spender",
      "value": "{{sdk.AccAddress of the module burning coins}}",
      "index": true
    },
    {
      "key": "amount",
      "value": "{{sdk.Coins being burned}}",
      "index": true
    }
  ]
}
```

#### addCoins

```json
{
  "type": "coin_received",
  "attributes": [
    {
      "key": "receiver",
      "value": "{{sdk.AccAddress of the address beneficiary of the coins}}",
      "index": true
    },
    {
      "key": "amount",
      "value": "{{sdk.Coins being received}}",
      "index": true
    }
  ]
}
```

#### subUnlockedCoins/DelegateCoins

```json
{
  "type": "coin_spent",
  "attributes": [
    {
      "key": "spender",
      "value": "{{sdk.AccAddress of the address which is spending coins}}",
      "index": true
    },
    {
      "key": "amount",
      "value": "{{sdk.Coins being spent}}",
      "index": true
    }
  ]
}
```

## Parameters

Module bank chứa các tham số sau

### SendEnabled

Tham số SendEnabled hiện đã deprecated và không được sử dụng. Nó được thay thế bằng các bản ghi state store.


### DefaultSendEnabled

Giá trị default send enabled kiểm soát khả năng gửi cho tất cả các denomination coin trừ khi được chỉ định cụ thể trong mảng tham số `SendEnabled`.

## Client

### CLI

Người dùng có thể truy vấn và tương tác với module `bank` bằng CLI.

#### Query

Các lệnh `query` cho phép người dùng truy vấn state `bank`.

```shell
simd query bank --help
```

##### balances

Lệnh `balances` cho phép người dùng truy vấn số dư tài khoản theo địa chỉ.

```shell
simd query bank balances [address] [flags]
```

Ví dụ:

```shell
simd query bank balances cosmos1..
```

Kết quả mẫu:

```yml
balances:
- amount: "1000000000"
  denom: stake
pagination:
  next_key: null
  total: "0"
```

##### denom-metadata

Lệnh `denom-metadata` cho phép người dùng truy vấn metadata cho các denomination coin. Người dùng có thể truy vấn metadata cho một denomination bằng cờ `--denom` hoặc tất cả denominations mà không có nó.

```shell
simd query bank denom-metadata [flags]
```

Ví dụ:

```shell
simd query bank denom-metadata --denom stake
```

Kết quả mẫu:

```yml
metadata:
  base: stake
  denom_units:
  - aliases:
    - STAKE
    denom: stake
  description: native staking token of simulation app
  display: stake
  name: SimApp Token
  symbol: STK
```

##### total

Lệnh `total` cho phép người dùng truy vấn tổng nguồn cung coin. Người dùng có thể truy vấn tổng nguồn cung cho một coin bằng cờ `--denom` hoặc tất cả coin mà không có nó.

```shell
simd query bank total [flags]
```

Ví dụ:

```shell
simd query bank total --denom stake
```

Kết quả mẫu:

```yml
amount: "10000000000"
denom: stake
```

##### send-enabled

Lệnh `send-enabled` cho phép người dùng truy vấn tất cả hoặc một số mục SendEnabled.

```shell
simd query bank send-enabled [denom1 ...] [flags]
```

Ví dụ:

```shell
simd query bank send-enabled
```

Kết quả mẫu:

```yml
send_enabled:
- denom: foocoin
  enabled: true
- denom: barcoin
pagination:
  next-key: null
  total: 2 
```

#### Transactions

Các lệnh `tx` cho phép người dùng tương tác với module `bank`.

```shell
simd tx bank --help
```

##### send

Lệnh `send` cho phép người dùng gửi quỹ từ tài khoản này đến tài khoản khác.

```shell
simd tx bank send [from_key_or_address] [to_address] [amount] [flags]
```

Ví dụ:

```shell
simd tx bank send cosmos1.. cosmos1.. 100stake
```

## gRPC

Người dùng có thể truy vấn module `bank` bằng các endpoint gRPC.

### Balance

Endpoint `Balance` cho phép người dùng truy vấn số dư tài khoản theo địa chỉ cho một denomination cụ thể.

```shell
cosmos.bank.v1beta1.Query/Balance
```

Ví dụ:

```shell
grpcurl -plaintext \
    -d '{"address":"cosmos1..","denom":"stake"}' \
    localhost:9090 \
    cosmos.bank.v1beta1.Query/Balance
```

Kết quả mẫu:

```json
{
  "balance": {
    "denom": "stake",
    "amount": "1000000000"
  }
}
```

### AllBalances

Endpoint `AllBalances` cho phép người dùng truy vấn số dư tài khoản theo địa chỉ cho tất cả các denomination.

```shell
cosmos.bank.v1beta1.Query/AllBalances
```

Ví dụ:

```shell
grpcurl -plaintext \
    -d '{"address":"cosmos1.."}' \
    localhost:9090 \
    cosmos.bank.v1beta1.Query/AllBalances
```

Kết quả mẫu:

```json
{
  "balances": [
    {
      "denom": "stake",
      "amount": "1000000000"
    }
  ],
  "pagination": {
    "total": "1"
  }
}
```

### DenomMetadata

Endpoint `DenomMetadata` cho phép người dùng truy vấn metadata cho một denomination coin.

```shell
cosmos.bank.v1beta1.Query/DenomMetadata
```

Ví dụ:

```shell
grpcurl -plaintext \
    -d '{"denom":"stake"}' \
    localhost:9090 \
    cosmos.bank.v1beta1.Query/DenomMetadata
```

Kết quả mẫu:

```json
{
  "metadata": {
    "description": "native staking token of simulation app",
    "denomUnits": [
      {
        "denom": "stake",
        "aliases": [
          "STAKE"
        ]
      }
    ],
    "base": "stake",
    "display": "stake",
    "name": "SimApp Token",
    "symbol": "STK"
  }
}
```

### DenomsMetadata

Endpoint `DenomsMetadata` cho phép người dùng truy vấn metadata cho tất cả các denomination coin.

```shell
cosmos.bank.v1beta1.Query/DenomsMetadata
```

Ví dụ:

```shell
grpcurl -plaintext \
    localhost:9090 \
    cosmos.bank.v1beta1.Query/DenomsMetadata
```

Kết quả mẫu:

```json
{
  "metadatas": [
    {
      "description": "native staking token of simulation app",
      "denomUnits": [
        {
          "denom": "stake",
          "aliases": [
            "STAKE"
          ]
        }
      ],
      "base": "stake",
      "display": "stake",
      "name": "SimApp Token",
      "symbol": "STK"
    }
  ],
  "pagination": {
    "total": "1"
  }
}
```

### DenomOwners

Endpoint `DenomOwners` cho phép người dùng truy vấn metadata cho một denomination coin.

```shell
cosmos.bank.v1beta1.Query/DenomOwners
```

Ví dụ:

```shell
grpcurl -plaintext \
    -d '{"denom":"stake"}' \
    localhost:9090 \
    cosmos.bank.v1beta1.Query/DenomOwners
```

Kết quả mẫu:

```json
{
  "denomOwners": [
    {
      "address": "cosmos1..",
      "balance": {
        "denom": "stake",
        "amount": "5000000000"
      }
    },
    {
      "address": "cosmos1..",
      "balance": {
        "denom": "stake",
        "amount": "5000000000"
      }
    },
  ],
  "pagination": {
    "total": "2"
  }
}
```

### TotalSupply

Endpoint `TotalSupply` cho phép người dùng truy vấn tổng nguồn cung của tất cả coin.

```shell
cosmos.bank.v1beta1.Query/TotalSupply
```

Ví dụ:

```shell
grpcurl -plaintext \
    localhost:9090 \
    cosmos.bank.v1beta1.Query/TotalSupply
```

Kết quả mẫu:

```json
{
  "supply": [
    {
      "denom": "stake",
      "amount": "10000000000"
    }
  ],
  "pagination": {
    "total": "1"
  }
}
```

### SupplyOf

Endpoint `SupplyOf` cho phép người dùng truy vấn tổng nguồn cung của một coin.

```shell
cosmos.bank.v1beta1.Query/SupplyOf
```

Ví dụ:

```shell
grpcurl -plaintext \
    -d '{"denom":"stake"}' \
    localhost:9090 \
    cosmos.bank.v1beta1.Query/SupplyOf
```

Kết quả mẫu:

```json
{
  "amount": {
    "denom": "stake",
    "amount": "10000000000"
  }
}
```

### Params

Endpoint `Params` cho phép người dùng truy vấn các tham số của module `bank`.

```shell
cosmos.bank.v1beta1.Query/Params
```

Ví dụ:

```shell
grpcurl -plaintext \
    localhost:9090 \
    cosmos.bank.v1beta1.Query/Params
```

Kết quả mẫu:

```json
{
  "params": {
    "defaultSendEnabled": true
  }
}
```

### SendEnabled

Các endpoint `SendEnabled` cho phép người dùng truy vấn các mục SendEnabled của module `bank`.

Bất kỳ denomination nào KHÔNG được trả về, sử dụng giá trị `Params.DefaultSendEnabled`.

```shell
cosmos.bank.v1beta1.Query/SendEnabled
```

Ví dụ:

```shell
grpcurl -plaintext \
    localhost:9090 \
    cosmos.bank.v1beta1.Query/SendEnabled
```

Kết quả mẫu:

```json
{
  "send_enabled": [
    {
      "denom": "foocoin",
      "enabled": true
    },
    {
      "denom": "barcoin"
    }
  ],
  "pagination": {
    "next-key": null,
    "total": 2
  }
}
```
