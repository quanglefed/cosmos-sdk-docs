---
sidebar_position: 1
---

# `x/slashing`

## Tóm tắt

Phần này đặc tả module slashing của Cosmos SDK, triển khai các chức năng lần đầu được phác thảo trong [Cosmos Whitepaper](https://cosmos.network/about/whitepaper) (tháng 6/2016).

Module slashing cho phép các blockchain dựa trên Cosmos SDK tạo cơ chế bất lợi hoá (disincentivize) các hành động có thể quy trách nhiệm (attributable) của một tác nhân được giao thức công nhận và có “giá trị đang stake”, bằng cách áp dụng hình phạt (“slashing”).

Hình phạt có thể bao gồm (nhưng không giới hạn):

* Đốt một phần stake
* Tước quyền bỏ phiếu cho các block trong một khoảng thời gian

Module này được dùng bởi Cosmos Hub, hub đầu tiên trong hệ sinh thái Cosmos.

## Mục lục

* [Khái niệm](#khái-niệm)
    * [Trạng thái](#trạng-thái)
    * [Giới hạn tombstone](#giới-hạn-tombstone)
    * [Dòng thời gian vi phạm](#dòng-thời-gian-vi-phạm)
* [State](#state)
    * [Signing Info (Liveness)](#signing-info-liveness)
    * [Params](#params)
* [Messages](#messages)
    * [Unjail](#unjail)
* [BeginBlock](#beginblock)
    * [Theo dõi liveness](#theo-dõi-liveness)
* [Hooks](#hooks)
* [Events](#events)
* [Staking Tombstone](#staking-tombstone)
* [Parameters](#parameters)
* [CLI](#cli)
    * [Query](#query)
    * [Transactions](#transactions)
    * [gRPC](#grpc)
    * [REST](#rest)

## Khái niệm

### Trạng thái

Tại mọi thời điểm, state machine có thể có bất kỳ số lượng validator nào đã đăng ký. Mỗi block, `MaxValidators` validator đứng đầu (định nghĩa bởi `x/staking`) và **không bị jailed** sẽ trở thành _bonded_, nghĩa là họ có thể đề xuất (propose) và bỏ phiếu (vote) cho các block. Validator _bonded_ là _at stake_, nghĩa là một phần hoặc toàn bộ stake của họ (và stake của delegator uỷ quyền cho họ) có rủi ro bị phạt nếu họ gây ra lỗi giao thức.

Với mỗi validator như vậy, ta lưu một bản ghi `ValidatorSigningInfo` chứa thông tin liên quan đến liveness và các thuộc tính khác về vi phạm.

### Giới hạn tombstone

Để giảm tác động của các nhóm lỗi giao thức ban đầu vốn có khả năng là “không cố ý”, Cosmos Hub áp dụng cho mỗi validator một giới hạn _tombstone_ (đóng mộ), chỉ cho phép một validator bị slash **một lần** cho lỗi double-sign. Ví dụ: nếu bạn cấu hình sai HSM và double-sign nhiều block cũ, bạn sẽ chỉ bị phạt cho lần double-sign đầu tiên (và sau đó bị tombstoned ngay lập tức). Dù vẫn rất tốn kém và nên tránh, giới hạn tombstone phần nào làm “cùn” tác động kinh tế của việc cấu hình sai ngoài ý muốn.

Lỗi liveness không có giới hạn (cap) vì chúng không thể “cộng dồn” lên nhau. Lỗi liveness được “phát hiện” ngay khi vi phạm xảy ra, và validator bị jailed ngay lập tức, nên không thể gây ra nhiều lỗi liveness mà không unjail ở giữa.

### Dòng thời gian vi phạm

Để minh hoạ cách module `x/slashing` xử lý evidence được submit thông qua consensus CometBFT, hãy xem các ví dụ sau.

**Định nghĩa**:

_[_ : bắt đầu timeline  
_]_ : kết thúc timeline  
_C<sub>n</sub>_ : vi phạm `n` được thực hiện  
_D<sub>n</sub>_ : vi phạm `n` được phát hiện  
_V<sub>b</sub>_ : validator bonded  
_V<sub>u</sub>_ : validator unbonded

#### Một vi phạm double-sign

\[----------C<sub>1</sub>----D<sub>1</sub>,V<sub>u</sub>-----\]

Một vi phạm được thực hiện và sau đó được phát hiện; tại thời điểm phát hiện, validator đã unbonded và bị slash đầy đủ theo mức của vi phạm.

#### Nhiều vi phạm double-sign

\[----------C<sub>1</sub>--C<sub>2</sub>---C<sub>3</sub>---D<sub>1</sub>,D<sub>2</sub>,D<sub>3</sub>V<sub>u</sub>-----\]

Nhiều vi phạm được thực hiện rồi sau đó mới bị phát hiện; tại thời điểm phát hiện, validator bị jailed và chỉ bị slash cho **một** vi phạm. Vì validator cũng bị tombstoned, họ không thể quay lại validator set.

## State

### Signing Info (Liveness)

Mỗi block đều chứa một tập precommit của validator cho block trước đó, được CometBFT cung cấp dưới dạng `LastCommitInfo`. Một `LastCommitInfo` hợp lệ miễn là nó chứa precommit từ hơn \(2/3\) tổng voting power.

Proposer được khuyến khích đưa precommit từ mọi validator vào `LastCommitInfo` của CometBFT bằng việc nhận thêm phí tỉ lệ với chênh lệch giữa voting power được đưa vào `LastCommitInfo` và \(2/3\) (xem [fee distribution](../distribution/README.md#begin-block)).

```go
type LastCommitInfo struct {
	Round int32
	Votes []VoteInfo
}
```

Validator sẽ bị phạt nếu không được đưa vào `LastCommitInfo` trong một số block nhất định, bằng cách tự động bị jailed, có thể bị slash, và unbonded.

Thông tin về hoạt động liveness của validator được theo dõi qua `ValidatorSigningInfo`. Nó được index trong store như sau:

* ValidatorSigningInfo: `0x01 | ConsAddrLen (1 byte) | ConsAddress -> ProtocolBuffer(ValSigningInfo)`
* MissedBlocksBitArray: `0x02 | ConsAddrLen (1 byte) | ConsAddress | LittleEndianUint64(signArrayIndex) -> VarInt(didMiss)` (varint là một định dạng mã hoá số)

Mapping đầu tiên giúp ta dễ dàng tra cứu signing info gần đây của validator dựa trên consensus address.

Mapping thứ hai (`MissedBlocksBitArray`) hoạt động như một bit-array có kích thước `SignedBlocksWindow`, cho biết validator có bỏ lỡ block ở một index nhất định trong bit-array hay không. Index trong bit-array được biểu diễn dưới dạng little endian uint64. Kết quả là một `varint` nhận giá trị `0` hoặc `1`, trong đó `0` nghĩa là validator **không** bỏ lỡ (có ký), và `1` nghĩa là họ bỏ lỡ block (không ký).

Lưu ý `MissedBlocksBitArray` không được khởi tạo tường minh từ đầu. Key sẽ được thêm dần khi ta tiến qua `SignedBlocksWindow` block đầu tiên của một validator mới bonded. Tham số `SignedBlocksWindow` xác định kích thước (số block) của cửa sổ trượt dùng để theo dõi liveness.

Thông tin được lưu để theo dõi liveness của validator như sau:

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/slashing/v1beta1/slashing.proto#L13-L35
```

### Params

Module slashing lưu params trong state với prefix `0x00`, và có thể được cập nhật thông qua governance hoặc địa chỉ có authority.

* Params: `0x00 | ProtocolBuffer(Params)`

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/slashing/v1beta1/slashing.proto#L37-L59
```

## Messages

Phần này mô tả xử lý các message cho module `slashing`.

### Unjail

Nếu một validator bị tự động unbonded do downtime và muốn quay lại online và/hoặc có thể gia nhập lại bonded set, họ phải gửi `MsgUnjail`:

```protobuf
// MsgUnjail is an sdk.Msg used for unjailing a jailed validator, thus returning
// them into the bonded validator set, so they can begin receiving provisions
// and rewards again.
message MsgUnjail {
  string validator_addr = 1;
}
```

Dưới đây là pseudocode cho RPC `MsgSrv/Unjail`:

```go
unjail(tx MsgUnjail)
    validator = getValidator(tx.ValidatorAddr)
    if validator == nil
      fail with "No validator found"

    if getSelfDelegation(validator) == 0
      fail with "validator must self delegate before unjailing"

    if !validator.Jailed
      fail with "Validator not jailed, cannot unjail"

    info = GetValidatorSigningInfo(operator)
    if info.Tombstoned
      fail with "Tombstoned validator cannot be unjailed"
    if block time < info.JailedUntil
      fail with "Validator still jailed, cannot unjail until period has expired"

    validator.Jailed = false
    setValidator(validator)

    return
```

Nếu validator có đủ stake để nằm trong top `n = MaximumBondedValidators`, họ sẽ tự động được rebonded, và mọi delegator còn đang uỷ quyền cho validator đó sẽ được rebonded và bắt đầu nhận provisions và rewards trở lại.

## BeginBlock

### Theo dõi liveness

Ở đầu mỗi block, ta cập nhật `ValidatorSigningInfo` cho từng validator và kiểm tra xem họ có vượt ngưỡng liveness trong một cửa sổ trượt hay không. Cửa sổ trượt này được định nghĩa bởi `SignedBlocksWindow` và index trong cửa sổ được xác định bởi `IndexOffset` trong `ValidatorSigningInfo` của validator. Với mỗi block được xử lý, `IndexOffset` sẽ tăng bất kể validator có ký hay không. Khi đã xác định index, `MissedBlocksBitArray` và `MissedBlocksCounter` được cập nhật tương ứng.

Cuối cùng, để xác định validator có rơi dưới ngưỡng liveness hay không, ta lấy số block tối đa được phép miss là `maxMissed`, được tính bằng `SignedBlocksWindow - (MinSignedPerWindow * SignedBlocksWindow)`, và chiều cao tối thiểu để có thể xác định liveness là `minHeight`. Nếu block hiện tại lớn hơn `minHeight` và `MissedBlocksCounter` của validator lớn hơn `maxMissed`, họ sẽ bị slash theo `SlashFractionDowntime`, bị jailed trong `DowntimeJailDuration`, và các giá trị sau sẽ được reset: `MissedBlocksBitArray`, `MissedBlocksCounter`, và `IndexOffset`.

**Lưu ý**: Slash do liveness **KHÔNG** dẫn tới tombstoning.

```go
height := block.Height

for vote in block.LastCommitInfo.Votes {
  signInfo := GetValidatorSigningInfo(vote.Validator.Address)

  // This is a relative index, so we count blocks the validator SHOULD have
  // signed. We use the 0-value default signing info if not present, except for
  // start height.
  index := signInfo.IndexOffset % SignedBlocksWindow()
  signInfo.IndexOffset++

  // Update MissedBlocksBitArray and MissedBlocksCounter. The MissedBlocksCounter
  // just tracks the sum of MissedBlocksBitArray. That way we avoid needing to
  // read/write the whole array each time.
  missedPrevious := GetValidatorMissedBlockBitArray(vote.Validator.Address, index)
  missed := !signed

  switch {
  case !missedPrevious && missed:
    // array index has changed from not missed to missed, increment counter
    SetValidatorMissedBlockBitArray(vote.Validator.Address, index, true)
    signInfo.MissedBlocksCounter++

  case missedPrevious && !missed:
    // array index has changed from missed to not missed, decrement counter
    SetValidatorMissedBlockBitArray(vote.Validator.Address, index, false)
    signInfo.MissedBlocksCounter--

  default:
    // array index at this index has not changed; no need to update counter
  }

  if missed {
    // emit events...
  }

  minHeight := signInfo.StartHeight + SignedBlocksWindow()
  maxMissed := SignedBlocksWindow() - MinSignedPerWindow()

  // If we are past the minimum height and the validator has missed too many
  // jail and slash them.
  if height > minHeight && signInfo.MissedBlocksCounter > maxMissed {
    validator := ValidatorByConsAddr(vote.Validator.Address)

    // emit events...

    // We need to retrieve the stake distribution which signed the block, so we
    // subtract ValidatorUpdateDelay from the block height, and subtract an
    // additional 1 since this is the LastCommit.
    //
    // Note, that this CAN result in a negative "distributionHeight" up to
    // -ValidatorUpdateDelay-1, i.e. at the end of the pre-genesis block (none) = at the beginning of the genesis block.
    // That's fine since this is just used to filter unbonding delegations & redelegations.
    distributionHeight := height - sdk.ValidatorUpdateDelay - 1

    SlashWithInfractionReason(vote.Validator.Address, distributionHeight, vote.Validator.Power, SlashFractionDowntime(), stakingtypes.Downtime)
    Jail(vote.Validator.Address)

    signInfo.JailedUntil = block.Time.Add(DowntimeJailDuration())

    // We need to reset the counter & array so that the validator won't be
    // immediately slashed for downtime upon rebonding.
    signInfo.MissedBlocksCounter = 0
    signInfo.IndexOffset = 0
    ClearValidatorMissedBlockBitArray(vote.Validator.Address)
  }

  SetValidatorSigningInfo(vote.Validator.Address, signInfo)
}
```

## Hooks

Phần này mô tả các `hooks` của module. Hook là các thao tác được thực thi tự động khi sự kiện được phát ra.

### Staking hooks

Module slashing triển khai `StakingHooks` được định nghĩa trong `x/staking` và được dùng để ghi nhận thông tin validator. Trong quá trình khởi tạo app, các hook này nên được đăng ký trong struct của module staking.

Các hook sau tác động lên slashing state:

* `AfterValidatorBonded` tạo một instance `ValidatorSigningInfo` như mô tả ở phần dưới.
* `AfterValidatorCreated` lưu consensus key của validator.
* `AfterValidatorRemoved` xoá consensus key của validator.

### Validator Bonded

Khi một validator mới lần đầu bonded thành công, ta tạo một cấu trúc `ValidatorSigningInfo` mới cho validator vừa bonded, với `StartHeight` là height của block hiện tại.

Nếu validator rời khỏi validator set và bonded lại, height bonded mới của nó sẽ được đặt lại.

```go
onValidatorBonded(address sdk.ValAddress)

  signingInfo, found = GetValidatorSigningInfo(address)
  if !found {
    signingInfo = ValidatorSigningInfo {
      StartHeight         : CurrentHeight,
      IndexOffset         : 0,
      JailedUntil         : time.Unix(0, 0),
      Tombstone           : false,
      MissedBlockCounter  : 0
    } else {
      signingInfo.StartHeight = CurrentHeight
    }

    setValidatorSigningInfo(signingInfo)
  }

  return
```

## Events

Module slashing phát ra các event sau:

### MsgServer

#### MsgUnjail

| Type    | Attribute Key | Attribute Value    |
| ------- | ------------- | ------------------ |
| message | module        | slashing           |
| message | sender        | {validatorAddress} |

### Keeper

### BeginBlocker: HandleValidatorSignature

| Type  | Attribute Key | Attribute Value             |
| ----- | ------------- | --------------------------- |
| slash | address       | {validatorConsensusAddress} |
| slash | power         | {validatorPower}            |
| slash | reason        | {slashReason}               |
| slash | jailed [0]    | {validatorConsensusAddress} |
| slash | burned coins  | {math.Int}                   |

* [0] Chỉ xuất hiện nếu validator bị jailed.

| Type     | Attribute Key | Attribute Value             |
| -------- | ------------- | --------------------------- |
| liveness | address       | {validatorConsensusAddress} |
| liveness | missed_blocks | {missedBlocksCounter}       |
| liveness | height        | {blockHeight}               |

#### Slash

* giống event `"slash"` từ `HandleValidatorSignature`, nhưng không có thuộc tính `jailed`.

#### Jail

| Type  | Attribute Key | Attribute Value    |
| ----- | ------------- | ------------------ |
| slash | jailed        | {validatorAddress} |

## Staking Tombstone

### Tóm tắt

Trong triển khai hiện tại của module `slashing`, khi consensus engine thông báo cho state machine về một lỗi consensus của validator, validator sẽ bị slash một phần và bị đưa vào một “giai đoạn jail” (jail period) — khoảng thời gian mà họ không được phép tái gia nhập validator set. Tuy nhiên, do bản chất của lỗi consensus và ABCI, có thể có độ trễ giữa thời điểm vi phạm xảy ra và thời điểm evidence của vi phạm đến được state machine (đây là một trong các lý do chính của việc tồn tại unbonding period).

> Note: Khái niệm tombstone chỉ áp dụng cho các lỗi có độ trễ giữa thời điểm vi phạm xảy ra và thời điểm evidence đến state machine. Ví dụ, evidence của việc double-sign có thể mất thời gian để đến được state machine do độ trễ không thể dự đoán của lớp gossip evidence và khả năng validator chọn lọc tiết lộ chữ ký double-sign (ví dụ, cho các light client online không thường xuyên). Liveness slashing thì được phát hiện ngay lập tức khi vi phạm xảy ra, nên không cần slashing period. Validator bị đưa vào jail ngay lập tức và họ không thể gây ra thêm lỗi liveness cho đến khi unjail. Trong tương lai, có thể có các loại lỗi byzantine khác có độ trễ (ví dụ: submit evidence về một proposal không hợp lệ dưới dạng giao dịch). Khi được triển khai, sẽ cần quyết định các loại lỗi byzantine tương lai này có dẫn tới tombstoning hay không (nếu không, mức slash sẽ không bị cap bởi slashing period).

Trong thiết kế hiện tại, một khi validator bị jailed vì lỗi an toàn consensus (consensus fault), sau `JailPeriod` họ được phép gửi một giao dịch để tự `unjail` và tái gia nhập validator set.

Một trong các “mong muốn thiết kế” của module `slashing` là nếu nhiều vi phạm xảy ra trước khi evidence được thực thi (và validator bị jailed), thì họ chỉ nên bị phạt cho **một** vi phạm tệ nhất, thay vì cộng dồn. Ví dụ, nếu chuỗi sự kiện là:

1. Validator A thực hiện Vi phạm 1 (slash 30%)
2. Validator A thực hiện Vi phạm 2 (slash 40%)
3. Validator A thực hiện Vi phạm 3 (slash 35%)
4. Evidence của Vi phạm 1 đến state machine (và validator bị jailed)
5. Evidence của Vi phạm 2 đến state machine
6. Evidence của Vi phạm 3 đến state machine

Chỉ Vi phạm 2 nên có tác dụng slash vì nó lớn nhất. Điều này nhằm đảm bảo trong trường hợp consensus key của validator bị lộ, họ chỉ bị phạt một lần, ngay cả khi kẻ tấn công double-sign nhiều block. Vì việc unjail phải được thực hiện bằng operator key của validator, họ có cơ hội bảo vệ lại consensus key và sau đó dùng operator key để báo hiệu rằng họ đã sẵn sàng. Ta gọi khoảng thời gian mà ta chỉ theo dõi vi phạm lớn nhất là “slashing period”.

Khi validator gia nhập lại bằng cách tự unjail, ta bắt đầu một slashing period mới; nếu họ gây ra vi phạm mới sau khi unjail, vi phạm đó sẽ bị slash cộng dồn lên trên vi phạm tệ nhất từ slashing period trước.

Tuy nhiên, dù các vi phạm được nhóm theo slashing period, vì evidence có thể được submit đến `unbondingPeriod` sau khi vi phạm xảy ra, ta vẫn phải cho phép submit evidence cho các slashing period trước. Ví dụ, nếu chuỗi sự kiện là:

1. Validator A thực hiện Vi phạm 1 (slash 30%)
2. Validator A thực hiện Vi phạm 2 (slash 40%)
3. Evidence của Vi phạm 1 đến state machine (và Validator A bị jailed)
4. Validator A unjail

Giờ ta đang ở slashing period mới, nhưng ta vẫn phải “mở cửa” cho vi phạm trước đó, vì evidence của Vi phạm 2 có thể vẫn đến. Khi số slashing period tăng lên, độ phức tạp tăng vì ta phải theo dõi mức slash lớn nhất cho từng slashing period cho mỗi validator.

> Note: Hiện tại, theo đặc tả module `slashing`, một slashing period mới được tạo mỗi khi validator unbonded rồi rebonded. Điều này có lẽ nên đổi thành jailed/unjail. Xem issue [#3205](https://github.com/cosmos/cosmos-sdk/issues/3205). Ở phần còn lại, ta giả định rằng slashing period mới chỉ bắt đầu khi validator được unjail.

Số slashing period tối đa là `len(UnbondingPeriod) / len(JailPeriod)`. Giá trị mặc định hiện tại trong Gaia cho `UnbondingPeriod` và `JailPeriod` lần lượt là 3 tuần và 2 ngày. Điều này nghĩa là có thể có tới 11 slashing period được theo dõi đồng thời cho mỗi validator. Nếu đặt `JailPeriod >= UnbondingPeriod`, ta chỉ cần theo dõi 1 slashing period (tức là không cần theo dõi slashing period).

Hiện tại, trong triển khai jail period, một khi validator unjail, tất cả delegator đang uỷ quyền cho họ (chưa unbond/redelegate đi) vẫn ở lại với họ. Vì các lỗi an toàn consensus là cực kỳ nghiêm trọng (nghiêm trọng hơn liveness), có lẽ nên cân nhắc để delegator không “auto-rebond” với validator.

#### Đề xuất: jail vĩnh viễn

Ta đề xuất đặt “jail time” cho validator gây ra lỗi an toàn consensus thành `infinite` (vô hạn), tức là trạng thái tombstone. Điều này đẩy validator ra khỏi validator set và không cho phép họ quay lại. Tất cả delegator của họ (bao gồm chính operator) phải unbond hoặc redelegate đi. Nếu operator muốn, họ có thể tạo một validator mới với operator key và consensus key mới, nhưng họ phải “kiếm lại” các uỷ quyền.

Việc triển khai tombstone system và bỏ theo dõi slashing period sẽ làm module `slashing` đơn giản hơn nhiều, đặc biệt vì ta có thể loại bỏ các hook được định nghĩa trong `slashing` và được `staking` tiêu thụ (module `slashing` vẫn tiêu thụ hook do `staking` định nghĩa).

#### Một mức slash duy nhất

Một tối ưu khác: nếu ta giả định mọi lỗi ABCI cho consensus CometBFT đều bị slash ở cùng một mức, ta không cần theo dõi “max slash”. Một khi lỗi ABCI xảy ra, ta không cần lo so sánh các lỗi tương lai để tìm mức lớn nhất.

Hiện tại, lỗi CometBFT ABCI duy nhất là:

* Unjustified precommits (double signs)

Trong tương lai gần, dự kiến bổ sung lỗi:

* Ký precommit khi đang trong giai đoạn unbonding (cần để light client bisection an toàn)

Vì các lỗi này đều là byzantine fault có thể quy trách nhiệm, nhiều khả năng ta muốn slash chúng như nhau, và do đó có thể áp dụng thay đổi trên.

> Note: Thay đổi này có thể phù hợp với consensus CometBFT hiện tại, nhưng có thể không phù hợp với thuật toán consensus khác hoặc các phiên bản CometBFT tương lai muốn trừng phạt ở các mức khác nhau (ví dụ: partial slashing).

## Parameters

Module slashing có các tham số sau:

| Key                     | Type           | Example                |
| ----------------------- | -------------- | ---------------------- |
| SignedBlocksWindow      | string (int64) | "100"                  |
| MinSignedPerWindow      | string (dec)   | "0.500000000000000000" |
| DowntimeJailDuration    | string (ns)    | "600000000000"         |
| SlashFractionDoubleSign | string (dec)   | "0.050000000000000000" |
| SlashFractionDowntime   | string (dec)   | "0.010000000000000000" |

## CLI

Người dùng có thể truy vấn và tương tác với module `slashing` bằng CLI.

### Query

Các lệnh `query` cho phép truy vấn slashing state.

```shell
simd query slashing --help
```

#### params

Lệnh `params` cho phép truy vấn tham số genesis cho module slashing.

```shell
simd query slashing params [flags]
```

Ví dụ:

```shell
simd query slashing params
```

Ví dụ output:

```yml
downtime_jail_duration: 600s
min_signed_per_window: "0.500000000000000000"
signed_blocks_window: "100"
slash_fraction_double_sign: "0.050000000000000000"
slash_fraction_downtime: "0.010000000000000000"
```

#### signing-info

Lệnh `signing-info` cho phép truy vấn signing-info của validator bằng consensus public key.

```shell
simd query slashing signing-infos [flags]
```

Ví dụ:

```shell
simd query slashing signing-info '{"@type":"/cosmos.crypto.ed25519.PubKey","key":"Auxs3865HpB/EfssYOzfqNhEJjzys6jD5B6tPgC8="}'

```

Ví dụ output:

```yml
address: cosmosvalcons1nrqsld3aw6lh6t082frdqc84uwxn0t958c
index_offset: "2068"
jailed_until: "1970-01-01T00:00:00Z"
missed_blocks_counter: "0"
start_height: "0"
tombstoned: false
```

#### signing-infos

Lệnh `signing-infos` cho phép truy vấn signing info của tất cả validator.

```shell
simd query slashing signing-infos [flags]
```

Ví dụ:

```shell
simd query slashing signing-infos
```

Ví dụ output:

```yml
info:
- address: cosmosvalcons1nrqsld3aw6lh6t082frdqc84uwxn0t958c
  index_offset: "2075"
  jailed_until: "1970-01-01T00:00:00Z"
  missed_blocks_counter: "0"
  start_height: "0"
  tombstoned: false
pagination:
  next_key: null
  total: "0"
```

### Transactions

Các lệnh `tx` cho phép tương tác với module `slashing`.

```bash
simd tx slashing --help
```

#### unjail

Lệnh `unjail` cho phép unjail một validator trước đó bị jailed do downtime.

```bash
simd tx slashing unjail --from mykey [flags]
```

Ví dụ:

```bash
simd tx slashing unjail --from mykey
```

### gRPC

Người dùng có thể truy vấn module `slashing` bằng các endpoint gRPC.

#### Params

Endpoint `Params` cho phép truy vấn tham số của module slashing.

```shell
cosmos.slashing.v1beta1.Query/Params
```

Ví dụ:

```shell
grpcurl -plaintext localhost:9090 cosmos.slashing.v1beta1.Query/Params
```

Ví dụ output:

```json
{
  "params": {
    "signedBlocksWindow": "100",
    "minSignedPerWindow": "NTAwMDAwMDAwMDAwMDAwMDAw",
    "downtimeJailDuration": "600s",
    "slashFractionDoubleSign": "NTAwMDAwMDAwMDAwMDAwMDA=",
    "slashFractionDowntime": "MTAwMDAwMDAwMDAwMDAwMDA="
  }
}
```

#### SigningInfo

`SigningInfo` truy vấn signing info của một cons address.

```shell
cosmos.slashing.v1beta1.Query/SigningInfo
```

Ví dụ:

```shell
grpcurl -plaintext -d '{"cons_address":"cosmosvalcons1nrqsld3aw6lh6t082frdqc84uwxn0t958c"}' localhost:9090 cosmos.slashing.v1beta1.Query/SigningInfo
```

Ví dụ output:

```json
{
  "valSigningInfo": {
    "address": "cosmosvalcons1nrqsld3aw6lh6t082frdqc84uwxn0t958c",
    "indexOffset": "3493",
    "jailedUntil": "1970-01-01T00:00:00Z"
  }
}
```

#### SigningInfos

`SigningInfos` truy vấn signing info của tất cả validator.

```shell
cosmos.slashing.v1beta1.Query/SigningInfos
```

Ví dụ:

```shell
grpcurl -plaintext localhost:9090 cosmos.slashing.v1beta1.Query/SigningInfos
```

Ví dụ output:

```json
{
  "info": [
    {
      "address": "cosmosvalcons1nrqslkwd3pz096lh6t082frdqc84uwxn0t958c",
      "indexOffset": "2467",
      "jailedUntil": "1970-01-01T00:00:00Z"
    }
  ],
  "pagination": {
    "total": "1"
  }
}
```

### REST

Người dùng có thể truy vấn module `slashing` bằng các endpoint REST.

#### Params

```shell
/cosmos/slashing/v1beta1/params
```

Ví dụ:

```shell
curl "localhost:1317/cosmos/slashing/v1beta1/params"
```

Ví dụ output:

```json
{
  "params": {
    "signed_blocks_window": "100",
    "min_signed_per_window": "0.500000000000000000",
    "downtime_jail_duration": "600s",
    "slash_fraction_double_sign": "0.050000000000000000",
    "slash_fraction_downtime": "0.010000000000000000"
}
```

#### signing_info

```shell
/cosmos/slashing/v1beta1/signing_infos/%s
```

Ví dụ:

```shell
curl "localhost:1317/cosmos/slashing/v1beta1/signing_infos/cosmosvalcons1nrqslkwd3pz096lh6t082frdqc84uwxn0t958c"
```

Ví dụ output:

```json
{
  "val_signing_info": {
    "address": "cosmosvalcons1nrqslkwd3pz096lh6t082frdqc84uwxn0t958c",
    "start_height": "0",
    "index_offset": "4184",
    "jailed_until": "1970-01-01T00:00:00Z",
    "tombstoned": false,
    "missed_blocks_counter": "0"
  }
}
```

#### signing_infos

```shell
/cosmos/slashing/v1beta1/signing_infos
```

Ví dụ:

```shell
curl "localhost:1317/cosmos/slashing/v1beta1/signing_infos"
```

Ví dụ output:

```json
{
  "info": [
    {
      "address": "cosmosvalcons1nrqslkwd3pz096lh6t082frdqc84uwxn0t958c",
      "start_height": "0",
      "index_offset": "4169",
      "jailed_until": "1970-01-01T00:00:00Z",
      "tombstoned": false,
      "missed_blocks_counter": "0"
    }
  ],
  "pagination": {
    "next_key": null,
    "total": "1"
  }
}
```

