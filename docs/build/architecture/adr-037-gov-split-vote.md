# ADR 037: Phiếu Bầu Quản Trị Chia Phần

## Nhật Ký Thay Đổi

* 28/10/2020: Bản nháp đầu tiên

## Trạng Thái

Đã Chấp Nhận

## Tóm Tắt

ADR này định nghĩa một sửa đổi cho module quản trị sẽ cho phép người stake chia phiếu bầu của họ thành nhiều tùy chọn bầu. Ví dụ, có thể sử dụng 70% quyền bầu để bỏ phiếu Có và 30% quyền bầu để bỏ phiếu Không.

## Bối Cảnh

Hiện tại, một địa chỉ chỉ có thể đặt phiếu với một tùy chọn (Có/Không/Kiêm/KhôngKèmVeto) và sử dụng toàn bộ quyền bầu phía sau lựa chọn đó.

Tuy nhiên, thực thể sở hữu địa chỉ đó thường có thể không phải là một cá nhân. Ví dụ, một công ty có thể có các bên liên quan khác nhau muốn bầu khác nhau, và vì vậy việc cho phép họ chia quyền bầu sẽ có ý nghĩa. Một trường hợp sử dụng ví dụ khác là các sàn giao dịch. Nhiều sàn giao dịch tập trung thường stake một phần token của người dùng trong sự giám hộ của họ. Hiện tại, không thể thực hiện "bỏ phiếu hộ" và trao quyền bầu cho người dùng đối với token của họ. Tuy nhiên, với hệ thống này, các sàn giao dịch có thể thăm dò người dùng về sở thích bầu chọn, và sau đó bầu on-chain theo tỷ lệ với kết quả của cuộc thăm dò.

## Quyết Định

Chúng ta sửa đổi các struct phiếu thành

```go
type WeightedVoteOption struct {
  Option string
  Weight sdk.Dec
}

type Vote struct {
  ProposalID int64
  Voter      sdk.Address
  Options    []WeightedVoteOption
}
```

Và để tương thích ngược, chúng ta giới thiệu `MsgVoteWeighted` trong khi vẫn giữ `MsgVote`.

```go
type MsgVote struct {
  ProposalID int64
  Voter      sdk.Address
  Option     Option
}

type MsgVoteWeighted struct {
  ProposalID int64
  Voter      sdk.Address
  Options    []WeightedVoteOption
}
```

`ValidateBasic` của struct `MsgVoteWeighted` sẽ yêu cầu rằng

1. Tổng của tất cả các tỷ lệ bằng 1.0
2. Không có Option nào bị lặp lại

Hàm tally quản trị sẽ lặp qua tất cả các tùy chọn trong một phiếu và thêm vào tally kết quả của quyền bầu của voter * tỷ lệ cho tùy chọn đó.

```go
tally() {
    results := map[types.VoteOption]sdk.Dec

    for _, vote := range votes {
        for i, weightedOption := range vote.Options {
            results[weightedOption.Option] += getVotingPower(vote.voter) * weightedOption.Weight
        }
    }
}
```

Lệnh CLI để tạo phiếu đa tùy chọn sẽ như sau:

```shell
simd tx gov vote 1 "yes=0.6,no=0.3,abstain=0.05,no_with_veto=0.05" --from mykey
```

Để tạo phiếu một tùy chọn, người dùng có thể thực hiện theo hai cách:

```shell
simd tx gov vote 1 "yes=1" --from mykey
```

hoặc

```shell
simd tx gov vote 1 yes --from mykey
```

để duy trì tương thích ngược.

## Hậu Quả

### Tương Thích Ngược

* Các kiểu VoteMsg trước đây sẽ giữ nguyên và do đó các client sẽ không phải cập nhật quy trình của họ trừ khi họ muốn hỗ trợ tính năng WeightedVoteMsg.
* Khi truy vấn struct Vote từ trạng thái, cấu trúc của nó sẽ khác, và do đó các client muốn hiển thị tất cả voters và phiếu bầu tương ứng của họ sẽ phải xử lý định dạng mới và thực tế rằng một voter có thể có phiếu bầu chia.
* Kết quả của việc truy vấn hàm tally nên có cùng API với các client.

### Tích Cực

* Có thể làm cho quá trình bầu chính xác hơn đối với các địa chỉ đại diện cho nhiều bên liên quan, thường là một trong những địa chỉ lớn nhất.

### Tiêu Cực

* Phức tạp hơn so với bầu đơn giản, và vì vậy có thể khó giải thích cho người dùng hơn. Tuy nhiên, điều này phần lớn được giảm nhẹ vì tính năng là tùy chọn.

### Trung Lập

* Thay đổi tương đối nhỏ đối với hàm tally quản trị.
