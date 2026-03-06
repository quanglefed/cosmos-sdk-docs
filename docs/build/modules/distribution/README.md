---
sidebar_position: 1
---

# `x/distribution`

## Tổng quan

Cơ chế phân phối _đơn giản_ này mô tả một cách thức chức năng để phân phối thụ động
phần thưởng giữa các validator và delegator. Lưu ý rằng cơ chế này không phân phối
quỹ chính xác như các cơ chế phân phối phần thưởng chủ động và do đó sẽ được nâng cấp
trong tương lai.

Cơ chế hoạt động như sau. Phần thưởng thu được được gom vào một pool toàn cục và
phân phối thụ động cho các validator và delegator. Mỗi validator có cơ hội thu
hoa hồng từ các delegator trên phần thưởng thu được thay mặt delegator. Phí được thu
trực tiếp vào pool phần thưởng toàn cục và pool phần thưởng proposer của validator.
Do bản chất của kế toán thụ động, bất cứ khi nào có thay đổi tham số ảnh hưởng đến
tỷ lệ phân phối phần thưởng thì việc rút phần thưởng cũng phải diễn ra.

* Bất cứ khi nào rút, người dùng phải rút tối đa số lượng họ được hưởng, không để
   lại gì trong pool.
* Bất cứ khi nào bonding, unbonding, hoặc re-delegating token cho một tài khoản hiện có,
   phải thực hiện rút toàn bộ phần thưởng (vì quy tắc kế toán lười thay đổi).
* Bất cứ khi nào validator chọn thay đổi hoa hồng trên phần thưởng, tất cả hoa hồng
   tích lũy phải được rút đồng thời.

Các tình huống trên được mô tả trong `hooks.md`.

Cơ chế phân phối được mô tả ở đây được sử dụng để phân phối lười biếng các phần thưởng
sau giữa các validator và delegator liên kết:

* phí đa token để phân phối xã hội
* cung cấp tài sản stake bị lạm phát
* hoa hồng validator trên tất cả phần thưởng kiếm được từ stake của delegator

Phí được gom trong một pool toàn cục. Các cơ chế sử dụng cho phép validator và delegator
rút phần thưởng độc lập và thụ động.

## Hạn chế

Một phần của các tính toán lười, mỗi delegator giữ một accumulation term cụ thể cho
mỗi validator được dùng để ước tính phần công bằng gần đúng của họ trong token được
giữ trong pool phí toàn cục mà họ được hưởng.

```text
entitlement = delegator-accumulation / all-delegators-accumulation
```

Trong trường hợp có dòng token phần thưởng đến liên tục và bằng nhau mỗi block, cơ chế
phân phối này sẽ tương đương với phân phối chủ động (phân phối riêng cho tất cả delegator
mỗi block). Tuy nhiên, điều này không thực tế nên sẽ có sai lệch so với phân phối chủ động
dựa trên biến động của token phần thưởng đến cũng như thời điểm rút phần thưởng của
các delegator khác.

