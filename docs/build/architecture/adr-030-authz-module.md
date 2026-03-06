# ADR 030: Module Authorization (Authz)

## Nhật Ký Thay Đổi

* 06/11/2019: Bản nháp đầu tiên
* 12/10/2020: Cập nhật bản nháp
* 13/11/2020: Đã chấp nhận
* 06/05/2020: Cập nhật API proto, sử dụng `sdk.Msg` thay vì `sdk.ServiceMsg` (khái niệm sau đã bị xóa khỏi Cosmos SDK)
* 20/04/2022: Cập nhật tài liệu proto `SendAuthorization` để làm rõ `SpendLimit` là trường bắt buộc.

## Trạng Thái

Đã Chấp Nhận

## Tóm Tắt

ADR này định nghĩa module `x/authz` cho phép các tài khoản cấp ủy quyền để thực hiện các hành động thay mặt cho tài khoản đó cho các tài khoản khác.

## Bối Cảnh

Các trường hợp sử dụng cụ thể thúc đẩy module này bao gồm:

* Mong muốn ủy quyền khả năng bỏ phiếu về đề xuất cho các tài khoản khác ngoài tài khoản mà mình đã ủy thác stake
* Chức năng "sub-keys", như được đề xuất ban đầu trong [\#4480](https://github.com/cosmos/cosmos-sdk/issues/4480) là thuật ngữ mô tả chức năng được cung cấp bởi module này cùng với module `fee_grant` từ [ADR 029](./adr-029-fee-grant-module.md) và [group module](https://github.com/cosmos/cosmos-sdk/tree/main/x/group).

Chức năng "sub-keys" đại khái đề cập đến khả năng cho một tài khoản cấp một tập con khả năng của nó cho các tài khoản khác với các biện pháp bảo mật có thể kém mạnh mẽ hơn nhưng dễ sử dụng hơn. Ví dụ, một tài khoản master đại diện cho một tổ chức có thể cấp khả năng chi tiêu một số tiền nhỏ từ quỹ của tổ chức cho các tài khoản nhân viên cá nhân. Hoặc một cá nhân (hoặc nhóm) với ví multisig có thể cấp khả năng bỏ phiếu về đề xuất cho bất kỳ một trong các khóa thành viên nào.

Triển khai hiện tại dựa trên công việc được thực hiện bởi [nhóm Gaian tại Hackatom Berlin 2019](https://github.com/cosmos-gaians/cosmos-sdk/tree/hackatom/x/delegation).

## Quyết Định

Chúng ta sẽ tạo một module có tên `authz` cung cấp chức năng cấp quyền tùy ý từ một tài khoản (_granter_) cho tài khoản khác (_grantee_). Các ủy quyền phải được cấp cho từng phương thức `Msg` service cụ thể một, sử dụng một triển khai của interface `Authorization`.

### Các Kiểu Dữ Liệu

Các ủy quyền xác định chính xác quyền gì được cấp. Chúng có thể mở rộng và có thể được định nghĩa cho bất kỳ phương thức `Msg` service nào ngay cả bên ngoài module nơi phương thức `Msg` được định nghĩa. `Authorization` tham chiếu `Msg` sử dụng TypeURL của chúng.

#### Authorization

```go
type Authorization interface {
	proto.Message

	// MsgTypeURL trả về TypeURL Msg đầy đủ (như được mô tả trong ADR 020),
	// sẽ xử lý và chấp nhận hoặc từ chối yêu cầu.
	MsgTypeURL() string

	// Accept xác định liệu khoản cấp này có cho phép sdk.Msg được cung cấp thực hiện hay không, và nếu
	// có, cung cấp một thể hiện ủy quyền được nâng cấp.
	Accept(ctx sdk.Context, msg sdk.Msg) (AcceptResponse, error)

	// ValidateBasic thực hiện kiểm tra xác nhận đơn giản không
	// cần truy cập vào bất kỳ thông tin nào khác.
	ValidateBasic() error
}

// AcceptResponse hướng dẫn controller của một authz message nếu yêu cầu được chấp nhận
// và nếu nó nên được cập nhật hay xóa.
type AcceptResponse struct {
	// Nếu Accept=true, controller có thể chấp nhận và ủy quyền và xử lý cập nhật.
	Accept bool
	// Nếu Delete=true, controller phải xóa đối tượng ủy quyền và giải phóng
	// tài nguyên storage.
	Delete bool
	// Controller gọi Authorization.Accept phải kiểm tra xem `Updated != nil`. Nếu có,
	// phải sử dụng phiên bản được cập nhật và xử lý cập nhật ở cấp storage.
	Updated Authorization
}
```

Ví dụ `SendAuthorization` như thế này được định nghĩa cho `MsgSend` lấy `SpendLimit` và cập nhật nó xuống về không:

```go
type SendAuthorization struct {
	// SpendLimit chỉ định số lượng token tối đa có thể được chi tiêu
	// bởi ủy quyền này và sẽ được cập nhật khi token được chi tiêu. Trường này là bắt buộc.
	SpendLimit sdk.Coins
}

func (a SendAuthorization) MsgTypeURL() string {
	return sdk.MsgTypeURL(&MsgSend{})
}

func (a SendAuthorization) Accept(ctx sdk.Context, msg sdk.Msg) (authz.AcceptResponse, error) {
	mSend, ok := msg.(*MsgSend)
	if !ok {
		return authz.AcceptResponse{}, sdkerrors.ErrInvalidType.Wrap("type mismatch")
	}
	limitLeft, isNegative := a.SpendLimit.SafeSub(mSend.Amount)
	if isNegative {
		return authz.AcceptResponse{}, sdkerrors.ErrInsufficientFunds.Wrapf("requested amount is more than spend limit")
	}
	if limitLeft.IsZero() {
		return authz.AcceptResponse{Accept: true, Delete: true}, nil
	}

	return authz.AcceptResponse{Accept: true, Delete: false, Updated: &SendAuthorization{SpendLimit: limitLeft}}, nil
}
```

Một loại khả năng khác cho `MsgSend` có thể được triển khai sử dụng interface `Authorization` mà không cần thay đổi module `bank` cơ bản.

##### Ghi Chú Nhỏ về `AcceptResponse`

* Trường `AcceptResponse.Accept` sẽ được đặt thành `true` nếu ủy quyền được chấp nhận. Tuy nhiên, nếu bị từ chối, hàm `Accept` sẽ phát sinh lỗi (mà không đặt `AcceptResponse.Accept` thành `false`).

* Trường `AcceptResponse.Updated` sẽ được đặt thành giá trị không phải nil chỉ khi có thay đổi thực sự đối với ủy quyền. Nếu ủy quyền giữ nguyên (như luôn luôn là trường hợp với [`GenericAuthorization`](#genericauthorization)), trường sẽ là `nil`.

### `Msg` Service

```protobuf
service Msg {
  // Grant cấp ủy quyền được cung cấp cho grantee trên
  // tài khoản của granter với thời gian hết hạn được cung cấp.
  rpc Grant(MsgGrant) returns (MsgGrantResponse);

  // Exec cố gắng thực thi các message được cung cấp sử dụng
  // các ủy quyền được cấp cho grantee. Mỗi message chỉ nên có một
  // signer tương ứng với granter của ủy quyền.
  rpc Exec(MsgExec) returns (MsgExecResponse);

  // Revoke thu hồi bất kỳ ủy quyền nào tương ứng với tên phương thức được cung cấp trên
  // tài khoản của granter đã được cấp cho grantee.
  rpc Revoke(MsgRevoke) returns (MsgRevokeResponse);
}

// Grant cấp quyền thực thi
// phương thức được cung cấp với thời gian hết hạn.
message Grant {
  google.protobuf.Any       authorization = 1 [(cosmos_proto.accepts_interface) = "cosmos.authz.v1beta1.Authorization"];
  google.protobuf.Timestamp expiration    = 2 [(gogoproto.stdtime) = true, (gogoproto.nullable) = false];
}

message MsgGrant {
  string granter = 1;
  string grantee = 2;

  Grant grant = 3 [(gogoproto.nullable) = false];
}

message MsgExecResponse {
  cosmos.base.abci.v1beta1.Result result = 1;
}

message MsgExec {
  string   grantee                  = 1;
  // Các yêu cầu Msg ủy quyền để thực thi. Mỗi msg phải triển khai interface Authorization
  repeated google.protobuf.Any msgs = 2 [(cosmos_proto.accepts_interface) = "cosmos.base.v1beta1.Msg"];
}
```

### Router Middleware

`authz` `Keeper` sẽ lộ một phương thức `DispatchActions` cho phép các module khác gửi `Msg` tới router dựa trên cấp ủy quyền `Authorization`:

```go
type Keeper interface {
	// DispatchActions định tuyến các msg được cung cấp tới các handler tương ứng nếu grantee được cấp ủy quyền
	// để gửi các message đó bởi người ký đầu tiên (và duy nhất) của mỗi msg.
    DispatchActions(ctx sdk.Context, grantee sdk.AccAddress, msgs []sdk.Msg) sdk.Result
}
```

### CLI

#### Phương Thức `tx exec`

Khi người dùng CLI muốn chạy một giao dịch thay mặt cho tài khoản khác sử dụng `MsgExec`, họ có thể sử dụng phương thức `exec`. Ví dụ `gaiacli tx gov vote 1 yes --from <grantee> --generate-only | gaiacli tx authz exec --send-as <granter> --from <grantee>` sẽ gửi giao dịch như thế này:

```go
MsgExec {
  Grantee: mykey,
  Msgs: []sdk.Msg{
    MsgVote {
      ProposalID: 1,
      Voter: cosmos3thsdgh983egh823
      Option: Yes
    }
  }
}
```

#### `tx grant <grantee> <authorization> --from <granter>`

Lệnh CLI này sẽ gửi giao dịch `MsgGrant`. `authorization` nên được mã hóa dưới dạng JSON trên CLI.

#### `tx revoke <grantee> <method-name> --from <granter>`

Lệnh CLI này sẽ gửi giao dịch `MsgRevoke`.

### Các Ủy Quyền Được Tích Hợp Sẵn

#### `SendAuthorization`

```protobuf
// SendAuthorization cho phép grantee chi tiêu tối đa spend_limit coin từ
// tài khoản của granter.
message SendAuthorization {
  repeated cosmos.base.v1beta1.Coin spend_limit = 1;
}
```

#### `GenericAuthorization`

```protobuf
// GenericAuthorization cấp cho grantee quyền không hạn chế để thực thi
// phương thức được cung cấp thay mặt cho tài khoản của granter.
message GenericAuthorization {
  option (cosmos_proto.implements_interface) = "Authorization";

  // Msg, được xác định bởi type URL của nó, để cấp quyền không hạn chế để thực thi
  string msg = 1;
}
```

## Hậu Quả

### Tích Cực

* Người dùng có thể ủy quyền các hành động tùy ý thay mặt cho tài khoản của họ cho người dùng khác, cải thiện quản lý khóa cho nhiều trường hợp sử dụng
* Giải pháp tổng quát hơn các cách tiếp cận được xem xét trước đây và cách tiếp cận interface `Authorization` có thể được mở rộng để bao gồm các trường hợp sử dụng khác bởi người dùng SDK

### Tiêu Cực

### Trung Lập

## Tham Khảo

* Triển khai Hackatom ban đầu: https://github.com/cosmos-gaians/cosmos-sdk/tree/hackatom/x/delegation
* Đặc tả sau Hackatom: https://gist.github.com/aaronc/b60628017352df5983791cad30babe56#delegation-module
* Đặc tả subkeys của B-Harvest: https://github.com/cosmos/cosmos-sdk/issues/4480
