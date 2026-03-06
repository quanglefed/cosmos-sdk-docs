# ADR 019: Mã Hóa Trạng Thái Bằng Protocol Buffer

## Changelog

* 2020 15 tháng 2: Bản nháp đầu tiên
* 2020 24 tháng 2: Cập nhật để xử lý các message có trường interface
* 2020 27 tháng 4: Chuyển đổi cách dùng `oneof` cho interface sang `Any`
* 2020 15 tháng 5: Mô tả các extension `cosmos_proto` và tương thích amino
* 2020 4 tháng 12: Di chuyển và đổi tên `MarshalAny` và `UnmarshalAny` vào interface `codec.Codec`.
* 2021 24 tháng 2: Xóa đề cập về `HybridCodec`, đã bị loại bỏ trong [#6843](https://github.com/cosmos/cosmos-sdk/pull/6843).

## Trạng Thái

Đã Chấp Nhận

## Bối Cảnh

Hiện tại, Cosmos SDK sử dụng [go-amino](https://github.com/tendermint/go-amino/) để mã hóa đối tượng nhị phân và JSON qua mạng, mang lại sự đồng nhất giữa các đối tượng logic và đối tượng lưu trữ.

Từ tài liệu Amino:

> Amino là một đặc tả mã hóa đối tượng. Đây là một tập con của Proto3 với phần mở rộng để hỗ trợ interface. Xem [đặc tả Proto3](https://developers.google.com/protocol-buffers/docs/proto3) để biết thêm thông tin về Proto3, mà Amino phần lớn tương thích (nhưng không tương thích với Proto2).
>
> Mục tiêu của giao thức mã hóa Amino là mang lại sự đồng nhất giữa các đối tượng logic và đối tượng lưu trữ.

Amino cũng hướng đến các mục tiêu sau (không phải danh sách đầy đủ):

* Các byte nhị phân phải có thể giải mã được với một schema.
* Schema phải có thể nâng cấp được.
* Logic của encoder và decoder phải tương đối đơn giản.

Tuy nhiên, chúng tôi tin rằng Amino không đáp ứng đầy đủ các mục tiêu này và không thỏa mãn hoàn toàn nhu cầu của một giao thức mã hóa thực sự linh hoạt, đa ngôn ngữ và tương thích với nhiều client trong Cosmos SDK. Cụ thể, Amino đã cho thấy là một điểm gây khó khăn lớn trong việc hỗ trợ tuần tự hóa đối tượng đa ngôn ngữ trong khi gần như không cung cấp khả năng tương thích ngược thực sự và khả năng nâng cấp. Hơn nữa, thông qua việc profiling và các benchmark, Amino đã được chứng minh là một điểm thắt cổ chai hiệu suất cực kỳ lớn trong Cosmos SDK <sup>1</sup>. Điều này phần lớn được phản ánh trong hiệu suất của các mô phỏng và thông lượng giao dịch của ứng dụng.

Do đó, chúng ta cần áp dụng một giao thức mã hóa đáp ứng các tiêu chí sau cho tuần tự hóa trạng thái:

* Độc lập ngôn ngữ
* Độc lập nền tảng
* Hỗ trợ client phong phú và hệ sinh thái sầm uất
* Hiệu suất cao
* Kích thước message mã hóa tối thiểu
* Dựa trên codegen thay vì reflection
* Hỗ trợ tương thích ngược và tương thích xuôi

Lưu ý, việc di chuyển khỏi Amino nên được xem như một cách tiếp cận hai hướng: mã hóa trạng thái và mã hóa client. ADR này tập trung vào tuần tự hóa trạng thái trong state machine của Cosmos SDK. Một ADR tương ứng sẽ được tạo ra để giải quyết mã hóa phía client.

## Quyết Định

Chúng ta sẽ áp dụng [Protocol Buffers](https://developers.google.com/protocol-buffers) để tuần tự hóa dữ liệu có cấu trúc được lưu trữ trong Cosmos SDK đồng thời cung cấp cơ chế rõ ràng và trải nghiệm nhà phát triển tốt cho các ứng dụng muốn tiếp tục sử dụng Amino. Chúng ta sẽ cung cấp cơ chế này bằng cách cập nhật các module để chấp nhận một codec interface, `Marshaler`, thay vì một Amino codec cụ thể. Hơn nữa, Cosmos SDK sẽ cung cấp hai triển khai cụ thể của interface `Marshaler`: `AminoCodec` và `ProtoCodec`.

* `AminoCodec`: Sử dụng Amino cho cả mã hóa nhị phân và JSON.
* `ProtoCodec`: Sử dụng Protobuf cho cả mã hóa nhị phân và JSON.

Các module sẽ sử dụng codec nào được khởi tạo trong app. Theo mặc định, `simapp` của Cosmos SDK khởi tạo một `ProtoCodec` là triển khai cụ thể của `Marshaler`, bên trong hàm `MakeTestEncodingConfig`. Điều này có thể được ghi đè dễ dàng bởi nhà phát triển app nếu họ muốn.

Mục tiêu cuối cùng sẽ là thay thế mã hóa Amino JSON bằng mã hóa Protobuf và do đó có các module chấp nhận và/hoặc mở rộng `ProtoCodec`. Cho đến khi đó, Amino JSON vẫn được cung cấp cho các trường hợp sử dụng cũ. Một vài chỗ trong Cosmos SDK vẫn có Amino JSON được hardcode, chẳng hạn như các Legacy API REST endpoint và store `x/params`. Chúng được lên kế hoạch chuyển đổi sang Protobuf theo cách tuần tự.

### Codec Của Module

Đối với các module không yêu cầu khả năng làm việc và tuần tự hóa interface, con đường di chuyển sang Protobuf khá đơn giản. Các module này chỉ cần di chuyển bất kỳ kiểu hiện có nào được mã hóa và lưu trữ qua Amino codec cụ thể sang Protobuf và để keeper của chúng chấp nhận một `Marshaler` sẽ là `ProtoCodec`. Việc di chuyển này đơn giản vì mọi thứ sẽ hoạt động ngay như cũ.

Lưu ý, bất kỳ logic nghiệp vụ nào cần mã hóa các kiểu nguyên thủy như `bool` hoặc `int64` nên sử dụng các kiểu Value của [gogoprotobuf](https://github.com/cosmos/gogoproto).

Ví dụ:

```go
  ts, err := gogotypes.TimestampProto(completionTime)
  if err != nil {
    // ...
  }

  bz := cdc.MustMarshal(ts)
```

Tuy nhiên, các module có thể khác nhau rất nhiều về mục đích và thiết kế, vì vậy chúng ta phải hỗ trợ khả năng để các module có thể mã hóa và làm việc với interface (ví dụ: `Account` hoặc `Content`). Đối với những module này, chúng phải định nghĩa codec interface riêng mở rộng `Marshaler`. Các interface cụ thể này là duy nhất cho module và sẽ chứa các hợp đồng phương thức biết cách tuần tự hóa các interface cần thiết.

Ví dụ:

```go
// x/auth/types/codec.go

type Codec interface {
  codec.Codec

  MarshalAccount(acc exported.Account) ([]byte, error)
  UnmarshalAccount(bz []byte) (exported.Account, error)

  MarshalAccountJSON(acc exported.Account) ([]byte, error)
  UnmarshalAccountJSON(bz []byte) (exported.Account, error)
}
```

### Sử Dụng `Any` Để Mã Hóa Interface

Nói chung, các file .proto ở cấp module nên định nghĩa các message mã hóa interface bằng [`google.protobuf.Any`](https://github.com/protocolbuffers/protobuf/blob/master/src/google/protobuf/any.proto). Sau [thảo luận mở rộng](https://github.com/cosmos/cosmos-sdk/issues/6030), đây được chọn là lựa chọn ưu tiên thay thế cho `oneof` ở cấp ứng dụng như trong thiết kế protobuf ban đầu của chúng ta. Các lập luận ủng hộ `Any` có thể được tóm tắt như sau:

* `Any` cung cấp trải nghiệm client đơn giản hơn, nhất quán hơn khi xử lý interface so với `oneof` ở cấp app cần được phối hợp cẩn thận hơn giữa các ứng dụng. Việc tạo một thư viện ký giao dịch chung sử dụng `oneof` có thể cồng kềnh và logic quan trọng có thể cần được triển khai lại cho mỗi chain.
* `Any` cung cấp khả năng chống lỗi của con người tốt hơn so với `oneof`.
* `Any` nhìn chung đơn giản hơn để triển khai cho cả module lẫn app.

Lập luận phản bác chính khi sử dụng `Any` xoay quanh không gian bổ sung của nó và có thể ảnh hưởng đến hiệu suất. Overhead về không gian có thể được giải quyết bằng cách nén ở lớp lưu trữ trong tương lai và tác động đến hiệu suất có thể là nhỏ. Do đó, không sử dụng `Any` được coi là tối ưu hóa non-mature, với trải nghiệm người dùng là mối quan tâm bậc cao hơn.

Lưu ý rằng, theo quyết định của Cosmos SDK về việc áp dụng các interface `Codec` được mô tả ở trên, các app vẫn có thể chọn sử dụng `oneof` để mã hóa trạng thái và giao dịch nhưng đây không phải là cách tiếp cận được khuyến nghị. Nếu các app chọn sử dụng `oneof` thay vì `Any`, họ có thể mất khả năng tương thích với các app client hỗ trợ nhiều chain. Do đó, các nhà phát triển nên suy nghĩ cẩn thận về việc họ quan tâm nhiều hơn đến tối ưu hóa có thể là non-mature hay trải nghiệm người dùng cuối và nhà phát triển client.

### Sử Dụng An Toàn `Any`

Theo mặc định, [triển khai gogo protobuf của `Any`](https://pkg.go.dev/github.com/cosmos/gogoproto/types) sử dụng [đăng ký kiểu toàn cục](https://github.com/cosmos/gogoproto/blob/master/proto/properties.go#L540) để giải mã các giá trị được đóng gói trong `Any` thành các kiểu go cụ thể. Điều này tạo ra một lỗ hổng trong đó bất kỳ module độc hại nào trong dependency tree có thể đăng ký một kiểu với registry protobuf toàn cục và khiến nó được tải và unmarshaled bởi một giao dịch tham chiếu đến nó trong trường `type_url`.

Để ngăn chặn điều này, chúng ta giới thiệu một cơ chế đăng ký kiểu để giải mã các giá trị `Any` thành các kiểu cụ thể thông qua interface `InterfaceRegistry`, có một số điểm tương đồng với đăng ký kiểu trong Amino:

```go
type InterfaceRegistry interface {
    // RegisterInterface liên kết protoName là tên công khai cho
    // interface được truyền vào là iface
    // Ví dụ:
    //   registry.RegisterInterface("cosmos_sdk.Msg", (*sdk.Msg)(nil))
    RegisterInterface(protoName string, iface interface{})

    // RegisterImplementations đăng ký impls là các triển khai cụ thể của
    // interface iface
    // Ví dụ:
    //  registry.RegisterImplementations((*sdk.Msg)(nil), &MsgSend{}, &MsgMultiSend{})
    RegisterImplementations(iface interface{}, impls ...proto.Message)

}
```

Ngoài việc đóng vai trò như một whitelist, `InterfaceRegistry` cũng có thể đóng vai trò truyền đạt danh sách các kiểu cụ thể thỏa mãn một interface đến các client.

Trong các file .proto:

* các trường chấp nhận interface nên được chú thích với `cosmos_proto.accepts_interface` sử dụng cùng tên đầy đủ được truyền là `protoName` cho `InterfaceRegistry.RegisterInterface`
* các triển khai interface nên được chú thích với `cosmos_proto.implements_interface` sử dụng cùng tên đầy đủ được truyền là `protoName` cho `InterfaceRegistry.RegisterInterface`

Trong tương lai, `protoName`, `cosmos_proto.accepts_interface`, `cosmos_proto.implements_interface` có thể được sử dụng qua code generation, reflection và/hoặc static linting.

Cùng struct triển khai `InterfaceRegistry` cũng sẽ triển khai một interface `InterfaceUnpacker` được dùng để unpack các `Any`:

```go
type InterfaceUnpacker interface {
    // UnpackAny unpack giá trị trong any vào con trỏ interface được truyền vào là
    // iface. Lưu ý rằng kiểu trong any phải đã được đăng ký với
    // RegisterImplementations là kiểu cụ thể cho interface đó
    // Ví dụ:
    //    var msg sdk.Msg
    //    err := ctx.UnpackAny(any, &msg)
    //    ...
    UnpackAny(any *Any, iface interface{}) error
}
```

Lưu ý rằng việc sử dụng `InterfaceRegistry` không khác với việc sử dụng protobuf tiêu chuẩn của `Any`, nó chỉ giới thiệu một lớp bảo mật và introspection cho việc sử dụng golang.

`InterfaceRegistry` sẽ là thành viên của `ProtoCodec` được mô tả ở trên. Để các module đăng ký các kiểu interface, app module có thể tùy chọn triển khai interface sau:

```go
type InterfaceModule interface {
    RegisterInterfaceTypes(InterfaceRegistry)
}
```

Module manager sẽ bao gồm một phương thức để gọi `RegisterInterfaceTypes` trên mỗi module triển khai nó để điền vào `InterfaceRegistry`.

### Sử Dụng `Any` Để Mã Hóa Trạng Thái

Cosmos SDK sẽ cung cấp các phương thức hỗ trợ `MarshalInterface` và `UnmarshalInterface` để ẩn độ phức tạp của việc bao các kiểu interface vào `Any` và cho phép tuần tự hóa dễ dàng.

```go
import "github.com/cosmos/cosmos-sdk/codec"

// lưu ý: eviexported.Evidence là kiểu interface
func MarshalEvidence(cdc codec.BinaryCodec, e eviexported.Evidence) ([]byte, error) {
	return cdc.MarshalInterface(e)
}

func UnmarshalEvidence(cdc codec.BinaryCodec, bz []byte) (eviexported.Evidence, error) {
	var evi eviexported.Evidence
	err := cdc.UnmarshalInterface(&evi, bz)
    return err, nil
}
```

### Sử Dụng `Any` Trong `sdk.Msg`

Một khái niệm tương tự được áp dụng cho các message chứa các trường interface. Ví dụ, chúng ta có thể định nghĩa `MsgSubmitEvidence` như sau, trong đó `Evidence` là một interface:

```protobuf
// x/evidence/types/types.proto

message MsgSubmitEvidence {
  bytes submitter = 1
    [
      (gogoproto.casttype) = "github.com/cosmos/cosmos-sdk/types.AccAddress"
    ];
  google.protobuf.Any evidence = 2;
}
```

Lưu ý rằng để unpack evidence từ `Any`, chúng ta cần một tham chiếu đến `InterfaceRegistry`. Để tham chiếu evidence trong các phương thức như `ValidateBasic` không cần biết về `InterfaceRegistry`, chúng ta giới thiệu một giai đoạn `UnpackInterfaces` để deserialization, unpack các interface trước khi chúng cần được sử dụng.

### Giải Nén Interface

Để triển khai giai đoạn `UnpackInterfaces` của deserialization, giai đoạn unpack các interface được bao trong `Any` trước khi chúng cần thiết, chúng ta tạo một interface mà `sdk.Msg` và các kiểu khác có thể triển khai:

```go
type UnpackInterfacesMessage interface {
  UnpackInterfaces(InterfaceUnpacker) error
}
```

Chúng ta cũng giới thiệu một trường `cachedValue interface{}` private trên chính struct `Any` với một getter công khai `GetCachedValue() interface{}`.

Phương thức `UnpackInterfaces` được gọi trong quá trình deserialization message ngay sau `Unmarshal` và bất kỳ giá trị interface nào được đóng gói trong `Any` sẽ được giải mã và lưu trữ trong `cachedValue` để tham chiếu sau này.

Sau đó, các giá trị interface đã được unpack có thể được sử dụng an toàn trong bất kỳ code nào sau đó mà không cần biết về `InterfaceRegistry`, và các message có thể giới thiệu một getter đơn giản để cast giá trị được cache thành kiểu interface đúng.

Điều này có lợi ích bổ sung là việc unmarshaling các giá trị `Any` chỉ xảy ra một lần trong quá trình deserialization ban đầu thay vì mỗi lần giá trị được đọc. Ngoài ra, khi các giá trị `Any` lần đầu tiên được đóng gói (ví dụ: trong một lần gọi `NewMsgSubmitEvidence`), giá trị interface ban đầu được cache để không cần unmarshaling để đọc lại.

`MsgSubmitEvidence` có thể triển khai `UnpackInterfaces`, cùng với một getter tiện lợi `GetEvidence` như sau:

```go
func (msg MsgSubmitEvidence) UnpackInterfaces(ctx sdk.InterfaceRegistry) error {
  var evi eviexported.Evidence
  return ctx.UnpackAny(msg.Evidence, *evi)
}

func (msg MsgSubmitEvidence) GetEvidence() eviexported.Evidence {
  return msg.Evidence.GetCachedValue().(eviexported.Evidence)
}
```

### Tương Thích Với Amino

Triển khai tùy chỉnh của chúng ta về `Any` có thể được sử dụng minh bạch với Amino nếu được sử dụng với instance codec phù hợp. Điều này có nghĩa là các interface được đóng gói trong `Any` sẽ được amino marshal như các interface Amino thông thường (giả sử chúng đã được đăng ký đúng cách với Amino).

Để chức năng này hoạt động:

* **tất cả code cũ phải sử dụng `*codec.LegacyAmino` thay vì `*amino.Codec` hiện là một wrapper xử lý đúng `Any`**
* **tất cả code mới nên sử dụng `Marshaler` tương thích với cả amino và protobuf**
* Ngoài ra, trước v0.39, `codec.LegacyAmino` sẽ được đổi tên thành `codec.LegacyAmino`.

### Tại Sao X Không Được Chọn Thay Thế

Để so sánh đầy đủ hơn với các giao thức thay thế, xem [tại đây](https://codeburst.io/json-vs-protocol-buffers-vs-flatbuffers-a4247f8bda6f).

### Cap'n Proto

Mặc dù [Cap'n Proto](https://capnproto.org/) có vẻ là một lựa chọn thay thế có lợi thế hơn Protobuf do hỗ trợ gốc cho interface/generics và tính năng canonical hóa tích hợp sẵn, nó thiếu hệ sinh thái client phong phú so với Protobuf và kém trưởng thành hơn một chút.

### FlatBuffers

[FlatBuffers](https://google.github.io/flatbuffers/) cũng là một lựa chọn thay thế khả thi, với sự khác biệt chính là FlatBuffers không cần bước phân tích/giải nén sang biểu diễn thứ cấp trước khi bạn có thể truy cập dữ liệu, thường kết hợp với cấp phát bộ nhớ per-object.

Tuy nhiên, nó sẽ đòi hỏi nỗ lực lớn vào nghiên cứu và hiểu đầy đủ phạm vi di chuyển và con đường tiến lên — điều không rõ ràng ngay lập tức. Ngoài ra, FlatBuffers không được thiết kế cho các đầu vào không đáng tin cậy.

## Cải Tiến Tương Lai & Lộ Trình

Trong tương lai, chúng ta có thể xem xét một lớp nén ngay trên lớp lưu trữ mà không thay đổi hash tx hoặc merkle tree, nhưng giảm overhead lưu trữ của `Any`. Ngoài ra, chúng ta có thể áp dụng các quy ước đặt tên protobuf làm cho các type URL ngắn gọn hơn trong khi vẫn mô tả.

Hỗ trợ tạo code bổ sung xung quanh việc sử dụng `Any` là điều cũng có thể được khám phá trong tương lai để làm cho UX cho các nhà phát triển Go liền mạch hơn.

## Hậu Quả

### Tích Cực

* Cải thiện hiệu suất đáng kể.
* Hỗ trợ tương thích kiểu ngược và xuôi.
* Hỗ trợ tốt hơn cho các client đa ngôn ngữ.

### Tiêu Cực

* Cần thời gian học để hiểu và triển khai các message Protobuf.
* Kích thước message lớn hơn đôi chút do sử dụng `Any`, mặc dù điều này có thể được bù đắp bởi một lớp nén trong tương lai.

### Trung Lập

## Tài Liệu Tham Khảo

1. https://github.com/cosmos/cosmos-sdk/issues/4977
2. https://github.com/cosmos/cosmos-sdk/issues/5444
