# ADR 042: Module Group

## Changelog

* 2020/04/09: Bản nháp ban đầu

## Trạng thái

BẢN NHÁP

## Tóm tắt

ADR này định nghĩa module `x/group`, cho phép tạo và quản lý các tài khoản đa chữ
ký (multi-signature) on-chain và cho phép bỏ phiếu để thực thi message dựa trên
các chính sách ra quyết định có thể cấu hình.

## Bối cảnh

Cơ chế multi-signature legacy amino của Cosmos SDK có một số hạn chế:

* Không thể xoay khoá (key rotation), dù có thể giải quyết bằng [account rekeying](adr-034-account-rekeying.md).
* Không thể thay đổi ngưỡng (threshold).
* UX rườm rà cho người dùng không chuyên kỹ thuật ([#5661](https://github.com/cosmos/cosmos-sdk/issues/5661)).
* Yêu cầu sign mode `legacy_amino` ([#8141](https://github.com/cosmos/cosmos-sdk/issues/8141)).

Dù module group không nhằm thay thế hoàn toàn các tài khoản multi-signature hiện
tại, nó cung cấp lời giải cho các hạn chế nêu trên, với một hệ thống quản lý khoá
linh hoạt hơn: khoá có thể được thêm, cập nhật hoặc xoá, đồng thời ngưỡng có thể
cấu hình.
Nó được kỳ vọng dùng cùng với các module kiểm soát truy cập khác như
[`x/feegrant`](./adr-029-fee-grant-module.md) và [`x/authz`](adr-030-authz-module.md)
để đơn giản hoá quản lý khoá cho cá nhân và tổ chức.

Bản proof of concept của module group có thể xem tại
https://github.com/cosmos/cosmos-sdk/tree/main/proto/cosmos/group/v1 và
https://github.com/cosmos/cosmos-sdk/tree/main/x/group.

## Quyết định

Chúng ta đề xuất gộp module `x/group` cùng với gói hỗ trợ
[ORM/Table Store](https://github.com/cosmos/cosmos-sdk/tree/main/x/group/internal/orm)
([#7098](https://github.com/cosmos/cosmos-sdk/issues/7098)) vào Cosmos SDK và tiếp
tục phát triển ở đây. Sẽ có một ADR riêng cho gói ORM.

### Group

Một group là một tập hợp các tài khoản kèm theo trọng số (weight) tương ứng. Nó
không phải là một tài khoản và không có số dư. Bản thân nó cũng không có bất kỳ
trọng số bỏ phiếu hay ra quyết định nào.
Thành viên group có thể tạo proposal và bỏ phiếu cho chúng thông qua các group
account, dùng các decision policy khác nhau.

Group có một tài khoản `admin` có thể quản lý thành viên trong group, cập nhật
metadata của group và đặt admin mới.

```protobuf
message GroupInfo {

    // group_id is the unique ID of this group.
    uint64 group_id = 1;

    // admin is the account address of the group's admin.
    string admin = 2;

    // metadata is any arbitrary metadata to attached to the group.
    bytes metadata = 3;

    // version is used to track changes to a group's membership structure that
    // would break existing proposals. Whenever a member weight has changed,
    // or any member is added or removed, the version is incremented and will
    // invalidate all proposals from older versions.
    uint64 version = 4;

    // total_weight is the sum of the group members' weights.
    string total_weight = 5;
}
```

```protobuf
message GroupMember {

    // group_id is the unique ID of the group.
    uint64 group_id = 1;

    // member is the member data.
    Member member = 2;
}

// Member represents a group member with an account address,
// non-zero weight and metadata.
message Member {

    // address is the member's account address.
    string address = 1;

    // weight is the member's voting weight that should be greater than 0.
    string weight = 2;

    // metadata is any arbitrary metadata to attached to the member.
    bytes metadata = 3;
}
```

### Group Account

Group account là một tài khoản gắn với một group và một decision policy.
Group account có số dư.

Group account được tách khỏi group bởi vì một group đơn lẻ có thể có nhiều decision
policy cho các loại hành động khác nhau. Quản lý membership tách biệt với decision
policy giúp giảm overhead tối đa và giữ membership nhất quán giữa các policy khác
nhau. Mẫu (pattern) được khuyến nghị là có một master group account duy nhất cho
một group nhất định, sau đó tạo các group account riêng với decision policy khác
nhau và uỷ quyền (delegate) các quyền mong muốn từ master account sang các
“sub-account” đó bằng module [`x/authz`](adr-030-authz-module.md).

```protobuf
message GroupAccountInfo {

    // address is the group account address.
    string address = 1;

    // group_id is the ID of the Group the GroupAccount belongs to.
    uint64 group_id = 2;

    // admin is the account address of the group admin.
    string admin = 3;

    // metadata is any arbitrary metadata of this group account.
    bytes metadata = 4;

    // version is used to track changes to a group's GroupAccountInfo structure that
    // invalidates active proposal from old versions.
    uint64 version = 5;

    // decision_policy specifies the group account's decision policy.
    google.protobuf.Any decision_policy = 6 [(cosmos_proto.accepts_interface) = "cosmos.group.v1.DecisionPolicy"];
}
```

Tương tự group admin, group account admin có thể cập nhật metadata, decision policy,
hoặc đặt group account admin mới.

Một group account cũng có thể là admin hoặc thành viên của một group.
Ví dụ: group admin có thể là một group account khác có thể “bầu chọn” thành viên,
hoặc cũng có thể chính group đó tự bầu chính mình.

### Decision Policy

Decision policy là cơ chế để các thành viên của một group bỏ phiếu đối với proposal.

Mọi decision policy nên có cửa sổ bỏ phiếu tối thiểu và tối đa.
Cửa sổ tối thiểu là khoảng thời gian tối thiểu phải trôi qua để một proposal
có thể “đậu” (pass), và có thể đặt bằng 0. Cửa sổ tối đa là thời gian tối đa mà
một proposal có thể được bỏ phiếu và được thực thi nếu nó đạt đủ ủng hộ trước khi
nó bị đóng.
Cả hai giá trị này phải nhỏ hơn tham số cửa sổ bỏ phiếu tối đa toàn chain.

Chúng ta định nghĩa interface `DecisionPolicy` mà mọi decision policy phải triển khai:

```go
type DecisionPolicy interface {
	codec.ProtoMarshaler

	ValidateBasic() error
	GetTimeout() types.Duration
	Allow(tally Tally, totalPower string, votingDuration time.Duration) (DecisionPolicyResult, error)
	Validate(g GroupInfo) error
}

type DecisionPolicyResult struct {
	Allow bool
	Final bool
}
```

#### Threshold decision policy

Threshold decision policy định nghĩa mức tối thiểu của phiếu ủng hộ (_yes_), dựa
trên tally trọng số phiếu bầu, để một proposal được thông qua. Với decision policy
này, abstain và veto được xem như không ủng hộ (_no_).

```protobuf
message ThresholdDecisionPolicy {

    // threshold is the minimum weighted sum of support votes for a proposal to succeed.
    string threshold = 1;

    // voting_period is the duration from submission of a proposal to the end of voting period
    // Within this period, votes and exec messages can be submitted.
    google.protobuf.Duration voting_period = 2 [(gogoproto.nullable) = false];
}
```

### Proposal

Bất kỳ thành viên nào của một group đều có thể gửi proposal để một group account
quyết định. Một proposal gồm một tập các `sdk.Msg` sẽ được thực thi nếu proposal
được thông qua, cùng với metadata gắn với proposal. Các `sdk.Msg` này được validate
như một phần của validation cho request `Msg/CreateProposal`. Chúng cũng nên có
signer được đặt là group account.

Nội bộ, một proposal cũng theo dõi:

* `Status` hiện tại: submitted, closed hoặc aborted
* `Result`: unfinalized, accepted hoặc rejected
* `VoteState` dưới dạng một `Tally`, được tính khi có phiếu mới và khi thực thi proposal

```protobuf
// Tally represents the sum of weighted votes.
message Tally {
    option (gogoproto.goproto_getters) = false;

    // yes_count is the weighted sum of yes votes.
    string yes_count = 1;

    // no_count is the weighted sum of no votes.
    string no_count = 2;

    // abstain_count is the weighted sum of abstainers.
    string abstain_count = 3;

    // veto_count is the weighted sum of vetoes.
    string veto_count = 4;
}
```

### Bỏ phiếu (Voting)

Thành viên group có thể bỏ phiếu cho proposal. Có bốn lựa chọn khi bỏ phiếu: yes,
no, abstain và veto. Không phải decision policy nào cũng hỗ trợ tất cả lựa chọn.
Phiếu bầu có thể chứa metadata tuỳ chọn.
Trong triển khai hiện tại, cửa sổ bỏ phiếu bắt đầu ngay khi proposal được gửi.

Việc bỏ phiếu nội bộ sẽ cập nhật `VoteState` của proposal cũng như `Status` và
`Result` nếu cần.

### Thực thi proposal

Trong thiết kế hiện tại, proposal sẽ không được chain tự động thực thi; thay vào
đó, người dùng phải gửi một giao dịch `Msg/Exec` để cố gắng thực thi proposal dựa
trên các phiếu hiện tại và decision policy. Một nâng cấp tương lai có thể tự động
hoá điều này và để group account (hoặc một fee granter) chi trả.

#### Thay đổi membership của group

Trong triển khai hiện tại, nếu cập nhật group hoặc group account sau khi đã gửi
proposal thì proposal sẽ trở nên không hợp lệ. Nó sẽ đơn giản thất bại nếu ai đó
gọi `Msg/Exec` và cuối cùng sẽ bị garbage collected.

### Ghi chú về triển khai hiện tại

Phần này phác thảo triển khai hiện tại dùng trong proof of concept của module
group, nhưng có thể thay đổi và được lặp/điều chỉnh.

#### ORM

[Gói ORM](https://github.com/cosmos/cosmos-sdk/discussions/9156) định nghĩa các
table, sequence, và secondary index được dùng trong module group.

Group được lưu trong state như một phần của `groupTable`, trong đó `group_id` là
một số nguyên tự tăng. Thành viên group được lưu trong `groupMemberTable`.

Group account được lưu trong `groupAccountTable`. Địa chỉ group account được tạo
dựa trên một số nguyên tự tăng, số này được dùng để suy dẫn (derive) `RootModuleKey`
của module group thành `DerivedModuleKey`, như nêu trong [ADR-033](adr-033-protobuf-inter-module-comm.md#modulekeys-and-moduleids).
Group account được thêm như một `ModuleAccount` mới thông qua `x/auth`.

Proposal được lưu như một phần của `proposalTable` dùng kiểu `Proposal`. `proposal_id`
là một số nguyên tự tăng.

Phiếu bầu được lưu trong `voteTable`. Primary key dựa trên `proposal_id` của phiếu
và địa chỉ tài khoản `voter`.

#### Dùng ADR-033 để định tuyến message của proposal

Giao tiếp liên module (inter-module communication) được giới thiệu bởi
[ADR-033](adr-033-protobuf-inter-module-comm.md) có thể dùng để định tuyến
message của proposal bằng `DerivedModuleKey` tương ứng với group account của proposal.

## Hệ quả

### Tích cực

* Cải thiện UX cho tài khoản multi-signature, cho phép xoay khoá và decision policy tuỳ biến.

### Tiêu cực

### Trung tính

* Nó dùng ADR 033 nên sẽ cần được triển khai trong Cosmos SDK, nhưng điều này
  không nhất thiết kéo theo refactor lớn đối với các module Cosmos SDK hiện có.
* Triển khai hiện tại của module group dùng gói ORM.

## Thảo luận thêm

* Hướng hội tụ giữa `/group` và `x/gov` vì cả hai đều hỗ trợ proposal và bỏ phiếu:
  https://github.com/cosmos/cosmos-sdk/discussions/9066
* Các cải tiến tương lai có thể có cho `x/group`:
  * Thực thi proposal ngay khi nộp (https://github.com/regen-network/regen-ledger/issues/288)
  * Rút lại một proposal (https://github.com/regen-network/cosmos-modules/issues/41)
  * Làm `Tally` linh hoạt hơn và hỗ trợ lựa chọn không nhị phân (non-binary)

## Tham khảo

* Đặc tả ban đầu:
  * https://gist.github.com/aaronc/b60628017352df5983791cad30babe56#group-module
  * [#5236](https://github.com/cosmos/cosmos-sdk/pull/5236)
* Đề xuất đưa `x/group` vào Cosmos SDK: [#7633](https://github.com/cosmos/cosmos-sdk/issues/7633)