Nếu bạn biết rằng phần thưởng sắp đến sẽ tăng đáng kể, bạn có động lực không rút cho đến
sau sự kiện này, làm tăng giá trị _accum_ hiện có của bạn. Xem [#2764](https://github.com/cosmos/cosmos-sdk/issues/2764)
để biết thêm chi tiết.

## Ảnh hưởng đến Staking

Thu hoa hồng trên cung cấp Atom trong khi cũng cho phép cung cấp Atom được auto-bonded
(phân phối trực tiếp vào stake bonded của validator) là có vấn đề trong BPoS. Về cơ bản,
hai cơ chế này loại trừ lẫn nhau. Nếu cả cơ chế hoa hồng và auto-bonding được áp dụng
đồng thời cho staking-token thì việc phân phối staking-token giữa bất kỳ validator và
delegator của nó sẽ thay đổi mỗi block. Điều này đòi hỏi phải tính toán cho mỗi bản ghi
delegation cho mỗi block - được coi là tốn kém về mặt tính toán.

Kết luận, chúng ta chỉ có thể có hoa hồng Atom và cung cấp atom unbonded hoặc cung cấp
atom bonded mà không có hoa hồng Atom, và chúng ta chọn triển khai phương án trước.
Các bên liên quan muốn rebond phần cung cấp của họ có thể thiết lập script để định kỳ
rút và rebond phần thưởng.

## Nội dung

* [Concepts](#concepts)
* [State](#state)
    * [FeePool](#feepool)
    * [Validator Distribution](#validator-distribution)
    * [Delegation Distribution](#delegation-distribution)
    * [Params](#params)
* [Begin Block](#begin-block)
* [Messages](#messages)
* [Hooks](#hooks)
* [Events](#events)
* [Parameters](#parameters)
* [Client](#client)
    * [CLI](#cli)
    * [gRPC](#grpc)

## Concepts

Trong các blockchain Proof of Stake (PoS), phần thưởng từ phí giao dịch được trả cho các validator. Module phân phối phí phân phối công bằng phần thưởng cho các delegator cấu thành của validator.

Phần thưởng được tính theo từng kỳ. Kỳ được cập nhật mỗi khi delegation của validator thay đổi, ví dụ khi validator nhận delegation mới.
Phần thưởng cho một validator có thể được tính bằng cách lấy tổng phần thưởng cho kỳ trước khi delegation bắt đầu, trừ đi tổng phần thưởng hiện tại.
Để tìm hiểu thêm, xem [F1 Fee Distribution paper](https://github.com/cosmos/cosmos-sdk/tree/main/docs/spec/fee_distribution/f1_fee_distr.pdf).

Hoa hồng cho validator được trả khi validator bị loại bỏ hoặc khi validator yêu cầu rút.
Hoa hồng được tính và tăng dần ở mỗi thao tác `BeginBlock` để cập nhật số lượng phí tích lũy.

Phần thưởng cho delegator được phân phối khi delegation thay đổi hoặc bị loại bỏ, hoặc khi có yêu cầu rút.
Trước khi phân phối phần thưởng, tất cả các slash đối với validator xảy ra trong delegation hiện tại được áp dụng.

### Reference Counting trong F1 Fee Distribution

Trong phân phối phí F1, phần thưởng delegator nhận được tính khi delegation của họ được rút. Tính toán này phải đọc các term của tổng phần thưởng chia cho tỷ lệ token từ kỳ mà họ kết thúc khi họ delegate, và kỳ cuối cùng được tạo cho việc rút.

Ngoài ra, vì slash thay đổi số lượng token mà delegation sẽ có (nhưng chúng ta tính điều này một cách lười biếng, chỉ khi delegator un-delegate), chúng ta phải tính phần thưởng trong các kỳ riêng biệt trước/sau bất kỳ slash nào xảy ra giữa khi delegator delegate và khi họ rút phần thưởng. Do đó slash, giống như delegation, tham chiếu đến kỳ được kết thúc bởi sự kiện slash.

Tất cả các bản ghi phần thưởng lịch sử được lưu trữ cho các kỳ không còn được tham chiếu bởi bất kỳ delegation hoặc slash nào có thể được xóa an toàn, vì chúng sẽ không bao giờ được đọc (delegation và slash tương lai sẽ luôn tham chiếu các kỳ tương lai). Điều này được triển khai bằng cách theo dõi `ReferenceCount` cùng với mỗi mục lưu trữ phần thưởng lịch sử. Mỗi khi một đối tượng mới (delegation hoặc slash) được tạo có thể cần tham chiếu bản ghi lịch sử, reference count được tăng. Mỗi khi một đối tượng trước đây cần tham chiếu bản ghi lịch sử bị xóa, reference count được giảm. Nếu reference count về 0, bản ghi lịch sử bị xóa.

### External Community Pool Keepers

External pool community keeper được định nghĩa như sau:

```go
// ExternalCommunityPoolKeeper is the interface that an external community pool module keeper must fulfill
// for x/distribution to properly accept it as a community pool fund destination.
type ExternalCommunityPoolKeeper interface {
	// GetCommunityPoolModule gets the module name that funds should be sent to for the community pool.
	// This is the address that x/distribution will send funds to for external management.
	GetCommunityPoolModule() string
	// FundCommunityPool allows an account to directly fund the community fund pool.
	FundCommunityPool(ctx sdk.Context, amount sdk.Coins, senderAddr sdk.AccAddress) error
	// DistributeFromCommunityPool distributes funds from the community pool module account to
	// a receiver address.
	DistributeFromCommunityPool(ctx sdk.Context, amount sdk.Coins, receiveAddr sdk.AccAddress) error
}
```

Mặc định, module distribution sẽ sử dụng triển khai community pool nội bộ. Một external community pool có thể được cung cấp cho module và quỹ sẽ được chuyển hướng đến nó thay vì triển khai nội bộ. External community pool tham chiếu được Cosmos SDK duy trì là [`x/protocolpool`](../protocolpool/README.md).

## State

### FeePool

Tất cả các tham số theo dõi toàn cục cho distribution được lưu trong `FeePool`. Phần thưởng được thu và thêm vào reward pool và phân phối cho validator/delegator từ đây.

Lưu ý rằng reward pool giữ decimal coins (`DecCoins`) để cho phép nhận phân số coin từ các thao tác như inflation.
Khi coin được phân phối từ pool, chúng được cắt bớt trở lại thành `sdk.Coins` không phải decimal.

* FeePool: `0x00 -> ProtocolBuffer(FeePool)`

```go
// coins with decimal
type DecCoins []DecCoin

type DecCoin struct {
    Amount math.LegacyDec
    Denom  string
}
```

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/distribution/v1beta1/distribution.proto#L116-L123
```

### Validator Distribution

Thông tin phân phối validator cho validator liên quan được cập nhật mỗi khi:

1. số lượng delegation cho validator được cập nhật,
2. bất kỳ delegator nào rút từ validator, hoặc
3. validator rút hoa hồng của mình.

* ValidatorDistInfo: `0x02 | ValOperatorAddrLen (1 byte) | ValOperatorAddr -> ProtocolBuffer(validatorDistribution)`

```go
type ValidatorDistInfo struct {
    OperatorAddress     sdk.AccAddress
    SelfBondRewards     sdkmath.DecCoins
    ValidatorCommission types.ValidatorAccumulatedCommission
}
```

### Delegation Distribution

Mỗi delegation distribution chỉ cần ghi lại chiều cao mà nó rút phí lần cuối. Vì delegation phải rút phí mỗi khi thuộc tính của nó thay đổi (tức là bonded tokens, v.v.) nên thuộc tính của nó sẽ không đổi và hệ số _accumulation_ của delegator có thể được tính thụ động chỉ cần biết chiều cao của lần rút cuối và thuộc tính hiện tại.

* DelegationDistInfo: `0x02 | DelegatorAddrLen (1 byte) | DelegatorAddr | ValOperatorAddrLen (1 byte) | ValOperatorAddr -> ProtocolBuffer(delegatorDist)`

```go
type DelegationDistInfo struct {
    WithdrawalHeight int64    // last time this delegation withdrew rewards
}
```

### Params

Module distribution lưu params trong state với prefix `0x09`, có thể cập nhật bằng governance hoặc địa chỉ có quyền.

* Params: `0x09 | ProtocolBuffer(Params)`

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/distribution/v1beta1/distribution.proto#L12-L42
```

## Begin Block

Ở mỗi `BeginBlock`, tất cả phí nhận được ở block trước được chuyển đến tài khoản `ModuleAccount` của distribution. Khi delegator hoặc validator rút phần thưởng, chúng được lấy ra từ `ModuleAccount`. Trong begin block, các khoản yêu cầu khác nhau về phí thu được được cập nhật như sau:

* Thuế cộng đồng dự trữ được thu.
* Phần còn lại được phân phối tỷ lệ theo voting power cho tất cả validator bonded

### The Distribution Scheme

Xem [params](#params) để biết mô tả tham số.

Gọi `fees` là tổng phí thu được trong block trước, bao gồm phần thưởng lạm phát cho stake. Tất cả phí được thu trong một tài khoản module cụ thể trong block. Trong `BeginBlock`, chúng được gửi đến `ModuleAccount` `"distribution"`. Không có việc gửi token nào khác xảy ra. Thay vào đó, phần thưởng mỗi tài khoản được hưởng được lưu trữ, và việc rút có thể được kích hoạt qua các message `FundCommunityPool`, `WithdrawValidatorCommission` và `WithdrawDelegatorReward`.

#### Reward to the Community Pool

Community pool nhận `community_tax * fees`, cộng với bất kỳ dust còn lại nào sau khi validator nhận phần thưởng luôn được làm tròn xuống giá trị nguyên gần nhất.

#### Using an External Community Pool

Bắt đầu từ Cosmos SDK v0.53.0, external community pool như `x/protocolpool` có thể được sử dụng thay cho community pool do `x/distribution` quản lý.


Vui lòng xem cảnh báo trong phần tiếp theo trước khi quyết định sử dụng external community pool.

```go
// ExternalCommunityPoolKeeper is the interface that an external community pool module keeper must fulfill
// for x/distribution to properly accept it as a community pool fund destination.
type ExternalCommunityPoolKeeper interface {
	// GetCommunityPoolModule gets the module name that funds should be sent to for the community pool.
	// This is the address that x/distribution will send funds to for external management.
	GetCommunityPoolModule() string
	// FundCommunityPool allows an account to directly fund the community fund pool.
	FundCommunityPool(ctx sdk.Context, amount sdk.Coins, senderAddr sdk.AccAddress) error
	// DistributeFromCommunityPool distributes funds from the community pool module account to
	// a receiver address.
	DistributeFromCommunityPool(ctx sdk.Context, amount sdk.Coins, receiveAddr sdk.AccAddress) error
}
```

```go
app.DistrKeeper = distrkeeper.NewKeeper(
    appCodec,
    runtime.NewKVStoreService(keys[distrtypes.StoreKey]),
    app.AccountKeeper,
    app.BankKeeper,
    app.StakingKeeper,
    authtypes.FeeCollectorName,
    authtypes.NewModuleAddress(govtypes.ModuleName).String(),
    distrkeeper.WithExternalCommunityPool(app.ProtocolPoolKeeper), // New option.
)
```

#### External Community Pool Usage Warning

Khi sử dụng external community pool với `x/distribution`, các handler sau sẽ trả về lỗi:

**QueryService**

* `CommunityPool`

**MsgService**

* `CommunityPoolSpend`
* `FundCommunityPool`

Nếu bạn có dịch vụ phụ thuộc vào chức năng này từ `x/distribution`, vui lòng cập nhật chúng để sử dụng tương đương `x/protocolpool`.

#### Reward To the Validators

Proposer không nhận phần thưởng thêm. Tất cả phí được phân phối cho tất cả validator bonded, bao gồm proposer, theo tỷ lệ consensus power của họ.

```text
powFrac = validator power / total bonded validator power
voteMul = 1 - community_tax
```

Tất cả validator nhận `fees * voteMul * powFrac`.

#### Rewards to Delegators

Phần thưởng của mỗi validator được phân phối cho delegator của nó. Validator cũng có self-delegation được xử lý như delegation thông thường trong tính toán distribution.

Validator đặt tỷ lệ hoa hồng. Tỷ lệ hoa hồng linh hoạt, nhưng mỗi validator đặt tỷ lệ tối đa và mức tăng hàng ngày tối đa. Các mức tối đa này không thể vượt quá và bảo vệ delegator khỏi việc tăng đột ngột tỷ lệ hoa hồng validator để ngăn validator chiếm hết phần thưởng.

Phần thưởng chưa thanh toán mà operator được hưởng được lưu trong `ValidatorAccumulatedCommission`, trong khi phần thưởng delegator được hưởng được lưu trong `ValidatorCurrentRewards`. [F1 fee distribution scheme](#concepts) được sử dụng để tính phần thưởng cho mỗi delegator khi họ rút hoặc cập nhật delegation, và do đó không được xử lý trong `BeginBlock`.

#### Example Distribution

Đối với ví dụ phân phối này, consensus engine cơ bản chọn block proposer theo tỷ lệ power của họ so với toàn bộ bonded power.

Tất cả validator đều thực hiện như nhau trong việc bao gồm pre-commit trong block đề xuất của họ. Sau đó giữ `(pre_commits included) / (total bonded validator power)` không đổi để phần thưởng block khấu hao cho validator là `( validator power / total bonded power) * (1 - community tax rate)` của tổng phần thưởng. Do đó, phần thưởng cho một delegator đơn lẻ là:

```text
(delegator proportion of the validator power / validator power) * (validator power / total bonded power)
  * (1 - community tax rate) * (1 - validator commission rate)
= (delegator proportion of the validator power / total bonded power) * (1 -
community tax rate) * (1 - validator commission rate)
```

## Messages

### MsgSetWithdrawAddress

Mặc định, địa chỉ rút là địa chỉ delegator. Để thay đổi địa chỉ rút, delegator phải gửi message `MsgSetWithdrawAddress`.
Thay đổi địa chỉ rút chỉ có thể nếu tham số `WithdrawAddrEnabled` được đặt là `true`.

Địa chỉ rút không thể là bất kỳ tài khoản module nào. Các tài khoản này bị chặn làm địa chỉ rút bằng cách được thêm vào mảng `blockedAddrs` của distribution keeper khi khởi tạo.

Response:

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/distribution/v1beta1/tx.proto#L49-L60
```

```go
func (k Keeper) SetWithdrawAddr(ctx context.Context, delegatorAddr sdk.AccAddress, withdrawAddr sdk.AccAddress) error
	if k.blockedAddrs[withdrawAddr.String()] {
		fail with "`{withdrawAddr}` is not allowed to receive external funds"
	}

	if !k.GetWithdrawAddrEnabled(ctx) {
		fail with `ErrSetWithdrawAddrDisabled`
	}

	k.SetDelegatorWithdrawAddr(ctx, delegatorAddr, withdrawAddr)
```

### MsgWithdrawDelegatorReward

Delegator có thể rút phần thưởng của mình.
Nội bộ trong module distribution, giao dịch này đồng thời loại bỏ delegation trước với phần thưởng liên kết, giống như nếu delegator đơn giản bắt đầu delegation mới có cùng giá trị.
Phần thưởng được gửi ngay lập tức từ `ModuleAccount` distribution đến địa chỉ rút.
Bất kỳ phần còn lại nào (decimal bị cắt) được gửi đến community pool.
Chiều cao bắt đầu của delegation được đặt thành validator period hiện tại, và reference count cho period trước được giảm.
Số lượng rút được trừ từ biến `ValidatorOutstandingRewards` cho validator.

Trong phân phối F1, tổng phần thưởng được tính theo validator period, và delegator nhận một phần phần thưởng đó theo tỷ lệ stake của họ trong validator.
Trong F1 cơ bản, tổng phần thưởng mà tất cả delegator được hưởng giữa hai period được tính theo cách sau.
Gọi `R(X)` là tổng phần thưởng tích lũy đến period `X` chia cho token staked tại thời điểm đó. Phân bổ delegator là `R(X) * delegator_stake`.
Sau đó phần thưởng cho tất cả delegator vì staking giữa period `A` và `B` là `(R(B) - R(A)) * total stake`.
Tuy nhiên, các phần thưởng tính toán này không tính đến slashing.

Tính đến slash cần lặp.
Gọi `F(X)` là phân số validator bị slash cho sự kiện slashing xảy ra tại period `X`.
Nếu validator bị slash tại period `P1, ..., PN`, trong đó `A < P1`, `PN < B`, module distribution tính phần thưởng delegator cá nhân, `T(A, B)`, như sau:

```go
stake := initial stake
rewards := 0
previous := A
for P in P1, ..., PN`:
    rewards = (R(P) - previous) * stake
    stake = stake * F(P)
    previous = P
rewards = rewards + (R(B) - R(PN)) * stake
```

Phần thưởng lịch sử được tính hồi tố bằng cách phát lại tất cả slash và sau đó làm giảm stake của delegator ở mỗi bước.
Stake tính toán cuối cùng tương đương với coin staked thực tế trong delegation với sai số do lỗi làm tròn.

Response:

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/distribution/v1beta1/tx.proto#L66-L77
```

### WithdrawValidatorCommission

Validator có thể gửi message WithdrawValidatorCommission để rút hoa hồng tích lũy của mình.
Hoa hồng được tính trong mỗi block trong `BeginBlock`, nên không cần lặp để rút.
Số lượng rút được trừ từ biến `ValidatorOutstandingRewards` cho validator.
Chỉ số lượng nguyên có thể được gửi. Nếu phần thưởng tích lũy có decimal, số lượng được cắt bớt trước khi gửi rút, và phần còn lại để rút sau.

### FundCommunityPool

:::warning

Handler này sẽ trả về lỗi nếu sử dụng `ExternalCommunityPool`.

:::

Message này gửi coin trực tiếp từ người gửi đến community pool.

Giao dịch thất bại nếu số lượng không thể chuyển từ người gửi đến tài khoản module distribution.

```go
func (k Keeper) FundCommunityPool(ctx context.Context, amount sdk.Coins, sender sdk.AccAddress) error {
  if err := k.bankKeeper.SendCoinsFromAccountToModule(ctx, sender, types.ModuleName, amount); err != nil {
    return err
  }

  feePool, err := k.FeePool.Get(ctx)
  if err != nil {
    return err
  }

  feePool.CommunityPool = feePool.CommunityPool.Add(sdk.NewDecCoinsFromCoins(amount...)...)
	
  if err := k.FeePool.Set(ctx, feePool); err != nil {
    return err
  }

  return nil
}
```

### Common distribution operations

Các thao tác này diễn ra trong nhiều message khác nhau.

#### Initialize delegation

Mỗi khi delegation thay đổi, phần thưởng được rút và delegation được khởi tạo lại.
Khởi tạo delegation tăng validator period và theo dõi period bắt đầu của delegation.

```go
// initialize starting info for a new delegation
func (k Keeper) initializeDelegation(ctx context.Context, val sdk.ValAddress, del sdk.AccAddress) {
    // period has already been incremented - we want to store the period ended by this delegation action
    previousPeriod := k.GetValidatorCurrentRewards(ctx, val).Period - 1

	// increment reference count for the period we're going to track
	k.incrementReferenceCount(ctx, val, previousPeriod)

	validator := k.stakingKeeper.Validator(ctx, val)
	delegation := k.stakingKeeper.Delegation(ctx, del, val)

	// calculate delegation stake in tokens
	// we don't store directly, so multiply delegation shares * (tokens per share)
	// note: necessary to truncate so we don't allow withdrawing more rewards than owed
	stake := validator.TokensFromSharesTruncated(delegation.GetShares())
	k.SetDelegatorStartingInfo(ctx, val, del, types.NewDelegatorStartingInfo(previousPeriod, stake, uint64(ctx.BlockHeight())))
}
```

### MsgUpdateParams

Params của module distribution có thể được cập nhật qua `MsgUpdateParams`, có thể thực hiện bằng governance proposal và người ký sẽ luôn là địa chỉ tài khoản module gov.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/distribution/v1beta1/tx.proto#L133-L147
```

Xử lý message có thể thất bại nếu:

* người ký không phải địa chỉ tài khoản module gov.

## Hooks

Các hook có sẵn có thể được gọi bởi và từ module này.

### Create or modify delegation distribution

* triggered-by: `staking.MsgDelegate`, `staking.MsgBeginRedelegate`, `staking.MsgUndelegate`

#### Before

* Phần thưởng delegation được rút đến địa chỉ rút của delegator.
  Phần thưởng bao gồm period hiện tại và loại trừ period bắt đầu.
* Validator period được tăng.
  Validator period được tăng vì power và phân phối share của validator có thể đã thay đổi.
* Reference count cho period bắt đầu của delegator được giảm.

#### After

Chiều cao bắt đầu của delegation được đặt thành period trước.
Do hook `Before`, period này là period cuối cùng mà delegator được thưởng.

### Validator created

* triggered-by: `staking.MsgCreateValidator`

Khi validator được tạo, các biến validator sau được khởi tạo:

* Historical rewards
* Current accumulated rewards
* Accumulated commission
* Total outstanding rewards
* Period

Mặc định, tất cả giá trị được đặt là `0`, ngoại trừ period được đặt là `1`.

### Validator removed

* triggered-by: `staking.RemoveValidator`

Hoa hồng chưa thanh toán được gửi đến địa chỉ rút self-delegation của validator.
Phần thưởng delegator còn lại được gửi đến community fee pool.

Lưu ý: Validator chỉ bị loại bỏ khi không còn delegation. Tại thời điểm đó, tất cả phần thưởng delegator chưa thanh toán sẽ đã được rút. Bất kỳ phần thưởng còn lại nào là số lượng dust.

### Validator is slashed

* triggered-by: `staking.Slash`
* Reference count của validator period hiện tại được tăng.
  Reference count được tăng vì sự kiện slash đã tạo tham chiếu đến nó.
* Validator period được tăng.
* Sự kiện slash được lưu để sử dụng sau.
  Sự kiện slash sẽ được tham chiếu khi tính phần thưởng delegator.

## Events

Module distribution phát ra các sự kiện sau:

### BeginBlocker

| Type            | Attribute Key | Attribute Value    |
|-----------------|---------------|--------------------|
| proposer_reward | validator     | {validatorAddress} |
| proposer_reward | reward        | {proposerReward}   |
| commission      | amount        | {commissionAmount} |
| commission      | validator     | {validatorAddress} |
| rewards         | amount        | {rewardAmount}     |
| rewards         | validator     | {validatorAddress} |

### Handlers

#### MsgSetWithdrawAddress

| Type                 | Attribute Key    | Attribute Value      |
|----------------------|------------------|----------------------|
| set_withdraw_address | withdraw_address | {withdrawAddress}    |
| message              | module           | distribution         |
| message              | action           | set_withdraw_address |
| message              | sender           | {senderAddress}      |

#### MsgWithdrawDelegatorReward

| Type    | Attribute Key | Attribute Value           |
|---------|---------------|---------------------------|
| withdraw_rewards | amount        | {rewardAmount}            |
| withdraw_rewards | validator     | {validatorAddress}        |
| message          | module        | distribution              |
| message          | action        | withdraw_delegator_reward |
| message          | sender        | {senderAddress}           |

#### MsgWithdrawValidatorCommission

| Type       | Attribute Key | Attribute Value               |
|------------|---------------|-------------------------------|
| withdraw_commission | amount        | {commissionAmount}            |
| message    | module        | distribution                  |
| message    | action        | withdraw_validator_commission |
| message    | sender        | {senderAddress}               |

## Parameters

Module distribution chứa các tham số sau:

| Key                 | Type         | Example                    |
| ------------------- | ------------ | -------------------------- |
| communitytax        | string (dec) | "0.020000000000000000" [0] |
| withdrawaddrenabled | bool         | true                       |

* [0] `communitytax` phải dương và không thể vượt quá 1.00.
* `baseproposerreward` và `bonusproposerreward` là các tham số đã bị deprecated trong v0.47 và không được sử dụng.

:::note
Reserve pool là pool quỹ thu được để governance sử dụng thông qua `CommunityTax`.
Hiện tại với Cosmos SDK, token thu bởi CommunityTax được ghi nhận nhưng không thể chi tiêu.
:::

## Client

## CLI

Người dùng có thể truy vấn và tương tác với module `distribution` bằng CLI.

#### Query

Các lệnh `query` cho phép người dùng truy vấn state `distribution`.

```shell
simd query distribution --help
```

##### commission

Lệnh `commission` cho phép người dùng truy vấn phần thưởng hoa hồng validator theo địa chỉ.

```shell
simd query distribution commission [address] [flags]
```

Ví dụ:

```shell
simd query distribution commission cosmosvaloper1...
```

Ví dụ đầu ra:

```yml
commission:
- amount: "1000000.000000000000000000"
  denom: stake
```

##### community-pool

Lệnh `community-pool` cho phép người dùng truy vấn tất cả số dư coin trong community pool.

```shell
simd query distribution community-pool [flags]
```

Ví dụ:

```shell
simd query distribution community-pool
```

Ví dụ đầu ra:

```yml
pool:
- amount: "1000000.000000000000000000"
  denom: stake
```

##### params

Lệnh `params` cho phép người dùng truy vấn tham số của module `distribution`.

```shell
simd query distribution params [flags]
```

Ví dụ:

```shell
simd query distribution params
```

Ví dụ đầu ra:

```yml
base_proposer_reward: "0.000000000000000000"
bonus_proposer_reward: "0.000000000000000000"
community_tax: "0.020000000000000000"
withdraw_addr_enabled: true
```

##### rewards

Lệnh `rewards` cho phép người dùng truy vấn phần thưởng delegator. Người dùng có thể tùy chọn bao gồm địa chỉ validator để truy vấn phần thưởng kiếm được từ validator cụ thể.

```shell
simd query distribution rewards [delegator-addr] [validator-addr] [flags]
```

Ví dụ:

```shell
simd query distribution rewards cosmos1...
```

Ví dụ đầu ra:

```yml
rewards:
- reward:
  - amount: "1000000.000000000000000000"
    denom: stake
  validator_address: cosmosvaloper1..
total:
- amount: "1000000.000000000000000000"
  denom: stake
```

##### slashes

Lệnh `slashes` cho phép người dùng truy vấn tất cả slash cho một khoảng block cho trước.

```shell
simd query distribution slashes [validator] [start-height] [end-height] [flags]
```

Ví dụ:

```shell
simd query distribution slashes cosmosvaloper1... 1 1000
```

Ví dụ đầu ra:

```yml
pagination:
  next_key: null
  total: "0"
slashes:
- validator_period: 20,
  fraction: "0.009999999999999999"
```

##### validator-outstanding-rewards

Lệnh `validator-outstanding-rewards` cho phép người dùng truy vấn tất cả phần thưởng chưa thanh toán (chưa rút) cho validator và tất cả delegation của họ.

```shell
simd query distribution validator-outstanding-rewards [validator] [flags]
```

Ví dụ:

```shell
simd query distribution validator-outstanding-rewards cosmosvaloper1...
```

Ví dụ đầu ra:

```yml
rewards:
- amount: "1000000.000000000000000000"
  denom: stake
```

##### validator-distribution-info

Lệnh `validator-distribution-info` cho phép người dùng truy vấn hoa hồng validator và phần thưởng self-delegation cho validator.

```shell
simd query distribution validator-distribution-info cosmosvaloper1...
```

Ví dụ đầu ra:

```yml
commission:
- amount: "100000.000000000000000000"
  denom: stake
operator_address: cosmosvaloper1...
self_bond_rewards:
- amount: "100000.000000000000000000"
  denom: stake
```

#### Transactions

Các lệnh `tx` cho phép người dùng tương tác với module `distribution`.

```shell
simd tx distribution --help
```

##### fund-community-pool

Lệnh `fund-community-pool` cho phép người dùng gửi quỹ đến community pool.

```shell
simd tx distribution fund-community-pool [amount] [flags]
```

Ví dụ:

```shell
simd tx distribution fund-community-pool 100stake --from cosmos1...
```

##### set-withdraw-addr

Lệnh `set-withdraw-addr` cho phép người dùng đặt địa chỉ rút cho phần thưởng liên kết với địa chỉ delegator.

```shell
simd tx distribution set-withdraw-addr [withdraw-addr] [flags]
```

Ví dụ:

```shell
simd tx distribution set-withdraw-addr cosmos1... --from cosmos1...
```

##### withdraw-all-rewards

Lệnh `withdraw-all-rewards` cho phép người dùng rút tất cả phần thưởng cho delegator.

```shell
simd tx distribution withdraw-all-rewards [flags]
```

Ví dụ:

```shell
simd tx distribution withdraw-all-rewards --from cosmos1...
```

##### withdraw-rewards

Lệnh `withdraw-rewards` cho phép người dùng rút tất cả phần thưởng từ địa chỉ delegation cho trước, và tùy chọn rút hoa hồng validator nếu địa chỉ delegation cho trước là validator operator và người dùng chứng minh cờ `--commission`.

```shell
simd tx distribution withdraw-rewards [validator-addr] [flags]
```

Ví dụ:

```shell
simd tx distribution withdraw-rewards cosmosvaloper1... --from cosmos1... --commission
```

### gRPC

Người dùng có thể truy vấn module `distribution` bằng các endpoint gRPC.

#### Params

Endpoint `Params` cho phép người dùng truy vấn tham số của module `distribution`.

Ví dụ:

```shell
grpcurl -plaintext \
    localhost:9090 \
    cosmos.distribution.v1beta1.Query/Params
```

Ví dụ đầu ra:

```json
{
  "params": {
    "communityTax": "20000000000000000",
    "baseProposerReward": "00000000000000000",
    "bonusProposerReward": "00000000000000000",
    "withdrawAddrEnabled": true
  }
}
```

#### ValidatorDistributionInfo

ValidatorDistributionInfo truy vấn hoa hồng validator và phần thưởng self-delegation cho validator.

Ví dụ:

```shell
grpcurl -plaintext \
    -d '{"validator_address":"cosmosvalop1..."}' \
    localhost:9090 \
    cosmos.distribution.v1beta1.Query/ValidatorDistributionInfo
```

Ví dụ đầu ra:

```json
{
  "commission": {
    "commission": [
      {
        "denom": "stake",
        "amount": "1000000000000000"
      }
    ]
  },
  "self_bond_rewards": [
    {
      "denom": "stake",
      "amount": "1000000000000000"
    }
  ],
  "validator_address": "cosmosvalop1..."
}
```

#### ValidatorOutstandingRewards

Endpoint `ValidatorOutstandingRewards` cho phép người dùng truy vấn phần thưởng của địa chỉ validator.

Ví dụ:

```shell
grpcurl -plaintext \
    -d '{"validator_address":"cosmosvalop1.."}' \
    localhost:9090 \
    cosmos.distribution.v1beta1.Query/ValidatorOutstandingRewards
```

Ví dụ đầu ra:

```json
{
  "rewards": {
    "rewards": [
      {
        "denom": "stake",
        "amount": "1000000000000000"
      }
    ]
  }
}
```

#### ValidatorCommission

Endpoint `ValidatorCommission` cho phép người dùng truy vấn hoa hồng tích lũy cho validator.

Ví dụ:

```shell
grpcurl -plaintext \
    -d '{"validator_address":"cosmosvalop1.."}' \
    localhost:9090 \
    cosmos.distribution.v1beta1.Query/ValidatorCommission
```

Ví dụ đầu ra:

```json
{
  "commission": {
    "commission": [
      {
        "denom": "stake",
        "amount": "1000000000000000"
      }
    ]
  }
}
```

#### ValidatorSlashes

Endpoint `ValidatorSlashes` cho phép người dùng truy vấn sự kiện slash của validator.

Ví dụ:

```shell
grpcurl -plaintext \
    -d '{"validator_address":"cosmosvalop1.."}' \
    localhost:9090 \
    cosmos.distribution.v1beta1.Query/ValidatorSlashes
```

Ví dụ đầu ra:

```json
{
  "slashes": [
    {
      "validator_period": "20",
      "fraction": "0.009999999999999999"
    }
  ],
  "pagination": {
    "total": "1"
  }
}
```

#### DelegationRewards

Endpoint `DelegationRewards` cho phép người dùng truy vấn tổng phần thưởng tích lũy bởi delegation.

Ví dụ:

```shell
grpcurl -plaintext \
    -d '{"delegator_address":"cosmos1...","validator_address":"cosmosvalop1..."}' \
    localhost:9090 \
    cosmos.distribution.v1beta1.Query/DelegationRewards
```

Ví dụ đầu ra:

```json
{
  "rewards": [
    {
      "denom": "stake",
      "amount": "1000000000000000"
    }
  ]
}
```

#### DelegationTotalRewards

Endpoint `DelegationTotalRewards` cho phép người dùng truy vấn tổng phần thưởng tích lũy bởi mỗi validator.

Ví dụ:

```shell
grpcurl -plaintext \
    -d '{"delegator_address":"cosmos1..."}' \
    localhost:9090 \
    cosmos.distribution.v1beta1.Query/DelegationTotalRewards
```

Ví dụ đầu ra:

```json
{
  "rewards": [
    {
      "validatorAddress": "cosmosvaloper1...",
      "reward": [
        {
          "denom": "stake",
          "amount": "1000000000000000"
        }
      ]
    }
  ],
  "total": [
    {
      "denom": "stake",
      "amount": "1000000000000000"
    }
  ]
}
```

#### DelegatorValidators

Endpoint `DelegatorValidators` cho phép người dùng truy vấn tất cả validator cho delegator cho trước.

Ví dụ:

```shell
grpcurl -plaintext \
    -d '{"delegator_address":"cosmos1..."}' \
    localhost:9090 \
    cosmos.distribution.v1beta1.Query/DelegatorValidators
```

Ví dụ đầu ra:

```json
{
  "validators": ["cosmosvaloper1..."]
}
```

#### DelegatorWithdrawAddress

Endpoint `DelegatorWithdrawAddress` cho phép người dùng truy vấn địa chỉ rút của delegator.

Ví dụ:

```shell
grpcurl -plaintext \
    -d '{"delegator_address":"cosmos1..."}' \
    localhost:9090 \
    cosmos.distribution.v1beta1.Query/DelegatorWithdrawAddress
```

Ví dụ đầu ra:

```json
{
  "withdrawAddress": "cosmos1..."
}
```

#### CommunityPool

Endpoint `CommunityPool` cho phép người dùng truy vấn coin của community pool.

Ví dụ:

```shell
grpcurl -plaintext \
    localhost:9090 \
    cosmos.distribution.v1beta1.Query/CommunityPool
```

Ví dụ đầu ra:

```json
{
  "pool": [
    {
      "denom": "stake",
      "amount": "1000000000000000000"
    }
  ]
}
```
