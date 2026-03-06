---
sidebar_position: 1
---

# Encoding (Mã hóa)

:::note Tóm tắt
Trong khi việc mã hóa trong Cosmos SDK trước đây chủ yếu được xử lý bởi codec `go-amino`, Cosmos SDK đang chuyển sang dùng `gogoprotobuf` cho cả mã hóa state lẫn phía client.
:::

:::note Tài liệu cần đọc trước

* [Cấu trúc của một ứng dụng Cosmos SDK](../beginner/00-app-anatomy.md)

:::

## Mã hóa

Cosmos SDK sử dụng hai giao thức mã hóa wire nhị phân, [Amino](https://github.com/tendermint/go-amino/) — là một đặc tả mã hóa đối tượng — và [Protocol Buffers](https://developers.google.com/protocol-buffers), một tập hợp con của Proto3 với phần mở rộng hỗ trợ interface. Xem [đặc tả Proto3](https://developers.google.com/protocol-buffers/docs/proto3) để biết thêm thông tin về Proto3, mà Amino phần lớn tương thích (nhưng không tương thích với Proto2).

Do Amino có những hạn chế đáng kể về hiệu suất, dựa trên reflection, và không có hỗ trợ đa ngôn ngữ/client có ý nghĩa, Protocol Buffers — cụ thể là [gogoprotobuf](https://github.com/cosmos/gogoproto/) — đang được dùng thay thế Amino. Lưu ý, quá trình chuyển sang dùng Protocol Buffers thay vì Amino vẫn đang diễn ra.

Mã hóa wire nhị phân của các kiểu trong Cosmos SDK có thể được chia thành hai danh mục chính: mã hóa phía client và mã hóa store. Mã hóa phía client chủ yếu liên quan đến xử lý và ký giao dịch, trong khi mã hóa store liên quan đến các kiểu được sử dụng trong chuyển đổi state-machine và những gì cuối cùng được lưu trong Merkle tree.

Đối với mã hóa store, định nghĩa protobuf có thể tồn tại cho bất kỳ kiểu nào và thường có kiểu "trung gian" dựa trên Amino. Cụ thể, định nghĩa kiểu dựa trên protobuf được dùng để tuần tự hóa và lưu trữ lâu dài, trong khi kiểu dựa trên Amino được dùng cho business logic trong state-machine nơi chúng có thể chuyển đổi qua lại. Lưu ý rằng các kiểu dựa trên Amino có thể dần bị loại bỏ trong tương lai, vì vậy các nhà phát triển nên chú ý dùng định nghĩa Protobuf message khi có thể.

Trong package `codec`, có hai interface cốt lõi: `BinaryCodec` và `JSONCodec`, trong đó cái trước bao gồm interface Amino hiện tại ngoại trừ nó hoạt động trên các kiểu triển khai cái sau thay vì các kiểu `interface{}` chung.

`ProtoCodec` — nơi cả tuần tự hóa nhị phân lẫn JSON đều được xử lý qua Protobuf. Điều này có nghĩa là các module có thể dùng mã hóa Protobuf, nhưng các kiểu phải triển khai `ProtoMarshaler`. Nếu các module muốn tránh triển khai interface này cho kiểu của họ, interface này được tự động tạo qua [buf](https://buf.build/).

Nếu các module dùng [Collections](../../build/packages/02-collections.md), việc mã hóa và giải mã được xử lý tự động — marshal và unmarshal không nên thực hiện thủ công trừ các trường hợp cụ thể do nhà phát triển xác định.

### Gogoproto

Các module được khuyến khích sử dụng mã hóa Protobuf cho các kiểu tương ứng của họ. Trong Cosmos SDK, chúng tôi dùng triển khai [Gogoproto](https://github.com/cosmos/gogoproto) cụ thể của đặc tả Protobuf, vì nó cung cấp cải tiến về tốc độ và trải nghiệm nhà phát triển (DX) so với [triển khai protobuf chính thức của Google](https://github.com/protocolbuffers/protobuf).

### Hướng dẫn định nghĩa Protobuf message

Ngoài việc [tuân theo hướng dẫn chính thức của Protocol Buffer](https://developers.google.com/protocol-buffers/docs/proto3#simple), chúng tôi khuyến nghị dùng các annotation sau trong file `.proto` khi làm việc với interface:

* dùng `cosmos_proto.accepts_interface` để chú thích các trường `Any` chấp nhận interface
    * truyền tên đầy đủ (fully qualified name) giống với `protoName` vào `InterfaceRegistry.RegisterInterface`
    * ví dụ: `(cosmos_proto.accepts_interface) = "cosmos.gov.v1beta1.Content"` (chứ không phải chỉ `Content`)
* chú thích các triển khai interface bằng `cosmos_proto.implements_interface`
    * truyền tên đầy đủ giống với `protoName` vào `InterfaceRegistry.RegisterInterface`
    * ví dụ: `(cosmos_proto.implements_interface) = "cosmos.authz.v1beta1.Authorization"` (chứ không phải chỉ `Authorization`)

Các code generator sau đó có thể khớp các annotation `accepts_interface` và `implements_interface` để biết liệu một số Protobuf message có được phép đóng gói trong một trường `Any` nhất định hay không.

### Mã hóa giao dịch

Một cách dùng quan trọng khác của Protobuf là mã hóa và giải mã [giao dịch](./01-transactions.md). Giao dịch được định nghĩa bởi ứng dụng hoặc Cosmos SDK nhưng sau đó được chuyển đến consensus engine bên dưới để chuyển tiếp đến các peer khác. Vì consensus engine bên dưới không biết gì về ứng dụng, nó chỉ chấp nhận giao dịch dưới dạng byte thô.

* Đối tượng `TxEncoder` thực hiện mã hóa.
* Đối tượng `TxDecoder` thực hiện giải mã.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/types/tx_msg.go#L109-L113
```

Một triển khai tiêu chuẩn của cả hai đối tượng này có thể tìm thấy trong [`auth/tx` module](../../build/modules/auth/2-tx.md):

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/x/auth/tx/decoder.go
```

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/x/auth/tx/encoder.go
```

Xem [ADR-020](https://github.com/cosmos/cosmos-sdk/blob/release/v0.53.x/docs/architecture/adr-020-protobuf-transaction-encoding.md) để biết chi tiết về cách một giao dịch được mã hóa.

### Mã hóa Interface và cách dùng `Any`

Protobuf DSL có kiểu mạnh (strongly typed), điều này có thể làm cho việc chèn các trường có kiểu biến động (variable-typed) trở nên khó khăn. Hãy tưởng tượng chúng ta muốn tạo một Protobuf message `Profile` đóng vai trò là wrapper cho [một tài khoản](../beginner/03-accounts.md):

```protobuf
message Profile {
  // account là tài khoản liên kết với profile.
  cosmos.auth.v1beta1.BaseAccount account = 1;
  // bio là mô tả ngắn về tài khoản.
  string bio = 4;
}
```

Trong ví dụ `Profile` này, chúng ta hardcode `account` là `BaseAccount`. Tuy nhiên, có nhiều kiểu [tài khoản người dùng liên quan đến vesting](../../build/modules/auth/1-vesting.md) khác, chẳng hạn như `BaseVestingAccount` hoặc `ContinuousVestingAccount`. Tất cả các tài khoản này đều khác nhau, nhưng tất cả đều triển khai interface `AccountI`. Làm thế nào để tạo `Profile` cho phép tất cả các kiểu tài khoản này với trường `account` chấp nhận interface `AccountI`?

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/types/account.go#L15-L32
```

Trong [ADR-019](https://github.com/cosmos/cosmos-sdk/blob/release/v0.53.x/docs/architecture/adr-019-protobuf-state-encoding.md), người ta quyết định dùng [`Any`](https://github.com/protocolbuffers/protobuf/blob/master/src/google/protobuf/any.proto) để mã hóa interface trong protobuf. `Any` chứa một message tuần tự hóa tùy ý dưới dạng bytes, cùng với một URL hoạt động như định danh duy nhất toàn cầu và giải quyết đến kiểu của message đó. Chiến lược này cho phép chúng ta đóng gói các kiểu Go tùy ý bên trong các Protobuf message. `Profile` mới của chúng ta trông như sau:

```protobuf
message Profile {
  // account là tài khoản liên kết với profile.
  google.protobuf.Any account = 1 [
    (cosmos_proto.accepts_interface) = "cosmos.auth.v1beta1.AccountI"; // Khẳng định trường này chỉ chấp nhận các kiểu Go triển khai `AccountI`. Hiện tại chỉ mang tính thông tin.
  ];
  // bio là mô tả ngắn về tài khoản.
  string bio = 4;
}
```

Để thêm tài khoản vào profile, chúng ta cần "đóng gói" nó vào một `Any` trước, bằng `codectypes.NewAnyWithValue`:

```go
var myAccount AccountI
myAccount = ... // Có thể là BaseAccount, ContinuousVestingAccount hoặc bất kỳ struct nào triển khai `AccountI`

// Đóng gói account vào Any
accAny, err := codectypes.NewAnyWithValue(myAccount)
if err != nil {
  return nil, err
}

// Tạo Profile mới với any.
profile := Profile {
  Account: accAny,
  Bio: "some bio",
}

// Sau đó ta có thể marshal profile như thường.
bz, err := cdc.Marshal(profile)
jsonBz, err := cdc.MarshalJSON(profile)
```

Tóm lại, để mã hóa một interface, bạn phải: 1/ đóng gói interface vào `Any` và 2/ marshal `Any`. Để tiện lợi, Cosmos SDK cung cấp phương thức `MarshalInterface` để gộp hai bước này. Xem [ví dụ thực tế trong module x/auth](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/x/auth/keeper/keeper.go#L239-L242).

Thao tác ngược lại — lấy kiểu Go cụ thể từ bên trong `Any` — được gọi là "unpacking" và thực hiện bằng `GetCachedValue()` trên `Any`.

```go
profileBz := ... // Các byte proto-encoded của Profile, ví dụ lấy qua gRPC.
var myProfile Profile
// Unmarshal các byte vào struct myProfile.
err := cdc.Unmarshal(profilebz, &myProfile)

// Hãy xem các kiểu của trường Account.
fmt.Printf("%T\n", myProfile.Account)                  // In ra "Any"
fmt.Printf("%T\n", myProfile.Account.GetCachedValue()) // In ra "BaseAccount", "ContinuousVestingAccount" hoặc bất cứ thứ gì ban đầu được đóng gói trong Any.

// Lấy địa chỉ của tài khoản.
accAddr := myProfile.Account.GetCachedValue().(AccountI).GetAddress()
```

Điều quan trọng cần lưu ý là để `GetCachedValue()` hoạt động, `Profile` (và bất kỳ struct nào nhúng `Profile`) phải triển khai phương thức `UnpackInterfaces`:

```go
func (p *Profile) UnpackInterfaces(unpacker codectypes.AnyUnpacker) error {
  if p.Account != nil {
    var account AccountI
    return unpacker.UnpackAny(p.Account, &account)
  }

  return nil
}
```

`UnpackInterfaces` được gọi đệ quy trên tất cả các struct triển khai phương thức này, để cho phép tất cả `Any` có `GetCachedValue()` được điền đúng.

Để biết thêm thông tin về mã hóa interface, và đặc biệt về `UnpackInterfaces` và cách `type_url` của `Any` được giải quyết bằng `InterfaceRegistry`, hãy tham khảo [ADR-019](https://github.com/cosmos/cosmos-sdk/blob/release/v0.53.x/docs/architecture/adr-019-protobuf-state-encoding.md).

#### Mã hóa `Any` trong Cosmos SDK

Ví dụ `Profile` ở trên là ví dụ hư cấu dùng cho mục đích giáo dục. Trong Cosmos SDK, chúng ta dùng mã hóa `Any` ở nhiều nơi (danh sách không đầy đủ):

* interface `cryptotypes.PubKey` để mã hóa các kiểu khóa công khai khác nhau,
* interface `sdk.Msg` để mã hóa các `Msg` khác nhau trong một giao dịch,
* interface `AccountI` để mã hóa các kiểu tài khoản khác nhau (tương tự ví dụ trên) trong phản hồi query của x/auth,
* interface `EvidenceI` để mã hóa các kiểu bằng chứng khác nhau trong module x/evidence,
* interface `AuthorizationI` để mã hóa các kiểu ủy quyền x/authz khác nhau,
* struct [`Validator`](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/x/staking/types/staking.pb.go#L340-L375) chứa thông tin về validator.

Ví dụ thực tế về mã hóa pubkey dưới dạng `Any` bên trong struct Validator của x/staking được hiển thị trong ví dụ sau:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/x/staking/types/validator.go#L43-L66
```

#### TypeURL của `Any`

Khi đóng gói một protobuf message vào `Any`, kiểu của message được định danh duy nhất bởi type URL của nó, là tên đầy đủ của message có tiền tố là ký tự `/` (slash). Trong một số triển khai của `Any`, như triển khai gogoproto, thường có [một tiền tố có thể giải quyết được, ví dụ `type.googleapis.com`](https://github.com/gogo/protobuf/blob/b03c65ea87cdc3521ede29f62fe3ce239267c1bc/protobuf/google/protobuf/any.proto#L87-L91). Tuy nhiên, trong Cosmos SDK, chúng tôi quyết định không bao gồm tiền tố như vậy, để có type URL ngắn hơn. Triển khai `Any` của Cosmos SDK có thể tìm thấy trong `github.com/cosmos/cosmos-sdk/codec/types`.

Cosmos SDK cũng đang chuyển từ gogoproto sang `google.golang.org/protobuf` chính thức (còn gọi là Protobuf API v2). Triển khai `Any` mặc định của nó cũng chứa tiền tố [`type.googleapis.com`](https://github.com/protocolbuffers/protobuf-go/blob/v1.28.1/types/known/anypb/any.pb.go#L266). Để duy trì khả năng tương thích với SDK, các phương thức sau từ `"google.golang.org/protobuf/types/known/anypb"` không nên dùng:

* `anypb.New`
* `anypb.MarshalFrom`
* `anypb.Any#MarshalFrom`

Thay vào đó, Cosmos SDK cung cấp các hàm helper trong `"github.com/cosmos/cosmos-proto/anyutil"`, tạo một `anypb.Any` chính thức mà không chèn tiền tố:

* `anyutil.New`
* `anyutil.MarshalFrom`

Ví dụ, để đóng gói một `sdk.Msg` tên là `internalMsg`, hãy dùng:

```diff
import (
- 	"google.golang.org/protobuf/types/known/anypb"
+	"github.com/cosmos/cosmos-proto/anyutil"
)

- anyMsg, err := anypb.New(internalMsg.Message().Interface())
+ anyMsg, err := anyutil.New(internalMsg.Message().Interface())

- fmt.Println(anyMsg.TypeURL) // type.googleapis.com/cosmos.bank.v1beta1.MsgSend
+ fmt.Println(anyMsg.TypeURL) // /cosmos.bank.v1beta1.MsgSend
```

## FAQ

### Cách tạo module dùng mã hóa protobuf

#### Định nghĩa kiểu module

Các kiểu Protobuf có thể được định nghĩa để mã hóa:

* trạng thái (state)
* [`Msg`](../../build/building-modules/02-messages-and-queries.md#messages)
* [Query services](../../build/building-modules/04-query-services.md)
* [genesis](../../build/building-modules/08-genesis.md)

#### Đặt tên và quy ước

Chúng tôi khuyến khích các nhà phát triển tuân theo hướng dẫn của ngành: [Protocol Buffers style guide](https://developers.google.com/protocol-buffers/docs/style) và [Buf](https://buf.build/docs/style-guide), xem thêm chi tiết trong [ADR 023](https://github.com/cosmos/cosmos-sdk/blob/release/v0.53.x/docs/architecture/adr-023-protobuf-naming.md).

### Cách cập nhật module sang mã hóa protobuf

Nếu các module không chứa bất kỳ interface nào (ví dụ: `Account` hoặc `Content`), thì chúng có thể đơn giản chuyển đổi các kiểu hiện có được mã hóa và lưu trữ qua Amino codec cụ thể của chúng sang Protobuf (xem mục 1 để biết thêm hướng dẫn) và chấp nhận `Marshaler` làm codec được triển khai thông qua `ProtoCodec` mà không cần tùy chỉnh thêm.

Tuy nhiên, nếu kiểu của module chứa một interface, nó phải bọc nó trong kiểu `sdk.Any` (từ package `/types`). Để làm điều đó, file `.proto` cấp module phải dùng [`google.protobuf.Any`](https://github.com/protocolbuffers/protobuf/blob/master/src/google/protobuf/any.proto) cho các kiểu interface message tương ứng.

Ví dụ, trong module `x/evidence` định nghĩa một interface `Evidence`, được dùng bởi `MsgSubmitEvidence`. Định nghĩa cấu trúc phải dùng `sdk.Any` để bọc evidence (đối tượng bằng chứng). Trong file proto, chúng tôi định nghĩa như sau:

```protobuf
// proto/cosmos/evidence/v1beta1/tx.proto

message MsgSubmitEvidence {
  string              submitter = 1;
  google.protobuf.Any evidence  = 2 [(cosmos_proto.accepts_interface) = "cosmos.evidence.v1beta1.Evidence"];
}
```

Interface `codec.Codec` của Cosmos SDK cung cấp các phương thức hỗ trợ `MarshalInterface` và `UnmarshalInterface` để mã hóa state thành `Any` dễ dàng.

Module nên đăng ký interface bằng `InterfaceRegistry` — cung cấp cơ chế đăng ký interface: `RegisterInterface(protoName string, iface interface{}, impls ...proto.Message)` và các triển khai: `RegisterImplementations(iface interface{}, impls ...proto.Message)` có thể được unpack an toàn từ `Any`, tương tự như đăng ký kiểu với Amino:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/codec/types/interface_registry.go#L40-L87
```

Ngoài ra, một giai đoạn `UnpackInterfaces` nên được đưa vào quá trình deserialization để unpack interface trước khi chúng cần. Các kiểu Protobuf chứa protobuf `Any` trực tiếp hoặc thông qua một trong các thành viên của chúng nên triển khai interface `UnpackInterfacesMessage`:

```go
type UnpackInterfacesMessage interface {
  UnpackInterfaces(InterfaceUnpacker) error
}
```
