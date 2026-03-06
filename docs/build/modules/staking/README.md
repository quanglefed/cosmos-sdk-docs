---
sidebar_position: 1
---

# `x/staking`

## Tóm tắt

Tài liệu này mô tả chi tiết module Staking của Cosmos SDK, lần đầu được giới thiệu trong [Cosmos Whitepaper](https://cosmos.network/about/whitepaper) vào tháng 6 năm 2016.

Module cho phép blockchain dựa trên Cosmos SDK hỗ trợ hệ thống Proof-of-Stake (PoS) tiên tiến. Trong hệ thống này, người nắm giữ token staking gốc của chain có thể trở thành validator và có thể delegate token cho validator, qua đó xác định tập hợp validator hiệu quả cho hệ thống.

Module này được sử dụng trong Cosmos Hub, Hub đầu tiên trong mạng Cosmos.

## Nội dung

* [State](#state)
    * [Pool](#pool)
    * [LastTotalPower](#lasttotalpower)
    * [ValidatorUpdates](#validatorupdates)
    * [UnbondingID](#unbondingid)
    * [Params](#params)
    * [Validator](#validator)
    * [Delegation](#delegation)
    * [UnbondingDelegation](#unbondingdelegation)
    * [Redelegation](#redelegation)
    * [Queues](#queues)
    * [HistoricalInfo](#historicalinfo)
* [State Transitions](#state-transitions)
    * [Validators](#validators)
    * [Delegations](#delegations)
    * [Slashing](#slashing)
    * [How Shares are calculated](#how-shares-are-calculated)
* [Messages](#messages)
    * [MsgCreateValidator](#msgcreatevalidator)
    * [MsgEditValidator](#msgeditvalidator)
    * [MsgDelegate](#msgdelegate)
    * [MsgUndelegate](#msgundelegate)
    * [MsgCancelUnbondingDelegation](#msgcancelunbondingdelegation)
    * [MsgBeginRedelegate](#msgbeginredelegate)
    * [MsgUpdateParams](#msgupdateparams)
* [Begin-Block](#begin-block)
    * [Historical Info Tracking](#historical-info-tracking)
* [End-Block](#end-block)
    * [Validator Set Changes](#validator-set-changes)
    * [Queues](#queues-1)
* [Hooks](#hooks)
* [Events](#events)
    * [EndBlocker](#endblocker)
    * [Msg's](#msgs)
* [Parameters](#parameters)
* [Client](#client)
    * [CLI](#cli)
    * [gRPC](#grpc)
    * [REST](#rest)

## State

### Pool

Pool được sử dụng để theo dõi nguồn cung token bonded và not-bonded của bond denomination.

### LastTotalPower

LastTotalPower theo dõi tổng lượng token bonded được ghi nhận trong end block trước đó. Các mục store có tiền tố "Last" phải giữ nguyên cho đến EndBlock.

* LastTotalPower: `0x12 -> ProtocolBuffer(math.Int)`

### ValidatorUpdates

ValidatorUpdates chứa các cập nhật validator được trả về cho ABCI vào cuối mỗi block. Các giá trị được ghi đè trong mỗi block.

* ValidatorUpdates `0x61 -> []abci.ValidatorUpdate`

### UnbondingID

UnbondingID lưu trữ ID của thao tác unbonding mới nhất. Nó cho phép tạo ID duy nhất cho các thao tác unbonding, tức là UnbondingID được tăng mỗi khi một thao tác unbonding mới (validator unbonding, unbonding delegation, redelegation) được khởi tạo.

* UnbondingID: `0x37 -> uint64`

### Params

Module staking lưu trữ params của nó trong state với tiền tố `0x51`, có thể được cập nhật thông qua governance hoặc địa chỉ có quyền.

* Params: `0x51 | ProtocolBuffer(Params)`

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/staking/v1beta1/staking.proto#L310-L333
```

### Validator

Validator có thể có một trong ba trạng thái:

* `Unbonded`: Validator không nằm trong active set. Họ không thể ký block và không nhận rewards. Họ có thể nhận delegations.
* `Bonded`: Khi validator nhận đủ token bonded, họ tự động tham gia active set trong [`EndBlock`](#validator-set-changes) và trạng thái của họ được cập nhật thành `Bonded`. Họ đang ký block và nhận rewards. Họ có thể nhận thêm delegations. Họ có thể bị slashed do hành vi sai trái. Delegator của validator này khi unbond delegation của họ phải chờ trong thời gian UnbondingTime, một tham số đặc thù chain, trong thời gian đó họ vẫn có thể bị slash cho các vi phạm của validator nguồn nếu các vi phạm đó xảy ra trong khoảng thời gian token được bonded.
* `Unbonding`: Khi validator rời khỏi active set, dù do lựa chọn hay do slashing, jailing hoặc tombstoning, quá trình unbonding của tất cả delegations của họ bắt đầu. Tất cả delegations phải chờ UnbondingTime trước khi token được chuyển vào tài khoản của họ từ `BondedPool`.

:::warning
Tombstoning là vĩnh viễn, một khi bị tombstoned thì consensus key của validator không thể tái sử dụng trong chain nơi tombstoning xảy ra.
:::

Các đối tượng Validator nên được lưu trữ và truy cập chủ yếu qua `OperatorAddr`, địa chỉ validator SDK cho operator của validator. Hai chỉ mục bổ sung được duy trì cho mỗi đối tượng validator để thực hiện các tra cứu cần thiết cho slashing và cập nhật validator-set. Một chỉ mục đặc biệt thứ ba (`LastValidatorPower`) cũng được duy trì, tuy nhiên giữ nguyên trong suốt mỗi block, không giống hai chỉ mục đầu phản ánh các bản ghi validator trong block.

* Validators: `0x21 | OperatorAddrLen (1 byte) | OperatorAddr -> ProtocolBuffer(validator)`
* ValidatorsByConsAddr: `0x22 | ConsAddrLen (1 byte) | ConsAddr -> OperatorAddr`
* ValidatorsByPower: `0x23 | BigEndian(ConsensusPower) | OperatorAddrLen (1 byte) | OperatorAddr -> OperatorAddr`
* LastValidatorsPower: `0x11 | OperatorAddrLen (1 byte) | OperatorAddr -> ProtocolBuffer(ConsensusPower)`
* ValidatorsByUnbondingID: `0x38 | UnbondingID ->  0x21 | OperatorAddrLen (1 byte) | OperatorAddr`

`Validators` là chỉ mục chính - nó đảm bảo mỗi operator chỉ có một validator liên kết, trong khi public key của validator đó có thể thay đổi trong tương lai. Delegator có thể tham chiếu đến operator bất biến của validator mà không lo lắng về public key thay đổi.

`ValidatorsByUnbondingID` là chỉ mục bổ sung cho phép tra cứu validator theo unbonding ID tương ứng với unbonding hiện tại của họ.

`ValidatorByConsAddr` là chỉ mục bổ sung cho phép tra cứu cho slashing. Khi CometBFT báo cáo evidence, nó cung cấp địa chỉ validator, do đó cần map này để tìm operator. Lưu ý rằng `ConsAddr` tương ứng với địa chỉ có thể được suy ra từ `ConsPubKey` của validator.

`ValidatorsByPower` là chỉ mục bổ sung cung cấp danh sách validator tiềm năng được sắp xếp để nhanh chóng xác định active set hiện tại. Ở đây ConsensusPower mặc định là validator.Tokens/10^6. Lưu ý rằng tất cả validator có `Jailed` là true không được lưu trong chỉ mục này.

`LastValidatorsPower` là chỉ mục đặc biệt cung cấp danh sách lịch sử các validator bonded của block trước. Chỉ mục này giữ nguyên trong suốt block nhưng được cập nhật trong quá trình cập nhật validator set diễn ra trong [`EndBlock`](#end-block).

Trạng thái của mỗi validator được lưu trong struct `Validator`:

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/staking/v1beta1/staking.proto#L82-L138
```

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/staking/v1beta1/staking.proto#L26-L80
```

### Delegation

Delegations được xác định bằng cách kết hợp `DelegatorAddr` (địa chỉ của delegator) với `ValidatorAddr`. Delegators được lập chỉ mục trong store như sau:

* Delegation: `0x31 | DelegatorAddrLen (1 byte) | DelegatorAddr | ValidatorAddrLen (1 byte) | ValidatorAddr -> ProtocolBuffer(delegation)`

Người nắm giữ stake có thể delegate coin cho validator; trong trường hợp này quỹ của họ được giữ trong cấu trúc dữ liệu `Delegation`. Nó thuộc sở hữu của một delegator và liên kết với shares cho một validator. Người gửi giao dịch là chủ sở hữu của bond.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/staking/v1beta1/staking.proto#L198-L216
```

#### Delegator Shares

Khi một người delegate token cho Validator, họ được cấp một số delegator shares dựa trên tỷ giá hối đoái động, được tính như sau từ tổng số token được delegate cho validator và số shares đã phát hành:

`Shares per Token = validator.TotalShares() / validator.Tokens()`

Chỉ số shares nhận được được lưu trên DelegationEntry. Khi delegator Undelegate, số lượng token họ nhận được tính từ số shares họ hiện nắm giữ và tỷ giá nghịch đảo:

`Tokens per Share = validator.Tokens() / validatorShares()`

Các `Shares` này đơn giản là cơ chế kế toán. Chúng không phải là tài sản có thể thay thế. Lý do cho cơ chế này là để đơn giản hóa kế toán xung quanh slashing. Thay vì lặp lại việc slash token của mỗi delegation entry, thay vào đó tổng token bonded của Validator có thể bị slash, qua đó giảm hiệu quả giá trị của mỗi delegator share đã phát hành.

### UnbondingDelegation

Shares trong `Delegation` có thể được unbond, nhưng chúng phải tồn tại trong một thời gian dưới dạng `UnbondingDelegation`, nơi shares có thể bị giảm nếu phát hiện hành vi Byzantine.

`UnbondingDelegation` được lập chỉ mục trong store như sau:

* UnbondingDelegation: `0x32 | DelegatorAddrLen (1 byte) | DelegatorAddr | ValidatorAddrLen (1 byte) | ValidatorAddr -> ProtocolBuffer(unbondingDelegation)`
* UnbondingDelegationsFromValidator: `0x33 | ValidatorAddrLen (1 byte) | ValidatorAddr | DelegatorAddrLen (1 byte) | DelegatorAddr -> nil`
* UnbondingDelegationByUnbondingId: `0x38 | UnbondingId -> 0x32 | DelegatorAddrLen (1 byte) | DelegatorAddr | ValidatorAddrLen (1 byte) | ValidatorAddr`
 `UnbondingDelegation` được sử dụng trong queries để tra cứu tất cả unbonding delegations cho một delegator nhất định.

`UnbondingDelegationsFromValidator` được sử dụng trong slashing để tra cứu tất cả unbonding delegations liên kết với một validator nhất định cần bị slash.

 `UnbondingDelegationByUnbondingId` là chỉ mục bổ sung cho phép tra cứu unbonding delegations theo unbonding ID của các unbonding delegation entries chứa.

Đối tượng UnbondingDelegation được tạo mỗi khi unbonding được khởi tạo.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/staking/v1beta1/staking.proto#L218-L261
```

### Redelegation

Giá trị token bonded của `Delegation` có thể được redelegate ngay lập tức từ validator nguồn sang validator khác (validator đích). Tuy nhiên khi điều này xảy ra chúng phải được theo dõi trong đối tượng `Redelegation`, qua đó shares của chúng có thể bị slash nếu token của chúng đã đóng góp vào lỗi Byzantine do validator nguồn gây ra.

`Redelegation` được lập chỉ mục trong store như sau:

* Redelegations: `0x34 | DelegatorAddrLen (1 byte) | DelegatorAddr | ValidatorAddrLen (1 byte) | ValidatorSrcAddr | ValidatorDstAddr -> ProtocolBuffer(redelegation)`
* RedelegationsBySrc: `0x35 | ValidatorSrcAddrLen (1 byte) | ValidatorSrcAddr | ValidatorDstAddrLen (1 byte) | ValidatorDstAddr | DelegatorAddrLen (1 byte) | DelegatorAddr -> nil`
* RedelegationsByDst: `0x36 | ValidatorDstAddrLen (1 byte) | ValidatorDstAddr | ValidatorSrcAddrLen (1 byte) | ValidatorSrcAddr | DelegatorAddrLen (1 byte) | DelegatorAddr -> nil`
* RedelegationByUnbondingId: `0x38 | UnbondingId -> 0x34 | DelegatorAddrLen (1 byte) | DelegatorAddr | ValidatorAddrLen (1 byte) | ValidatorSrcAddr | ValidatorDstAddr`

 `Redelegations` được sử dụng cho queries để tra cứu tất cả redelegations cho một delegator nhất định.

 `RedelegationsBySrc` được sử dụng cho slashing dựa trên `ValidatorSrcAddr`.

 `RedelegationsByDst` được sử dụng cho slashing dựa trên `ValidatorDstAddr`

Map đầu tiên ở đây được sử dụng cho queries để tra cứu tất cả redelegations cho một delegator nhất định. Map thứ hai được sử dụng cho slashing dựa trên `ValidatorSrcAddr`, trong khi map thứ ba dùng cho slashing dựa trên `ValidatorDstAddr`.

`RedelegationByUnbondingId` là chỉ mục bổ sung cho phép tra cứu redelegations theo unbonding ID của các redelegation entries chứa.

Đối tượng redelegation được tạo mỗi khi redelegation xảy ra. Để ngăn chặn "redelegation hopping", redelegations không thể xảy ra trong tình huống:

* (re)delegator đã có redelegation chưa trưởng thành khác đang tiến hành với đích đến là một validator (gọi là `Validator X`)
* và, (re)delegator đang cố gắng tạo redelegation _mới_ trong đó validator nguồn của redelegation mới này là `Validator X`.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/staking/v1beta1/staking.proto#L263-L308
```

### Queues

Tất cả đối tượng queue được sắp xếp theo timestamp. Thời gian sử dụng trong bất kỳ queue nào trước tiên được chuyển đổi sang UTC, làm tròn đến nanosecond gần nhất rồi sắp xếp. Định dạng thời gian có thể sắp xếp sử dụng là phiên bản sửa đổi nhẹ của RFC3339Nano và sử dụng chuỗi định dạng `"2006-01-02T15:04:05.000000000"`. Đáng chú ý định dạng này:

* đệm số không bên phải (right pads all zeros)
* bỏ thông tin múi giờ (đã sử dụng UTC)

Trong mọi trường hợp, timestamp được lưu trữ đại diện cho thời điểm trưởng thành của phần tử queue.

#### UnbondingDelegationQueue

Để theo dõi tiến trình unbonding delegations, queue unbonding delegations được duy trì.

* UnbondingDelegation: `0x41 | format(time) -> []DVPair`

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/staking/v1beta1/staking.proto#L162-L172
```

#### RedelegationQueue

Để theo dõi tiến trình redelegations, queue redelegation được duy trì.

* RedelegationQueue: `0x42 | format(time) -> []DVVTriplet`

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/staking/v1beta1/staking.proto#L179-L191
```

#### ValidatorQueue

Để theo dõi tiến trình unbonding validators, queue validator được duy trì.

* ValidatorQueueTime: `0x43 | format(time) -> []sdk.ValAddress`

Đối tượng được lưu trữ bởi mỗi key là mảng địa chỉ validator operator mà từ đó có thể truy cập đối tượng validator. Thông thường chỉ mong đợi một bản ghi validator duy nhất sẽ liên kết với một timestamp nhất định, tuy nhiên có thể có nhiều validator tồn tại trong queue tại cùng vị trí.

### HistoricalInfo

Các đối tượng HistoricalInfo được lưu trữ và cắt tỉa ở mỗi block sao cho staking keeper lưu giữ `n` historical info gần nhất được xác định bởi tham số module staking: `HistoricalEntries`.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/staking/v1beta1/staking.proto#L17-L24
```

Ở mỗi BeginBlock, staking keeper sẽ lưu trữ Header hiện tại và các Validator đã commit block hiện tại trong đối tượng `HistoricalInfo`. Các Validator được sắp xếp theo địa chỉ để đảm bảo chúng theo thứ tự xác định. Các HistoricalEntries cũ nhất sẽ được cắt tỉa để đảm bảo chỉ tồn tại số lượng historical entries được xác định bởi tham số.

## State Transitions

### Validators

Các chuyển đổi trạng thái trong validators được thực hiện ở mỗi [`EndBlock`](#validator-set-changes) để kiểm tra thay đổi trong `ValidatorSet` active.

Một validator có thể là `Unbonded`, `Unbonding` hoặc `Bonded`. `Unbonded` và `Unbonding` được gọi chung là `Not Bonded`. Một validator có thể chuyển trực tiếp giữa tất cả các trạng thái, ngoại trừ từ `Bonded` sang `Unbonded`.

#### Not bonded to Bonded

Chuyển đổi sau xảy ra khi thứ hạng của validator trong `ValidatorPowerIndex` vượt qua `LastValidator`:

* đặt `validator.Status` thành `Bonded`
* chuyển `validator.Tokens` từ `NotBondedTokens` sang `BondedPool` `ModuleAccount`
* xóa bản ghi hiện có khỏi `ValidatorByPowerIndex`
* thêm bản ghi mới đã cập nhật vào `ValidatorByPowerIndex`
* cập nhật đối tượng `Validator` cho validator này
* nếu tồn tại, xóa bất kỳ bản ghi `ValidatorQueue` nào cho validator này

#### Bonded to Unbonding

Khi validator bắt đầu quá trình unbonding, các thao tác sau xảy ra:

* chuyển `validator.Tokens` từ `BondedPool` sang `NotBondedTokens` `ModuleAccount`
* đặt `validator.Status` thành `Unbonding`
* xóa bản ghi hiện có khỏi `ValidatorByPowerIndex`
* thêm bản ghi mới đã cập nhật vào `ValidatorByPowerIndex`
* cập nhật đối tượng `Validator` cho validator này
* chèn bản ghi mới vào `ValidatorQueue` cho validator này

#### Unbonding to Unbonded

Validator chuyển từ unbonding sang unbonded khi đối tượng `ValidatorQueue` chuyển từ bonded sang unbonded:

* cập nhật đối tượng `Validator` cho validator này
* đặt `validator.Status` thành `Unbonded`

#### Jail/Unjail

Khi validator bị jailed, họ thực tế bị loại khỏi tập CometBFT. Quá trình này cũng có thể được đảo ngược. Các thao tác sau xảy ra:

* đặt `Validator.Jailed` và cập nhật đối tượng
* nếu jailed thì xóa bản ghi khỏi `ValidatorByPowerIndex`
* nếu unjailed thì thêm bản ghi vào `ValidatorByPowerIndex`

Validator bị jailed không có mặt trong bất kỳ store nào sau:

* power store (từ consensus power đến địa chỉ)

### Delegations

#### Delegate

Khi delegation xảy ra, cả validator và đối tượng delegation đều bị ảnh hưởng:

* xác định shares của delegator dựa trên token được delegate và tỷ giá của validator
* loại bỏ token khỏi tài khoản gửi
* thêm shares vào đối tượng delegation hoặc thêm vào đối tượng validator được tạo
* thêm delegator shares mới và cập nhật đối tượng `Validator`
* chuyển `delegation.Amount` từ tài khoản delegator sang `BondedPool` hoặc `NotBondedPool` `ModuleAccount` tùy thuộc `validator.Status` là `Bonded` hay không
* xóa bản ghi hiện có khỏi `ValidatorByPowerIndex`
* thêm bản ghi mới đã cập nhật vào `ValidatorByPowerIndex`

#### Begin Unbonding

Như một phần của chuyển đổi trạng thái Undelegate và Complete Unbonding, Unbond Delegation có thể được gọi.

* trừ shares unbonded khỏi delegator
* thêm token unbonded vào `UnbondingDelegationEntry`
* cập nhật delegation hoặc xóa delegation nếu không còn shares
* nếu delegation là operator của validator và không còn shares thì kích hoạt jail validator
* cập nhật validator với việc loại bỏ delegator shares và coin liên kết
* nếu trạng thái validator là `Bonded`, chuyển giá trị `Coins` của shares unbonded từ `BondedPool` sang `NotBondedPool` `ModuleAccount`
* xóa validator nếu unbonded và không còn delegation shares
* lấy `unbondingId` duy nhất và map nó với `UnbondingDelegationEntry` trong `UnbondingDelegationByUnbondingId`
* gọi hook `AfterUnbondingInitiated(unbondingId)`
* thêm unbonding delegation vào `UnbondingDelegationQueue` với thời gian hoàn thành đặt là `UnbondingTime`

#### Cancel an `UnbondingDelegation` Entry

Khi `cancel unbond delegation` xảy ra, cả `validator`, `delegation` và trạng thái `UnbondingDelegationQueue` sẽ được cập nhật.

* nếu số lượng cancel unbonding delegation bằng với `balance` của entry `UnbondingDelegation`, thì entry `UnbondingDelegation` bị xóa khỏi `UnbondingDelegationQueue`
* nếu số lượng cancel unbonding delegation nhỏ hơn balance của entry `UnbondingDelegation`, thì entry `UnbondingDelegation` sẽ được cập nhật với balance mới trong `UnbondingDelegationQueue`
* số lượng `amount` bị cancel được [Delegated](#delegations) trở lại `validator` gốc

#### Complete Unbonding

Đối với undelegations không hoàn thành ngay lập tức, các thao tác sau xảy ra khi phần tử unbonding delegation queue trưởng thành:

* xóa entry khỏi đối tượng `UnbondingDelegation`
* chuyển token từ `NotBondedPool` `ModuleAccount` sang `Account` của delegator

#### Begin Redelegation

Redelegations ảnh hưởng đến delegation, validator nguồn và đích.

* thực hiện `unbond` delegation từ validator nguồn để lấy giá trị token của shares unbonded
* sử dụng token unbonded, `Delegate` chúng cho validator đích
* nếu `sourceValidator.Status` là `Bonded` và `destinationValidator` không phải, chuyển token mới được delegate từ `BondedPool` sang `NotBondedPool` `ModuleAccount`
* ngược lại, nếu `sourceValidator.Status` không phải `Bonded` và `destinationValidator` là `Bonded`, chuyển token mới được delegate từ `NotBondedPool` sang `BondedPool` `ModuleAccount`
* ghi số lượng token vào entry mới trong `Redelegation` liên quan

Từ khi redelegation bắt đầu cho đến khi hoàn thành, delegator ở trạng thái "pseudo-unbonding" và vẫn có thể bị slash cho các vi phạm xảy ra trước khi redelegation bắt đầu.

#### Complete Redelegation

Khi redelegations hoàn thành, điều sau xảy ra:

* xóa entry khỏi đối tượng `Redelegation`

### Slashing

#### Slash Validator

Khi Validator bị slash, điều sau xảy ra:

* Tổng `slashAmount` được tính là `slashFactor` (tham số chain) \* `TokensFromConsensusPower`, tổng số token bonded cho validator tại thời điểm vi phạm.
* Mọi unbonding delegation và pseudo-unbonding redelegation mà vi phạm xảy ra trước khi unbonding hoặc redelegation bắt đầu từ validator đều bị slash theo phần trăm `slashFactor` của initialBalance.
* Mỗi số lượng bị slash từ redelegations và unbonding delegations được trừ khỏi tổng slash amount.
* `remainingSlashAmount` sau đó được slash từ token của validator trong `BondedPool` hoặc `NonBondedPool` tùy thuộc trạng thái validator. Điều này giảm tổng cung token.

Trong trường hợp slash do bất kỳ vi phạm nào yêu cầu evidence phải được gửi (ví dụ double-sign), slash xảy ra tại block nơi evidence được bao gồm, không phải tại block nơi vi phạm xảy ra. Nói cách khác, validator không bị slash hồi tố, chỉ khi họ bị bắt.

#### Slash Unbonding Delegation

Khi validator bị slash, các unbonding delegations từ validator đó bắt đầu unbonding sau thời điểm vi phạm cũng bị slash. Mọi entry trong mọi unbonding delegation từ validator bị slash theo `slashFactor`. Số lượng bị slash được tính từ `InitialBalance` của delegation và được giới hạn để ngăn balance âm. Unbondings đã hoàn thành (hoặc trưởng thành) không bị slash.

#### Slash Redelegation

Khi validator bị slash, tất cả redelegations từ validator bắt đầu sau vi phạm cũng bị slash. Redelegations bị slash theo `slashFactor`. Redelegations bắt đầu trước vi phạm không bị slash. Số lượng bị slash được tính từ `InitialBalance` của delegation và được giới hạn để ngăn balance âm. Redelegations trưởng thành (đã hoàn thành pseudo-unbonding) không bị slash.

### How Shares are calculated

Tại bất kỳ thời điểm nào, mỗi validator có số token `T` và có số shares đã phát hành `S`. Mỗi delegator `i` nắm giữ số shares `S_i`. Số token là tổng tất cả token được delegate cho validator, cộng rewards, trừ slashes.

Delegator có quyền nhận một phần token cơ bản tỷ lệ với tỷ lệ shares của họ. Vậy delegator `i` có quyền nhận `T * S_i / S` token của validator.

Khi delegator delegate token mới cho validator, họ nhận số shares tỷ lệ với đóng góp của họ. Vậy khi delegator `j` delegate `T_j` token, họ nhận `S_j = S * T_j / T` shares. Tổng số token là `T + T_j` và tổng số shares là `S + S_j`. Tỷ lệ shares của `j` giống với tỷ lệ tổng token đóng góp: `(S + S_j) / S = (T + T_j) / T`.

Trường hợp đặc biệt là delegation ban đầu, khi `T = 0` và `S = 0`, nên `T_j / T` không xác định. Đối với delegation ban đầu, delegator `j` delegate `T_j` token nhận `S_j = T_j` shares. Vậy validator chưa nhận rewards và chưa bị slash sẽ có `T = S`.

## Messages

Trong phần này chúng ta mô tả xử lý các thông tin staking và các cập nhật tương ứng cho state. Tất cả đối tượng state được tạo/sửa đổi được xác định bởi mỗi message đều được định nghĩa trong phần [state](#state).

### MsgCreateValidator

Validator được tạo bằng message `MsgCreateValidator`. Validator phải được tạo với delegation ban đầu từ operator.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/staking/v1beta1/tx.proto#L20-L21
```

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/staking/v1beta1/tx.proto#L50-L73
```

Message này được kỳ vọng sẽ thất bại nếu:

* đã có validator khác với operator address này được đăng ký
* đã có validator khác với pubkey này được đăng ký
* token self-delegation ban đầu có denom không được chỉ định là bonding denom
* tham số commission không hợp lệ, cụ thể:
    * `MaxRate` hoặc > 1 hoặc < 0
    * `Rate` ban đầu hoặc âm hoặc > `MaxRate`
    * `MaxChangeRate` ban đầu hoặc âm hoặc > `MaxRate`
* các trường description quá lớn

Message này tạo và lưu đối tượng `Validator` tại các chỉ mục phù hợp. Ngoài ra self-delegation được thực hiện với token delegation ban đầu `Delegation`. Validator luôn bắt đầu ở trạng thái unbonded nhưng có thể bonded trong end-block đầu tiên.

### MsgEditValidator

`Description`, `CommissionRate` của validator có thể được cập nhật bằng message `MsgEditValidator`.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/staking/v1beta1/tx.proto#L23-L24
```

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/staking/v1beta1/tx.proto#L78-L97
```

Message này được kỳ vọng sẽ thất bại nếu:

* `CommissionRate` ban đầu hoặc âm hoặc > `MaxRate`
* `CommissionRate` đã được cập nhật trong 24 giờ trước
* `CommissionRate` > `MaxChangeRate`
* các trường description quá lớn

Message này lưu đối tượng `Validator` đã cập nhật.

### MsgDelegate

Trong message này delegator cung cấp coin và nhận lại một số lượng delegator-shares (mới tạo) của validator được gán cho `Delegation.Shares`.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/staking/v1beta1/tx.proto#L26-L28
```

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/staking/v1beta1/tx.proto#L102-L114
```

Message này được kỳ vọng sẽ thất bại nếu:

* validator không tồn tại
* `Amount` `Coin` có denomination khác với định nghĩa trong `params.BondDenom`
* tỷ giá không hợp lệ, nghĩa là validator không có token (do slashing) nhưng có shares outstanding
* số lượng delegate nhỏ hơn delegation tối thiểu cho phép

Nếu đối tượng `Delegation` hiện có cho các địa chỉ cung cấp chưa tồn tại thì nó được tạo như một phần của message này, nếu không thì `Delegation` hiện có được cập nhật để bao gồm shares mới nhận được.

Delegator nhận shares mới tạo ở tỷ giá hiện tại. Tỷ giá là số shares hiện có trong validator chia cho số token hiện đang được delegate.

Validator được cập nhật trong chỉ mục `ValidatorByPower` và delegation được theo dõi trong đối tượng validator trong chỉ mục `Validators`.

Có thể delegate cho validator bị jailed, điểm khác biệt duy nhất là nó sẽ không được thêm vào power index cho đến khi unjailed.

![Delegation sequence](https://raw.githubusercontent.com/cosmos/cosmos-sdk/release/v0.46.x/docs/uml/svg/delegation_sequence.svg)

### MsgUndelegate

Message `MsgUndelegate` cho phép delegator undelegate token của họ khỏi validator.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/staking/v1beta1/tx.proto#L34-L36
```

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/staking/v1beta1/tx.proto#L140-L152
```

Message này trả về response chứa thời gian hoàn thành undelegation:

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/staking/v1beta1/tx.proto#L154-L158
```

Message này được kỳ vọng sẽ thất bại nếu:

* delegation không tồn tại
* validator không tồn tại
* delegation có ít shares hơn giá trị của `Amount`
* `UnbondingDelegation` hiện có đã có số entries tối đa theo định nghĩa `params.MaxEntries`
* `Amount` có denomination khác với định nghĩa trong `params.BondDenom`

Khi message này được xử lý, các hành động sau xảy ra:

* `DelegatorShares` của validator và `Shares` của delegation đều bị giảm bởi `SharesAmount` của message
* tính giá trị token của shares, loại bỏ số lượng token đó khỏi validator
* với những token bị loại bỏ, nếu validator là:
    * `Bonded` - thêm chúng vào entry trong `UnbondingDelegation` (tạo `UnbondingDelegation` nếu chưa tồn tại) với thời gian hoàn thành là một chu kỳ unbonding đầy đủ từ thời điểm hiện tại. Cập nhật pool shares để giảm BondedTokens và tăng NotBondedTokens theo giá trị token của shares.
    * `Unbonding` - thêm chúng vào entry trong `UnbondingDelegation` (tạo `UnbondingDelegation` nếu chưa tồn tại) với thời gian hoàn thành giống validator (`UnbondingMinTime`).
    * `Unbonded` - sau đó chuyển coin đến `DelegatorAddr` của message
* nếu không còn `Shares` trong delegation thì đối tượng delegation bị xóa khỏi store
    * trong tình huống này nếu delegation là self-delegation của validator thì cũng jail validator.

![Unbond sequence](https://raw.githubusercontent.com/cosmos/cosmos-sdk/release/v0.46.x/docs/uml/svg/unbond_sequence.svg)

### MsgCancelUnbondingDelegation

Message `MsgCancelUnbondingDelegation` cho phép delegator hủy entry `unbondingDelegation` và delegate trở lại validator trước đó.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/staking/v1beta1/tx.proto#L38-L42
```

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/staking/v1beta1/tx.proto#L160-L175
```

Message này được kỳ vọng sẽ thất bại nếu:

* entry `unbondingDelegation` đã được xử lý
* số lượng `cancel unbonding delegation` lớn hơn balance của entry `unbondingDelegation`
* height `cancel unbonding delegation` không tồn tại trong `unbondingDelegationQueue` của delegator

Khi message này được xử lý, các hành động sau xảy ra:

* nếu balance của Entry `unbondingDelegation` bằng không
    * trong điều kiện này entry `unbondingDelegation` sẽ bị xóa khỏi `unbondingDelegationQueue`
    * nếu không thì `unbondingDelegationQueue` sẽ được cập nhật với balance và initial balance mới của entry `unbondingDelegation`
* `DelegatorShares` của validator và `Shares` của delegation đều tăng bởi `Amount` của message

### MsgBeginRedelegate

Lệnh redelegation cho phép delegator chuyển validator ngay lập tức. Sau khi thời gian unbonding trôi qua, redelegation tự động hoàn thành trong EndBlocker.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/staking/v1beta1/tx.proto#L30-L32
```

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/staking/v1beta1/tx.proto#L119-L132
```

Message này trả về response chứa thời gian hoàn thành redelegation:

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/staking/v1beta1/tx.proto#L133-L138
```

Message này được kỳ vọng sẽ thất bại nếu:

* delegation không tồn tại
* validator nguồn hoặc đích không tồn tại
* delegation có ít shares hơn giá trị của `Amount`
* validator nguồn có redelegation nhận chưa trưởng thành (được gọi là redelegation có thể transitive)
* `Redelegation` hiện có đã có số entries tối đa theo định nghĩa `params.MaxEntries`
* `Amount` `Coin` có denomination khác với định nghĩa trong `params.BondDenom`

Khi message này được xử lý, các hành động sau xảy ra:

* `DelegatorShares` của validator nguồn và `Shares` của delegations đều bị giảm bởi `SharesAmount` của message
* tính giá trị token của shares, loại bỏ số lượng token đó khỏi validator nguồn
* nếu validator nguồn là:
    * `Bonded` - thêm entry vào `Redelegation` (tạo `Redelegation` nếu chưa tồn tại) với thời gian hoàn thành là một chu kỳ unbonding đầy đủ từ thời điểm hiện tại. Cập nhật pool shares để giảm BondedTokens và tăng NotBondedTokens theo giá trị token của shares (có thể được đảo ngược hiệu quả trong bước tiếp theo).
    * `Unbonding` - thêm entry vào `Redelegation` (tạo `Redelegation` nếu chưa tồn tại) với thời gian hoàn thành giống validator (`UnbondingMinTime`).
    * `Unbonded` - không cần hành động trong bước này
* Delegate giá trị token cho validator đích, có thể chuyển token trở lại trạng thái bonded
* nếu không còn `Shares` trong delegation nguồn thì đối tượng delegation nguồn bị xóa khỏi store
    * trong tình huống này nếu delegation là self-delegation của validator thì cũng jail validator

![Begin redelegation sequence](https://raw.githubusercontent.com/cosmos/cosmos-sdk/release/v0.46.x/docs/uml/svg/begin_redelegation_sequence.svg)


### MsgUpdateParams

`MsgUpdateParams` cập nhật tham số module staking. Params được cập nhật thông qua governance proposal trong đó signer là địa chỉ tài khoản gov module.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/staking/v1beta1/tx.proto#L182-L195
```

Xử lý message có thể thất bại nếu:

* signer không phải authority được xác định trong staking keeper (thường là tài khoản gov module).

## Begin-Block

Mỗi lần gọi abci begin block, historical info sẽ được lưu trữ và cắt tỉa theo tham số `HistoricalEntries`.

### Historical Info Tracking

Nếu tham số `HistoricalEntries` là 0 thì `BeginBlock` thực hiện no-op.

Nếu không, historical info mới nhất được lưu dưới key `historicalInfoKey|height`, trong khi các entries cũ hơn `height - HistoricalEntries` bị xóa. Trong hầu hết trường hợp, điều này dẫn đến một entry duy nhất bị cắt tỉa mỗi block. Tuy nhiên, nếu tham số `HistoricalEntries` đã thay đổi sang giá trị thấp hơn thì sẽ có nhiều entries trong store cần được cắt tỉa.

## End-Block

Mỗi lần gọi abci end block, các thao tác cập nhật queues và thay đổi validator set được chỉ định để thực thi.

### Validator Set Changes

Validator set staking được cập nhật trong quá trình này qua các chuyển đổi trạng thái chạy vào cuối mỗi block. Như một phần của quá trình này, mọi validator được cập nhật cũng được trả về cho CometBFT để đưa vào validator set CometBFT chịu trách nhiệm xác thực thông điệp CometBFT ở tầng consensus. Các thao tác như sau:

* validator set mới được lấy là top `params.MaxValidators` number of validators từ chỉ mục `ValidatorsByPower`
* validator set trước được so sánh với validator set mới:
    * các validator thiếu bắt đầu unbonding và `Tokens` của họ được chuyển từ `BondedPool` sang `NotBondedPool` `ModuleAccount`
    * các validator mới được bonded ngay lập tức và `Tokens` của họ được chuyển từ `NotBondedPool` sang `BondedPool` `ModuleAccount`

Trong mọi trường hợp, mọi validator rời khỏi hoặc vào validator set bonded hoặc thay đổi balance và vẫn ở trong validator set bonded đều phát sinh message cập nhật báo cáo consensus power mới của họ được truyền cho CometBFT.

`LastTotalPower` và `LastValidatorsPower` giữ trạng thái tổng power và validator power từ cuối block trước, và được sử dụng để kiểm tra thay đổi đã xảy ra trong `ValidatorsByPower` và tổng power mới, được tính trong `EndBlock`.

### Queues

Trong staking, một số chuyển đổi trạng thái không tức thì mà diễn ra trong khoảng thời gian (thường là unbonding period). Khi các chuyển đổi này trưởng thành, các thao tác nhất định phải diễn ra để hoàn thành thao tác state. Điều này đạt được thông qua việc sử dụng queues được kiểm tra/xử lý vào cuối mỗi block.

#### Unbonding Validators

Khi validator bị loại khỏi validator set bonded (dù do bị jailed hoặc không có đủ token bonded), nó bắt đầu quá trình unbonding cùng với tất cả delegations của nó bắt đầu unbonding (trong khi vẫn được delegate cho validator này). Tại thời điểm này validator được gọi là "unbonding validator", qua đó nó sẽ trưởng thành để trở thành "unbonded validator" sau khi unbonding period trôi qua.

Mỗi block, validator queue được kiểm tra cho các unbonding validators trưởng thành (cụ thể với completion time <= thời gian hiện tại và completion height <= block height hiện tại). Tại thời điểm này mọi validator trưởng thành không còn delegations nào bị xóa khỏi state. Đối với tất cả unbonding validators trưởng thành khác vẫn còn delegations, `validator.Status` được chuyển từ `types.Unbonding` sang `types.Unbonded`.

Các thao tác unbonding có thể bị tạm giữ bởi các module bên ngoài qua phương thức `PutUnbondingOnHold(unbondingId)`. Kết quả là, thao tác unbonding (ví dụ unbonding delegation) đang bị tạm giữ không thể hoàn thành ngay cả khi đạt trưởng thành. Để thao tác unbonding với `unbondingId` cuối cùng hoàn thành (sau khi đạt trưởng thành), mỗi lần gọi `PutUnbondingOnHold(unbondingId)` phải được khớp với lần gọi `UnbondingCanComplete(unbondingId)`.

#### Unbonding Delegations

Hoàn thành unbonding của tất cả `UnbondingDelegations.Entries` trưởng thành trong queue `UnbondingDelegations` với quy trình sau:

* chuyển balance coins sang địa chỉ ví của delegator
* xóa entry trưởng thành khỏi `UnbondingDelegation.Entries`
* xóa đối tượng `UnbondingDelegation` khỏi store nếu không còn entries

#### Redelegations

Hoàn thành unbonding của tất cả `Redelegation.Entries` trưởng thành trong queue `Redelegations` với quy trình sau:

* xóa entry trưởng thành khỏi `Redelegation.Entries`
* xóa đối tượng `Redelegation` khỏi store nếu không còn entries

## Hooks

Các module khác có thể đăng ký thao tác để thực thi khi một sự kiện nhất định đã xảy ra trong staking. Các sự kiện này có thể được đăng ký để thực thi ngay `Before` hoặc `After` sự kiện staking (theo tên hook). Các hooks sau có thể đăng ký với staking:

* `AfterValidatorCreated(Context, ValAddress) error`
    * được gọi khi validator được tạo
* `BeforeValidatorModified(Context, ValAddress) error`
    * được gọi khi trạng thái validator thay đổi
* `AfterValidatorRemoved(Context, ConsAddress, ValAddress) error`
    * được gọi khi validator bị xóa
* `AfterValidatorBonded(Context, ConsAddress, ValAddress) error`
    * được gọi khi validator được bonded
* `AfterValidatorBeginUnbonding(Context, ConsAddress, ValAddress) error`
    * được gọi khi validator bắt đầu unbonding
* `BeforeDelegationCreated(Context, AccAddress, ValAddress) error`
    * được gọi khi delegation được tạo
* `BeforeDelegationSharesModified(Context, AccAddress, ValAddress) error`
    * được gọi khi shares của delegation được sửa đổi
* `AfterDelegationModified(Context, AccAddress, ValAddress) error`
    * được gọi khi delegation được tạo hoặc sửa đổi
* `BeforeDelegationRemoved(Context, AccAddress, ValAddress) error`
    * được gọi khi delegation bị xóa
* `AfterUnbondingInitiated(Context, UnbondingID)`
    * được gọi khi thao tác unbonding (validator unbonding, unbonding delegation, redelegation) được khởi tạo


## Events

Module staking phát ra các sự kiện sau:

### EndBlocker

| Type                  | Attribute Key         | Attribute Value           |
| --------------------- | --------------------- | ------------------------- |
| complete_unbonding    | amount                | {totalUnbondingAmount}    |
| complete_unbonding    | validator             | {validatorAddress}        |
| complete_unbonding    | delegator             | {delegatorAddress}        |
| complete_redelegation | amount                | {totalRedelegationAmount} |
| complete_redelegation | source_validator      | {srcValidatorAddress}     |
| complete_redelegation | destination_validator | {dstValidatorAddress}     |
| complete_redelegation | delegator             | {delegatorAddress}        |

## Msg's

### MsgCreateValidator

| Type             | Attribute Key | Attribute Value    |
| ---------------- | ------------- | ------------------ |
| create_validator | validator     | {validatorAddress} |
| create_validator | amount        | {delegationAmount} |
| message          | module        | staking            |
| message          | action        | create_validator   |
| message          | sender        | {senderAddress}    |

### MsgEditValidator

| Type           | Attribute Key       | Attribute Value     |
| -------------- | ------------------- | ------------------- |
| edit_validator | commission_rate     | {commissionRate}    |
| edit_validator | min_self_delegation | {minSelfDelegation} |
| message        | module              | staking             |
| message        | action              | edit_validator      |
| message        | sender              | {senderAddress}     |

### MsgDelegate

| Type     | Attribute Key | Attribute Value    |
| -------- | ------------- | ------------------ |
| delegate | validator     | {validatorAddress} |
| delegate | amount        | {delegationAmount} |
| message  | module        | staking            |
| message  | action        | delegate           |
| message  | sender        | {senderAddress}    |

### MsgUndelegate

| Type    | Attribute Key       | Attribute Value    |
| ------- | ------------------- | ------------------ |
| unbond  | validator           | {validatorAddress} |
| unbond  | amount              | {unbondAmount}     |
| unbond  | completion_time [0] | {completionTime}   |
| message | module              | staking            |
| message | action              | begin_unbonding    |
| message | sender              | {senderAddress}    |

* [0] Time is formatted in the RFC3339 standard

### MsgCancelUnbondingDelegation

| Type                          | Attribute Key       | Attribute Value                     |
| ----------------------------- | ------------------  | ------------------------------------|
| cancel_unbonding_delegation   | validator           | {validatorAddress}                  |
| cancel_unbonding_delegation   | delegator           | {delegatorAddress}                  |
| cancel_unbonding_delegation   | amount              | {cancelUnbondingDelegationAmount}   |
| cancel_unbonding_delegation   | creation_height     | {unbondingCreationHeight}           |
| message                       | module              | staking                             |
| message                       | action              | cancel_unbond                       |
| message                       | sender              | {senderAddress}                     |

### MsgBeginRedelegate

| Type       | Attribute Key         | Attribute Value       |
| ---------- | --------------------- | --------------------- |
| redelegate | source_validator      | {srcValidatorAddress} |
| redelegate | destination_validator | {dstValidatorAddress} |
| redelegate | amount                | {unbondAmount}        |
| redelegate | completion_time [0]   | {completionTime}      |
| message    | module                | staking               |
| message    | action                | begin_redelegate      |
| message    | sender                | {senderAddress}       |

* [0] Time is formatted in the RFC3339 standard

## Parameters

Module staking chứa các tham số sau:

| Key               | Type             | Example                |
|-------------------|------------------|------------------------|
| UnbondingTime     | string (time ns) | "259200000000000"      |
| MaxValidators     | uint16           | 100                    |
| KeyMaxEntries     | uint16           | 7                      |
| HistoricalEntries | uint16           | 3                      |
| BondDenom         | string           | "stake"                |
| MinCommissionRate | string           | "0.000000000000000000" |

## Client

### CLI

Người dùng có thể truy vấn và tương tác với module `staking` bằng CLI.

#### Query

Các lệnh `query` cho phép người dùng truy vấn state `staking`.

```bash
simd query staking --help
```

##### delegation

Lệnh `delegation` cho phép người dùng truy vấn delegations cho một delegator nhất định trên một validator nhất định.

Usage:

```bash
simd query staking delegation [delegator-addr] [validator-addr] [flags]
```

Ví dụ:

```bash
simd query staking delegation cosmos1gghjut3ccd8ay0zduzj64hwre2fxs9ld75ru9p cosmosvaloper1gghjut3ccd8ay0zduzj64hwre2fxs9ldmqhffj
```

Ví dụ đầu ra:

```bash
balance:
  amount: "10000000000"
  denom: stake
delegation:
  delegator_address: cosmos1gghjut3ccd8ay0zduzj64hwre2fxs9ld75ru9p
  shares: "10000000000.000000000000000000"
  validator_address: cosmosvaloper1gghjut3ccd8ay0zduzj64hwre2fxs9ldmqhffj
```

##### delegations

Lệnh `delegations` cho phép người dùng truy vấn delegations cho một delegator nhất định trên tất cả validators.

Usage:

```bash
simd query staking delegations [delegator-addr] [flags]
```

Ví dụ:

```bash
simd query staking delegations cosmos1gghjut3ccd8ay0zduzj64hwre2fxs9ld75ru9p
```

Ví dụ đầu ra:

```bash
delegation_responses:
- balance:
    amount: "10000000000"
    denom: stake
  delegation:
    delegator_address: cosmos1gghjut3ccd8ay0zduzj64hwre2fxs9ld75ru9p
    shares: "10000000000.000000000000000000"
    validator_address: cosmosvaloper1gghjut3ccd8ay0zduzj64hwre2fxs9ldmqhffj
- balance:
    amount: "10000000000"
    denom: stake
  delegation:
    delegator_address: cosmos1gghjut3ccd8ay0zduzj64hwre2fxs9ld75ru9p
    shares: "10000000000.000000000000000000"
    validator_address: cosmosvaloper1x20lytyf6zkcrv5edpkfkn8sz578qg5sqfyqnp
pagination:
  next_key: null
  total: "0"
```

##### delegations-to

Lệnh `delegations-to` cho phép người dùng truy vấn delegations trên một validator nhất định.

Usage:

```bash
simd query staking delegations-to [validator-addr] [flags]
```

Ví dụ:

```bash
simd query staking delegations-to cosmosvaloper1gghjut3ccd8ay0zduzj64hwre2fxs9ldmqhffj
```

Ví dụ đầu ra:

```bash
- balance:
    amount: "504000000"
    denom: stake
  delegation:
    delegator_address: cosmos1q2qwwynhv8kh3lu5fkeex4awau9x8fwt45f5cp
    shares: "504000000.000000000000000000"
    validator_address: cosmosvaloper1gghjut3ccd8ay0zduzj64hwre2fxs9ldmqhffj
- balance:
    amount: "78125000000"
    denom: uixo
  delegation:
    delegator_address: cosmos1qvppl3479hw4clahe0kwdlfvf8uvjtcd99m2ca
    shares: "78125000000.000000000000000000"
    validator_address: cosmosvaloper1gghjut3ccd8ay0zduzj64hwre2fxs9ldmqhffj
pagination:
  next_key: null
  total: "0"
```

##### historical-info

Lệnh `historical-info` cho phép người dùng truy vấn thông tin lịch sử tại height cụ thể.

Usage:

```bash
simd query staking historical-info [height] [flags]
```

Ví dụ:

```bash
simd query staking historical-info 10
```

Ví dụ đầu ra:

```bash
header:
  app_hash: Lbx8cXpI868wz8sgp4qPYVrlaKjevR5WP/IjUxwp3oo=
  chain_id: testnet
  consensus_hash: BICRvH3cKD93v7+R1zxE2ljD34qcvIZ0Bdi389qtoi8=
  data_hash: 47DEQpj8HBSa+/TImW+5JCeuQeRkm5NMpJWZG3hSuFU=
  evidence_hash: 47DEQpj8HBSa+/TImW+5JCeuQeRkm5NMpJWZG3hSuFU=
  height: "10"
  last_block_id:
    hash: RFbkpu6pWfSThXxKKl6EZVDnBSm16+U0l0xVjTX08Fk=
    part_set_header:
      hash: vpIvXD4rxD5GM4MXGz0Sad9I7//iVYLzZsEU4BVgWIU=
      total: 1
  last_commit_hash: Ne4uXyx4QtNp4Zx89kf9UK7oG9QVbdB6e7ZwZkhy8K0=
  last_results_hash: 47DEQpj8HBSa+/TImW+5JCeuQeRkm5NMpJWZG3hSuFU=
  next_validators_hash: nGBgKeWBjoxeKFti00CxHsnULORgKY4LiuQwBuUrhCs=
  proposer_address: mMEP2c2IRPLr99LedSRtBg9eONM=
  time: "2021-10-01T06:00:49.785790894Z"
  validators_hash: nGBgKeWBjoxeKFti00CxHsnULORgKY4LiuQwBuUrhCs=
  version:
    app: "0"
    block: "11"
valset:
- commission:
    commission_rates:
      max_change_rate: "0.010000000000000000"
      max_rate: "0.200000000000000000"
      rate: "0.100000000000000000"
    update_time: "2021-10-01T05:52:50.380144238Z"
  consensus_pubkey:
    '@type': /cosmos.crypto.ed25519.PubKey
    key: Auxs3865HpB/EfssYOzfqNhEJjzys2Fo6jD5B8tPgC8=
  delegator_shares: "10000000.000000000000000000"
  description:
    details: ""
    identity: ""
    moniker: myvalidator
    security_contact: ""
    website: ""
  jailed: false
  min_self_delegation: "1"
  operator_address: cosmosvaloper1rne8lgs98p0jqe82sgt0qr4rdn4hgvmgp9ggcc
  status: BOND_STATUS_BONDED
  tokens: "10000000"
  unbonding_height: "0"
  unbonding_time: "1970-01-01T00:00:00Z"
```

##### params

Lệnh `params` cho phép người dùng truy vấn các giá trị được đặt làm tham số staking.

Usage:

```bash
simd query staking params [flags]
```

Ví dụ:

```bash
simd query staking params
```

Ví dụ đầu ra:

```bash
bond_denom: stake
historical_entries: 10000
max_entries: 7
max_validators: 50
unbonding_time: 1814400s
```

##### pool

Lệnh `pool` cho phép người dùng truy vấn các giá trị cho số lượng được lưu trong staking pool.

Usage:

```bash
simd q staking pool [flags]
```

Ví dụ:

```bash
simd q staking pool
```

Ví dụ đầu ra:

```bash
bonded_tokens: "10000000"
not_bonded_tokens: "0"
```

##### redelegation

Lệnh `redelegation` cho phép người dùng truy vấn bản ghi redelegation dựa trên delegator và địa chỉ validator nguồn và đích.

Usage:

```bash
simd query staking redelegation [delegator-addr] [src-validator-addr] [dst-validator-addr] [flags]
```

Ví dụ:

```bash
simd query staking redelegation cosmos1gghjut3ccd8ay0zduzj64hwre2fxs9ld75ru9p cosmosvaloper1l2rsakp388kuv9k8qzq6lrm9taddae7fpx59wm cosmosvaloper1gghjut3ccd8ay0zduzj64hwre2fxs9ldmqhffj
```

Ví dụ đầu ra:

```bash
pagination: null
redelegation_responses:
- entries:
  - balance: "50000000"
    redelegation_entry:
      completion_time: "2021-10-24T20:33:21.960084845Z"
      creation_height: 2.382847e+06
      initial_balance: "50000000"
      shares_dst: "50000000.000000000000000000"
  - balance: "5000000000"
    redelegation_entry:
      completion_time: "2021-10-25T21:33:54.446846862Z"
      creation_height: 2.397271e+06
      initial_balance: "5000000000"
      shares_dst: "5000000000.000000000000000000"
  redelegation:
    delegator_address: cosmos1gghjut3ccd8ay0zduzj64hwre2fxs9ld75ru9p
    entries: null
    validator_dst_address: cosmosvaloper1l2rsakp388kuv9k8qzq6lrm9taddae7fpx59wm
    validator_src_address: cosmosvaloper1l2rsakp388kuv9k8qzq6lrm9taddae7fpx59wm
```

##### redelegations

Lệnh `redelegations` cho phép người dùng truy vấn tất cả bản ghi redelegation cho một delegator nhất định.

Usage:

```bash
simd query staking redelegations [delegator-addr] [flags]
```

Ví dụ:

```bash
simd query staking redelegation cosmos1gghjut3ccd8ay0zduzj64hwre2fxs9ld75ru9p
```

Ví dụ đầu ra:

```bash
pagination:
  next_key: null
  total: "0"
redelegation_responses:
- entries:
  - balance: "50000000"
    redelegation_entry:
      completion_time: "2021-10-24T20:33:21.960084845Z"
      creation_height: 2.382847e+06
      initial_balance: "50000000"
      shares_dst: "50000000.000000000000000000"
  - balance: "5000000000"
    redelegation_entry:
      completion_time: "2021-10-25T21:33:54.446846862Z"
      creation_height: 2.397271e+06
      initial_balance: "5000000000"
      shares_dst: "5000000000.000000000000000000"
  redelegation:
    delegator_address: cosmos1gghjut3ccd8ay0zduzj64hwre2fxs9ld75ru9p
    entries: null
    validator_dst_address: cosmosvaloper1uccl5ugxrm7vqlzwqr04pjd320d2fz0z3hc6vm
    validator_src_address: cosmosvaloper1zppjyal5emta5cquje8ndkpz0rs046m7zqxrpp
- entries:
  - balance: "562770000000"
    redelegation_entry:
      completion_time: "2021-10-25T21:42:07.336911677Z"
      creation_height: 2.39735e+06
      initial_balance: "562770000000"
      shares_dst: "562770000000.000000000000000000"
  redelegation:
    delegator_address: cosmos1gghjut3ccd8ay0zduzj64hwre2fxs9ld75ru9p
    entries: null
    validator_dst_address: cosmosvaloper1uccl5ugxrm7vqlzwqr04pjd320d2fz0z3hc6vm
    validator_src_address: cosmosvaloper1zppjyal5emta5cquje8ndkpz0rs046m7zqxrpp
```

##### redelegations-from

Lệnh `redelegations-from` cho phép người dùng truy vấn các delegations đang redelegate _từ_ một validator.

Usage:

```bash
simd query staking redelegations-from [validator-addr] [flags]
```

Ví dụ:

```bash
simd query staking redelegations-from cosmosvaloper1y4rzzrgl66eyhzt6gse2k7ej3zgwmngeleucjy
```

Ví dụ đầu ra:

```bash
pagination:
  next_key: null
  total: "0"
redelegation_responses:
- entries:
  - balance: "50000000"
    redelegation_entry:
      completion_time: "2021-10-24T20:33:21.960084845Z"
      creation_height: 2.382847e+06
      initial_balance: "50000000"
      shares_dst: "50000000.000000000000000000"
  - balance: "5000000000"
    redelegation_entry:
      completion_time: "2021-10-25T21:33:54.446846862Z"
      creation_height: 2.397271e+06
      initial_balance: "5000000000"
      shares_dst: "5000000000.000000000000000000"
  redelegation:
    delegator_address: cosmos1pm6e78p4pgn0da365plzl4t56pxy8hwtqp2mph
    entries: null
    validator_dst_address: cosmosvaloper1uccl5ugxrm7vqlzwqr04pjd320d2fz0z3hc6vm
    validator_src_address: cosmosvaloper1y4rzzrgl66eyhzt6gse2k7ej3zgwmngeleucjy
- entries:
  - balance: "221000000"
    redelegation_entry:
      completion_time: "2021-10-05T21:05:45.669420544Z"
      creation_height: 2.120693e+06
      initial_balance: "221000000"
      shares_dst: "221000000.000000000000000000"
  redelegation:
    delegator_address: cosmos1zqv8qxy2zgn4c58fz8jt8jmhs3d0attcussrf6
    entries: null
    validator_dst_address: cosmosvaloper10mseqwnwtjaqfrwwp2nyrruwmjp6u5jhah4c3y
    validator_src_address: cosmosvaloper1y4rzzrgl66eyhzt6gse2k7ej3zgwmngeleucjy
```

##### unbonding-delegation

Lệnh `unbonding-delegation` cho phép người dùng truy vấn unbonding delegations cho một delegator nhất định trên một validator nhất định.

Usage:

```bash
simd query staking unbonding-delegation [delegator-addr] [validator-addr] [flags]
```

Ví dụ:

```bash
simd query staking unbonding-delegation cosmos1gghjut3ccd8ay0zduzj64hwre2fxs9ld75ru9p cosmosvaloper1gghjut3ccd8ay0zduzj64hwre2fxs9ldmqhffj
```

Ví dụ đầu ra:

```bash
delegator_address: cosmos1gghjut3ccd8ay0zduzj64hwre2fxs9ld75ru9p
entries:
- balance: "52000000"
  completion_time: "2021-11-02T11:35:55.391594709Z"
  creation_height: "55078"
  initial_balance: "52000000"
validator_address: cosmosvaloper1gghjut3ccd8ay0zduzj64hwre2fxs9ldmqhffj
```

##### unbonding-delegations

Lệnh `unbonding-delegations` cho phép người dùng truy vấn tất cả bản ghi unbonding-delegations cho một delegator.

Usage:

```bash
simd query staking unbonding-delegations [delegator-addr] [flags]
```

Ví dụ:

```bash
simd query staking unbonding-delegations cosmos1gghjut3ccd8ay0zduzj64hwre2fxs9ld75ru9p
```

Ví dụ đầu ra:

```bash
pagination:
  next_key: null
  total: "0"
unbonding_responses:
- delegator_address: cosmos1gghjut3ccd8ay0zduzj64hwre2fxs9ld75ru9p
  entries:
  - balance: "52000000"
    completion_time: "2021-11-02T11:35:55.391594709Z"
    creation_height: "55078"
    initial_balance: "52000000"
  validator_address: cosmosvaloper1t8ehvswxjfn3ejzkjtntcyrqwvmvuknzmvtaaa

```

##### unbonding-delegations-from

Lệnh `unbonding-delegations-from` cho phép người dùng truy vấn các delegations đang unbonding _từ_ một validator.

Usage:

```bash
simd query staking unbonding-delegations-from [validator-addr] [flags]
```

Ví dụ:

```bash
simd query staking unbonding-delegations-from cosmosvaloper1gghjut3ccd8ay0zduzj64hwre2fxs9ldmqhffj
```

Ví dụ đầu ra:

```bash
pagination:
  next_key: null
  total: "0"
unbonding_responses:
- delegator_address: cosmos1qqq9txnw4c77sdvzx0tkedsafl5s3vk7hn53fn
  entries:
  - balance: "150000000"
    completion_time: "2021-11-01T21:41:13.098141574Z"
    creation_height: "46823"
    initial_balance: "150000000"
  validator_address: cosmosvaloper1gghjut3ccd8ay0zduzj64hwre2fxs9ldmqhffj
- delegator_address: cosmos1peteje73eklqau66mr7h7rmewmt2vt99y24f5z
  entries:
  - balance: "24000000"
    completion_time: "2021-10-31T02:57:18.192280361Z"
    creation_height: "21516"
    initial_balance: "24000000"
  validator_address: cosmosvaloper1gghjut3ccd8ay0zduzj64hwre2fxs9ldmqhffj
```

##### validator

Lệnh `validator` cho phép người dùng truy vấn chi tiết về một validator nhất định.

Usage:

```bash
simd query staking validator [validator-addr] [flags]
```

Ví dụ:

```bash
simd query staking validator cosmosvaloper1gghjut3ccd8ay0zduzj64hwre2fxs9ldmqhffj
```

Ví dụ đầu ra:

```bash
commission:
  commission_rates:
    max_change_rate: "0.020000000000000000"
    max_rate: "0.200000000000000000"
    rate: "0.050000000000000000"
  update_time: "2021-10-01T19:24:52.663191049Z"
consensus_pubkey:
  '@type': /cosmos.crypto.ed25519.PubKey
  key: sIiexdJdYWn27+7iUHQJDnkp63gq/rzUq1Y+fxoGjXc=
delegator_shares: "32948270000.000000000000000000"
description:
  details: Witval is the validator arm from Vitwit. Vitwit is into software consulting
    and services business since 2015. We are working closely with Cosmos ecosystem
    since 2018. We are also building tools for the ecosystem, Aneka is our explorer
    for the cosmos ecosystem.
  identity: 51468B615127273A
  moniker: Witval
  security_contact: ""
  website: ""
jailed: false
min_self_delegation: "1"
operator_address: cosmosvaloper1gghjut3ccd8ay0zduzj64hwre2fxs9ldmqhffj
status: BOND_STATUS_BONDED
tokens: "32948270000"
unbonding_height: "0"
unbonding_time: "1970-01-01T00:00:00Z"
```

##### validators

Lệnh `validators` cho phép người dùng truy vấn chi tiết về tất cả validators trên mạng.

Usage:

```bash
simd query staking validators [flags]
```

Ví dụ:

```bash
simd query staking validators
```

Ví dụ đầu ra:

```bash
pagination:
  next_key: FPTi7TKAjN63QqZh+BaXn6gBmD5/
  total: "0"
validators:
commission:
  commission_rates:
    max_change_rate: "0.020000000000000000"
    max_rate: "0.200000000000000000"
    rate: "0.050000000000000000"
  update_time: "2021-10-01T19:24:52.663191049Z"
consensus_pubkey:
  '@type': /cosmos.crypto.ed25519.PubKey
  key: sIiexdJdYWn27+7iUHQJDnkp63gq/rzUq1Y+fxoGjXc=
delegator_shares: "32948270000.000000000000000000"
description:
    details: Witval is the validator arm from Vitwit. Vitwit is into software consulting
      and services business since 2015. We are working closely with Cosmos ecosystem
      since 2018. We are also building tools for the ecosystem, Aneka is our explorer
      for the cosmos ecosystem.
    identity: 51468B615127273A
    moniker: Witval
    security_contact: ""
    website: ""
  jailed: false
  min_self_delegation: "1"
  operator_address: cosmosvaloper1gghjut3ccd8ay0zduzj64hwre2fxs9ldmqhffj
  status: BOND_STATUS_BONDED
  tokens: "32948270000"
  unbonding_height: "0"
  unbonding_time: "1970-01-01T00:00:00Z"
- commission:
    commission_rates:
      max_change_rate: "0.100000000000000000"
      max_rate: "0.200000000000000000"
      rate: "0.050000000000000000"
    update_time: "2021-10-04T18:02:21.446645619Z"
  consensus_pubkey:
    '@type': /cosmos.crypto.ed25519.PubKey
    key: GDNpuKDmCg9GnhnsiU4fCWktuGUemjNfvpCZiqoRIYA=
  delegator_shares: "559343421.000000000000000000"
  description:
    details: Noderunners is a professional validator in POS networks. We have a huge
      node running experience, reliable soft and hardware. Our commissions are always
      low, our support to delegators is always full. Stake with us and start receiving
      your Cosmos rewards now!
    identity: 812E82D12FEA3493
    moniker: Noderunners
    security_contact: info@noderunners.biz
    website: http://noderunners.biz
  jailed: false
  min_self_delegation: "1"
  operator_address: cosmosvaloper1q5ku90atkhktze83j9xjaks2p7uruag5zp6wt7
  status: BOND_STATUS_BONDED
  tokens: "559343421"
  unbonding_height: "0"
  unbonding_time: "1970-01-01T00:00:00Z"
```

#### Transactions

Các lệnh `tx` cho phép người dùng tương tác với module `staking`.

```bash
simd tx staking --help
```

##### create-validator

Lệnh `create-validator` cho phép người dùng tạo validator mới được khởi tạo với self-delegation cho nó.

Usage:

```bash
simd tx staking create-validator [path/to/validator.json] [flags]
```

Ví dụ:

```bash
simd tx staking create-validator /path/to/validator.json \
  --chain-id="name_of_chain_id" \
  --gas="auto" \
  --gas-adjustment="1.2" \
  --gas-prices="0.025stake" \
  --from=mykey
```

trong đó `validator.json` chứa:

```json
{
  "pubkey": {"@type":"/cosmos.crypto.ed25519.PubKey","key":"BnbwFpeONLqvWqJb3qaUbL5aoIcW3fSuAp9nT3z5f20="},
  "amount": "1000000stake",
  "moniker": "my-moniker",
  "website": "https://myweb.site",
  "security": "security-contact@gmail.com",
  "details": "description of your validator",
  "commission-rate": "0.10",
  "commission-max-rate": "0.20",
  "commission-max-change-rate": "0.01",
  "min-self-delegation": "1"
}
```

và pubkey có thể lấy bằng lệnh `simd tendermint show-validator`.

##### delegate

Lệnh `delegate` cho phép người dùng delegate liquid token cho validator.

Usage:

```bash
simd tx staking delegate [validator-addr] [amount] [flags]
```

Ví dụ:

```bash
simd tx staking delegate cosmosvaloper1l2rsakp388kuv9k8qzq6lrm9taddae7fpx59wm 1000stake --from mykey
```

##### edit-validator

Lệnh `edit-validator` cho phép người dùng chỉnh sửa tài khoản validator hiện có.

Usage:

```bash
simd tx staking edit-validator [flags]
```

Ví dụ:

```bash
simd tx staking edit-validator --moniker "new_moniker_name" --website "new_website_url" --from mykey
```

##### redelegate

Lệnh `redelegate` cho phép người dùng redelegate illiquid token từ validator này sang validator khác.

Usage:

```bash
simd tx staking redelegate [src-validator-addr] [dst-validator-addr] [amount] [flags]
```

Ví dụ:

```bash
simd tx staking redelegate cosmosvaloper1gghjut3ccd8ay0zduzj64hwre2fxs9ldmqhffj cosmosvaloper1l2rsakp388kuv9k8qzq6lrm9taddae7fpx59wm 100stake --from mykey
```

##### unbond

Lệnh `unbond` cho phép người dùng unbond shares từ validator.

Usage:

```bash
simd tx staking unbond [validator-addr] [amount] [flags]
```

Ví dụ:

```bash
simd tx staking unbond cosmosvaloper1gghjut3ccd8ay0zduzj64hwre2fxs9ldmqhffj 100stake --from mykey
```

##### cancel unbond

Lệnh `cancel-unbond` cho phép người dùng hủy entry unbonding delegation và delegate trở lại validator gốc.

Usage:

```bash
simd tx staking cancel-unbond [validator-addr] [amount] [creation-height]
```

Ví dụ:

```bash
simd tx staking cancel-unbond cosmosvaloper1gghjut3ccd8ay0zduzj64hwre2fxs9ldmqhffj 100stake 123123 --from mykey
```


### gRPC

Người dùng có thể truy vấn module `staking` bằng các endpoint gRPC.

#### Validators

Endpoint `Validators` truy vấn tất cả validators khớp với trạng thái đã cho.

```bash
cosmos.staking.v1beta1.Query/Validators
```

Ví dụ:

```bash
grpcurl -plaintext localhost:9090 cosmos.staking.v1beta1.Query/Validators
```

Ví dụ đầu ra:

```bash
{
  "validators": [
    {
      "operatorAddress": "cosmosvaloper1rne8lgs98p0jqe82sgt0qr4rdn4hgvmgp9ggcc",
      "consensusPubkey": {"@type":"/cosmos.crypto.ed25519.PubKey","key":"Auxs3865HpB/EfssYOzfqNhEJjzys2Fo6jD5B8tPgC8="},
      "status": "BOND_STATUS_BONDED",
      "tokens": "10000000",
      "delegatorShares": "10000000000000000000000000",
      "description": {
        "moniker": "myvalidator"
      },
      "unbondingTime": "1970-01-01T00:00:00Z",
      "commission": {
        "commissionRates": {
          "rate": "100000000000000000",
          "maxRate": "200000000000000000",
          "maxChangeRate": "10000000000000000"
        },
        "updateTime": "2021-10-01T05:52:50.380144238Z"
      },
      "minSelfDelegation": "1"
    }
  ],
  "pagination": {
    "total": "1"
  }
}
```

#### Validator

Endpoint `Validator` truy vấn thông tin validator cho địa chỉ validator đã cho.

```bash
cosmos.staking.v1beta1.Query/Validator
```

Ví dụ:

```bash
grpcurl -plaintext -d '{"validator_addr":"cosmosvaloper1rne8lgs98p0jqe82sgt0qr4rdn4hgvmgp9ggcc"}' \
localhost:9090 cosmos.staking.v1beta1.Query/Validator
```

Ví dụ đầu ra:

```bash
{
  "validator": {
    "operatorAddress": "cosmosvaloper1rne8lgs98p0jqe82sgt0qr4rdn4hgvmgp9ggcc",
    "consensusPubkey": {"@type":"/cosmos.crypto.ed25519.PubKey","key":"Auxs3865HpB/EfssYOzfqNhEJjzys2Fo6jD5B8tPgC8="},
    "status": "BOND_STATUS_BONDED",
    "tokens": "10000000",
    "delegatorShares": "10000000000000000000000000",
    "description": {
      "moniker": "myvalidator"
    },
    "unbondingTime": "1970-01-01T00:00:00Z",
    "commission": {
      "commissionRates": {
        "rate": "100000000000000000",
        "maxRate": "200000000000000000",
        "maxChangeRate": "10000000000000000"
      },
      "updateTime": "2021-10-01T05:52:50.380144238Z"
    },
    "minSelfDelegation": "1"
  }
}
```

#### ValidatorDelegations

Endpoint `ValidatorDelegations` truy vấn thông tin delegate cho validator đã cho.

```bash
cosmos.staking.v1beta1.Query/ValidatorDelegations
```

Ví dụ:

```bash
grpcurl -plaintext -d '{"validator_addr":"cosmosvaloper1rne8lgs98p0jqe82sgt0qr4rdn4hgvmgp9ggcc"}' \
localhost:9090 cosmos.staking.v1beta1.Query/ValidatorDelegations
```

Ví dụ đầu ra:

```bash
{
  "delegationResponses": [
    {
      "delegation": {
        "delegatorAddress": "cosmos1rne8lgs98p0jqe82sgt0qr4rdn4hgvmgy3ua5t",
        "validatorAddress": "cosmosvaloper1rne8lgs98p0jqe82sgt0qr4rdn4hgvmgp9ggcc",
        "shares": "10000000000000000000000000"
      },
      "balance": {
        "denom": "stake",
        "amount": "10000000"
      }
    }
  ],
  "pagination": {
    "total": "1"
  }
}
```

#### ValidatorUnbondingDelegations

Endpoint `ValidatorUnbondingDelegations` truy vấn thông tin delegate cho validator đã cho.

```bash
cosmos.staking.v1beta1.Query/ValidatorUnbondingDelegations
```

Ví dụ:

```bash
grpcurl -plaintext -d '{"validator_addr":"cosmosvaloper1rne8lgs98p0jqe82sgt0qr4rdn4hgvmgp9ggcc"}' \
localhost:9090 cosmos.staking.v1beta1.Query/ValidatorUnbondingDelegations
```

Ví dụ đầu ra:

```bash
{
  "unbonding_responses": [
    {
      "delegator_address": "cosmos1z3pzzw84d6xn00pw9dy3yapqypfde7vg6965fy",
      "validator_address": "cosmosvaloper1rne8lgs98p0jqe82sgt0qr4rdn4hgvmgp9ggcc",
      "entries": [
        {
          "creation_height": "25325",
          "completion_time": "2021-10-31T09:24:36.797320636Z",
          "initial_balance": "20000000",
          "balance": "20000000"
        }
      ]
    },
    {
      "delegator_address": "cosmos1y8nyfvmqh50p6ldpzljk3yrglppdv3t8phju77",
      "validator_address": "cosmosvaloper1rne8lgs98p0jqe82sgt0qr4rdn4hgvmgp9ggcc",
      "entries": [
        {
          "creation_height": "13100",
          "completion_time": "2021-10-30T12:53:02.272266791Z",
          "initial_balance": "1000000",
          "balance": "1000000"
        }
      ]
    },
  ],
  "pagination": {
    "next_key": null,
    "total": "8"
  }
}
```

#### Delegation

Endpoint `Delegation` truy vấn thông tin delegate cho cặp validator delegator đã cho.

```bash
cosmos.staking.v1beta1.Query/Delegation
```

Ví dụ:

```bash
grpcurl -plaintext \
-d '{"delegator_addr": "cosmos1y8nyfvmqh50p6ldpzljk3yrglppdv3t8phju77", validator_addr":"cosmosvaloper1rne8lgs98p0jqe82sgt0qr4rdn4hgvmgp9ggcc"}' \
localhost:9090 cosmos.staking.v1beta1.Query/Delegation
```

Ví dụ đầu ra:

```bash
{
  "delegation_response":
  {
    "delegation":
      {
        "delegator_address":"cosmos1y8nyfvmqh50p6ldpzljk3yrglppdv3t8phju77",
        "validator_address":"cosmosvaloper1rne8lgs98p0jqe82sgt0qr4rdn4hgvmgp9ggcc",
        "shares":"25083119936.000000000000000000"
      },
    "balance":
      {
        "denom":"stake",
        "amount":"25083119936"
      }
  }
}
```

#### UnbondingDelegation

Endpoint `UnbondingDelegation` truy vấn thông tin unbonding cho validator delegator đã cho.

```bash
cosmos.staking.v1beta1.Query/UnbondingDelegation
```

Ví dụ:

```bash
grpcurl -plaintext \
-d '{"delegator_addr": "cosmos1y8nyfvmqh50p6ldpzljk3yrglppdv3t8phju77", validator_addr":"cosmosvaloper1rne8lgs98p0jqe82sgt0qr4rdn4hgvmgp9ggcc"}' \
localhost:9090 cosmos.staking.v1beta1.Query/UnbondingDelegation
```

Ví dụ đầu ra:

```bash
{
  "unbond": {
    "delegator_address": "cosmos1y8nyfvmqh50p6ldpzljk3yrglppdv3t8phju77",
    "validator_address": "cosmosvaloper1rne8lgs98p0jqe82sgt0qr4rdn4hgvmgp9ggcc",
    "entries": [
      {
        "creation_height": "136984",
        "completion_time": "2021-11-08T05:38:47.505593891Z",
        "initial_balance": "400000000",
        "balance": "400000000"
      },
      {
        "creation_height": "137005",
        "completion_time": "2021-11-08T05:40:53.526196312Z",
        "initial_balance": "385000000",
        "balance": "385000000"
      }
    ]
  }
}
```

#### DelegatorDelegations

Endpoint `DelegatorDelegations` truy vấn tất cả delegations của địa chỉ delegator đã cho.

```bash
cosmos.staking.v1beta1.Query/DelegatorDelegations
```

Ví dụ:

```bash
grpcurl -plaintext \
-d '{"delegator_addr": "cosmos1y8nyfvmqh50p6ldpzljk3yrglppdv3t8phju77"}' \
localhost:9090 cosmos.staking.v1beta1.Query/DelegatorDelegations
```

Ví dụ đầu ra:

```bash
{
  "delegation_responses": [
    {"delegation":{"delegator_address":"cosmos1y8nyfvmqh50p6ldpzljk3yrglppdv3t8phju77","validator_address":"cosmosvaloper1eh5mwu044gd5ntkkc2xgfg8247mgc56fww3vc8","shares":"25083339023.000000000000000000"},"balance":{"denom":"stake","amount":"25083339023"}}
  ],
  "pagination": {
    "next_key": null,
    "total": "1"
  }
}
```

#### DelegatorUnbondingDelegations

Endpoint `DelegatorUnbondingDelegations` truy vấn tất cả unbonding delegations của địa chỉ delegator đã cho.

```bash
cosmos.staking.v1beta1.Query/DelegatorUnbondingDelegations
```

Ví dụ:

```bash
grpcurl -plaintext \
-d '{"delegator_addr": "cosmos1y8nyfvmqh50p6ldpzljk3yrglppdv3t8phju77"}' \
localhost:9090 cosmos.staking.v1beta1.Query/DelegatorUnbondingDelegations
```

Ví dụ đầu ra:

```bash
{
  "unbonding_responses": [
    {
      "delegator_address": "cosmos1y8nyfvmqh50p6ldpzljk3yrglppdv3t8phju77",
      "validator_address": "cosmosvaloper1sjllsnramtg3ewxqwwrwjxfgc4n4ef9uxyejze",
      "entries": [
        {
          "creation_height": "136984",
          "completion_time": "2021-11-08T05:38:47.505593891Z",
          "initial_balance": "400000000",
          "balance": "400000000"
        },
        {
          "creation_height": "137005",
          "completion_time": "2021-11-08T05:40:53.526196312Z",
          "initial_balance": "385000000",
          "balance": "385000000"
        }
      ]
    }
  ],
  "pagination": {
    "next_key": null,
    "total": "1"
  }
}
```

#### Redelegations

Endpoint `Redelegations` truy vấn redelegations của địa chỉ đã cho.

```bash
cosmos.staking.v1beta1.Query/Redelegations
```

Ví dụ:

```bash
grpcurl -plaintext \
-d '{"delegator_addr": "cosmos1ld5p7hn43yuh8ht28gm9pfjgj2fctujp2tgwvf", "src_validator_addr" : "cosmosvaloper1j7euyj85fv2jugejrktj540emh9353ltgppc3g", "dst_validator_addr" : "cosmosvaloper1yy3tnegzmkdcm7czzcy3flw5z0zyr9vkkxrfse"}' \
localhost:9090 cosmos.staking.v1beta1.Query/Redelegations
```

Ví dụ đầu ra:

```bash
{
  "redelegation_responses": [
    {
      "redelegation": {
        "delegator_address": "cosmos1ld5p7hn43yuh8ht28gm9pfjgj2fctujp2tgwvf",
        "validator_src_address": "cosmosvaloper1j7euyj85fv2jugejrktj540emh9353ltgppc3g",
        "validator_dst_address": "cosmosvaloper1yy3tnegzmkdcm7czzcy3flw5z0zyr9vkkxrfse",
        "entries": null
      },
      "entries": [
        {
          "redelegation_entry": {
            "creation_height": 135932,
            "completion_time": "2021-11-08T03:52:55.299147901Z",
            "initial_balance": "2900000",
            "shares_dst": "2900000.000000000000000000"
          },
          "balance": "2900000"
        }
      ]
    }
  ],
  "pagination": null
}
```

#### DelegatorValidators

Endpoint `DelegatorValidators` truy vấn tất cả thông tin validators cho delegator đã cho.

```bash
cosmos.staking.v1beta1.Query/DelegatorValidators
```

Ví dụ:

```bash
grpcurl -plaintext \
-d '{"delegator_addr": "cosmos1ld5p7hn43yuh8ht28gm9pfjgj2fctujp2tgwvf"}' \
localhost:9090 cosmos.staking.v1beta1.Query/DelegatorValidators
```

Ví dụ đầu ra:

```bash
{
  "validators": [
    {
      "operator_address": "cosmosvaloper1eh5mwu044gd5ntkkc2xgfg8247mgc56fww3vc8",
      "consensus_pubkey": {
        "@type": "/cosmos.crypto.ed25519.PubKey",
        "key": "UPwHWxH1zHJWGOa/m6JB3f5YjHMvPQPkVbDqqi+U7Uw="
      },
      "jailed": false,
      "status": "BOND_STATUS_BONDED",
      "tokens": "347260647559",
      "delegator_shares": "347260647559.000000000000000000",
      "description": {
        "moniker": "BouBouNode",
        "identity": "",
        "website": "https://boubounode.com",
        "security_contact": "",
        "details": "AI-based Validator. #1 AI Validator on Game of Stakes. Fairly priced. Don't trust (humans), verify. Made with BouBou love."
      },
      "unbonding_height": "0",
      "unbonding_time": "1970-01-01T00:00:00Z",
      "commission": {
        "commission_rates": {
          "rate": "0.061000000000000000",
          "max_rate": "0.300000000000000000",
          "max_change_rate": "0.150000000000000000"
        },
        "update_time": "2021-10-01T15:00:00Z"
      },
      "min_self_delegation": "1"
    }
  ],
  "pagination": {
    "next_key": null,
    "total": "1"
  }
}
```

#### DelegatorValidator

Endpoint `DelegatorValidator` truy vấn thông tin validator cho delegator validator đã cho

```bash
cosmos.staking.v1beta1.Query/DelegatorValidator
```

Ví dụ:

```bash
grpcurl -plaintext \
-d '{"delegator_addr": "cosmos1eh5mwu044gd5ntkkc2xgfg8247mgc56f3n8rr7", "validator_addr": "cosmosvaloper1eh5mwu044gd5ntkkc2xgfg8247mgc56fww3vc8"}' \
localhost:9090 cosmos.staking.v1beta1.Query/DelegatorValidator
```

Ví dụ đầu ra:

```bash
{
  "validator": {
    "operator_address": "cosmosvaloper1eh5mwu044gd5ntkkc2xgfg8247mgc56fww3vc8",
    "consensus_pubkey": {
      "@type": "/cosmos.crypto.ed25519.PubKey",
      "key": "UPwHWxH1zHJWGOa/m6JB3f5YjHMvPQPkVbDqqi+U7Uw="
    },
    "jailed": false,
    "status": "BOND_STATUS_BONDED",
    "tokens": "347262754841",
    "delegator_shares": "347262754841.000000000000000000",
    "description": {
      "moniker": "BouBouNode",
      "identity": "",
      "website": "https://boubounode.com",
      "security_contact": "",
      "details": "AI-based Validator. #1 AI Validator on Game of Stakes. Fairly priced. Don't trust (humans), verify. Made with BouBou love."
    },
    "unbonding_height": "0",
    "unbonding_time": "1970-01-01T00:00:00Z",
    "commission": {
      "commission_rates": {
        "rate": "0.061000000000000000",
        "max_rate": "0.300000000000000000",
        "max_change_rate": "0.150000000000000000"
      },
      "update_time": "2021-10-01T15:00:00Z"
    },
    "min_self_delegation": "1"
  }
}
```

#### HistoricalInfo

```bash
cosmos.staking.v1beta1.Query/HistoricalInfo
```

Ví dụ:

```bash
grpcurl -plaintext -d '{"height" : 1}' localhost:9090 cosmos.staking.v1beta1.Query/HistoricalInfo
```

Ví dụ đầu ra:

```bash
{
  "hist": {
    "header": {
      "version": {
        "block": "11",
        "app": "0"
      },
      "chain_id": "simd-1",
      "height": "140142",
      "time": "2021-10-11T10:56:29.720079569Z",
      "last_block_id": {
        "hash": "9gri/4LLJUBFqioQ3NzZIP9/7YHR9QqaM6B2aJNQA7o=",
        "part_set_header": {
          "total": 1,
          "hash": "Hk1+C864uQkl9+I6Zn7IurBZBKUevqlVtU7VqaZl1tc="
        }
      },
      "last_commit_hash": "VxrcS27GtvGruS3I9+AlpT7udxIT1F0OrRklrVFSSKc=",
      "data_hash": "80BjOrqNYUOkTnmgWyz9AQ8n7SoEmPVi4QmAe8RbQBY=",
      "validators_hash": "95W49n2hw8RWpr1GPTAO5MSPi6w6Wjr3JjjS7AjpBho=",
      "next_validators_hash": "95W49n2hw8RWpr1GPTAO5MSPi6w6Wjr3JjjS7AjpBho=",
      "consensus_hash": "BICRvH3cKD93v7+R1zxE2ljD34qcvIZ0Bdi389qtoi8=",
      "app_hash": "ZZaxnSY3E6Ex5Bvkm+RigYCK82g8SSUL53NymPITeOE=",
      "last_results_hash": "47DEQpj8HBSa+/TImW+5JCeuQeRkm5NMpJWZG3hSuFU=",
      "evidence_hash": "47DEQpj8HBSa+/TImW+5JCeuQeRkm5NMpJWZG3hSuFU=",
      "proposer_address": "aH6dO428B+ItuoqPq70efFHrSMY="
    },
  "valset": [
      {
        "operator_address": "cosmosvaloper196ax4vc0lwpxndu9dyhvca7jhxp70rmcqcnylw",
        "consensus_pubkey": {
          "@type": "/cosmos.crypto.ed25519.PubKey",
          "key": "/O7BtNW0pafwfvomgR4ZnfldwPXiFfJs9mHg3gwfv5Q="
        },
        "jailed": false,
        "status": "BOND_STATUS_BONDED",
        "tokens": "1426045203613",
        "delegator_shares": "1426045203613.000000000000000000",
        "description": {
          "moniker": "SG-1",
          "identity": "48608633F99D1B60",
          "website": "https://sg-1.online",
          "security_contact": "",
          "details": "SG-1 - your favorite validator on Witval. We offer 100% Soft Slash protection."
        },
        "unbonding_height": "0",
        "unbonding_time": "1970-01-01T00:00:00Z",
        "commission": {
          "commission_rates": {
            "rate": "0.037500000000000000",
            "max_rate": "0.200000000000000000",
            "max_change_rate": "0.030000000000000000"
          },
          "update_time": "2021-10-01T15:00:00Z"
        },
        "min_self_delegation": "1"
      }
    ]
  }
}

```

#### Pool

Endpoint `Pool` truy vấn thông tin pool.

```bash
cosmos.staking.v1beta1.Query/Pool
```

Ví dụ:

```bash
grpcurl -plaintext -d localhost:9090 cosmos.staking.v1beta1.Query/Pool
```

Ví dụ đầu ra:

```bash
{
  "pool": {
    "not_bonded_tokens": "369054400189",
    "bonded_tokens": "15657192425623"
  }
}
```

#### Params

Endpoint `Params` truy vấn thông tin pool.

```bash
cosmos.staking.v1beta1.Query/Params
```

Ví dụ:

```bash
grpcurl -plaintext localhost:9090 cosmos.staking.v1beta1.Query/Params
```

Ví dụ đầu ra:

```bash
{
  "params": {
    "unbondingTime": "1814400s",
    "maxValidators": 100,
    "maxEntries": 7,
    "historicalEntries": 10000,
    "bondDenom": "stake"
  }
}
```

### REST

Người dùng có thể truy vấn module `staking` bằng các endpoint REST.

#### DelegatorDelegations

Endpoint REST `DelegatorDelegations` truy vấn tất cả delegations của địa chỉ delegator đã cho.

```bash
/cosmos/staking/v1beta1/delegations/{delegatorAddr}
```

Ví dụ:

```bash
curl -X GET "http://localhost:1317/cosmos/staking/v1beta1/delegations/cosmos1vcs68xf2tnqes5tg0khr0vyevm40ff6zdxatp5" -H  "accept: application/json"
```

Ví dụ đầu ra:

```bash
{
  "delegation_responses": [
    {
      "delegation": {
        "delegator_address": "cosmos1vcs68xf2tnqes5tg0khr0vyevm40ff6zdxatp5",
        "validator_address": "cosmosvaloper1quqxfrxkycr0uzt4yk0d57tcq3zk7srm7sm6r8",
        "shares": "256250000.000000000000000000"
      },
      "balance": {
        "denom": "stake",
        "amount": "256250000"
      }
    },
    {
      "delegation": {
        "delegator_address": "cosmos1vcs68xf2tnqes5tg0khr0vyevm40ff6zdxatp5",
        "validator_address": "cosmosvaloper194v8uwee2fvs2s8fa5k7j03ktwc87h5ym39jfv",
        "shares": "255150000.000000000000000000"
      },
      "balance": {
        "denom": "stake",
        "amount": "255150000"
      }
    }
  ],
  "pagination": {
    "next_key": null,
    "total": "2"
  }
}
```

#### Redelegations

Endpoint REST `Redelegations` truy vấn redelegations của địa chỉ đã cho.

```bash
/cosmos/staking/v1beta1/delegators/{delegatorAddr}/redelegations
```

Ví dụ:

```bash
curl -X GET \
"http://localhost:1317/cosmos/staking/v1beta1/delegators/cosmos1thfntksw0d35n2tkr0k8v54fr8wxtxwxl2c56e/redelegations?srcValidatorAddr=cosmosvaloper1lzhlnpahvznwfv4jmay2tgaha5kmz5qx4cuznf&dstValidatorAddr=cosmosvaloper1vq8tw77kp8lvxq9u3c8eeln9zymn68rng8pgt4" \
-H  "accept: application/json"
```

Ví dụ đầu ra:

```bash
{
  "redelegation_responses": [
    {
      "redelegation": {
        "delegator_address": "cosmos1thfntksw0d35n2tkr0k8v54fr8wxtxwxl2c56e",
        "validator_src_address": "cosmosvaloper1lzhlnpahvznwfv4jmay2tgaha5kmz5qx4cuznf",
        "validator_dst_address": "cosmosvaloper1vq8tw77kp8lvxq9u3c8eeln9zymn68rng8pgt4",
        "entries": null
      },
      "entries": [
        {
          "redelegation_entry": {
            "creation_height": 151523,
            "completion_time": "2021-11-09T06:03:25.640682116Z",
            "initial_balance": "200000000",
            "shares_dst": "200000000.000000000000000000"
          },
          "balance": "200000000"
        }
      ]
    }
  ],
  "pagination": null
}
```

#### DelegatorUnbondingDelegations

Endpoint REST `DelegatorUnbondingDelegations` truy vấn tất cả unbonding delegations của địa chỉ delegator đã cho.

```bash
/cosmos/staking/v1beta1/delegators/{delegatorAddr}/unbonding_delegations
```

Ví dụ:

```bash
curl -X GET \
"http://localhost:1317/cosmos/staking/v1beta1/delegators/cosmos1nxv42u3lv642q0fuzu2qmrku27zgut3n3z7lll/unbonding_delegations" \
-H  "accept: application/json"
```

Ví dụ đầu ra:

```bash
{
  "unbonding_responses": [
    {
      "delegator_address": "cosmos1nxv42u3lv642q0fuzu2qmrku27zgut3n3z7lll",
      "validator_address": "cosmosvaloper1e7mvqlz50ch6gw4yjfemsc069wfre4qwmw53kq",
      "entries": [
        {
          "creation_height": "2442278",
          "completion_time": "2021-10-12T10:59:03.797335857Z",
          "initial_balance": "50000000000",
          "balance": "50000000000"
        }
      ]
    }
  ],
  "pagination": {
    "next_key": null,
    "total": "1"
  }
}
```

#### DelegatorValidators

Endpoint REST `DelegatorValidators` truy vấn tất cả thông tin validators cho địa chỉ delegator đã cho.

```bash
/cosmos/staking/v1beta1/delegators/{delegatorAddr}/validators
```

Ví dụ:

```bash
curl -X GET \
"http://localhost:1317/cosmos/staking/v1beta1/delegators/cosmos1xwazl8ftks4gn00y5x3c47auquc62ssune9ppv/validators" \
-H  "accept: application/json"
```

Ví dụ đầu ra:

```bash
{
  "validators": [
    {
      "operator_address": "cosmosvaloper1xwazl8ftks4gn00y5x3c47auquc62ssuvynw64",
      "consensus_pubkey": {
        "@type": "/cosmos.crypto.ed25519.PubKey",
        "key": "5v4n3px3PkfNnKflSgepDnsMQR1hiNXnqOC11Y72/PQ="
      },
      "jailed": false,
      "status": "BOND_STATUS_BONDED",
      "tokens": "21592843799",
      "delegator_shares": "21592843799.000000000000000000",
      "description": {
        "moniker": "jabbey",
        "identity": "",
        "website": "https://twitter.com/JoeAbbey",
        "security_contact": "",
        "details": "just another dad in the cosmos"
      },
      "unbonding_height": "0",
      "unbonding_time": "1970-01-01T00:00:00Z",
      "commission": {
        "commission_rates": {
          "rate": "0.100000000000000000",
          "max_rate": "0.200000000000000000",
          "max_change_rate": "0.100000000000000000"
        },
        "update_time": "2021-10-09T19:03:54.984821705Z"
      },
      "min_self_delegation": "1"
    }
  ],
  "pagination": {
    "next_key": null,
    "total": "1"
  }
}
```

#### DelegatorValidator

Endpoint REST `DelegatorValidator` truy vấn thông tin validator cho cặp delegator validator đã cho.

```bash
/cosmos/staking/v1beta1/delegators/{delegatorAddr}/validators/{validatorAddr}
```

Ví dụ:

```bash
curl -X GET \
"http://localhost:1317/cosmos/staking/v1beta1/delegators/cosmos1xwazl8ftks4gn00y5x3c47auquc62ssune9ppv/validators/cosmosvaloper1xwazl8ftks4gn00y5x3c47auquc62ssuvynw64" \
-H  "accept: application/json"
```

Ví dụ đầu ra:

```bash
{
  "validator": {
    "operator_address": "cosmosvaloper1xwazl8ftks4gn00y5x3c47auquc62ssuvynw64",
    "consensus_pubkey": {
      "@type": "/cosmos.crypto.ed25519.PubKey",
      "key": "5v4n3px3PkfNnKflSgepDnsMQR1hiNXnqOC11Y72/PQ="
    },
    "jailed": false,
    "status": "BOND_STATUS_BONDED",
    "tokens": "21592843799",
    "delegator_shares": "21592843799.000000000000000000",
    "description": {
      "moniker": "jabbey",
      "identity": "",
      "website": "https://twitter.com/JoeAbbey",
      "security_contact": "",
      "details": "just another dad in the cosmos"
    },
    "unbonding_height": "0",
    "unbonding_time": "1970-01-01T00:00:00Z",
    "commission": {
      "commission_rates": {
        "rate": "0.100000000000000000",
        "max_rate": "0.200000000000000000",
        "max_change_rate": "0.100000000000000000"
      },
      "update_time": "2021-10-09T19:03:54.984821705Z"
    },
    "min_self_delegation": "1"
  }
}
```

#### HistoricalInfo

Endpoint REST `HistoricalInfo` truy vấn thông tin lịch sử cho height đã cho.

```bash
/cosmos/staking/v1beta1/historical_info/{height}
```

Ví dụ:

```bash
curl -X GET "http://localhost:1317/cosmos/staking/v1beta1/historical_info/153332" -H  "accept: application/json"
```

Ví dụ đầu ra:

```bash
{
  "hist": {
    "header": {
      "version": {
        "block": "11",
        "app": "0"
      },
      "chain_id": "cosmos-1",
      "height": "153332",
      "time": "2021-10-12T09:05:35.062230221Z",
      "last_block_id": {
        "hash": "NX8HevR5khb7H6NGKva+jVz7cyf0skF1CrcY9A0s+d8=",
        "part_set_header": {
          "total": 1,
          "hash": "zLQ2FiKM5tooL3BInt+VVfgzjlBXfq0Hc8Iux/xrhdg="
        }
      },
      "last_commit_hash": "P6IJrK8vSqU3dGEyRHnAFocoDGja0bn9euLuy09s350=",
      "data_hash": "eUd+6acHWrNXYju8Js449RJ99lOYOs16KpqQl4SMrEM=",
      "validators_hash": "mB4pravvMsJKgi+g8aYdSeNlt0kPjnRFyvtAQtaxcfw=",
      "next_validators_hash": "mB4pravvMsJKgi+g8aYdSeNlt0kPjnRFyvtAQtaxcfw=",
      "consensus_hash": "BICRvH3cKD93v7+R1zxE2ljD34qcvIZ0Bdi389qtoi8=",
      "app_hash": "fuELArKRK+CptnZ8tu54h6xEleSWenHNmqC84W866fU=",
      "last_results_hash": "p/BPexV4LxAzlVcPRvW+lomgXb6Yze8YLIQUo/4Kdgc=",
      "evidence_hash": "47DEQpj8HBSa+/TImW+5JCeuQeRkm5NMpJWZG3hSuFU=",
      "proposer_address": "G0MeY8xQx7ooOsni8KE/3R/Ib3Q="
    },
    "valset": [
      {
        "operator_address": "cosmosvaloper196ax4vc0lwpxndu9dyhvca7jhxp70rmcqcnylw",
        "consensus_pubkey": {
          "@type": "/cosmos.crypto.ed25519.PubKey",
          "key": "/O7BtNW0pafwfvomgR4ZnfldwPXiFfJs9mHg3gwfv5Q="
        },
        "jailed": false,
        "status": "BOND_STATUS_BONDED",
        "tokens": "1416521659632",
        "delegator_shares": "1416521659632.000000000000000000",
        "description": {
          "moniker": "SG-1",
          "identity": "48608633F99D1B60",
          "website": "https://sg-1.online",
          "security_contact": "",
          "details": "SG-1 - your favorite validator on cosmos. We offer 100% Soft Slash protection."
        },
        "unbonding_height": "0",
        "unbonding_time": "1970-01-01T00:00:00Z",
        "commission": {
          "commission_rates": {
            "rate": "0.037500000000000000",
            "max_rate": "0.200000000000000000",
            "max_change_rate": "0.030000000000000000"
          },
          "update_time": "2021-10-01T15:00:00Z"
        },
        "min_self_delegation": "1"
      },
      {
        "operator_address": "cosmosvaloper1t8ehvswxjfn3ejzkjtntcyrqwvmvuknzmvtaaa",
        "consensus_pubkey": {
          "@type": "/cosmos.crypto.ed25519.PubKey",
          "key": "uExZyjNLtr2+FFIhNDAMcQ8+yTrqE7ygYTsI7khkA5Y="
        },
        "jailed": false,
        "status": "BOND_STATUS_BONDED",
        "tokens": "1348298958808",
        "delegator_shares": "1348298958808.000000000000000000",
        "description": {
          "moniker": "Cosmostation",
          "identity": "AE4C403A6E7AA1AC",
          "website": "https://www.cosmostation.io",
          "security_contact": "admin@stamper.network",
          "details": "Cosmostation validator node. Delegate your tokens and Start Earning Staking Rewards"
        },
        "unbonding_height": "0",
        "unbonding_time": "1970-01-01T00:00:00Z",
        "commission": {
          "commission_rates": {
            "rate": "0.050000000000000000",
            "max_rate": "1.000000000000000000",
            "max_change_rate": "0.200000000000000000"
          },
          "update_time": "2021-10-01T15:06:38.821314287Z"
        },
        "min_self_delegation": "1"
      }
    ]
  }
}
```

#### Parameters

Endpoint REST `Parameters` truy vấn tham số staking.

```bash
/cosmos/staking/v1beta1/params
```

Ví dụ:

```bash
curl -X GET "http://localhost:1317/cosmos/staking/v1beta1/params" -H  "accept: application/json"
```

Ví dụ đầu ra:

```bash
{
  "params": {
    "unbonding_time": "2419200s",
    "max_validators": 100,
    "max_entries": 7,
    "historical_entries": 10000,
    "bond_denom": "stake"
  }
}
```

#### Pool

Endpoint REST `Pool` truy vấn thông tin pool.

```bash
/cosmos/staking/v1beta1/pool
```

Ví dụ:

```bash
curl -X GET "http://localhost:1317/cosmos/staking/v1beta1/pool" -H  "accept: application/json"
```

Ví dụ đầu ra:

```bash
{
  "pool": {
    "not_bonded_tokens": "432805737458",
    "bonded_tokens": "15783637712645"
  }
}
```

#### Validators

Endpoint REST `Validators` truy vấn tất cả validators khớp với trạng thái đã cho.

```bash
/cosmos/staking/v1beta1/validators
```

Ví dụ:

```bash
curl -X GET "http://localhost:1317/cosmos/staking/v1beta1/validators" -H  "accept: application/json"
```

Ví dụ đầu ra:

```bash
{
  "validators": [
    {
      "operator_address": "cosmosvaloper1q3jsx9dpfhtyqqgetwpe5tmk8f0ms5qywje8tw",
      "consensus_pubkey": {
        "@type": "/cosmos.crypto.ed25519.PubKey",
        "key": "N7BPyek2aKuNZ0N/8YsrqSDhGZmgVaYUBuddY8pwKaE="
      },
      "jailed": false,
      "status": "BOND_STATUS_BONDED",
      "tokens": "383301887799",
      "delegator_shares": "383301887799.000000000000000000",
      "description": {
        "moniker": "SmartNodes",
        "identity": "D372724899D1EDC8",
        "website": "https://smartnodes.co",
        "security_contact": "",
        "details": "Earn Rewards with Crypto Staking & Node Deployment"
      },
      "unbonding_height": "0",
      "unbonding_time": "1970-01-01T00:00:00Z",
      "commission": {
        "commission_rates": {
          "rate": "0.050000000000000000",
          "max_rate": "0.200000000000000000",
          "max_change_rate": "0.100000000000000000"
        },
        "update_time": "2021-10-01T15:51:31.596618510Z"
      },
      "min_self_delegation": "1"
    },
    {
      "operator_address": "cosmosvaloper1q5ku90atkhktze83j9xjaks2p7uruag5zp6wt7",
      "consensus_pubkey": {
        "@type": "/cosmos.crypto.ed25519.PubKey",
        "key": "GDNpuKDmCg9GnhnsiU4fCWktuGUemjNfvpCZiqoRIYA="
      },
      "jailed": false,
      "status": "BOND_STATUS_UNBONDING",
      "tokens": "1017819654",
      "delegator_shares": "1017819654.000000000000000000",
      "description": {
        "moniker": "Noderunners",
        "identity": "812E82D12FEA3493",
        "website": "http://noderunners.biz",
        "security_contact": "info@noderunners.biz",
        "details": "Noderunners is a professional validator in POS networks. We have a huge node running experience, reliable soft and hardware. Our commissions are always low, our support to delegators is always full. Stake with us and start receiving your cosmos rewards now!"
      },
      "unbonding_height": "147302",
      "unbonding_time": "2021-11-08T22:58:53.718662452Z",
      "commission": {
        "commission_rates": {
          "rate": "0.050000000000000000",
          "max_rate": "0.200000000000000000",
          "max_change_rate": "0.100000000000000000"
        },
        "update_time": "2021-10-04T18:02:21.446645619Z"
      },
      "min_self_delegation": "1"
    }
  ],
  "pagination": {
    "next_key": "FONDBFkE4tEEf7yxWWKOD49jC2NK",
    "total": "2"
  }
}
```

#### Validator

Endpoint REST `Validator` truy vấn thông tin validator cho địa chỉ validator đã cho.

```bash
/cosmos/staking/v1beta1/validators/{validatorAddr}
```

Ví dụ:

```bash
curl -X GET \
"http://localhost:1317/cosmos/staking/v1beta1/validators/cosmosvaloper16msryt3fqlxtvsy8u5ay7wv2p8mglfg9g70e3q" \
-H  "accept: application/json"
```

Ví dụ đầu ra:

```bash
{
  "validator": {
    "operator_address": "cosmosvaloper16msryt3fqlxtvsy8u5ay7wv2p8mglfg9g70e3q",
    "consensus_pubkey": {
      "@type": "/cosmos.crypto.ed25519.PubKey",
      "key": "sIiexdJdYWn27+7iUHQJDnkp63gq/rzUq1Y+fxoGjXc="
    },
    "jailed": false,
    "status": "BOND_STATUS_BONDED",
    "tokens": "33027900000",
    "delegator_shares": "33027900000.000000000000000000",
    "description": {
      "moniker": "Witval",
      "identity": "51468B615127273A",
      "website": "",
      "security_contact": "",
      "details": "Witval is the validator arm from Vitwit. Vitwit is into software consulting and services business since 2015. We are working closely with Cosmos ecosystem since 2018. We are also building tools for the ecosystem, Aneka is our explorer for the cosmos ecosystem."
    },
    "unbonding_height": "0",
    "unbonding_time": "1970-01-01T00:00:00Z",
    "commission": {
      "commission_rates": {
        "rate": "0.050000000000000000",
        "max_rate": "0.200000000000000000",
        "max_change_rate": "0.020000000000000000"
      },
      "update_time": "2021-10-01T19:24:52.663191049Z"
    },
    "min_self_delegation": "1"
  }
}
```

#### ValidatorDelegations

Endpoint REST `ValidatorDelegations` truy vấn thông tin delegate cho validator đã cho.

```bash
/cosmos/staking/v1beta1/validators/{validatorAddr}/delegations
```

Ví dụ:

```bash
curl -X GET "http://localhost:1317/cosmos/staking/v1beta1/validators/cosmosvaloper16msryt3fqlxtvsy8u5ay7wv2p8mglfg9g70e3q/delegations" -H  "accept: application/json"
```

Ví dụ đầu ra:

```bash
{
  "delegation_responses": [
    {
      "delegation": {
        "delegator_address": "cosmos190g5j8aszqhvtg7cprmev8xcxs6csra7xnk3n3",
        "validator_address": "cosmosvaloper16msryt3fqlxtvsy8u5ay7wv2p8mglfg9g70e3q",
        "shares": "31000000000.000000000000000000"
      },
      "balance": {
        "denom": "stake",
        "amount": "31000000000"
      }
    },
    {
      "delegation": {
        "delegator_address": "cosmos1ddle9tczl87gsvmeva3c48nenyng4n56qwq4ee",
        "validator_address": "cosmosvaloper16msryt3fqlxtvsy8u5ay7wv2p8mglfg9g70e3q",
        "shares": "628470000.000000000000000000"
      },
      "balance": {
        "denom": "stake",
        "amount": "628470000"
      }
    },
    {
      "delegation": {
        "delegator_address": "cosmos10fdvkczl76m040smd33lh9xn9j0cf26kk4s2nw",
        "validator_address": "cosmosvaloper16msryt3fqlxtvsy8u5ay7wv2p8mglfg9g70e3q",
        "shares": "838120000.000000000000000000"
      },
      "balance": {
        "denom": "stake",
        "amount": "838120000"
      }
    },
    {
      "delegation": {
        "delegator_address": "cosmos1n8f5fknsv2yt7a8u6nrx30zqy7lu9jfm0t5lq8",
        "validator_address": "cosmosvaloper16msryt3fqlxtvsy8u5ay7wv2p8mglfg9g70e3q",
        "shares": "500000000.000000000000000000"
      },
      "balance": {
        "denom": "stake",
        "amount": "500000000"
      }
    },
    {
      "delegation": {
        "delegator_address": "cosmos16msryt3fqlxtvsy8u5ay7wv2p8mglfg9hrek2e",
        "validator_address": "cosmosvaloper16msryt3fqlxtvsy8u5ay7wv2p8mglfg9g70e3q",
        "shares": "61310000.000000000000000000"
      },
      "balance": {
        "denom": "stake",
        "amount": "61310000"
      }
    }
  ],
  "pagination": {
    "next_key": null,
    "total": "5"
  }
}
```

#### Delegation

Endpoint REST `Delegation` truy vấn thông tin delegate cho cặp validator delegator đã cho.

```bash
/cosmos/staking/v1beta1/validators/{validatorAddr}/delegations/{delegatorAddr}
```

Ví dụ:

```bash
curl -X GET \
"http://localhost:1317/cosmos/staking/v1beta1/validators/cosmosvaloper16msryt3fqlxtvsy8u5ay7wv2p8mglfg9g70e3q/delegations/cosmos1n8f5fknsv2yt7a8u6nrx30zqy7lu9jfm0t5lq8" \
-H  "accept: application/json"
```

Ví dụ đầu ra:

```bash
{
  "delegation_response": {
    "delegation": {
      "delegator_address": "cosmos1n8f5fknsv2yt7a8u6nrx30zqy7lu9jfm0t5lq8",
      "validator_address": "cosmosvaloper16msryt3fqlxtvsy8u5ay7wv2p8mglfg9g70e3q",
      "shares": "500000000.000000000000000000"
    },
    "balance": {
      "denom": "stake",
      "amount": "500000000"
    }
  }
}
```

#### UnbondingDelegation

Endpoint REST `UnbondingDelegation` truy vấn thông tin unbonding cho cặp validator delegator đã cho.

```bash
/cosmos/staking/v1beta1/validators/{validatorAddr}/delegations/{delegatorAddr}/unbonding_delegation
```

Ví dụ:

```bash
curl -X GET \
"http://localhost:1317/cosmos/staking/v1beta1/validators/cosmosvaloper13v4spsah85ps4vtrw07vzea37gq5la5gktlkeu/delegations/cosmos1ze2ye5u5k3qdlexvt2e0nn0508p04094ya0qpm/unbonding_delegation" \
-H  "accept: application/json"
```

Ví dụ đầu ra:

```bash
{
  "unbond": {
    "delegator_address": "cosmos1ze2ye5u5k3qdlexvt2e0nn0508p04094ya0qpm",
    "validator_address": "cosmosvaloper13v4spsah85ps4vtrw07vzea37gq5la5gktlkeu",
    "entries": [
      {
        "creation_height": "153687",
        "completion_time": "2021-11-09T09:41:18.352401903Z",
        "initial_balance": "525111",
        "balance": "525111"
      }
    ]
  }
}
```

#### ValidatorUnbondingDelegations

Endpoint REST `ValidatorUnbondingDelegations` truy vấn unbonding delegations của validator.

```bash
/cosmos/staking/v1beta1/validators/{validatorAddr}/unbonding_delegations
```

Ví dụ:

```bash
curl -X GET \
"http://localhost:1317/cosmos/staking/v1beta1/validators/cosmosvaloper13v4spsah85ps4vtrw07vzea37gq5la5gktlkeu/unbonding_delegations" \
-H  "accept: application/json"
```

Ví dụ đầu ra:

```bash
{
  "unbonding_responses": [
    {
      "delegator_address": "cosmos1q9snn84jfrd9ge8t46kdcggpe58dua82vnj7uy",
      "validator_address": "cosmosvaloper13v4spsah85ps4vtrw07vzea37gq5la5gktlkeu",
      "entries": [
        {
          "creation_height": "90998",
          "completion_time": "2021-11-05T00:14:37.005841058Z",
          "initial_balance": "24000000",
          "balance": "24000000"
        }
      ]
    },
    {
      "delegator_address": "cosmos1qf36e6wmq9h4twhdvs6pyq9qcaeu7ye0s3dqq2",
      "validator_address": "cosmosvaloper13v4spsah85ps4vtrw07vzea37gq5la5gktlkeu",
      "entries": [
        {
          "creation_height": "47478",
          "completion_time": "2021-11-01T22:47:26.714116854Z",
          "initial_balance": "8000000",
          "balance": "8000000"
        }
      ]
    }
  ],
  "pagination": {
    "next_key": null,
    "total": "2"
  }
}
```
