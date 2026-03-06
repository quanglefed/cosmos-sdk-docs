---
sidebar_position: 1
---

# `x/group`

⚠️ **DEPRECATED**: This package is deprecated and will be removed in the next major release.  The `x/group` module will be moved to a separate repo `github.com/cosmos/cosmos-sdk-legacy`.

## Abstract

Các tài liệu sau đây mô tả chi tiết module group.

Module này cho phép tạo và quản lý các tài khoản multisig trên chuỗi, đồng thời hỗ trợ bỏ phiếu cho việc thực thi message dựa trên các chính sách quyết định có thể cấu hình.

## Contents

* [Concepts](#concepts)
    * [Group](#group)
    * [Group Policy](#group-policy)
    * [Decision Policy](#decision-policy)
    * [Proposal](#proposal)
    * [Pruning](#pruning)
* [State](#state)
    * [Group Table](#group-table)
    * [Group Member Table](#group-member-table)
    * [Group Policy Table](#group-policy-table)
    * [Proposal Table](#proposal-table)
    * [Vote Table](#vote-table)
* [Msg Service](#msg-service)
    * [Msg/CreateGroup](#msgcreategroup)
    * [Msg/UpdateGroupMembers](#msgupdategroupmembers)
    * [Msg/UpdateGroupAdmin](#msgupdategroupadmin)
    * [Msg/UpdateGroupMetadata](#msgupdategroupmetadata)
    * [Msg/CreateGroupPolicy](#msgcreategrouppolicy)
    * [Msg/CreateGroupWithPolicy](#msgcreategroupwithpolicy)
    * [Msg/UpdateGroupPolicyAdmin](#msgupdategrouppolicyadmin)
    * [Msg/UpdateGroupPolicyDecisionPolicy](#msgupdategrouppolicydecisionpolicy)
    * [Msg/UpdateGroupPolicyMetadata](#msgupdategrouppolicymetadata)
    * [Msg/SubmitProposal](#msgsubmitproposal)
    * [Msg/WithdrawProposal](#msgwithdrawproposal)
    * [Msg/Vote](#msgvote)
    * [Msg/Exec](#msgexec)
    * [Msg/LeaveGroup](#msgleavegroup)
* [Events](#events)
    * [EventCreateGroup](#eventcreategroup)
    * [EventUpdateGroup](#eventupdategroup)
    * [EventCreateGroupPolicy](#eventcreategrouppolicy)
    * [EventUpdateGroupPolicy](#eventupdategrouppolicy)
    * [EventCreateProposal](#eventcreateproposal)
    * [EventWithdrawProposal](#eventwithdrawproposal)
    * [EventVote](#eventvote)
    * [EventExec](#eventexec)
    * [EventLeaveGroup](#eventleavegroup)
    * [EventProposalPruned](#eventproposalpruned)
* [Client](#client)
    * [CLI](#cli)
    * [gRPC](#grpc)
    * [REST](#rest)
* [Metadata](#metadata)

## Concepts

### Group

Một group đơn giản là tập hợp các tài khoản với trọng số liên kết. Nó không phải là một tài khoản và không có số dư. Bản thân nó không có bất kỳ quyền bỏ phiếu hay trọng số quyết định nào. Group có một "administrator" có khả năng thêm, xóa và cập nhật thành viên trong group. Lưu ý rằng tài khoản group policy có thể là administrator của một group, và administrator không nhất thiết phải là thành viên của group.

### Group Policy

Group policy là một tài khoản liên kết với một group và một decision policy. Group policy được tách biệt khỏi group vì một group có thể có nhiều decision policy cho các loại hành động khác nhau. Quản lý thành viên group tách biệt khỏi decision policy giúp giảm thiểu chi phí và duy trì tính nhất quán của thành viên giữa các policy khác nhau. Mô hình được khuyến nghị là có một group policy chính duy nhất cho một group, sau đó tạo các group policy riêng biệt với các decision policy khác nhau và ủy quyền quyền mong muốn từ tài khoản chính sang các "sub-account" đó bằng module `x/authz`.

### Decision Policy

Decision policy là cơ chế mà qua đó các thành viên của group có thể bỏ phiếu cho các proposal, cũng như các quy tắc quy định proposal có được thông qua hay không dựa trên kết quả kiểm đếm.

Tất cả decision policy thường có thời gian thực thi tối thiểu và cửa sổ bỏ phiếu tối đa. Thời gian thực thi tối thiểu là khoảng thời gian tối thiểu phải trôi qua sau khi gửi để proposal có thể được thực thi, và có thể được đặt là 0. Cửa sổ bỏ phiếu tối đa là thời gian tối đa sau khi gửi mà proposal có thể được bỏ phiếu trước khi được kiểm đếm.

Nhà phát triển chuỗi cũng định nghĩa thời gian thực thi tối đa toàn ứng dụng, đây là khoảng thời gian tối đa sau khi kết thúc thời gian bỏ phiếu của proposal mà người dùng được phép thực thi proposal.

Module group hiện tại đi kèm hai decision policy: threshold và percentage. Bất kỳ nhà phát triển chuỗi nào cũng có thể mở rộng hai loại này bằng cách tạo decision policy tùy chỉnh, miễn là tuân thủ interface `DecisionPolicy`:

```go reference
https://github.com/cosmos/cosmos-sdk/tree/release/v0.50.x/x/group/types.go#L27-L45
```

#### Threshold decision policy

Threshold decision policy định nghĩa ngưỡng phiếu đồng ý (dựa trên kiểm đếm trọng số người bỏ phiếu) phải đạt được để proposal được thông qua. Với decision policy này, abstain và veto đơn giản được coi là phiếu không đồng ý.

Decision policy này cũng có cửa sổ VotingPeriod và cửa sổ MinExecutionPeriod. Cái trước định nghĩa khoảng thời gian sau khi gửi proposal mà các thành viên được phép bỏ phiếu, sau đó việc kiểm đếm được thực hiện. Cái sau chỉ định khoảng thời gian tối thiểu sau khi gửi proposal mà proposal có thể được thực thi. Nếu đặt là 0, thì proposal được phép thực thi ngay khi gửi (sử dụng tùy chọn `TRY_EXEC`). Rõ ràng, MinExecutionPeriod không thể lớn hơn VotingPeriod+MaxExecutionPeriod (trong đó MaxExecution là khoảng thời gian do ứng dụng định nghĩa chỉ định cửa sổ sau khi bỏ phiếu kết thúc mà proposal có thể được thực thi).

#### Percentage decision policy

Percentage decision policy tương tự threshold decision policy, ngoại trừ việc ngưỡng không được định nghĩa là trọng số cố định mà là phần trăm. Nó phù hợp hơn cho các group mà trọng số thành viên có thể được cập nhật, vì ngưỡng phần trăm giữ nguyên và không phụ thuộc vào cách các trọng số thành viên được cập nhật.

Giống như Threshold decision policy, percentage decision policy có hai tham số VotingPeriod và MinExecutionPeriod.

### Proposal

Bất kỳ thành viên nào của group đều có thể gửi proposal để tài khoản group policy quyết định. Một proposal bao gồm một tập hợp các message sẽ được thực thi nếu proposal được thông qua cũng như bất kỳ metadata nào liên kết với proposal.

#### Voting

Có bốn lựa chọn khi bỏ phiếu - yes, no, abstain và veto. Không phải tất cả decision policy đều tính đến bốn lựa chọn. Phiếu bầu có thể chứa một số metadata tùy chọn. Trong triển khai hiện tại, cửa sổ bỏ phiếu bắt đầu ngay khi proposal được gửi, và kết thúc được định nghĩa bởi decision policy của group policy.

#### Withdrawing Proposals

Proposal có thể được rút bất kỳ lúc nào trước khi kết thúc thời gian bỏ phiếu, bởi admin của group policy hoặc bởi một trong các proposer. Sau khi rút, nó được đánh dấu là `PROPOSAL_STATUS_WITHDRAWN`, và không còn bỏ phiếu hay thực thi nào được phép trên nó.

#### Aborted Proposals

Nếu group policy được cập nhật trong thời gian bỏ phiếu của proposal, thì proposal được đánh dấu là `PROPOSAL_STATUS_ABORTED`, và không còn bỏ phiếu hay thực thi nào được phép trên nó. Điều này là vì group policy định nghĩa các quy tắc bỏ phiếu và thực thi proposal, vì vậy nếu các quy tắc đó thay đổi trong vòng đời của proposal, thì proposal nên được đánh dấu là lỗi thời.

#### Tallying

Tallying là việc đếm tất cả phiếu bầu trên một proposal. Nó chỉ xảy ra một lần trong vòng đời của proposal, nhưng có thể được kích hoạt bởi hai yếu tố, cái nào xảy ra trước:

* hoặc ai đó cố gắng thực thi proposal (xem phần tiếp theo), có thể xảy ra trên giao dịch `Msg/Exec`, hoặc giao dịch `Msg/{SubmitProposal,Vote}` với trường `Exec` được đặt. Khi thực thi proposal được thử, việc kiểm đếm được thực hiện trước để đảm bảo proposal được thông qua.
* hoặc trên `EndBlock` khi thời gian bỏ phiếu của proposal vừa kết thúc.

Nếu kết quả kiểm đếm thỏa mãn các quy tắc của decision policy, thì proposal được đánh dấu là `PROPOSAL_STATUS_ACCEPTED`, nếu không thì được đánh dấu là `PROPOSAL_STATUS_REJECTED`. Trong mọi trường hợp, không còn bỏ phiếu nào được phép nữa, và kết quả kiểm đếm được lưu vào state trong `FinalTallyResult` của proposal.

#### Executing Proposals

Proposal chỉ được thực thi khi việc kiểm đếm hoàn tất và decision policy của tài khoản group cho phép proposal được thông qua dựa trên kết quả kiểm đếm. Chúng được đánh dấu bởi trạng thái `PROPOSAL_STATUS_ACCEPTED`. Việc thực thi phải diễn ra trước khoảng thời gian `MaxExecutionPeriod` (do nhà phát triển chuỗi đặt) sau khi kết thúc thời gian bỏ phiếu của mỗi proposal.

Proposal sẽ không được tự động thực thi bởi chuỗi trong thiết kế hiện tại này, mà người dùng phải gửi giao dịch `Msg/Exec` để thử thực thi proposal dựa trên phiếu hiện tại và decision policy. Bất kỳ người dùng nào (không chỉ thành viên group) đều có thể thực thi các proposal đã được chấp nhận, và phí thực thi do người thực thi proposal trả. Cũng có thể thử thực thi proposal ngay khi tạo hoặc khi có phiếu mới bằng trường `Exec` của request `Msg/SubmitProposal` và `Msg/Vote`. Trong trường hợp trước, chữ ký của proposer được coi là phiếu đồng ý. Trong các trường hợp này, nếu proposal không thể được thực thi (tức là không thỏa mãn quy tắc của decision policy), nó vẫn sẽ được mở cho phiếu mới và có thể được kiểm đếm và thực thi sau đó.

Một proposal thực thi thành công sẽ có `ExecutorResult` được đánh dấu là `PROPOSAL_EXECUTOR_RESULT_SUCCESS`. Proposal sẽ được tự động prune sau khi thực thi. Mặt khác, proposal thực thi thất bại sẽ được đánh dấu là `PROPOSAL_EXECUTOR_RESULT_FAILURE`. Proposal như vậy có thể được thực thi lại nhiều lần, cho đến khi hết hạn sau `MaxExecutionPeriod` sau khi kết thúc thời gian bỏ phiếu.

### Pruning

Proposal và vote được tự động prune để tránh state phình to.

Vote được prune:

* hoặc sau khi kiểm đếm thành công, tức là kiểm đếm có kết quả thỏa mãn quy tắc của decision policy, có thể được kích hoạt bởi `Msg/Exec` hoặc `Msg/{SubmitProposal,Vote}` với trường `Exec` được đặt,
* hoặc trên `EndBlock` ngay sau khi kết thúc thời gian bỏ phiếu của proposal. Điều này cũng áp dụng cho proposal có trạng thái `aborted` hoặc `withdrawn`.

cái nào xảy ra trước.

Proposal được prune:

* trên `EndBlock` mà proposal có trạng thái `withdrawn` hoặc `aborted` khi kết thúc thời gian bỏ phiếu của proposal trước khi kiểm đếm,
* và hoặc sau khi thực thi proposal thành công,
* hoặc trên `EndBlock` ngay sau khi `voting_period_end` + `max_execution_period` (định nghĩa là cấu hình toàn ứng dụng) của proposal trôi qua,

cái nào xảy ra trước.

## State

Module `group` sử dụng package `orm` cung cấp lưu trữ bảng với hỗ trợ khóa chính và chỉ mục phụ. `orm` cũng định nghĩa `Sequence` là bộ tạo khóa duy nhất liên tục dựa trên bộ đếm có thể được sử dụng cùng với `Table`s.

Đây là danh sách các bảng và sequence cùng chỉ mục liên kết được lưu trữ như một phần của module `group`.

### Group Table

`groupTable` lưu `GroupInfo`: `0x0 | BigEndian(GroupId) -> ProtocolBuffer(GroupInfo)`.

#### groupSeq

Giá trị của `groupSeq` được tăng khi tạo group mới và tương ứng với `GroupId` mới: `0x1 | 0x1 -> BigEndian`.

`0x1` thứ hai tương ứng với `sequenceStorageKey` của ORM.

#### groupByAdminIndex

`groupByAdminIndex` cho phép truy xuất group theo địa chỉ admin:
`0x2 | len([]byte(group.Admin)) | []byte(group.Admin) | BigEndian(GroupId) -> []byte()`.

### Group Member Table

`groupMemberTable` lưu `GroupMember`s: `0x10 | BigEndian(GroupId) | []byte(member.Address) -> ProtocolBuffer(GroupMember)`.

`groupMemberTable` là bảng khóa chính và `PrimaryKey` của nó được cho bởi `BigEndian(GroupId) | []byte(member.Address)` được sử dụng bởi các chỉ mục sau.

#### groupMemberByGroupIndex

`groupMemberByGroupIndex` cho phép truy xuất thành viên group theo group id:
`0x11 | BigEndian(GroupId) | PrimaryKey -> []byte()`.

#### groupMemberByMemberIndex

`groupMemberByMemberIndex` cho phép truy xuất thành viên group theo địa chỉ member:
`0x12 | len([]byte(member.Address)) | []byte(member.Address) | PrimaryKey -> []byte()`.

### Group Policy Table

`groupPolicyTable` lưu `GroupPolicyInfo`: `0x20 | len([]byte(Address)) | []byte(Address) -> ProtocolBuffer(GroupPolicyInfo)`.

`groupPolicyTable` là bảng khóa chính và `PrimaryKey` của nó được cho bởi `len([]byte(Address)) | []byte(Address)` được sử dụng bởi các chỉ mục sau.

#### groupPolicySeq

Giá trị của `groupPolicySeq` được tăng khi tạo group policy mới và được sử dụng để tạo địa chỉ tài khoản group policy mới `Address`:
`0x21 | 0x1 -> BigEndian`.

`0x1` thứ hai tương ứng với `sequenceStorageKey` của ORM.

#### groupPolicyByGroupIndex

`groupPolicyByGroupIndex` cho phép truy xuất group policy theo group id:
`0x22 | BigEndian(GroupId) | PrimaryKey -> []byte()`.

#### groupPolicyByAdminIndex

`groupPolicyByAdminIndex` cho phép truy xuất group policy theo địa chỉ admin:
`0x23 | len([]byte(Address)) | []byte(Address) | PrimaryKey -> []byte()`.

### Proposal Table

`proposalTable` lưu `Proposal`s: `0x30 | BigEndian(ProposalId) -> ProtocolBuffer(Proposal)`.

#### proposalSeq

Giá trị của `proposalSeq` được tăng khi tạo proposal mới và tương ứng với `ProposalId` mới: `0x31 | 0x1 -> BigEndian`.

`0x1` thứ hai tương ứng với `sequenceStorageKey` của ORM.

#### proposalByGroupPolicyIndex

`proposalByGroupPolicyIndex` cho phép truy xuất proposal theo địa chỉ tài khoản group policy:
`0x32 | len([]byte(account.Address)) | []byte(account.Address) | BigEndian(ProposalId) -> []byte()`.

#### ProposalsByVotingPeriodEndIndex

`proposalsByVotingPeriodEndIndex` cho phép truy xuất proposal được sắp xếp theo `voting_period_end` theo thứ tự thời gian:
`0x33 | sdk.FormatTimeBytes(proposal.VotingPeriodEnd) | BigEndian(ProposalId) -> []byte()`.

Chỉ mục này được sử dụng khi kiểm đếm phiếu proposal khi kết thúc thời gian bỏ phiếu, và để prune proposal tại `VotingPeriodEnd + MaxExecutionPeriod`.

### Vote Table

`voteTable` lưu `Vote`s: `0x40 | BigEndian(ProposalId) | []byte(voter.Address) -> ProtocolBuffer(Vote)`.

`voteTable` là bảng khóa chính và `PrimaryKey` của nó được cho bởi `BigEndian(ProposalId) | []byte(voter.Address)` được sử dụng bởi các chỉ mục sau.

#### voteByProposalIndex

`voteByProposalIndex` cho phép truy xuất vote theo proposal id:
`0x41 | BigEndian(ProposalId) | PrimaryKey -> []byte()`.

#### voteByVoterIndex

`voteByVoterIndex` cho phép truy xuất vote theo địa chỉ voter:
`0x42 | len([]byte(voter.Address)) | []byte(voter.Address) | PrimaryKey -> []byte()`.

## Msg Service

### Msg/CreateGroup

Một group mới có thể được tạo với `MsgCreateGroup`, có địa chỉ admin, danh sách thành viên và một số metadata tùy chọn.

Metadata có độ dài tối đa do nhà phát triển ứng dụng chọn và được truyền vào group keeper dưới dạng config.

```go reference
https://github.com/cosmos/cosmos-sdk/tree/release/v0.50.x/proto/cosmos/group/v1/tx.proto#L67-L80
```

Dự kiến sẽ thất bại nếu:

* độ dài metadata lớn hơn config `MaxMetadataLen`
* thành viên không được thiết lập đúng (ví dụ: định dạng địa chỉ sai, trùng lặp, hoặc trọng số 0).

### Msg/UpdateGroupMembers

Thành viên group có thể được cập nhật với `UpdateGroupMembers`.

```go reference
https://github.com/cosmos/cosmos-sdk/tree/release/v0.50.x/proto/cosmos/group/v1/tx.proto#L88-L102
```

Trong danh sách `MemberUpdates`, một thành viên hiện có có thể bị xóa bằng cách đặt trọng số của nó là 0.

Dự kiến sẽ thất bại nếu:

* người ký không phải là admin của group.
* đối với bất kỳ group policy liên kết nào, nếu phương thức `Validate()` của decision policy của nó thất bại với group đã cập nhật.

### Msg/UpdateGroupAdmin

`UpdateGroupAdmin` có thể được sử dụng để cập nhật admin của group.

```go reference
https://github.com/cosmos/cosmos-sdk/tree/release/v0.50.x/proto/cosmos/group/v1/tx.proto#L107-L120
```

Dự kiến sẽ thất bại nếu người ký không phải là admin của group.

### Msg/UpdateGroupMetadata

`UpdateGroupMetadata` có thể được sử dụng để cập nhật metadata của group.

```go reference
https://github.com/cosmos/cosmos-sdk/tree/release/v0.50.x/proto/cosmos/group/v1/tx.proto#L125-L138
```

Dự kiến sẽ thất bại nếu:

* độ dài metadata mới lớn hơn config `MaxMetadataLen`.
* người ký không phải là admin của group.

### Msg/CreateGroupPolicy

Một group policy mới có thể được tạo với `MsgCreateGroupPolicy`, có địa chỉ admin, group id, decision policy và một số metadata tùy chọn.

```go reference
https://github.com/cosmos/cosmos-sdk/tree/release/v0.50.x/proto/cosmos/group/v1/tx.proto#L147-L165
```

Dự kiến sẽ thất bại nếu:

* người ký không phải là admin của group.
* độ dài metadata lớn hơn config `MaxMetadataLen`.
* phương thức `Validate()` của decision policy không thỏa mãn với group.

### Msg/CreateGroupWithPolicy

Một group và policy mới có thể được tạo với `MsgCreateGroupWithPolicy`, có địa chỉ admin, danh sách thành viên, decision policy, trường `group_policy_as_admin` để tùy chọn đặt admin của group và group policy bằng địa chỉ group policy, và một số metadata tùy chọn cho group và group policy.

```go reference
https://github.com/cosmos/cosmos-sdk/tree/release/v0.50.x/proto/cosmos/group/v1/tx.proto#L191-L215
```

Dự kiến sẽ thất bại vì các lý do tương tự như `Msg/CreateGroup` và `Msg/CreateGroupPolicy`.

### Msg/UpdateGroupPolicyAdmin

`UpdateGroupPolicyAdmin` có thể được sử dụng để cập nhật admin của group policy.

```go reference
https://github.com/cosmos/cosmos-sdk/tree/release/v0.50.x/proto/cosmos/group/v1/tx.proto#L173-L186
```

Dự kiến sẽ thất bại nếu người ký không phải là admin của group policy.

### Msg/UpdateGroupPolicyDecisionPolicy

`UpdateGroupPolicyDecisionPolicy` có thể được sử dụng để cập nhật decision policy.

```go reference
https://github.com/cosmos/cosmos-sdk/tree/release/v0.50.x/proto/cosmos/group/v1/tx.proto#L226-L241
```

Dự kiến sẽ thất bại nếu:

* người ký không phải là admin của group policy.
* phương thức `Validate()` của decision policy mới không thỏa mãn với group.

### Msg/UpdateGroupPolicyMetadata

`UpdateGroupPolicyMetadata` có thể được sử dụng để cập nhật metadata của group policy.

```go reference
https://github.com/cosmos/cosmos-sdk/tree/release/v0.50.x/proto/cosmos/group/v1/tx.proto#L246-L259
```

Dự kiến sẽ thất bại nếu:

* độ dài metadata mới lớn hơn config `MaxMetadataLen`.
* người ký không phải là admin của group.

### Msg/SubmitProposal

Một proposal mới có thể được tạo với `MsgSubmitProposal`, có địa chỉ tài khoản group policy, danh sách địa chỉ proposer, danh sách message để thực thi nếu proposal được chấp nhận và một số metadata tùy chọn.
Giá trị `Exec` tùy chọn có thể được cung cấp để thử thực thi proposal ngay sau khi tạo proposal. Chữ ký của proposer được coi là phiếu đồng ý trong trường hợp này.

```go reference
https://github.com/cosmos/cosmos-sdk/tree/release/v0.50.x/proto/cosmos/group/v1/tx.proto#L281-L315
```

Dự kiến sẽ thất bại nếu:

* độ dài metadata, title hoặc summary lớn hơn config `MaxMetadataLen`.
* nếu bất kỳ proposer nào không phải là thành viên group.

### Msg/WithdrawProposal

Một proposal có thể được rút bằng `MsgWithdrawProposal` có `address` (có thể là proposer hoặc admin của group policy) và `proposal_id` (proposal cần rút).

```go reference
https://github.com/cosmos/cosmos-sdk/tree/release/v0.50.x/proto/cosmos/group/v1/tx.proto#L323-L333
```

Dự kiến sẽ thất bại nếu:

* người ký không phải là admin của group policy cũng không phải proposer của proposal.
* proposal đã đóng hoặc aborted.

### Msg/Vote

Một vote mới có thể được tạo với `MsgVote`, cho proposal id, địa chỉ voter, lựa chọn (yes, no, veto hoặc abstain) và một số metadata tùy chọn.
Giá trị `Exec` tùy chọn có thể được cung cấp để thử thực thi proposal ngay sau khi bỏ phiếu.

```go reference
https://github.com/cosmos/cosmos-sdk/tree/release/v0.50.x/proto/cosmos/group/v1/tx.proto#L338-L358
```

Dự kiến sẽ thất bại nếu:

* độ dài metadata lớn hơn config `MaxMetadataLen`.
* proposal không còn trong thời gian bỏ phiếu.

### Msg/Exec

Một proposal có thể được thực thi với `MsgExec`.

```go reference
https://github.com/cosmos/cosmos-sdk/tree/release/v0.50.x/proto/cosmos/group/v1/tx.proto#L363-L373
```

Các message là phần của proposal này sẽ không được thực thi nếu:

* proposal chưa được group policy chấp nhận.
* proposal đã được thực thi thành công.

### Msg/LeaveGroup

`MsgLeaveGroup` cho phép thành viên group rời khỏi group.

```go reference
https://github.com/cosmos/cosmos-sdk/tree/release/v0.50.x/proto/cosmos/group/v1/tx.proto#L381-L391
```

Dự kiến sẽ thất bại nếu:

* thành viên group không phải là thành viên của group.
* đối với bất kỳ group policy liên kết nào, nếu phương thức `Validate()` của decision policy của nó thất bại với group đã cập nhật.

## Events

Module group phát ra các event sau:

### EventCreateGroup

| Type                             | Attribute Key | Attribute Value                  |
| -------------------------------- | ------------- | -------------------------------- |
| message                          | action        | /cosmos.group.v1.Msg/CreateGroup |
| cosmos.group.v1.EventCreateGroup | group_id      | {groupId}                        |

### EventUpdateGroup

| Type                             | Attribute Key | Attribute Value                                            |
| -------------------------------- | ------------- | ---------------------------------------------------------- |
| message                          | action        | /cosmos.group.v1.Msg/UpdateGroup{Admin\|Metadata\|Members} |
| cosmos.group.v1.EventUpdateGroup | group_id      | {groupId}                                                  |

### EventCreateGroupPolicy

| Type                                   | Attribute Key | Attribute Value                        |
| -------------------------------------- | ------------- | -------------------------------------- |
| message                                | action        | /cosmos.group.v1.Msg/CreateGroupPolicy |
| cosmos.group.v1.EventCreateGroupPolicy | address       | {groupPolicyAddress}                   |

### EventUpdateGroupPolicy

| Type                                   | Attribute Key | Attribute Value                                                         |
| -------------------------------------- | ------------- | ----------------------------------------------------------------------- |
| message                                | action        | /cosmos.group.v1.Msg/UpdateGroupPolicy{Admin\|Metadata\|DecisionPolicy} |
| cosmos.group.v1.EventUpdateGroupPolicy | address       | {groupPolicyAddress}                                                    |

### EventCreateProposal

| Type                                | Attribute Key | Attribute Value                     |
| ----------------------------------- | ------------- | ----------------------------------- |
| message                             | action        | /cosmos.group.v1.Msg/CreateProposal |
| cosmos.group.v1.EventCreateProposal | proposal_id   | {proposalId}                        |

### EventWithdrawProposal

| Type                                  | Attribute Key | Attribute Value                       |
| ------------------------------------- | ------------- | ------------------------------------- |
| message                               | action        | /cosmos.group.v1.Msg/WithdrawProposal |
| cosmos.group.v1.EventWithdrawProposal | proposal_id   | {proposalId}                          |

### EventVote

| Type                      | Attribute Key | Attribute Value           |
| ------------------------- | ------------- | ------------------------- |
| message                   | action        | /cosmos.group.v1.Msg/Vote |
| cosmos.group.v1.EventVote | proposal_id   | {proposalId}              |

## EventExec

| Type                      | Attribute Key | Attribute Value           |
| ------------------------- | ------------- | ------------------------- |
| message                   | action        | /cosmos.group.v1.Msg/Exec |
| cosmos.group.v1.EventExec | proposal_id   | {proposalId}              |
| cosmos.group.v1.EventExec | logs          | {logs_string}             |

### EventLeaveGroup

| Type                            | Attribute Key | Attribute Value                 |
| ------------------------------- | ------------- | ------------------------------- |
| message                         | action        | /cosmos.group.v1.Msg/LeaveGroup |
| cosmos.group.v1.EventLeaveGroup | proposal_id   | {proposalId}                    |
| cosmos.group.v1.EventLeaveGroup | address       | {address}                       |

### EventProposalPruned

| Type                                | Attribute Key | Attribute Value                 |
|-------------------------------------|---------------|---------------------------------|
| message                             | action        | /cosmos.group.v1.Msg/LeaveGroup |
| cosmos.group.v1.EventProposalPruned | proposal_id   | {proposalId}                    |
| cosmos.group.v1.EventProposalPruned | status        | {ProposalStatus}                |
| cosmos.group.v1.EventProposalPruned | tally_result  | {TallyResult}                   |


## Client

### CLI

Người dùng có thể truy vấn và tương tác với module `group` bằng CLI.

#### Query

Các lệnh `query` cho phép người dùng truy vấn state của `group`.

```bash
simd query group --help
```

##### group-info

Lệnh `group-info` cho phép người dùng truy vấn thông tin group theo group id cho trước.

```bash
simd query group group-info [id] [flags]
```

Example:

```bash
simd query group group-info 1
```

Example Output:

```bash
admin: cosmos1..
group_id: "1"
metadata: AQ==
total_weight: "3"
version: "1"
```

##### group-policy-info

Lệnh `group-policy-info` cho phép người dùng truy vấn thông tin group policy theo địa chỉ tài khoản group policy.

```bash
simd query group group-policy-info [group-policy-account] [flags]
```

Example:

```bash
simd query group group-policy-info cosmos1..
```

Example Output:

```bash
address: cosmos1..
admin: cosmos1..
decision_policy:
  '@type': /cosmos.group.v1.ThresholdDecisionPolicy
  threshold: "1"
  windows:
      min_execution_period: 0s
      voting_period: 432000s
group_id: "1"
metadata: AQ==
version: "1"
```

##### group-members

Lệnh `group-members` cho phép người dùng truy vấn thành viên group theo group id với các cờ phân trang.

```bash
simd query group group-members [id] [flags]
```

Example:

```bash
simd query group group-members 1
```

Example Output:

```bash
members:
- group_id: "1"
  member:
    address: cosmos1..
    metadata: AQ==
    weight: "2"
- group_id: "1"
  member:
    address: cosmos1..
    metadata: AQ==
    weight: "1"
pagination:
  next_key: null
  total: "2"
```

##### groups-by-admin

Lệnh `groups-by-admin` cho phép người dùng truy vấn group theo địa chỉ tài khoản admin với các cờ phân trang.

```bash
simd query group groups-by-admin [admin] [flags]
```

Example:

```bash
simd query group groups-by-admin cosmos1..
```

Example Output:

```bash
groups:
- admin: cosmos1..
  group_id: "1"
  metadata: AQ==
  total_weight: "3"
  version: "1"
- admin: cosmos1..
  group_id: "2"
  metadata: AQ==
  total_weight: "3"
  version: "1"
pagination:
  next_key: null
  total: "2"
```

##### group-policies-by-group

Lệnh `group-policies-by-group` cho phép người dùng truy vấn group policy theo group id với các cờ phân trang.

```bash
simd query group group-policies-by-group [group-id] [flags]
```

Example:

```bash
simd query group group-policies-by-group 1
```

Example Output:

```bash
group_policies:
- address: cosmos1..
  admin: cosmos1..
  decision_policy:
    '@type': /cosmos.group.v1.ThresholdDecisionPolicy
    threshold: "1"
    windows:
      min_execution_period: 0s
      voting_period: 432000s
  group_id: "1"
  metadata: AQ==
  version: "1"
- address: cosmos1..
  admin: cosmos1..
  decision_policy:
    '@type': /cosmos.group.v1.ThresholdDecisionPolicy
    threshold: "1"
    windows:
      min_execution_period: 0s
      voting_period: 432000s
  group_id: "1"
  metadata: AQ==
  version: "1"
pagination:
  next_key: null
  total: "2"
```

##### group-policies-by-admin

Lệnh `group-policies-by-admin` cho phép người dùng truy vấn group policy theo địa chỉ tài khoản admin với các cờ phân trang.

```bash
simd query group group-policies-by-admin [admin] [flags]
```

Example:

```bash
simd query group group-policies-by-admin cosmos1..
```

Example Output:

```bash
group_policies:
- address: cosmos1..
  admin: cosmos1..
  decision_policy:
    '@type': /cosmos.group.v1.ThresholdDecisionPolicy
    threshold: "1"
    windows:
      min_execution_period: 0s
      voting_period: 432000s
  group_id: "1"
  metadata: AQ==
  version: "1"
- address: cosmos1..
  admin: cosmos1..
  decision_policy:
    '@type': /cosmos.group.v1.ThresholdDecisionPolicy
    threshold: "1"
    windows:
      min_execution_period: 0s
      voting_period: 432000s
  group_id: "1"
  metadata: AQ==
  version: "1"
pagination:
  next_key: null
  total: "2"
```

##### proposal

Lệnh `proposal` cho phép người dùng truy vấn proposal theo id.

```bash
simd query group proposal [id] [flags]
```

Example:

```bash
simd query group proposal 1
```

Example Output:

```bash
proposal:
  address: cosmos1..
  executor_result: EXECUTOR_RESULT_NOT_RUN
  group_policy_version: "1"
  group_version: "1"
  metadata: AQ==
  msgs:
  - '@type': /cosmos.bank.v1beta1.MsgSend
    amount:
    - amount: "100000000"
      denom: stake
    from_address: cosmos1..
    to_address: cosmos1..
  proposal_id: "1"
  proposers:
  - cosmos1..
  result: RESULT_UNFINALIZED
  status: STATUS_SUBMITTED
  submitted_at: "2021-12-17T07:06:26.310638964Z"
  windows:
    min_execution_period: 0s
    voting_period: 432000s
  vote_state:
    abstain_count: "0"
    no_count: "0"
    veto_count: "0"
    yes_count: "0"
  summary: "Summary"
  title: "Title"
```

##### proposals-by-group-policy

Lệnh `proposals-by-group-policy` cho phép người dùng truy vấn proposal theo địa chỉ tài khoản group policy với các cờ phân trang.

```bash
simd query group proposals-by-group-policy [group-policy-account] [flags]
```

Example:

```bash
simd query group proposals-by-group-policy cosmos1..
```

Example Output:

```bash
pagination:
  next_key: null
  total: "1"
proposals:
- address: cosmos1..
  executor_result: EXECUTOR_RESULT_NOT_RUN
  group_policy_version: "1"
  group_version: "1"
  metadata: AQ==
  msgs:
  - '@type': /cosmos.bank.v1beta1.MsgSend
    amount:
    - amount: "100000000"
      denom: stake
    from_address: cosmos1..
    to_address: cosmos1..
  proposal_id: "1"
  proposers:
  - cosmos1..
  result: RESULT_UNFINALIZED
  status: STATUS_SUBMITTED
  submitted_at: "2021-12-17T07:06:26.310638964Z"
  windows:
    min_execution_period: 0s
    voting_period: 432000s
  vote_state:
    abstain_count: "0"
    no_count: "0"
    veto_count: "0"
    yes_count: "0"
  summary: "Summary"
  title: "Title"
```

##### vote

Lệnh `vote` cho phép người dùng truy vấn vote theo proposal id và địa chỉ tài khoản voter.

```bash
simd query group vote [proposal-id] [voter] [flags]
```

Example:

```bash
simd query group vote 1 cosmos1..
```

Example Output:

```bash
vote:
  choice: CHOICE_YES
  metadata: AQ==
  proposal_id: "1"
  submitted_at: "2021-12-17T08:05:02.490164009Z"
  voter: cosmos1..
```

##### votes-by-proposal

Lệnh `votes-by-proposal` cho phép người dùng truy vấn vote theo proposal id với các cờ phân trang.

```bash
simd query group votes-by-proposal [proposal-id] [flags]
```

Example:

```bash
simd query group votes-by-proposal 1
```

Example Output:

```bash
pagination:
  next_key: null
  total: "1"
votes:
- choice: CHOICE_YES
  metadata: AQ==
  proposal_id: "1"
  submitted_at: "2021-12-17T08:05:02.490164009Z"
  voter: cosmos1..
```

##### votes-by-voter

Lệnh `votes-by-voter` cho phép người dùng truy vấn vote theo địa chỉ tài khoản voter với các cờ phân trang.

```bash
simd query group votes-by-voter [voter] [flags]
```

Example:

```bash
simd query group votes-by-voter cosmos1..
```

Example Output:

```bash
pagination:
  next_key: null
  total: "1"
votes:
- choice: CHOICE_YES
  metadata: AQ==
  proposal_id: "1"
  submitted_at: "2021-12-17T08:05:02.490164009Z"
  voter: cosmos1..
```

### Transactions

Các lệnh `tx` cho phép người dùng tương tác với module `group`.

```bash
simd tx group --help
```

#### create-group

Lệnh `create-group` cho phép người dùng tạo group là tập hợp các tài khoản thành viên với trọng số liên kết và tài khoản administrator.

```bash
simd tx group create-group [admin] [metadata] [members-json-file]
```

Example:

```bash
simd tx group create-group cosmos1.. "AQ==" members.json
```

#### update-group-admin

Lệnh `update-group-admin` cho phép người dùng cập nhật admin của group.

```bash
simd tx group update-group-admin [admin] [group-id] [new-admin] [flags]
```

Example:

```bash
simd tx group update-group-admin cosmos1.. 1 cosmos1..
```

#### update-group-members

Lệnh `update-group-members` cho phép người dùng cập nhật thành viên của group.

```bash
simd tx group update-group-members [admin] [group-id] [members-json-file] [flags]
```

Example:

```bash
simd tx group update-group-members cosmos1.. 1 members.json
```

#### update-group-metadata

Lệnh `update-group-metadata` cho phép người dùng cập nhật metadata của group.

```bash
simd tx group update-group-metadata [admin] [group-id] [metadata] [flags]
```

Example:

```bash
simd tx group update-group-metadata cosmos1.. 1 "AQ=="
```

#### create-group-policy

Lệnh `create-group-policy` cho phép người dùng tạo group policy là tài khoản liên kết với group và decision policy.

```bash
simd tx group create-group-policy [admin] [group-id] [metadata] [decision-policy] [flags]
```

Example:

```bash
simd tx group create-group-policy cosmos1.. 1 "AQ==" '{"@type":"/cosmos.group.v1.ThresholdDecisionPolicy", "threshold":"1", "windows": {"voting_period": "120h", "min_execution_period": "0s"}}'
```

#### create-group-with-policy

Lệnh `create-group-with-policy` cho phép người dùng tạo group là tập hợp các tài khoản thành viên với trọng số liên kết và tài khoản administrator với decision policy. Nếu cờ `--group-policy-as-admin` được đặt là `true`, địa chỉ group policy trở thành admin của group và group policy.

```bash
simd tx group create-group-with-policy [admin] [group-metadata] [group-policy-metadata] [members-json-file] [decision-policy] [flags]
```

Example:

```bash
simd tx group create-group-with-policy cosmos1.. "AQ==" "AQ==" members.json '{"@type":"/cosmos.group.v1.ThresholdDecisionPolicy", "threshold":"1", "windows": {"voting_period": "120h", "min_execution_period": "0s"}}'
```

#### update-group-policy-admin

Lệnh `update-group-policy-admin` cho phép người dùng cập nhật admin của group policy.

```bash
simd tx group update-group-policy-admin [admin] [group-policy-account] [new-admin] [flags]
```

Example:

```bash
simd tx group update-group-policy-admin cosmos1.. cosmos1.. cosmos1..
```

#### update-group-policy-metadata

Lệnh `update-group-policy-metadata` cho phép người dùng cập nhật metadata của group policy.

```bash
simd tx group update-group-policy-metadata [admin] [group-policy-account] [new-metadata] [flags]
```

Example:

```bash
simd tx group update-group-policy-metadata cosmos1.. cosmos1.. "AQ=="
```

#### update-group-policy-decision-policy

Lệnh `update-group-policy-decision-policy` cho phép người dùng cập nhật decision policy của group policy.

```bash
simd  tx group update-group-policy-decision-policy [admin] [group-policy-account] [decision-policy] [flags]
```

Example:

```bash
simd tx group update-group-policy-decision-policy cosmos1.. cosmos1.. '{"@type":"/cosmos.group.v1.ThresholdDecisionPolicy", "threshold":"2", "windows": {"voting_period": "120h", "min_execution_period": "0s"}}'
```

#### submit-proposal

Lệnh `submit-proposal` cho phép người dùng gửi proposal mới.

```bash
simd tx group submit-proposal [group-policy-account] [proposer[,proposer]*] [msg_tx_json_file] [metadata] [flags]
```

Example:

```bash
simd tx group submit-proposal cosmos1.. cosmos1.. msg_tx.json "AQ=="
```

#### withdraw-proposal

Lệnh `withdraw-proposal` cho phép người dùng rút proposal.

```bash
simd tx group withdraw-proposal [proposal-id] [group-policy-admin-or-proposer]
```

Example:

```bash
simd tx group withdraw-proposal 1 cosmos1..
```

#### vote

Lệnh `vote` cho phép người dùng bỏ phiếu cho proposal.

```bash
simd tx group vote proposal-id] [voter] [choice] [metadata] [flags]
```

Example:

```bash
simd tx group vote 1 cosmos1.. CHOICE_YES "AQ=="
```

#### exec

Lệnh `exec` cho phép người dùng thực thi proposal.

```bash
simd tx group exec [proposal-id] [flags]
```

Example:

```bash
simd tx group exec 1
```

#### leave-group

Lệnh `leave-group` cho phép thành viên group rời khỏi group.

```bash
simd tx group leave-group [member-address] [group-id]
```

Example:

```bash
simd tx group leave-group cosmos1... 1
```

### gRPC

Người dùng có thể truy vấn module `group` bằng các endpoint gRPC.

#### GroupInfo

Endpoint `GroupInfo` cho phép người dùng truy vấn thông tin group theo group id cho trước.

```bash
cosmos.group.v1.Query/GroupInfo
```

Example:

```bash
grpcurl -plaintext \
    -d '{"group_id":1}' localhost:9090 cosmos.group.v1.Query/GroupInfo
```

Example Output:

```bash
{
  "info": {
    "groupId": "1",
    "admin": "cosmos1..",
    "metadata": "AQ==",
    "version": "1",
    "totalWeight": "3"
  }
}
```

#### GroupPolicyInfo

Endpoint `GroupPolicyInfo` cho phép người dùng truy vấn thông tin group policy theo địa chỉ tài khoản group policy.

```bash
cosmos.group.v1.Query/GroupPolicyInfo
```

Example:

```bash
grpcurl -plaintext \
    -d '{"address":"cosmos1.."}'  localhost:9090 cosmos.group.v1.Query/GroupPolicyInfo
```

Example Output:

```bash
{
  "info": {
    "address": "cosmos1..",
    "groupId": "1",
    "admin": "cosmos1..",
    "version": "1",
    "decisionPolicy": {"@type":"/cosmos.group.v1.ThresholdDecisionPolicy","threshold":"1","windows": {"voting_period": "120h", "min_execution_period": "0s"}},
  }
}
```

#### GroupMembers

Endpoint `GroupMembers` cho phép người dùng truy vấn thành viên group theo group id với các cờ phân trang.

```bash
cosmos.group.v1.Query/GroupMembers
```

Example:

```bash
grpcurl -plaintext \
    -d '{"group_id":"1"}'  localhost:9090 cosmos.group.v1.Query/GroupMembers
```

Example Output:

```bash
{
  "members": [
    {
      "groupId": "1",
      "member": {
        "address": "cosmos1..",
        "weight": "1"
      }
    },
    {
      "groupId": "1",
      "member": {
        "address": "cosmos1..",
        "weight": "2"
      }
    }
  ],
  "pagination": {
    "total": "2"
  }
}
```

#### GroupsByAdmin

Endpoint `GroupsByAdmin` cho phép người dùng truy vấn group theo địa chỉ tài khoản admin với các cờ phân trang.

```bash
cosmos.group.v1.Query/GroupsByAdmin
```

Example:

```bash
grpcurl -plaintext \
    -d '{"admin":"cosmos1.."}'  localhost:9090 cosmos.group.v1.Query/GroupsByAdmin
```

Example Output:

```bash
{
  "groups": [
    {
      "groupId": "1",
      "admin": "cosmos1..",
      "metadata": "AQ==",
      "version": "1",
      "totalWeight": "3"
    },
    {
      "groupId": "2",
      "admin": "cosmos1..",
      "metadata": "AQ==",
      "version": "1",
      "totalWeight": "3"
    }
  ],
  "pagination": {
    "total": "2"
  }
}
```

#### GroupPoliciesByGroup

Endpoint `GroupPoliciesByGroup` cho phép người dùng truy vấn group policy theo group id với các cờ phân trang.

```bash
cosmos.group.v1.Query/GroupPoliciesByGroup
```

Example:

```bash
grpcurl -plaintext \
    -d '{"group_id":"1"}'  localhost:9090 cosmos.group.v1.Query/GroupPoliciesByGroup
```

Example Output:

```bash
{
  "GroupPolicies": [
    {
      "address": "cosmos1..",
      "groupId": "1",
      "admin": "cosmos1..",
      "version": "1",
      "decisionPolicy": {"@type":"/cosmos.group.v1.ThresholdDecisionPolicy","threshold":"1","windows":{"voting_period": "120h", "min_execution_period": "0s"}},
    },
    {
      "address": "cosmos1..",
      "groupId": "1",
      "admin": "cosmos1..",
      "version": "1",
      "decisionPolicy": {"@type":"/cosmos.group.v1.ThresholdDecisionPolicy","threshold":"1","windows":{"voting_period": "120h", "min_execution_period": "0s"}},
    }
  ],
  "pagination": {
    "total": "2"
  }
}
```

#### GroupPoliciesByAdmin

Endpoint `GroupPoliciesByAdmin` cho phép người dùng truy vấn group policy theo địa chỉ tài khoản admin với các cờ phân trang.

```bash
cosmos.group.v1.Query/GroupPoliciesByAdmin
```

Example:

```bash
grpcurl -plaintext \
    -d '{"admin":"cosmos1.."}'  localhost:9090 cosmos.group.v1.Query/GroupPoliciesByAdmin
```

Example Output:

```bash
{
  "GroupPolicies": [
    {
      "address": "cosmos1..",
      "groupId": "1",
      "admin": "cosmos1..",
      "version": "1",
      "decisionPolicy": {"@type":"/cosmos.group.v1.ThresholdDecisionPolicy","threshold":"1","windows":{"voting_period": "120h", "min_execution_period": "0s"}},
    },
    {
      "address": "cosmos1..",
      "groupId": "1",
      "admin": "cosmos1..",
      "version": "1",
      "decisionPolicy": {"@type":"/cosmos.group.v1.ThresholdDecisionPolicy","threshold":"1","windows":{"voting_period": "120h", "min_execution_period": "0s"}},
    }
  ],
  "pagination": {
    "total": "2"
  }
}
```

#### Proposal

Endpoint `Proposal` cho phép người dùng truy vấn proposal theo id.

```bash
cosmos.group.v1.Query/Proposal
```

Example:

```bash
grpcurl -plaintext \
    -d '{"proposal_id":"1"}'  localhost:9090 cosmos.group.v1.Query/Proposal
```

Example Output:

```bash
{
  "proposal": {
    "proposalId": "1",
    "address": "cosmos1..",
    "proposers": [
      "cosmos1.."
    ],
    "submittedAt": "2021-12-17T07:06:26.310638964Z",
    "groupVersion": "1",
    "GroupPolicyVersion": "1",
    "status": "STATUS_SUBMITTED",
    "result": "RESULT_UNFINALIZED",
    "voteState": {
      "yesCount": "0",
      "noCount": "0",
      "abstainCount": "0",
      "vetoCount": "0"
    },
    "windows": {
      "min_execution_period": "0s",
      "voting_period": "432000s"
    },
    "executorResult": "EXECUTOR_RESULT_NOT_RUN",
    "messages": [
      {"@type":"/cosmos.bank.v1beta1.MsgSend","amount":[{"denom":"stake","amount":"100000000"}],"fromAddress":"cosmos1..","toAddress":"cosmos1.."}
    ],
    "title": "Title",
    "summary": "Summary",
  }
}
```

#### ProposalsByGroupPolicy

Endpoint `ProposalsByGroupPolicy` cho phép người dùng truy vấn proposal theo địa chỉ tài khoản group policy với các cờ phân trang.

```bash
cosmos.group.v1.Query/ProposalsByGroupPolicy
```

Example:

```bash
grpcurl -plaintext \
    -d '{"address":"cosmos1.."}'  localhost:9090 cosmos.group.v1.Query/ProposalsByGroupPolicy
```

Example Output:

```bash
{
  "proposals": [
    {
      "proposalId": "1",
      "address": "cosmos1..",
      "proposers": [
        "cosmos1.."
      ],
      "submittedAt": "2021-12-17T08:03:27.099649352Z",
      "groupVersion": "1",
      "GroupPolicyVersion": "1",
      "status": "STATUS_CLOSED",
      "result": "RESULT_ACCEPTED",
      "voteState": {
        "yesCount": "1",
        "noCount": "0",
        "abstainCount": "0",
        "vetoCount": "0"
      },
      "windows": {
        "min_execution_period": "0s",
        "voting_period": "432000s"
      },
      "executorResult": "EXECUTOR_RESULT_NOT_RUN",
      "messages": [
        {"@type":"/cosmos.bank.v1beta1.MsgSend","amount":[{"denom":"stake","amount":"100000000"}],"fromAddress":"cosmos1..","toAddress":"cosmos1.."}
      ],
      "title": "Title",
      "summary": "Summary",
    }
  ],
  "pagination": {
    "total": "1"
  }
}
```

#### VoteByProposalVoter

Endpoint `VoteByProposalVoter` cho phép người dùng truy vấn vote theo proposal id và địa chỉ tài khoản voter.

```bash
cosmos.group.v1.Query/VoteByProposalVoter
```

Example:

```bash
grpcurl -plaintext \
    -d '{"proposal_id":"1","voter":"cosmos1.."}'  localhost:9090 cosmos.group.v1.Query/VoteByProposalVoter
```

Example Output:

```bash
{
  "vote": {
    "proposalId": "1",
    "voter": "cosmos1..",
    "choice": "CHOICE_YES",
    "submittedAt": "2021-12-17T08:05:02.490164009Z"
  }
}
```

#### VotesByProposal

Endpoint `VotesByProposal` cho phép người dùng truy vấn vote theo proposal id với các cờ phân trang.

```bash
cosmos.group.v1.Query/VotesByProposal
```

Example:

```bash
grpcurl -plaintext \
    -d '{"proposal_id":"1"}'  localhost:9090 cosmos.group.v1.Query/VotesByProposal
```

Example Output:

```bash
{
  "votes": [
    {
      "proposalId": "1",
      "voter": "cosmos1..",
      "choice": "CHOICE_YES",
      "submittedAt": "2021-12-17T08:05:02.490164009Z"
    }
  ],
  "pagination": {
    "total": "1"
  }
}
```

#### VotesByVoter

Endpoint `VotesByVoter` cho phép người dùng truy vấn vote theo địa chỉ tài khoản voter với các cờ phân trang.

```bash
cosmos.group.v1.Query/VotesByVoter
```

Example:

```bash
grpcurl -plaintext \
    -d '{"voter":"cosmos1.."}'  localhost:9090 cosmos.group.v1.Query/VotesByVoter
```

Example Output:

```bash
{
  "votes": [
    {
      "proposalId": "1",
      "voter": "cosmos1..",
      "choice": "CHOICE_YES",
      "submittedAt": "2021-12-17T08:05:02.490164009Z"
    }
  ],
  "pagination": {
    "total": "1"
  }
}
```

### REST

Người dùng có thể truy vấn module `group` bằng các endpoint REST.

#### GroupInfo

Endpoint `GroupInfo` cho phép người dùng truy vấn thông tin group theo group id cho trước.

```bash
/cosmos/group/v1/group_info/{group_id}
```

Example:

```bash
curl localhost:1317/cosmos/group/v1/group_info/1
```

Example Output:

```bash
{
  "info": {
    "id": "1",
    "admin": "cosmos1..",
    "metadata": "AQ==",
    "version": "1",
    "total_weight": "3"
  }
}
```

#### GroupPolicyInfo

Endpoint `GroupPolicyInfo` cho phép người dùng truy vấn thông tin group policy theo địa chỉ tài khoản group policy.

```bash
/cosmos/group/v1/group_policy_info/{address}
```

Example:

```bash
curl localhost:1317/cosmos/group/v1/group_policy_info/cosmos1..
```

Example Output:

```bash
{
  "info": {
    "address": "cosmos1..",
    "group_id": "1",
    "admin": "cosmos1..",
    "metadata": "AQ==",
    "version": "1",
    "decision_policy": {
      "@type": "/cosmos.group.v1.ThresholdDecisionPolicy",
      "threshold": "1",
      "windows": {
        "voting_period": "120h",
        "min_execution_period": "0s"
      }
    },
  }
}
```

#### GroupMembers

Endpoint `GroupMembers` cho phép người dùng truy vấn thành viên group theo group id với các cờ phân trang.

```bash
/cosmos/group/v1/group_members/{group_id}
```

Example:

```bash
curl localhost:1317/cosmos/group/v1/group_members/1
```

Example Output:

```bash
{
  "members": [
    {
      "group_id": "1",
      "member": {
        "address": "cosmos1..",
        "weight": "1",
        "metadata": "AQ=="
      }
    },
    {
      "group_id": "1",
      "member": {
        "address": "cosmos1..",
        "weight": "2",
        "metadata": "AQ=="
    }
  ],
  "pagination": {
    "next_key": null,
    "total": "2"
  }
}
```

#### GroupsByAdmin

Endpoint `GroupsByAdmin` cho phép người dùng truy vấn group theo địa chỉ tài khoản admin với các cờ phân trang.

```bash
/cosmos/group/v1/groups_by_admin/{admin}
```

Example:

```bash
curl localhost:1317/cosmos/group/v1/groups_by_admin/cosmos1..
```

Example Output:

```bash
{
  "groups": [
    {
      "id": "1",
      "admin": "cosmos1..",
      "metadata": "AQ==",
      "version": "1",
      "total_weight": "3"
    },
    {
      "id": "2",
      "admin": "cosmos1..",
      "metadata": "AQ==",
      "version": "1",
      "total_weight": "3"
    }
  ],
  "pagination": {
    "next_key": null,
    "total": "2"
  }
}
```

#### GroupPoliciesByGroup

Endpoint `GroupPoliciesByGroup` cho phép người dùng truy vấn group policy theo group id với các cờ phân trang.

```bash
/cosmos/group/v1/group_policies_by_group/{group_id}
```

Example:

```bash
curl localhost:1317/cosmos/group/v1/group_policies_by_group/1
```

Example Output:

```bash
{
  "group_policies": [
    {
      "address": "cosmos1..",
      "group_id": "1",
      "admin": "cosmos1..",
      "metadata": "AQ==",
      "version": "1",
      "decision_policy": {
        "@type": "/cosmos.group.v1.ThresholdDecisionPolicy",
        "threshold": "1",
        "windows": {
          "voting_period": "120h",
          "min_execution_period": "0s"
      }
      },
    },
    {
      "address": "cosmos1..",
      "group_id": "1",
      "admin": "cosmos1..",
      "metadata": "AQ==",
      "version": "1",
      "decision_policy": {
        "@type": "/cosmos.group.v1.ThresholdDecisionPolicy",
        "threshold": "1",
        "windows": {
          "voting_period": "120h",
          "min_execution_period": "0s"
      }
      },
    }
  ],
  "pagination": {
    "next_key": null,
    "total": "2"
  }
}
```

#### GroupPoliciesByAdmin

Endpoint `GroupPoliciesByAdmin` cho phép người dùng truy vấn group policy theo địa chỉ tài khoản admin với các cờ phân trang.

```bash
/cosmos/group/v1/group_policies_by_admin/{admin}
```

Example:

```bash
curl localhost:1317/cosmos/group/v1/group_policies_by_admin/cosmos1..
```

Example Output:

```bash
{
  "group_policies": [
    {
      "address": "cosmos1..",
      "group_id": "1",
      "admin": "cosmos1..",
      "metadata": "AQ==",
      "version": "1",
      "decision_policy": {
        "@type": "/cosmos.group.v1.ThresholdDecisionPolicy",
        "threshold": "1",
        "windows": {
          "voting_period": "120h",
          "min_execution_period": "0s"
      } 
      },
    },
    {
      "address": "cosmos1..",
      "group_id": "1",
      "admin": "cosmos1..",
      "metadata": "AQ==",
      "version": "1",
      "decision_policy": {
        "@type": "/cosmos.group.v1.ThresholdDecisionPolicy",
        "threshold": "1",
        "windows": {
          "voting_period": "120h",
          "min_execution_period": "0s"
      }
      },
    }
  ],
  "pagination": {
    "next_key": null,
    "total": "2"
  }
```

#### Proposal

Endpoint `Proposal` cho phép người dùng truy vấn proposal theo id.

```bash
/cosmos/group/v1/proposal/{proposal_id}
```

Example:

```bash
curl localhost:1317/cosmos/group/v1/proposal/1
```

Example Output:

```bash
{
  "proposal": {
    "proposal_id": "1",
    "address": "cosmos1..",
    "metadata": "AQ==",
    "proposers": [
      "cosmos1.."
    ],
    "submitted_at": "2021-12-17T07:06:26.310638964Z",
    "group_version": "1",
    "group_policy_version": "1",
    "status": "STATUS_SUBMITTED",
    "result": "RESULT_UNFINALIZED",
    "vote_state": {
      "yes_count": "0",
      "no_count": "0",
      "abstain_count": "0",
      "veto_count": "0"
    },
    "windows": {
      "min_execution_period": "0s",
      "voting_period": "432000s"
    },
    "executor_result": "EXECUTOR_RESULT_NOT_RUN",
    "messages": [
      {
        "@type": "/cosmos.bank.v1beta1.MsgSend",
        "from_address": "cosmos1..",
        "to_address": "cosmos1..",
        "amount": [
          {
            "denom": "stake",
            "amount": "100000000"
          }
        ]
      }
    ],
    "title": "Title",
    "summary": "Summary",
  }
}
```

#### ProposalsByGroupPolicy

Endpoint `ProposalsByGroupPolicy` cho phép người dùng truy vấn proposal theo địa chỉ tài khoản group policy với các cờ phân trang.

```bash
/cosmos/group/v1/proposals_by_group_policy/{address}
```

Example:

```bash
curl localhost:1317/cosmos/group/v1/proposals_by_group_policy/cosmos1..
```

Example Output:

```bash
{
  "proposals": [
    {
      "id": "1",
      "group_policy_address": "cosmos1..",
      "metadata": "AQ==",
      "proposers": [
        "cosmos1.."
      ],
      "submit_time": "2021-12-17T08:03:27.099649352Z",
      "group_version": "1",
      "group_policy_version": "1",
      "status": "STATUS_CLOSED",
      "result": "RESULT_ACCEPTED",
      "vote_state": {
        "yes_count": "1",
        "no_count": "0",
        "abstain_count": "0",
        "veto_count": "0"
      },
      "windows": {
        "min_execution_period": "0s",
        "voting_period": "432000s"
      },
      "executor_result": "EXECUTOR_RESULT_NOT_RUN",
      "messages": [
        {
          "@type": "/cosmos.bank.v1beta1.MsgSend",
          "from_address": "cosmos1..",
          "to_address": "cosmos1..",
          "amount": [
            {
              "denom": "stake",
              "amount": "100000000"
            }
          ]
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

#### VoteByProposalVoter

Endpoint `VoteByProposalVoter` cho phép người dùng truy vấn vote theo proposal id và địa chỉ tài khoản voter.

```bash
/cosmos/group/v1/vote_by_proposal_voter/{proposal_id}/{voter}
```

Example:

```bash
curl localhost:1317/cosmos/group/v1beta1/vote_by_proposal_voter/1/cosmos1..
```

Example Output:

```bash
{
  "vote": {
    "proposal_id": "1",
    "voter": "cosmos1..",
    "choice": "CHOICE_YES",
    "metadata": "AQ==",
    "submitted_at": "2021-12-17T08:05:02.490164009Z"
  }
}
```

#### VotesByProposal

Endpoint `VotesByProposal` cho phép người dùng truy vấn vote theo proposal id với các cờ phân trang.

```bash
/cosmos/group/v1/votes_by_proposal/{proposal_id}
```

Example:

```bash
curl localhost:1317/cosmos/group/v1/votes_by_proposal/1
```

Example Output:

```bash
{
  "votes": [
    {
      "proposal_id": "1",
      "voter": "cosmos1..",
      "option": "CHOICE_YES",
      "metadata": "AQ==",
      "submit_time": "2021-12-17T08:05:02.490164009Z"
    }
  ],
  "pagination": {
    "next_key": null,
    "total": "1"
  }
}
```

#### VotesByVoter

Endpoint `VotesByVoter` cho phép người dùng truy vấn vote theo địa chỉ tài khoản voter với các cờ phân trang.

```bash
/cosmos/group/v1/votes_by_voter/{voter}
```

Example:

```bash
curl localhost:1317/cosmos/group/v1/votes_by_voter/cosmos1..
```

Example Output:

```bash
{
  "votes": [
    {
      "proposal_id": "1",
      "voter": "cosmos1..",
      "choice": "CHOICE_YES",
      "metadata": "AQ==",
      "submitted_at": "2021-12-17T08:05:02.490164009Z"
    }
  ],
  "pagination": {
    "next_key": null,
    "total": "1"
  }
}
```

## Metadata

Module group có bốn vị trí cho metadata nơi người dùng có thể cung cấp thêm ngữ cảnh về các hành động trên chuỗi mà họ đang thực hiện. Theo mặc định tất cả các trường metadata có giới hạn độ dài 255 ký tự nơi metadata có thể được lưu trữ ở định dạng json, có thể on-chain hoặc off-chain tùy thuộc vào lượng dữ liệu cần thiết. Ở đây chúng tôi cung cấp khuyến nghị cho cấu trúc json và nơi dữ liệu nên được lưu trữ. Có hai yếu tố quan trọng trong việc đưa ra các khuyến nghị này. Thứ nhất, module group và gov phải nhất quán với nhau, lưu ý số lượng proposal do tất cả group tạo có thể khá lớn. Thứ hai, các ứng dụng client như block explorer và giao diện governance có thể tin tưởng vào tính nhất quán của cấu trúc metadata trên các chuỗi.

### Proposal

Vị trí: off-chain dưới dạng đối tượng json lưu trên IPFS (tương tự [gov proposal](../gov/README.md#metadata))

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
Trường `authors` là mảng chuỗi, điều này cho phép liệt kê nhiều tác giả trong metadata.
Trong v0.46, trường `authors` là chuỗi phân tách bằng dấu phẩy. Frontend được khuyến khích hỗ trợ cả hai định dạng để tương thích ngược.
:::

### Vote

Vị trí: on-chain dưới dạng json trong giới hạn 255 ký tự (tương tự [gov vote](../gov/README.md#metadata))

```json
{
  "justification": "",
}
```

### Group

Vị trí: off-chain dưới dạng đối tượng json lưu trên IPFS

```json
{
  "name": "",
  "description": "",
  "group_website_url": "",
  "group_forum_url": "",
}
```

### Decision policy

Vị trí: on-chain dưới dạng json trong giới hạn 255 ký tự

```json
{
  "name": "",
  "description": "",
}
```
