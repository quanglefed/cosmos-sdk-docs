---
sidebar_position: 1
---

# `x/gov`

## Abstract

Tài liệu này mô tả module Governance của Cosmos SDK, được trình bày lần đầu trong [Cosmos Whitepaper](https://cosmos.network/about/whitepaper) vào tháng 6 năm 2016.

Module cho phép blockchain dựa trên Cosmos SDK hỗ trợ hệ thống quản trị on-chain. Trong hệ thống này, những người nắm giữ token staking gốc của chain có thể bỏ phiếu cho các proposal theo nguyên tắc 1 token 1 vote. Dưới đây là danh sách các tính năng mà module hiện hỗ trợ:

* **Gửi proposal:** Người dùng có thể gửi proposal kèm deposit. Khi đạt minimum deposit, proposal chuyển sang giai đoạn voting. Minimum deposit có thể đạt được bằng cách thu thập deposit từ nhiều người dùng khác nhau (bao gồm người đề xuất) trong thời gian deposit.
* **Vote:** Người tham gia có thể bỏ phiếu cho các proposal đã đạt MinDeposit và đã vào giai đoạn voting.
* **Kế thừa và phạt:** Delegator kế thừa phiếu của validator nếu họ không tự bỏ phiếu.
* **Nhận lại deposit:** Người dùng đã deposit vào proposal có thể nhận lại deposit nếu proposal được chấp nhận hoặc từ chối. Nếu proposal bị veto, hoặc không bao giờ vào giai đoạn voting (không đạt minimum deposit trong thời gian deposit), deposit sẽ bị burn.

Module này đang được sử dụng trên Cosmos Hub (còn gọi là [gaia](https://github.com/cosmos/gaia)).
Các tính năng có thể được thêm trong tương lai được mô tả trong [Future Improvements](#future-improvements).

## Contents

Đặc tả sau sử dụng *ATOM* làm token staking gốc. Module có thể được điều chỉnh cho bất kỳ blockchain Proof-Of-Stake nào bằng cách thay *ATOM* bằng token staking gốc của chain đó.

* [Concepts](#concepts)
    * [Proposal submission](#proposal-submission)
    * [Deposit](#deposit)
    * [Vote](#vote)
    * [Software Upgrade](#software-upgrade)
* [State](#state)
    * [Proposals](#proposals)
    * [Parameters and base types](#parameters-and-base-types)
    * [Deposit](#deposit-1)
    * [ValidatorGovInfo](#validatorgovinfo)
    * [Stores](#stores)
    * [Proposal Processing Queue](#proposal-processing-queue)
    * [Legacy Proposal](#legacy-proposal)
* [Messages](#messages)
    * [Proposal Submission](#proposal-submission-1)
    * [Deposit](#deposit-2)
    * [Vote](#vote-1)
* [Events](#events)
    * [EndBlocker](#endblocker)
    * [Handlers](#handlers)
* [Parameters](#parameters)
* [Client](#client)
    * [CLI](#cli)
    * [gRPC](#grpc)
    * [REST](#rest)
* [Metadata](#metadata)
    * [Proposal](#proposal-3)
    * [Vote](#vote-5)
* [Future Improvements](#future-improvements)

## Concepts

*Disclaimer: Đây là công việc đang tiến hành. Các cơ chế có thể thay đổi.*

Quy trình quản trị được chia thành một số bước được phác thảo dưới đây:

* **Gửi proposal:** Proposal được gửi lên blockchain kèm deposit.
* **Vote:** Khi deposit đạt một giá trị nhất định (`MinDeposit`), proposal được xác nhận và vote mở. Người nắm giữ Atom đã bond có thể gửi giao dịch `TxGovVote` để bỏ phiếu cho proposal.
* **Thực thi** Sau một khoảng thời gian, phiếu được tổng hợp và tùy theo kết quả, các message trong proposal sẽ được thực thi.

### Proposal submission

#### Quyền gửi proposal

Mọi account đều có thể gửi proposal bằng cách gửi giao dịch `MsgSubmitProposal`.
Khi proposal được gửi, nó được xác định bởi `proposalID` duy nhất.

#### Proposal Messages

Một proposal bao gồm một mảng các `sdk.Msg` được thực thi tự động nếu proposal được thông qua. Các message được thực thi bởi chính `ModuleAccount` của governance. Các module như `x/upgrade`, muốn cho phép một số message nhất định chỉ được thực thi bởi governance, nên thêm whitelist trong msg server tương ứng, cấp quyền cho governance module thực thi message khi đã đạt quorum. Module governance sử dụng `MsgServiceRouter` để kiểm tra các message này được xây dựng đúng và có đường dẫn tương ứng để thực thi nhưng không thực hiện kiểm tra tính hợp lệ đầy đủ.

### Deposit

Để ngăn spam, proposal phải được gửi kèm deposit bằng các coin được định nghĩa bởi tham số `MinDeposit`.

Khi proposal được gửi, nó phải kèm theo deposit phải dương chặt, nhưng có thể nhỏ hơn `MinDeposit`. Người gửi không cần trả toàn bộ deposit một mình. Proposal mới tạo được lưu trong *inactive proposal queue* và ở đó cho đến khi deposit vượt qua `MinDeposit`.
Những người nắm giữ token khác có thể tăng deposit của proposal bằng cách gửi giao dịch `Deposit`. Nếu proposal không đạt `MinDeposit` trước thời điểm kết thúc deposit (thời điểm không còn chấp nhận deposit), proposal sẽ bị hủy: proposal sẽ bị xóa khỏi state và deposit sẽ bị burn (xem x/gov `EndBlocker`).
Khi deposit của proposal vượt qua ngưỡng `MinDeposit` (kể cả trong lúc gửi proposal) trước thời điểm kết thúc deposit, proposal sẽ được chuyển vào *active proposal queue* và giai đoạn voting sẽ bắt đầu.

Deposit được giữ trong escrow và nắm giữ bởi governance `ModuleAccount` cho đến khi proposal được hoàn tất (passed hoặc rejected).

#### Hoàn trả và burn deposit

Khi proposal được hoàn tất, coin từ deposit được hoàn trả hoặc burn tùy theo kết quả tổng hợp cuối cùng của proposal:

* Nếu proposal được chấp nhận hoặc từ chối nhưng *không* bị veto, mỗi deposit sẽ được hoàn trả tự động cho người deposit tương ứng (chuyển từ governance `ModuleAccount`).
* Khi proposal bị veto với hơn 1/3, deposit sẽ bị burn từ governance `ModuleAccount` và thông tin proposal cùng thông tin deposit sẽ bị xóa khỏi state.
* Tất cả deposit được hoàn trả hoặc burn đều bị xóa khỏi state. Event được phát ra khi burn hoặc hoàn trả deposit.

### Vote

#### Người tham gia

*Participants* là người dùng có quyền bỏ phiếu cho proposal. Trên Cosmos Hub, participants là người nắm giữ Atom đã bond. Người nắm giữ Atom chưa bond và người dùng khác không có quyền tham gia quản trị. Tuy nhiên, họ có thể gửi và deposit vào proposal.

Lưu ý rằng khi *participants* có cả Atom đã bond và chưa bond, quyền bỏ phiếu của họ được tính từ phần nắm giữ Atom đã bond.

#### Giai đoạn voting

Khi proposal đạt `MinDeposit`, nó ngay lập tức vào `Voting period`. Chúng ta định nghĩa `Voting period` là khoảng thời gian giữa lúc vote mở và lúc vote đóng. Giá trị ban đầu của `Voting period` là 2 tuần.

#### Tập option

Tập option của proposal đề cập đến tập các lựa chọn mà participant có thể chọn khi bỏ phiếu.

Tập option ban đầu bao gồm các option sau:

* `Yes`
* `No`
* `NoWithVeto`
* `Abstain`

`NoWithVeto` được tính là `No` nhưng cũng thêm một phiếu `Veto`. Option `Abstain` cho phép người bỏ phiếu báo hiệu rằng họ không có ý bỏ phiếu tán thành hay phản đối proposal nhưng chấp nhận kết quả của cuộc bỏ phiếu.

*Lưu ý: từ UI, đối với proposal khẩn cấp chúng ta có thể thêm option 'Not Urgent' bỏ phiếu `NoWithVeto`.*

#### Weighted Votes

[ADR-037](https://github.com/cosmos/cosmos-sdk/blob/main/docs/architecture/adr-037-gov-split-vote.md) giới thiệu tính năng weighted vote cho phép staker chia phiếu của họ thành nhiều voting option. Ví dụ, có thể sử dụng 70% quyền bỏ phiếu để vote Yes và 30% để vote No.

Thường thì thực thể sở hữu địa chỉ đó có thể không phải là một cá nhân duy nhất. Ví dụ, một công ty có thể có các stakeholder khác nhau muốn bỏ phiếu khác nhau, và do đó có ý nghĩa khi cho phép họ chia quyền bỏ phiếu. Hiện tại, họ không thể thực hiện "passthrough voting" và cấp quyền bỏ phiếu cho người dùng của họ đối với token của họ. Tuy nhiên, với hệ thống này, các sàn giao dịch có thể thăm dò người dùng về sở thích bỏ phiếu, sau đó bỏ phiếu on-chain theo tỷ lệ kết quả cuộc thăm dò.

Để biểu diễn weighted vote trên chain, chúng ta sử dụng Protobuf message sau.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/gov/v1beta1/gov.proto#L34-L47
```

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/gov/v1beta1/gov.proto#L181-L201
```

Để weighted vote hợp lệ, trường `options` không được chứa vote option trùng lặp, và tổng trọng số của tất cả option phải bằng 1.

#### Custom Vote Calculation

Cosmos SDK v0.53.0 giới thiệu tùy chọn cho developer định nghĩa hàm tính toán kết quả vote và quyền bỏ phiếu tùy chỉnh.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/main/x/gov/keeper/tally.go#L15-L24
```

Điều này cho developer cách biểu đạt linh hoạt hơn để xử lý quản trị trên appchain của họ.
Developer giờ có thể xây dựng hệ thống với:

* Quadratic Voting
* Time-weighted Voting
* Reputation-Based voting

##### Example

```go
func myCustomVotingFunction(
  ctx context.Context,
  k Keeper,
  proposal v1.Proposal,
  validators map[string]v1.ValidatorGovInfo,
) (totalVoterPower math.LegacyDec, results map[v1.VoteOption]math.LegacyDec, err error) {
  // ... tally logic
}

govKeeper := govkeeper.NewKeeper(
  appCodec,
  runtime.NewKVStoreService(keys[govtypes.StoreKey]),
  app.AccountKeeper,
  app.BankKeeper,
  app.StakingKeeper,
  app.DistrKeeper,
  app.MsgServiceRouter(),
  govConfig,
  authtypes.NewModuleAddress(govtypes.ModuleName).String(),
  govkeeper.WithCustomCalculateVoteResultsAndVotingPowerFn(myCustomVotingFunction),
)
```

### Quorum

Quorum được định nghĩa là tỷ lệ phần trăm tối thiểu của quyền bỏ phiếu cần được bỏ cho proposal để kết quả hợp lệ.

### Expedited Proposals

Proposal có thể được expedited, khiến proposal sử dụng thời gian voting ngắn hơn và ngưỡng tally cao hơn theo mặc định. Nếu expedited proposal không đạt ngưỡng trong phạm vi thời gian voting ngắn hơn, expedited proposal sẽ được chuyển thành proposal thường và bắt đầu lại voting trong điều kiện voting thường.

#### Threshold

Threshold được định nghĩa là tỷ lệ tối thiểu của phiếu `Yes` (không tính phiếu `Abstain`) để proposal được chấp nhận.

Ban đầu, threshold được đặt ở 50% phiếu `Yes`, không tính phiếu `Abstain`.
Có khả năng veto nếu hơn 1/3 tổng phiếu là phiếu `NoWithVeto`. Lưu ý, cả hai giá trị này đều được lấy từ tham số on-chain `TallyParams`, có thể thay đổi bởi governance.
Điều này có nghĩa proposal được chấp nhận iff:

* Tồn tại bonded token.
* Đã đạt Quorum.
* Tỷ lệ phiếu `Abstain` nhỏ hơn 1/1.
* Tỷ lệ phiếu `NoWithVeto` nhỏ hơn 1/3, bao gồm phiếu `Abstain`.
* Tỷ lệ phiếu `Yes`, không tính phiếu `Abstain`, vào cuối voting period lớn hơn 1/2.

Đối với expedited proposal, theo mặc định, threshold cao hơn so với *normal proposal*, cụ thể là 66.7%.

#### Inheritance

Nếu delegator không bỏ phiếu, họ sẽ kế thừa phiếu của validator.

* Nếu delegator bỏ phiếu trước validator, họ sẽ không kế thừa phiếu của validator.
* Nếu delegator bỏ phiếu sau validator, họ sẽ ghi đè phiếu của validator bằng phiếu của chính họ. Nếu proposal khẩn cấp, có thể vote sẽ đóng trước khi delegator có cơ hội phản ứng và ghi đè phiếu của validator. Điều này không phải vấn đề, vì proposal yêu cầu hơn 2/3 tổng quyền bỏ phiếu để thông qua, khi tổng hợp vào cuối voting period. Vì chỉ cần 1/3 + 1 quyền validation có thể cấu kết để kiểm duyệt giao dịch, việc không cấu kết đã được giả định cho các phạm vi vượt ngưỡng này.

#### Hình phạt validator vì không bỏ phiếu

Hiện tại, validator không bị phạt vì không bỏ phiếu.

#### Governance address

Sau này, chúng ta có thể thêm permissioned key chỉ có thể ký tx từ một số module nhất định. Đối với MVP, `Governance address` sẽ là địa chỉ validator chính được tạo khi tạo account. Địa chỉ này tương ứng với PrivKey khác với CometBFT PrivKey chịu trách nhiệm ký consensus message. Do đó validator không phải ký giao dịch governance bằng CometBFT PrivKey nhạy cảm.

#### Burnable Params

Có ba tham số định nghĩa deposit của proposal có nên bị burn hay hoàn trả cho người deposit.

* `BurnVoteVeto` burn deposit của proposal nếu proposal bị veto.
* `BurnVoteQuorum` burn deposit của proposal nếu vote không đạt quorum.
* `BurnProposalDepositPrevote` burn deposit của proposal nếu nó không vào giai đoạn voting.

> Lưu ý: Các tham số này có thể thay đổi qua governance.

## State

### Constitution

`Constitution` được tìm thấy trong genesis state. Đây là trường chuỗi dùng để mô tả mục đích của một blockchain cụ thể và các chuẩn mực mong đợi của nó. Một số ví dụ về cách sử dụng trường constitution:

* định nghĩa mục đích của chain, đặt nền tảng cho sự phát triển tương lai
* đặt kỳ vọng cho delegator
* đặt kỳ vọng cho validator
* định nghĩa mối quan hệ của chain với các thực thể "meatspace", như foundation hoặc corporation

Vì đây là tính năng xã hội hơn là kỹ thuật, chúng ta sẽ đi vào một số mục có thể hữu ích trong genesis constitution:

* Có giới hạn nào về governance không?
    * cộng đồng có được phép slash ví của whale mà họ không còn muốn xung quanh không? (ví dụ: Juno Proposal 4 và 16)
    * governance có thể "socially slash" validator đang sử dụng MEV chưa được phê duyệt không? (ví dụ: commonwealth.im/osmosis)
    * Trong trường hợp khẩn cấp kinh tế, validator nên làm gì?
        * Sự sụp đổ Terra tháng 5/2022, validator chọn chạy binary mới với code chưa được governance phê duyệt, vì governance token đã bị lạm phát về không.
* Mục đích cụ thể của chain là gì?
    * ví dụ điển hình nhất là Cosmos hub, nơi các nhóm sáng lập khác nhau có cách hiểu khác nhau về mục đích của mạng.

Mục genesis "constitution" này chưa được thiết kế cho các chain hiện có, những chain nên có thể chỉ phê chuẩn constitution bằng hệ thống governance của họ. Thay vào đó, đây dành cho chain mới. Nó cho phép validator có ý tưởng rõ ràng hơn nhiều về mục đích và kỳ vọng đặt lên họ khi vận hành node. Tương tự, đối với thành viên cộng đồng, constitution sẽ cho họ ý tưởng về những gì mong đợi từ cả "chain team" và validator.

Constitution này được thiết kế bất biến và chỉ đặt trong genesis, mặc dù có thể thay đổi theo thời gian qua pull request tới cosmos-sdk cho phép constitution được thay đổi bởi governance. Cộng đồng muốn sửa đổi constitution gốc nên sử dụng cơ chế governance và "signaling proposal" để làm điều đó.

**Kịch bản sử dụng lý tưởng cho constitution chain cosmos**

Là chain developer, bạn quyết định muốn cung cấp sự rõ ràng cho các nhóm người dùng chính:

* validator
* token holder
* developer (chính bạn)

Bạn sử dụng constitution để lưu trữ Markdown bất biến trong genesis, để khi có câu hỏi khó, constitution có thể cung cấp hướng dẫn cho cộng đồng.

### Proposals

Đối tượng `Proposal` được sử dụng để tổng hợp phiếu và theo dõi trạng thái proposal nói chung.
Chúng chứa mảng các `sdk.Msg` tùy ý mà governance module sẽ cố gắng resolve và thực thi nếu proposal được thông qua. `Proposal` được xác định bởi id duy nhất và chứa một chuỗi timestamp: `submit_time`, `deposit_end_time`, `voting_start_time`, `voting_end_time` theo dõi vòng đời của proposal

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/gov/v1/gov.proto#L51-L99
```

Proposal thường cần nhiều hơn chỉ một tập message để giải thích mục đích nhưng cần thêm lý do và cho phép phương tiện để người tham gia quan tâm thảo luận và tranh luận proposal.
Trong hầu hết trường hợp, **khuyến khích có hệ thống off-chain hỗ trợ quy trình governance on-chain**.
Để phục vụ điều này, proposal chứa trường **`metadata`** đặc biệt, một chuỗi, có thể dùng để thêm ngữ cảnh cho proposal. Trường `metadata` cho phép sử dụng tùy chỉnh cho các mạng, tuy nhiên, mong đợi trường chứa URL hoặc dạng CID sử dụng hệ thống như [IPFS](https://docs.ipfs.io/concepts/content-addressing/). Để hỗ trợ trường hợp tương tác giữa các mạng, SDK khuyến nghị `metadata` biểu diễn template `JSON` sau:

```json
{
  "title": "...",
  "description": "...",
  "forum": "...", // a link to the discussion platform (i.e. Discord)
  "other": "..." // any extra data that doesn't correspond to the other fields
}
```

Điều này giúp client hỗ trợ nhiều mạng dễ dàng hơn nhiều.

Metadata có độ dài tối đa do app developer chọn và truyền vào gov keeper dưới dạng config. Độ dài tối đa mặc định trong SDK là 255 ký tự.

#### Viết module sử dụng governance

Có nhiều khía cạnh của chain hoặc của các module riêng lẻ mà bạn có thể muốn sử dụng governance để thực hiện như thay đổi các tham số khác nhau. Điều này rất đơn giản. Đầu tiên, viết các message type và implementation `MsgServer`. Thêm trường `authority` vào keeper sẽ được điền trong constructor với governance module account: `govKeeper.GetGovernanceAccount().GetAddress()`. Sau đó với các method trong `msg_server.go`, thực hiện kiểm tra message rằng signer khớp với `authority`. Điều này sẽ ngăn bất kỳ user nào thực thi message đó.

### Parameters and base types

`Parameters` định nghĩa các quy tắc theo đó vote được chạy. Chỉ có thể có một bộ tham số active tại bất kỳ thời điểm nào. Nếu governance muốn thay đổi bộ tham số, để sửa giá trị hoặc thêm/xóa trường tham số, phải tạo bộ tham số mới và vô hiệu hóa bộ trước đó.

#### DepositParams

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/gov/v1/gov.proto#L152-L162
```

#### VotingParams

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/gov/v1/gov.proto#L164-L168
```

#### TallyParams

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/gov/v1/gov.proto#L170-L182
```

Parameters được lưu trong KVStore `GlobalParams` toàn cục.

Ngoài ra, chúng ta giới thiệu một số base type:

```go
type Vote byte

const (
    VoteYes         = 0x1
    VoteNo          = 0x2
    VoteNoWithVeto  = 0x3
    VoteAbstain     = 0x4
)

type ProposalType string

const (
    ProposalTypePlainText       = "Text"
    ProposalTypeSoftwareUpgrade = "SoftwareUpgrade"
)

type ProposalStatus byte


const (
    StatusNil           ProposalStatus = 0x00
    StatusDepositPeriod ProposalStatus = 0x01  // Proposal is submitted. Participants can deposit on it but not vote
    StatusVotingPeriod  ProposalStatus = 0x02  // MinDeposit is reached, participants can vote
    StatusPassed        ProposalStatus = 0x03  // Proposal passed and successfully executed
    StatusRejected      ProposalStatus = 0x04  // Proposal has been rejected
    StatusFailed        ProposalStatus = 0x05  // Proposal passed but failed execution
)
```

### Deposit

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/gov/v1/gov.proto#L38-L49
```

### ValidatorGovInfo

Type này được sử dụng trong temp map khi tổng hợp

```go
  type ValidatorGovInfo struct {
    Minus     sdk.Dec
    Vote      Vote
  }
```

## Stores

:::note
Stores là KVStores trong multi-store. Key để tìm store là tham số đầu tiên trong danh sách
:::

Chúng ta sẽ sử dụng một KVStore `Governance` để lưu bốn ánh xạ:

* Ánh xạ từ `proposalID|'proposal'` đến `Proposal`.
* Ánh xạ từ `proposalID|'addresses'|address` đến `Vote`. Ánh xạ này cho phép truy vấn tất cả địa chỉ đã bỏ phiếu cho proposal cùng phiếu của họ bằng range query trên `proposalID:addresses`.
* Ánh xạ từ `ParamsKey|'Params'` đến `Params`. Map này cho phép truy vấn tất cả params x/gov.
* Ánh xạ từ `VotingPeriodProposalKeyPrefix|proposalID` đến một byte. Điều này cho phép biết proposal có trong voting period hay không với chi phí gas rất thấp.

Với mục đích pseudocode, đây là hai function chúng ta sẽ dùng để đọc hoặc ghi trong stores:

* `load(StoreKey, Key)`: Lấy item lưu tại key `Key` trong store tìm tại key `StoreKey` trong multistore
* `store(StoreKey, Key, value)`: Ghi value `Value` tại key `Key` trong store tìm tại key `StoreKey` trong multistore

### Proposal Processing Queue

**Store:**

* `ProposalProcessingQueue`: Hàng đợi `queue[proposalID]` chứa tất cả `ProposalIDs` của proposal đã đạt `MinDeposit`. Trong mỗi `EndBlock`, tất cả proposal đã đến cuối voting period được xử lý. Để xử lý proposal hoàn thành, ứng dụng tổng hợp phiếu, tính phiếu của mỗi validator và kiểm tra mọi validator trong validator set đã bỏ phiếu chưa. Nếu proposal được chấp nhận, deposit được hoàn trả. Cuối cùng, `Handler` nội dung proposal được thực thi.

Và pseudocode cho `ProposalProcessingQueue`:

```go
  in EndBlock do

    for finishedProposalID in GetAllFinishedProposalIDs(block.Time)
      proposal = load(Governance, <proposalID|'proposal'>) // proposal is a const key

      validators = Keeper.getAllValidators()
      tmpValMap := map(sdk.AccAddress)ValidatorGovInfo

      // Initiate mapping at 0. This is the amount of shares of the validator's vote that will be overridden by their delegator's votes
      for each validator in validators
        tmpValMap(validator.OperatorAddr).Minus = 0

      // Tally
      voterIterator = rangeQuery(Governance, <proposalID|'addresses'>) //return all the addresses that voted on the proposal
      for each (voterAddress, vote) in voterIterator
        delegations = stakingKeeper.getDelegations(voterAddress) // get all delegations for current voter

        for each delegation in delegations
          // make sure delegation.Shares does NOT include shares being unbonded
          tmpValMap(delegation.ValidatorAddr).Minus += delegation.Shares
          proposal.updateTally(vote, delegation.Shares)

        _, isVal = stakingKeeper.getValidator(voterAddress)
        if (isVal)
          tmpValMap(voterAddress).Vote = vote

      tallyingParam = load(GlobalParams, 'TallyingParam')

      // Update tally if validator voted
      for each validator in validators
        if tmpValMap(validator).HasVoted
          proposal.updateTally(tmpValMap(validator).Vote, (validator.TotalShares - tmpValMap(validator).Minus))



      // Check if proposal is accepted or rejected
      totalNonAbstain := proposal.YesVotes + proposal.NoVotes + proposal.NoWithVetoVotes
      if (proposal.Votes.YesVotes/totalNonAbstain > tallyingParam.Threshold AND proposal.Votes.NoWithVetoVotes/totalNonAbstain  < tallyingParam.Veto)
        //  proposal was accepted at the end of the voting period
        //  refund deposits (non-voters already punished)
        for each (amount, depositor) in proposal.Deposits
          depositor.AtomBalance += amount

        stateWriter, err := proposal.Handler()
        if err != nil
            // proposal passed but failed during state execution
            proposal.CurrentStatus = ProposalStatusFailed
         else
            // proposal pass and state is persisted
            proposal.CurrentStatus = ProposalStatusAccepted
            stateWriter.save()
      else
        // proposal was rejected
        proposal.CurrentStatus = ProposalStatusRejected

      store(Governance, <proposalID|'proposal'>, proposal)
```

### Legacy Proposal

:::warning
Legacy proposals đã deprecated. Sử dụng luồng proposal mới bằng cách cấp quyền cho governance module thực thi message.
:::

Legacy proposal là implementation cũ của governance proposal.
Khác với proposal có thể chứa bất kỳ message nào, legacy proposal cho phép gửi tập proposal được định nghĩa trước. Các proposal này được định nghĩa theo type và xử lý bởi handler đăng ký trong gov v1beta1 router.

Thêm thông tin về cách gửi proposal trong [client section](#client).

## Messages

### Proposal Submission

Proposal có thể được gửi bởi bất kỳ account nào qua giao dịch `MsgSubmitProposal`.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/gov/v1/tx.proto#L42-L69
```

Tất cả `sdk.Msgs` truyền vào trường `messages` của message `MsgSubmitProposal` phải được đăng ký trong `MsgServiceRouter` của app. Mỗi message này phải có một signer, cụ thể là gov module account. Và cuối cùng, độ dài metadata không được lớn hơn config `maxMetadataLen` truyền vào gov keeper.
`initialDeposit` phải dương chặt và tuân theo denom được chấp nhận của tham số `MinDeposit`.

**Sửa đổi state:**

* Tạo `proposalID` mới
* Tạo `Proposal` mới
* Khởi tạo thuộc tính của `Proposal`
* Giảm balance của sender đi `InitialDeposit`
* Nếu đạt `MinDeposit`:
    * Đẩy `proposalID` vào `ProposalProcessingQueue`
* Chuyển `InitialDeposit` từ `Proposer` sang governance `ModuleAccount`

### Deposit

Khi proposal được gửi, nếu `Proposal.TotalDeposit < ActiveParam.MinDeposit`, người nắm giữ Atom có thể gửi giao dịch `MsgDeposit` để tăng deposit của proposal.

Deposit được chấp nhận iff:

* Proposal tồn tại
* Proposal không trong voting period
* Coin deposit tuân theo denom được chấp nhận từ tham số `MinDeposit`

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/gov/v1/tx.proto#L134-L147
```

**Sửa đổi state:**

* Giảm balance của sender đi `deposit`
* Thêm `deposit` của sender vào `proposal.Deposits`
* Tăng `proposal.TotalDeposit` bằng `deposit` của sender
* Nếu đạt `MinDeposit`:
    * Đẩy `proposalID` vào `ProposalProcessingQueueEnd`
* Chuyển `Deposit` từ `proposer` sang governance `ModuleAccount`

### Vote

Khi đạt `ActiveParam.MinDeposit`, voting period bắt đầu. Từ đó, người nắm giữ Atom đã bond có thể gửi giao dịch `MsgVote` để bỏ phiếu cho proposal.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/gov/v1/tx.proto#L92-L108
```

**Sửa đổi state:**

* Ghi `Vote` của sender

:::note
Chi phí gas cho message này phải tính đến việc tổng hợp phiếu trong EndBlocker sau này.
:::

## Events

Module governance phát ra các event sau:

### EndBlocker

| Type              | Attribute Key   | Attribute Value  |
|-------------------|-----------------|------------------|
| inactive_proposal | proposal_id     | {proposalID}     |
| inactive_proposal | proposal_result | {proposalResult} |
| active_proposal   | proposal_id     | {proposalID}     |
| active_proposal   | proposal_result | {proposalResult} |

### Handlers

#### MsgSubmitProposal

| Type                | Attribute Key       | Attribute Value |
|---------------------|---------------------|-----------------|
| submit_proposal     | proposal_id         | {proposalID}    |
| submit_proposal [0] | voting_period_start | {proposalID}    |
| proposal_deposit    | amount              | {depositAmount} |
| proposal_deposit    | proposal_id         | {proposalID}    |
| message             | module              | governance      |
| message             | action              | submit_proposal |
| message             | sender              | {senderAddress} |

* [0] Event chỉ phát ra nếu voting period bắt đầu trong lúc submission.

#### MsgVote

| Type          | Attribute Key | Attribute Value |
|---------------|---------------|-----------------|
| proposal_vote | option        | {voteOption}    |
| proposal_vote | proposal_id   | {proposalID}    |
| message       | module        | governance      |
| message       | action        | vote            |
| message       | sender        | {senderAddress} |

#### MsgVoteWeighted

| Type          | Attribute Key | Attribute Value       |
|---------------|---------------|-----------------------|
| proposal_vote | option        | {weightedVoteOptions} |
| proposal_vote | proposal_id   | {proposalID}          |
| message       | module        | governance            |
| message       | action        | vote                  |
| message       | sender        | {senderAddress}       |

#### MsgDeposit

| Type                 | Attribute Key       | Attribute Value |
|----------------------|---------------------|-----------------|
| proposal_deposit     | amount              | {depositAmount} |
| proposal_deposit     | proposal_id         | {proposalID}    |
| proposal_deposit [0] | voting_period_start | {proposalID}    |
| message              | module              | governance      |
| message              | action              | deposit         |
| message              | sender              | {senderAddress} |

* [0] Event chỉ phát ra nếu voting period bắt đầu trong lúc submission.

## Parameters

Module governance chứa các tham số sau:

| Key                           | Type             | Example                                 |
|-------------------------------|------------------|-----------------------------------------|
| min_deposit                   | array (coins)    | [{"denom":"uatom","amount":"10000000"}] |
| max_deposit_period            | string (time ns) | "172800000000000" (17280s)              |
| voting_period                 | string (time ns) | "172800000000000" (17280s)              |
| quorum                        | string (dec)     | "0.334000000000000000"                  |
| threshold                     | string (dec)     | "0.500000000000000000"                  |
| veto                          | string (dec)     | "0.334000000000000000"                  |
| expedited_threshold           | string (time ns) | "0.667000000000000000"                  |
| expedited_voting_period       | string (time ns) | "86400000000000" (8600s)                |
| expedited_min_deposit         | array (coins)    | [{"denom":"uatom","amount":"50000000"}] |
| burn_proposal_deposit_prevote | bool             | false                                    |
| burn_vote_quorum              | bool             | false                                   |
| burn_vote_veto                | bool             | true                                    |
| min_initial_deposit_ratio                | string             | "0.1"                                    |


**NOTE**: Module governance chứa các tham số là object khác với các module khác. Nếu chỉ muốn thay đổi một tập con tham số, chỉ cần bao gồm chúng và không cần toàn bộ cấu trúc object tham số.

## Client

### CLI

Người dùng có thể truy vấn và tương tác với module `gov` bằng CLI.

#### Query

Lệnh `query` cho phép người dùng truy vấn state `gov`.

```bash
simd query gov --help
```

##### deposit

Lệnh `deposit` cho phép người dùng truy vấn deposit cho proposal nhất định từ depositor nhất định.

```bash
simd query gov deposit [proposal-id] [depositor-addr] [flags]
```

Example:

```bash
simd query gov deposit 1 cosmos1..
```

Example Output:

```bash
amount:
- amount: "100"
  denom: stake
depositor: cosmos1..
proposal_id: "1"
```

##### deposits

Lệnh `deposits` cho phép người dùng truy vấn tất cả deposit cho proposal nhất định.

```bash
simd query gov deposits [proposal-id] [flags]
```

Example:

```bash
simd query gov deposits 1
```

Example Output:

```bash
deposits:
- amount:
  - amount: "100"
    denom: stake
  depositor: cosmos1..
  proposal_id: "1"
pagination:
  next_key: null
  total: "0"
```

##### param

Lệnh `param` cho phép người dùng truy vấn tham số nhất định cho module `gov`.

```bash
simd query gov param [param-type] [flags]
```

Example:

```bash
simd query gov param voting
```

Example Output:

```bash
voting_period: "172800000000000"
```

##### params

Lệnh `params` cho phép người dùng truy vấn tất cả tham số cho module `gov`.

```bash
simd query gov params [flags]
```

Example:

```bash
simd query gov params
```

Example Output:

```bash
deposit_params:
  max_deposit_period: 172800s
  min_deposit:
  - amount: "10000000"
    denom: stake
params:
  expedited_min_deposit:
  - amount: "50000000"
    denom: stake
  expedited_threshold: "0.670000000000000000"
  expedited_voting_period: 86400s
  max_deposit_period: 172800s
  min_deposit:
  - amount: "10000000"
    denom: stake
  min_initial_deposit_ratio: "0.000000000000000000"
  proposal_cancel_burn_rate: "0.500000000000000000"
  quorum: "0.334000000000000000"
  threshold: "0.500000000000000000"
  veto_threshold: "0.334000000000000000"
  voting_period: 172800s
tally_params:
  quorum: "0.334000000000000000"
  threshold: "0.500000000000000000"
  veto_threshold: "0.334000000000000000"
voting_params:
  voting_period: 172800s
```

##### proposal

Lệnh `proposal` cho phép người dùng truy vấn proposal nhất định.

```bash
simd query gov proposal [proposal-id] [flags]
```

Example:

```bash
simd query gov proposal 1
```

Example Output:

```bash
deposit_end_time: "2022-03-30T11:50:20.819676256Z"
final_tally_result:
  abstain_count: "0"
  no_count: "0"
  no_with_veto_count: "0"
  yes_count: "0"
id: "1"
messages:
- '@type': /cosmos.bank.v1beta1.MsgSend
  amount:
  - amount: "10"
    denom: stake
  from_address: cosmos1..
  to_address: cosmos1..
metadata: AQ==
status: PROPOSAL_STATUS_DEPOSIT_PERIOD
submit_time: "2022-03-28T11:50:20.819676256Z"
total_deposit:
- amount: "10"
  denom: stake
voting_end_time: null
voting_start_time: null
```

##### proposals

Lệnh `proposals` cho phép người dùng truy vấn tất cả proposal với bộ lọc tùy chọn.

```bash
simd query gov proposals [flags]
```

Example:

```bash
simd query gov proposals
```

Example Output:

```bash
pagination:
  next_key: null
  total: "0"
proposals:
- deposit_end_time: "2022-03-30T11:50:20.819676256Z"
  final_tally_result:
    abstain_count: "0"
    no_count: "0"
    no_with_veto_count: "0"
    yes_count: "0"
  id: "1"
  messages:
  - '@type': /cosmos.bank.v1beta1.MsgSend
    amount:
    - amount: "10"
      denom: stake
    from_address: cosmos1..
    to_address: cosmos1..
  metadata: AQ==
  status: PROPOSAL_STATUS_DEPOSIT_PERIOD
  submit_time: "2022-03-28T11:50:20.819676256Z"
  total_deposit:
  - amount: "10"
    denom: stake
  voting_end_time: null
  voting_start_time: null
- deposit_end_time: "2022-03-30T14:02:41.165025015Z"
  final_tally_result:
    abstain_count: "0"
    no_count: "0"
    no_with_veto_count: "0"
    yes_count: "0"
  id: "2"
  messages:
  - '@type': /cosmos.bank.v1beta1.MsgSend
    amount:
    - amount: "10"
      denom: stake
    from_address: cosmos1..
    to_address: cosmos1..
  metadata: AQ==
  status: PROPOSAL_STATUS_DEPOSIT_PERIOD
  submit_time: "2022-03-28T14:02:41.165025015Z"
  total_deposit:
  - amount: "10"
    denom: stake
  voting_end_time: null
  voting_start_time: null
```

##### proposer

Lệnh `proposer` cho phép người dùng truy vấn proposer cho proposal nhất định.

```bash
simd query gov proposer [proposal-id] [flags]
```

Example:

```bash
simd query gov proposer 1
```

Example Output:

```bash
proposal_id: "1"
proposer: cosmos1..
```

##### tally

Lệnh `tally` cho phép người dùng truy vấn tổng hợp phiếu của proposal vote nhất định.

```bash
simd query gov tally [proposal-id] [flags]
```

Example:

```bash
simd query gov tally 1
```

Example Output:

```bash
abstain: "0"
"no": "0"
no_with_veto: "0"
"yes": "1"
```

##### vote

Lệnh `vote` cho phép người dùng truy vấn phiếu cho proposal nhất định.

```bash
simd query gov vote [proposal-id] [voter-addr] [flags]
```

Example:

```bash
simd query gov vote 1 cosmos1..
```

Example Output:

```bash
option: VOTE_OPTION_YES
options:
- option: VOTE_OPTION_YES
  weight: "1.000000000000000000"
proposal_id: "1"
voter: cosmos1..
```

##### votes

Lệnh `votes` cho phép người dùng truy vấn tất cả phiếu cho proposal nhất định.

```bash
simd query gov votes [proposal-id] [flags]
```

Example:

```bash
simd query gov votes 1
```

Example Output:

```bash
pagination:
  next_key: null
  total: "0"
votes:
- option: VOTE_OPTION_YES
  options:
  - option: VOTE_OPTION_YES
    weight: "1.000000000000000000"
  proposal_id: "1"
  voter: cosmos1..
```

#### Transactions

Lệnh `tx` cho phép người dùng tương tác với module `gov`.

```bash
simd tx gov --help
```

##### deposit

Lệnh `deposit` cho phép người dùng deposit token cho proposal nhất định.

```bash
simd tx gov deposit [proposal-id] [deposit] [flags]
```

Example:

```bash
simd tx gov deposit 1 10000000stake --from cosmos1..
```

##### draft-proposal

Lệnh `draft-proposal` cho phép người dùng soạn thảo proposal bất kỳ loại nào.
Lệnh trả về `draft_proposal.json`, dùng cho `submit-proposal` sau khi hoàn thành.
`draft_metadata.json` dùng để upload lên [IPFS](#metadata).

```bash
simd tx gov draft-proposal
```

##### submit-proposal

Lệnh `submit-proposal` cho phép người dùng gửi governance proposal cùng một số message và metadata.
Messages, metadata và deposit được định nghĩa trong file JSON.

```bash
simd tx gov submit-proposal [path-to-proposal-json] [flags]
```

Example:

```bash
simd tx gov submit-proposal /path/to/proposal.json --from cosmos1..
```

trong đó `proposal.json` chứa:

```json
{
  "messages": [
    {
      "@type": "/cosmos.bank.v1beta1.MsgSend",
      "from_address": "cosmos1...", // The gov module address
      "to_address": "cosmos1...",
      "amount":[{"denom": "stake","amount": "10"}]
    }
  ],
  "metadata": "AQ==",
  "deposit": "10stake",
  "title": "Proposal Title",
  "summary": "Proposal Summary"
}
```

:::note
Theo mặc định metadata, summary và title đều giới hạn 255 ký tự, điều này có thể được ghi đè bởi application developer.
:::

:::tip
Khi metadata không được chỉ định, title giới hạn 255 ký tự và summary 40x độ dài title.
:::

##### submit-legacy-proposal

Lệnh `submit-legacy-proposal` cho phép người dùng gửi governance legacy proposal cùng deposit ban đầu.

```bash
simd tx gov submit-legacy-proposal [command] [flags]
```

Example:

```bash
simd tx gov submit-legacy-proposal --title="Test Proposal" --description="testing" --type="Text" --deposit="100000000stake" --from cosmos1..
```

Example (`param-change`):

```bash
simd tx gov submit-legacy-proposal param-change proposal.json --from cosmos1..
```

```json
{
  "title": "Test Proposal",
  "description": "testing, testing, 1, 2, 3",
  "changes": [
    {
      "subspace": "staking",
      "key": "MaxValidators",
      "value": 100
    }
  ],
  "deposit": "10000000stake"
}
```

#### cancel-proposal

Khi proposal bị hủy, từ deposit của proposal `deposits * proposal_cancel_ratio` sẽ bị burn hoặc gửi đến địa chỉ `ProposalCancelDest`, nếu `ProposalCancelDest` rỗng thì deposit sẽ bị burn. `remaining deposits` sẽ được gửi cho người deposit.

```bash
simd tx gov cancel-proposal [proposal-id] [flags]
```

Example:

```bash
simd tx gov cancel-proposal 1 --from cosmos1...
```

##### vote

Lệnh `vote` cho phép người dùng gửi phiếu cho governance proposal nhất định.

```bash
simd tx gov vote [command] [flags]
```

Example:

```bash
simd tx gov vote 1 yes --from cosmos1..
```

##### weighted-vote

Lệnh `weighted-vote` cho phép người dùng gửi weighted vote cho governance proposal nhất định.

```bash
simd tx gov weighted-vote [proposal-id] [weighted-options] [flags]
```

Example:

```bash
simd tx gov weighted-vote 1 yes=0.5,no=0.5 --from cosmos1..
```

### gRPC

Người dùng có thể truy vấn module `gov` bằng gRPC endpoint.

#### Proposal

Endpoint `Proposal` cho phép người dùng truy vấn proposal nhất định.

Using legacy v1beta1:

```bash
cosmos.gov.v1beta1.Query/Proposal
```

Example:

```bash
grpcurl -plaintext \
    -d '{"proposal_id":"1"}' \
    localhost:9090 \
    cosmos.gov.v1beta1.Query/Proposal
```

Example Output:

```bash
{
  "proposal": {
    "proposalId": "1",
    "content": {"@type":"/cosmos.gov.v1beta1.TextProposal","description":"testing, testing, 1, 2, 3","title":"Test Proposal"},
    "status": "PROPOSAL_STATUS_VOTING_PERIOD",
    "finalTallyResult": {
      "yes": "0",
      "abstain": "0",
      "no": "0",
      "noWithVeto": "0"
    },
    "submitTime": "2021-09-16T19:40:08.712440474Z",
    "depositEndTime": "2021-09-18T19:40:08.712440474Z",
    "totalDeposit": [
      {
        "denom": "stake",
        "amount": "10000000"
      }
    ],
    "votingStartTime": "2021-09-16T19:40:08.712440474Z",
    "votingEndTime": "2021-09-18T19:40:08.712440474Z",
    "title": "Test Proposal",
    "summary": "testing, testing, 1, 2, 3"
  }
}
```

Using v1:

```bash
cosmos.gov.v1.Query/Proposal
```

Example:

```bash
grpcurl -plaintext \
    -d '{"proposal_id":"1"}' \
    localhost:9090 \
    cosmos.gov.v1.Query/Proposal
```

Example Output:

```bash
{
  "proposal": {
    "id": "1",
    "messages": [
      {"@type":"/cosmos.bank.v1beta1.MsgSend","amount":[{"denom":"stake","amount":"10"}],"fromAddress":"cosmos1..","toAddress":"cosmos1.."}
    ],
    "status": "PROPOSAL_STATUS_VOTING_PERIOD",
    "finalTallyResult": {
      "yesCount": "0",
      "abstainCount": "0",
      "noCount": "0",
      "noWithVetoCount": "0"
    },
    "submitTime": "2022-03-28T11:50:20.819676256Z",
    "depositEndTime": "2022-03-30T11:50:20.819676256Z",
    "totalDeposit": [
      {
        "denom": "stake",
        "amount": "10000000"
      }
    ],
    "votingStartTime": "2022-03-28T14:25:26.644857113Z",
    "votingEndTime": "2022-03-30T14:25:26.644857113Z",
    "metadata": "AQ==",
    "title": "Test Proposal",
    "summary": "testing, testing, 1, 2, 3"
  }
}
```

#### Proposals

Endpoint `Proposals` cho phép người dùng truy vấn tất cả proposal với bộ lọc tùy chọn.

Using legacy v1beta1:

```bash
cosmos.gov.v1beta1.Query/Proposals
```

Example:

```bash
grpcurl -plaintext \
    localhost:9090 \
    cosmos.gov.v1beta1.Query/Proposals
```

Example Output:

```bash
{
  "proposals": [
    {
      "proposalId": "1",
      "status": "PROPOSAL_STATUS_VOTING_PERIOD",
      "finalTallyResult": {
        "yes": "0",
        "abstain": "0",
        "no": "0",
        "noWithVeto": "0"
      },
      "submitTime": "2022-03-28T11:50:20.819676256Z",
      "depositEndTime": "2022-03-30T11:50:20.819676256Z",
      "totalDeposit": [
        {
          "denom": "stake",
          "amount": "10000000010"
        }
      ],
      "votingStartTime": "2022-03-28T14:25:26.644857113Z",
      "votingEndTime": "2022-03-30T14:25:26.644857113Z"
    },
    {
      "proposalId": "2",
      "status": "PROPOSAL_STATUS_DEPOSIT_PERIOD",
      "finalTallyResult": {
        "yes": "0",
        "abstain": "0",
        "no": "0",
        "noWithVeto": "0"
      },
      "submitTime": "2022-03-28T14:02:41.165025015Z",
      "depositEndTime": "2022-03-30T14:02:41.165025015Z",
      "totalDeposit": [
        {
          "denom": "stake",
          "amount": "10"
        }
      ],
      "votingStartTime": "0001-01-01T00:00:00Z",
      "votingEndTime": "0001-01-01T00:00:00Z"
    }
  ],
  "pagination": {
    "total": "2"
  }
}
```

Using v1:

```bash
cosmos.gov.v1.Query/Proposals
```

Example:

```bash
grpcurl -plaintext \
    localhost:9090 \
    cosmos.gov.v1.Query/Proposals
```

Example Output:

```bash
{
  "proposals": [
    {
      "id": "1",
      "messages": [
        {"@type":"/cosmos.bank.v1beta1.MsgSend","amount":[{"denom":"stake","amount":"10"}],"fromAddress":"cosmos1..","toAddress":"cosmos1.."}
      ],
      "status": "PROPOSAL_STATUS_VOTING_PERIOD",
      "finalTallyResult": {
        "yesCount": "0",
        "abstainCount": "0",
        "noCount": "0",
        "noWithVetoCount": "0"
      },
      "submitTime": "2022-03-28T11:50:20.819676256Z",
      "depositEndTime": "2022-03-30T11:50:20.819676256Z",
      "totalDeposit": [
        {
          "denom": "stake",
          "amount": "10000000010"
        }
      ],
      "votingStartTime": "2022-03-28T14:25:26.644857113Z",
      "votingEndTime": "2022-03-30T14:25:26.644857113Z",
      "metadata": "AQ==",
      "title": "Proposal Title",
      "summary": "Proposal Summary"
    },
    {
      "id": "2",
      "messages": [
        {"@type":"/cosmos.bank.v1beta1.MsgSend","amount":[{"denom":"stake","amount":"10"}],"fromAddress":"cosmos1..","toAddress":"cosmos1.."}
      ],
      "status": "PROPOSAL_STATUS_DEPOSIT_PERIOD",
      "finalTallyResult": {
        "yesCount": "0",
        "abstainCount": "0",
        "noCount": "0",
        "noWithVetoCount": "0"
      },
      "submitTime": "2022-03-28T14:02:41.165025015Z",
      "depositEndTime": "2022-03-30T14:02:41.165025015Z",
      "totalDeposit": [
        {
          "denom": "stake",
          "amount": "10"
        }
      ],
      "metadata": "AQ==",
      "title": "Proposal Title",
      "summary": "Proposal Summary"
    }
  ],
  "pagination": {
    "total": "2"
  }
}
```

#### Vote

Endpoint `Vote` cho phép người dùng truy vấn phiếu cho proposal nhất định.

Using legacy v1beta1:

```bash
cosmos.gov.v1beta1.Query/Vote
```

Example:

```bash
grpcurl -plaintext \
    -d '{"proposal_id":"1","voter":"cosmos1.."}' \
    localhost:9090 \
    cosmos.gov.v1beta1.Query/Vote
```

Example Output:

```bash
{
  "vote": {
    "proposalId": "1",
    "voter": "cosmos1..",
    "option": "VOTE_OPTION_YES",
    "options": [
      {
        "option": "VOTE_OPTION_YES",
        "weight": "1000000000000000000"
      }
    ]
  }
}
```

Using v1:

```bash
cosmos.gov.v1.Query/Vote
```

Example:

```bash
grpcurl -plaintext \
    -d '{"proposal_id":"1","voter":"cosmos1.."}' \
    localhost:9090 \
    cosmos.gov.v1.Query/Vote
```

Example Output:

```bash
{
  "vote": {
    "proposalId": "1",
    "voter": "cosmos1..",
    "option": "VOTE_OPTION_YES",
    "options": [
      {
        "option": "VOTE_OPTION_YES",
        "weight": "1.000000000000000000"
      }
    ]
  }
}
```

#### Votes

Endpoint `Votes` cho phép người dùng truy vấn tất cả phiếu cho proposal nhất định.

Using legacy v1beta1:

```bash
cosmos.gov.v1beta1.Query/Votes
```

Example:

```bash
grpcurl -plaintext \
    -d '{"proposal_id":"1"}' \
    localhost:9090 \
    cosmos.gov.v1beta1.Query/Votes
```

Example Output:

```bash
{
  "votes": [
    {
      "proposalId": "1",
      "voter": "cosmos1..",
      "options": [
        {
          "option": "VOTE_OPTION_YES",
          "weight": "1000000000000000000"
        }
      ]
    }
  ],
  "pagination": {
    "total": "1"
  }
}
```

Using v1:

```bash
cosmos.gov.v1.Query/Votes
```

Example:

```bash
grpcurl -plaintext \
    -d '{"proposal_id":"1"}' \
    localhost:9090 \
    cosmos.gov.v1.Query/Votes
```

Example Output:

```bash
{
  "votes": [
    {
      "proposalId": "1",
      "voter": "cosmos1..",
      "options": [
        {
          "option": "VOTE_OPTION_YES",
          "weight": "1.000000000000000000"
        }
      ]
    }
  ],
  "pagination": {
    "total": "1"
  }
}
```

#### Params

Endpoint `Params` cho phép người dùng truy vấn tất cả tham số cho module `gov`.

<!-- TODO: #10197 Querying governance params outputs nil values -->

Using legacy v1beta1:

```bash
cosmos.gov.v1beta1.Query/Params
```

Example:

```bash
grpcurl -plaintext \
    -d '{"params_type":"voting"}' \
    localhost:9090 \
    cosmos.gov.v1beta1.Query/Params
```

Example Output:

```bash
{
  "votingParams": {
    "votingPeriod": "172800s"
  },
  "depositParams": {
    "maxDepositPeriod": "0s"
  },
  "tallyParams": {
    "quorum": "MA==",
    "threshold": "MA==",
    "vetoThreshold": "MA=="
  }
}
```

Using v1:

```bash
cosmos.gov.v1.Query/Params
```

Example:

```bash
grpcurl -plaintext \
    -d '{"params_type":"voting"}' \
    localhost:9090 \
    cosmos.gov.v1.Query/Params
```

Example Output:

```bash
{
  "votingParams": {
    "votingPeriod": "172800s"
  }
}
```

#### Deposit

Endpoint `Deposit` cho phép người dùng truy vấn deposit cho proposal nhất định từ depositor nhất định.

Using legacy v1beta1:

```bash
cosmos.gov.v1beta1.Query/Deposit
```

Example:

```bash
grpcurl -plaintext \
    '{"proposal_id":"1","depositor":"cosmos1.."}' \
    localhost:9090 \
    cosmos.gov.v1beta1.Query/Deposit
```

Example Output:

```bash
{
  "deposit": {
    "proposalId": "1",
    "depositor": "cosmos1..",
    "amount": [
      {
        "denom": "stake",
        "amount": "10000000"
      }
    ]
  }
}
```

Using v1:

```bash
cosmos.gov.v1.Query/Deposit
```

Example:

```bash
grpcurl -plaintext \
    '{"proposal_id":"1","depositor":"cosmos1.."}' \
    localhost:9090 \
    cosmos.gov.v1.Query/Deposit
```

Example Output:

```bash
{
  "deposit": {
    "proposalId": "1",
    "depositor": "cosmos1..",
    "amount": [
      {
        "denom": "stake",
        "amount": "10000000"
      }
    ]
  }
}
```

#### deposits

Endpoint `Deposits` cho phép người dùng truy vấn tất cả deposit cho proposal nhất định.

Using legacy v1beta1:

```bash
cosmos.gov.v1beta1.Query/Deposits
```

Example:

```bash
grpcurl -plaintext \
    -d '{"proposal_id":"1"}' \
    localhost:9090 \
    cosmos.gov.v1beta1.Query/Deposits
```

Example Output:

```bash
{
  "deposits": [
    {
      "proposalId": "1",
      "depositor": "cosmos1..",
      "amount": [
        {
          "denom": "stake",
          "amount": "10000000"
        }
      ]
    }
  ],
  "pagination": {
    "total": "1"
  }
}
```

Using v1:

```bash
cosmos.gov.v1.Query/Deposits
```

Example:

```bash
grpcurl -plaintext \
    -d '{"proposal_id":"1"}' \
    localhost:9090 \
    cosmos.gov.v1.Query/Deposits
```

Example Output:

```bash
{
  "deposits": [
    {
      "proposalId": "1",
      "depositor": "cosmos1..",
      "amount": [
        {
          "denom": "stake",
          "amount": "10000000"
        }
      ]
    }
  ],
  "pagination": {
    "total": "1"
  }
}
```

#### TallyResult

Endpoint `TallyResult` cho phép người dùng truy vấn tổng hợp phiếu của proposal nhất định.

Using legacy v1beta1:

```bash
cosmos.gov.v1beta1.Query/TallyResult
```

Example:

```bash
grpcurl -plaintext \
    -d '{"proposal_id":"1"}' \
    localhost:9090 \
    cosmos.gov.v1beta1.Query/TallyResult
```

Example Output:

```bash
{
  "tally": {
    "yes": "1000000",
    "abstain": "0",
    "no": "0",
    "noWithVeto": "0"
  }
}
```

Using v1:

```bash
cosmos.gov.v1.Query/TallyResult
```

Example:

```bash
grpcurl -plaintext \
    -d '{"proposal_id":"1"}' \
    localhost:9090 \
    cosmos.gov.v1.Query/TallyResult
```

Example Output:

```bash
{
  "tally": {
    "yes": "1000000",
    "abstain": "0",
    "no": "0",
    "noWithVeto": "0"
  }
}
```

### REST

Người dùng có thể truy vấn module `gov` bằng REST endpoint.

#### proposal

Endpoint `proposals` cho phép người dùng truy vấn proposal nhất định.

Using legacy v1beta1:

```bash
/cosmos/gov/v1beta1/proposals/{proposal_id}
```

Example:

```bash
curl localhost:1317/cosmos/gov/v1beta1/proposals/1
```

Example Output:

```bash
{
  "proposal": {
    "proposal_id": "1",
    "content": null,
    "status": "PROPOSAL_STATUS_VOTING_PERIOD",
    "final_tally_result": {
      "yes": "0",
      "abstain": "0",
      "no": "0",
      "no_with_veto": "0"
    },
    "submit_time": "2022-03-28T11:50:20.819676256Z",
    "deposit_end_time": "2022-03-30T11:50:20.819676256Z",
    "total_deposit": [
      {
        "denom": "stake",
        "amount": "10000000010"
      }
    ],
    "voting_start_time": "2022-03-28T14:25:26.644857113Z",
    "voting_end_time": "2022-03-30T14:25:26.644857113Z"
  }
}
```

Using v1:

```bash
/cosmos/gov/v1/proposals/{proposal_id}
```

Example:

```bash
curl localhost:1317/cosmos/gov/v1/proposals/1
```

Example Output:

```bash
{
  "proposal": {
    "id": "1",
    "messages": [
      {
        "@type": "/cosmos.bank.v1beta1.MsgSend",
        "from_address": "cosmos1..",
        "to_address": "cosmos1..",
        "amount": [
          {
            "denom": "stake",
            "amount": "10"
          }
        ]
      }
    ],
    "status": "PROPOSAL_STATUS_VOTING_PERIOD",
    "final_tally_result": {
      "yes_count": "0",
      "abstain_count": "0",
      "no_count": "0",
      "no_with_veto_count": "0"
    },
    "submit_time": "2022-03-28T11:50:20.819676256Z",
    "deposit_end_time": "2022-03-30T11:50:20.819676256Z",
    "total_deposit": [
      {
        "denom": "stake",
        "amount": "10000000"
      }
    ],
    "voting_start_time": "2022-03-28T14:25:26.644857113Z",
    "voting_end_time": "2022-03-30T14:25:26.644857113Z",
    "metadata": "AQ==",
    "title": "Proposal Title",
    "summary": "Proposal Summary"
  }
}
```

#### proposals

Endpoint `proposals` cũng cho phép người dùng truy vấn tất cả proposal với bộ lọc tùy chọn.

Using legacy v1beta1:

```bash
/cosmos/gov/v1beta1/proposals
```

Example:

```bash
curl localhost:1317/cosmos/gov/v1beta1/proposals
```

Example Output:

```bash
{
  "proposals": [
    {
      "proposal_id": "1",
      "content": null,
      "status": "PROPOSAL_STATUS_VOTING_PERIOD",
      "final_tally_result": {
        "yes": "0",
        "abstain": "0",
        "no": "0",
        "no_with_veto": "0"
      },
      "submit_time": "2022-03-28T11:50:20.819676256Z",
      "deposit_end_time": "2022-03-30T11:50:20.819676256Z",
      "total_deposit": [
        {
          "denom": "stake",
          "amount": "10000000"
        }
      ],
      "voting_start_time": "2022-03-28T14:25:26.644857113Z",
      "voting_end_time": "2022-03-30T14:25:26.644857113Z"
    },
    {
      "proposal_id": "2",
      "content": null,
      "status": "PROPOSAL_STATUS_DEPOSIT_PERIOD",
      "final_tally_result": {
        "yes": "0",
        "abstain": "0",
        "no": "0",
        "no_with_veto": "0"
      },
      "submit_time": "2022-03-28T14:02:41.165025015Z",
      "deposit_end_time": "2022-03-30T14:02:41.165025015Z",
      "total_deposit": [
        {
          "denom": "stake",
          "amount": "10"
        }
      ],
      "voting_start_time": "0001-01-01T00:00:00Z",
      "voting_end_time": "0001-01-01T00:00:00Z"
    }
  ],
  "pagination": {
    "next_key": null,
    "total": "2"
  }
}
```

Using v1:

```bash
/cosmos/gov/v1/proposals
```

Example:

```bash
curl localhost:1317/cosmos/gov/v1/proposals
```

Example Output:

```bash
{
  "proposals": [
    {
      "id": "1",
      "messages": [
        {
          "@type": "/cosmos.bank.v1beta1.MsgSend",
          "from_address": "cosmos1..",
          "to_address": "cosmos1..",
          "amount": [
            {
              "denom": "stake",
              "amount": "10"
            }
          ]
        }
      ],
      "status": "PROPOSAL_STATUS_VOTING_PERIOD",
      "final_tally_result": {
        "yes_count": "0",
        "abstain_count": "0",
        "no_count": "0",
        "no_with_veto_count": "0"
      },
      "submit_time": "2022-03-28T11:50:20.819676256Z",
      "deposit_end_time": "2022-03-30T11:50:20.819676256Z",
      "total_deposit": [
        {
          "denom": "stake",
          "amount": "10000000010"
        }
      ],
      "voting_start_time": "2022-03-28T14:25:26.644857113Z",
      "voting_end_time": "2022-03-30T14:25:26.644857113Z",
      "metadata": "AQ==",
      "title": "Proposal Title",
      "summary": "Proposal Summary"
    },
    {
      "id": "2",
      "messages": [
        {
          "@type": "/cosmos.bank.v1beta1.MsgSend",
          "from_address": "cosmos1..",
          "to_address": "cosmos1..",
          "amount": [
            {
              "denom": "stake",
              "amount": "10"
            }
          ]
        }
      ],
      "status": "PROPOSAL_STATUS_DEPOSIT_PERIOD",
      "final_tally_result": {
        "yes_count": "0",
        "abstain_count": "0",
        "no_count": "0",
        "no_with_veto_count": "0"
      },
      "submit_time": "2022-03-28T14:02:41.165025015Z",
      "deposit_end_time": "2022-03-30T14:02:41.165025015Z",
      "total_deposit": [
        {
          "denom": "stake",
          "amount": "10"
        }
      ],
      "voting_start_time": null,
      "voting_end_time": null,
      "metadata": "AQ==",
      "title": "Proposal Title",
      "summary": "Proposal Summary"
    }
  ],
  "pagination": {
    "next_key": null,
    "total": "2"
  }
}
```

#### voter vote

Endpoint `votes` cho phép người dùng truy vấn phiếu cho proposal nhất định.

Using legacy v1beta1:

```bash
/cosmos/gov/v1beta1/proposals/{proposal_id}/votes/{voter}
```

Example:

```bash
curl localhost:1317/cosmos/gov/v1beta1/proposals/1/votes/cosmos1..
```

Example Output:

```bash
{
  "vote": {
    "proposal_id": "1",
    "voter": "cosmos1..",
    "option": "VOTE_OPTION_YES",
    "options": [
      {
        "option": "VOTE_OPTION_YES",
        "weight": "1.000000000000000000"
      }
    ]
  }
}
```

Using v1:

```bash
/cosmos/gov/v1/proposals/{proposal_id}/votes/{voter}
```

Example:

```bash
curl localhost:1317/cosmos/gov/v1/proposals/1/votes/cosmos1..
```

Example Output:

```bash
{
  "vote": {
    "proposal_id": "1",
    "voter": "cosmos1..",
    "options": [
      {
        "option": "VOTE_OPTION_YES",
        "weight": "1.000000000000000000"
      }
    ],
    "metadata": ""
  }
}
```

#### votes

Endpoint `votes` cho phép người dùng truy vấn tất cả phiếu cho proposal nhất định.

Using legacy v1beta1:

```bash
/cosmos/gov/v1beta1/proposals/{proposal_id}/votes
```

Example:

```bash
curl localhost:1317/cosmos/gov/v1beta1/proposals/1/votes
```

Example Output:

```bash
{
  "votes": [
    {
      "proposal_id": "1",
      "voter": "cosmos1..",
      "option": "VOTE_OPTION_YES",
      "options": [
        {
          "option": "VOTE_OPTION_YES",
          "weight": "1.000000000000000000"
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

Using v1:

```bash
/cosmos/gov/v1/proposals/{proposal_id}/votes
```

Example:

```bash
curl localhost:1317/cosmos/gov/v1/proposals/1/votes
```

Example Output:

```bash
{
  "votes": [
    {
      "proposal_id": "1",
      "voter": "cosmos1..",
      "options": [
        {
          "option": "VOTE_OPTION_YES",
          "weight": "1.000000000000000000"
        }
      ],
      "metadata": ""
    }
  ],
  "pagination": {
    "next_key": null,
    "total": "1"
  }
}
```

#### params

Endpoint `params` cho phép người dùng truy vấn tất cả tham số cho module `gov`.

<!-- TODO: #10197 Querying governance params outputs nil values -->

Using legacy v1beta1:

```bash
/cosmos/gov/v1beta1/params/{params_type}
```

Example:

```bash
curl localhost:1317/cosmos/gov/v1beta1/params/voting
```

Example Output:

```bash
{
  "voting_params": {
    "voting_period": "172800s"
  },
  "deposit_params": {
    "min_deposit": [
    ],
    "max_deposit_period": "0s"
  },
  "tally_params": {
    "quorum": "0.000000000000000000",
    "threshold": "0.000000000000000000",
    "veto_threshold": "0.000000000000000000"
  }
}
```

Using v1:

```bash
/cosmos/gov/v1/params/{params_type}
```

Example:

```bash
curl localhost:1317/cosmos/gov/v1/params/voting
```

Example Output:

```bash
{
  "voting_params": {
    "voting_period": "172800s"
  },
  "deposit_params": {
    "min_deposit": [
    ],
    "max_deposit_period": "0s"
  },
  "tally_params": {
    "quorum": "0.000000000000000000",
    "threshold": "0.000000000000000000",
    "veto_threshold": "0.000000000000000000"
  }
}
```

#### deposits

Endpoint `deposits` cho phép người dùng truy vấn deposit cho proposal nhất định từ depositor nhất định.

Using legacy v1beta1:

```bash
/cosmos/gov/v1beta1/proposals/{proposal_id}/deposits/{depositor}
```

Example:

```bash
curl localhost:1317/cosmos/gov/v1beta1/proposals/1/deposits/cosmos1..
```

Example Output:

```bash
{
  "deposit": {
    "proposal_id": "1",
    "depositor": "cosmos1..",
    "amount": [
      {
        "denom": "stake",
        "amount": "10000000"
      }
    ]
  }
}
```

Using v1:

```bash
/cosmos/gov/v1/proposals/{proposal_id}/deposits/{depositor}
```

Example:

```bash
curl localhost:1317/cosmos/gov/v1/proposals/1/deposits/cosmos1..
```

Example Output:

```bash
{
  "deposit": {
    "proposal_id": "1",
    "depositor": "cosmos1..",
    "amount": [
      {
        "denom": "stake",
        "amount": "10000000"
      }
    ]
  }
}
```

#### proposal deposits

Endpoint `deposits` cho phép người dùng truy vấn tất cả deposit cho proposal nhất định.

Using legacy v1beta1:

```bash
/cosmos/gov/v1beta1/proposals/{proposal_id}/deposits
```

Example:

```bash
curl localhost:1317/cosmos/gov/v1beta1/proposals/1/deposits
```

Example Output:

```bash
{
  "deposits": [
    {
      "proposal_id": "1",
      "depositor": "cosmos1..",
      "amount": [
        {
          "denom": "stake",
          "amount": "10000000"
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

Using v1:

```bash
/cosmos/gov/v1/proposals/{proposal_id}/deposits
```

Example:

```bash
curl localhost:1317/cosmos/gov/v1/proposals/1/deposits
```

Example Output:

```bash
{
  "deposits": [
    {
      "proposal_id": "1",
      "depositor": "cosmos1..",
      "amount": [
        {
          "denom": "stake",
          "amount": "10000000"
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

#### tally

Endpoint `tally` cho phép người dùng truy vấn tổng hợp phiếu của proposal nhất định.

Using legacy v1beta1:

```bash
/cosmos/gov/v1beta1/proposals/{proposal_id}/tally
```

Example:

```bash
curl localhost:1317/cosmos/gov/v1beta1/proposals/1/tally
```

Example Output:

```bash
{
  "tally": {
    "yes": "1000000",
    "abstain": "0",
    "no": "0",
    "no_with_veto": "0"
  }
}
```

Using v1:

```bash
/cosmos/gov/v1/proposals/{proposal_id}/tally
```

Example:

```bash
curl localhost:1317/cosmos/gov/v1/proposals/1/tally
```

Example Output:

```bash
{
  "tally": {
    "yes": "1000000",
    "abstain": "0",
    "no": "0",
    "no_with_veto": "0"
  }
}
```

## Metadata

Module gov có hai vị trí cho metadata nơi người dùng có thể cung cấp thêm ngữ cảnh về các hành động on-chain họ đang thực hiện. Theo mặc định tất cả trường metadata có giới hạn 255 ký tự nơi metadata có thể lưu ở định dạng json, on-chain hoặc off-chain tùy theo lượng dữ liệu cần thiết. Ở đây chúng ta cung cấp khuyến nghị cho cấu trúc json và nơi dữ liệu nên được lưu. Có hai yếu tố quan trọng khi đưa ra các khuyến nghị này. Thứ nhất, module gov và group nhất quán với nhau, lưu ý số lượng proposal do tất cả group tạo có thể khá lớn. Thứ hai, ứng dụng client như block explorer và giao diện governance có thể tin tưởng vào tính nhất quán của cấu trúc metadata trên các chain.

### Proposal

Location: off-chain dưới dạng json object lưu trên IPFS (phản chiếu [group proposal](../group/README.md#metadata))

```json
{
  "title": "",
  "authors": [""],
  "summary": "",
  "details": "",
  "proposal_forum_url": "",
  "vote_option_context": "",
}
```

:::note
Trường `authors` là mảng chuỗi, cho phép liệt kê nhiều tác giả trong metadata.
Trong v0.46, trường `authors` là chuỗi phân tách bằng dấu phẩy. Khuyến khích frontend hỗ trợ cả hai định dạng để tương thích ngược.
:::

### Vote

Location: on-chain dưới dạng json trong giới hạn 255 ký tự (phản chiếu [group vote](../group/README.md#metadata))

```json
{
  "justification": "",
}
```

## Future Improvements

Tài liệu hiện tại chỉ mô tả sản phẩm tối thiểu khả thi cho module governance. Các cải tiến tương lai có thể bao gồm:

* **`BountyProposals`:** Nếu được chấp nhận, `BountyProposal` tạo bounty mở. `BountyProposal` chỉ định bao nhiêu Atom sẽ được trao khi hoàn thành. Các Atom này sẽ được lấy từ `reserve pool`. Sau khi `BountyProposal` được governance chấp nhận, bất kỳ ai cũng có thể gửi `SoftwareUpgradeProposal` với code để nhận bounty. Lưu ý rằng khi `BountyProposal` được chấp nhận, quỹ tương ứng trong `reserve pool` bị khóa để thanh toán luôn có thể được thực hiện. Để liên kết `SoftwareUpgradeProposal` với bounty mở, người gửi `SoftwareUpgradeProposal` sẽ sử dụng thuộc tính `Proposal.LinkedProposal`. Nếu `SoftwareUpgradeProposal` liên kết với bounty mở được governance chấp nhận, quỹ đã được reserve sẽ tự động chuyển cho người gửi.
* **Complex delegation:** Delegator có thể chọn đại diện khác ngoài validator của họ. Cuối cùng, chuỗi đại diện luôn kết thúc ở validator, nhưng delegator có thể kế thừa phiếu của đại diện họ chọn trước khi kế thừa phiếu của validator. Nói cách khác, họ chỉ kế thừa phiếu của validator nếu đại diện được chỉ định khác của họ không bỏ phiếu.
* **Better process for proposal review:** Sẽ có hai phần cho `proposal.Deposit`, một cho chống spam (giống MVP) và một để thưởng cho auditor bên thứ ba.
