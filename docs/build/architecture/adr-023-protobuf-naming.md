# ADR 023: Quy Ước Đặt Tên và Phiên Bản Protocol Buffer

## Nhật Ký Thay Đổi

* 27 tháng 4 năm 2020: Bản nháp đầu tiên
* 5 tháng 8 năm 2020: Cập nhật hướng dẫn

## Trạng Thái

Đã Chấp Nhận

## Bối Cảnh

Protocol Buffers cung cấp một [hướng dẫn phong cách](https://developers.google.com/protocol-buffers/docs/style) cơ bản và [Buf](https://buf.build/docs/style-guide) xây dựng dựa trên đó. Trong phạm vi có thể, chúng ta muốn tuân theo các hướng dẫn và kinh nghiệm được chấp nhận trong ngành để sử dụng protobuf hiệu quả, chỉ lệch ra ngoài những điều đó khi có lý do rõ ràng cho trường hợp sử dụng của chúng ta.

### Áp Dụng `Any`

Việc áp dụng `google.protobuf.Any` như cách tiếp cận được khuyến nghị để mã hóa các kiểu interface (thay vì `oneof`) làm cho việc đặt tên gói trở thành phần trung tâm của mã hóa vì các tên message đầy đủ bây giờ xuất hiện trong các message được mã hóa.

### Tổ Chức Thư Mục Hiện Tại

Cho đến nay chúng ta chủ yếu tuân theo các khuyến nghị [DEFAULT](https://buf.build/docs/lint-checkers#default) của [Buf](https://buf.build), với sự lệch nhỏ là tắt [`PACKAGE_DIRECTORY_MATCH`](https://buf.build/docs/lint-checkers#file_layout) mặc dù thuận tiện cho việc phát triển code nhưng đi kèm với cảnh báo từ Buf rằng:

> bạn sẽ gặp rất nhiều khó khăn với nhiều plugin Protobuf trên nhiều ngôn ngữ khác nhau nếu bạn không làm điều này

### Áp Dụng Truy Vấn gRPC

Trong [ADR 021](adr-021-protobuf-query-encoding.md), gRPC được áp dụng cho các truy vấn gốc Protobuf. Đường dẫn service gRPC đầy đủ do đó trở thành phần quan trọng của đường dẫn truy vấn ABCI. Trong tương lai, các truy vấn gRPC có thể được cho phép từ bên trong các script liên tục bởi các công nghệ như CosmWasm và các route truy vấn này sẽ được lưu trữ trong các tệp nhị phân script.

## Quyết Định

Mục tiêu của ADR này là cung cấp các quy ước đặt tên có suy nghĩ kỹ lưỡng để:

* Khuyến khích trải nghiệm người dùng tốt khi người dùng tương tác trực tiếp với các file .proto và các tên protobuf đầy đủ
* Cân bằng sự ngắn gọn so với khả năng tối ưu hóa quá mức (làm cho tên quá ngắn và khó hiểu) hoặc tối ưu hóa quá ít (chỉ chấp nhận các tên cồng kềnh với nhiều thông tin dư thừa)

Các hướng dẫn này nhằm mục đích hoạt động như một hướng dẫn phong cách cho cả Cosmos SDK và các module của bên thứ ba.

Là điểm khởi đầu, chúng ta nên áp dụng tất cả các trình kiểm tra [DEFAULT](https://buf.build/docs/lint-checkers#default) trong [Buf](https://buf.build) bao gồm [`PACKAGE_DIRECTORY_MATCH`](https://buf.build/docs/lint-checkers#file_layout), ngoại trừ:

* [PACKAGE_VERSION_SUFFIX](https://buf.build/docs/lint-checkers#package_version_suffix)
* [SERVICE_SUFFIX](https://buf.build/docs/lint-checkers#service_suffix)

Các hướng dẫn bổ sung sẽ được mô tả bên dưới.

### Nguyên Tắc

#### Tên Ngắn Gọn và Mô Tả

Tên nên đủ mô tả để truyền đạt ý nghĩa của chúng và phân biệt chúng với các tên khác.

Vì chúng ta đang sử dụng tên đầy đủ bên trong `google.protobuf.Any` cũng như trong các route truy vấn gRPC, chúng ta nên hướng tới việc giữ tên ngắn gọn mà không quá đà. Quy tắc chung nên là nếu một tên ngắn hơn truyền đạt nhiều hoặc ít hơn cùng một thứ, hãy chọn tên ngắn hơn.

Ví dụ, `cosmos.bank.MsgSend` (19 byte) truyền đạt thông tin tương tự như `cosmos_sdk.x.bank.v1.MsgSend` (28 byte) nhưng ngắn gọn hơn.

Sự ngắn gọn như vậy làm cho tên dễ làm việc hơn và chiếm ít không gian hơn trong các giao dịch và trên đường truyền.

Chúng ta cũng nên tránh cám dỗ tối ưu hóa quá mức, bằng cách tạo ra các tên bí ẩn ngắn với các chữ viết tắt. Ví dụ, chúng ta không nên cố gắng rút gọn `cosmos.bank.MsgSend` thành `csm.bk.MSnd` chỉ để tiết kiệm một vài byte.

Mục tiêu là làm cho tên **_ngắn gọn nhưng không khó hiểu_**.

#### Tên Phục Vụ Client Trước Tiên

Tên gói và kiểu nên được chọn vì lợi ích của người dùng, không nhất thiết vì các lo ngại về di sản liên quan đến cơ sở code go.

#### Lên Kế Hoạch Cho Sự Bền Lâu

Vì lợi ích của hỗ trợ dài hạn, chúng ta nên lên kế hoạch cho các tên mà chúng ta chọn được sử dụng trong một thời gian dài, vì vậy bây giờ là cơ hội để đưa ra các lựa chọn tốt nhất cho tương lai.

### Phiên Bản

#### Hướng Dẫn về Phiên Bản Gói Ổn Định

Nói chung, sự phát triển schema là cách cập nhật các schema protobuf. Điều đó có nghĩa là các trường, message và phương thức RPC mới được _thêm_ vào các schema hiện có và các trường, message và phương thức RPC cũ được duy trì càng lâu càng tốt.

Việc phá vỡ mọi thứ thường không thể chấp nhận trong một kịch bản blockchain. Ví dụ, các hợp đồng thông minh bất biến có thể phụ thuộc vào một số schema dữ liệu nhất định trên chain host. Nếu chain host phá vỡ các schema đó, hợp đồng thông minh có thể bị hỏng không thể khắc phục. Ngay cả khi mọi thứ có thể được sửa chữa (ví dụ trong phần mềm client), điều này thường đi kèm với chi phí cao.

Thay vì phá vỡ mọi thứ, chúng ta nên nỗ lực tối đa để phát triển các schema thay vì chỉ phá vỡ chúng. Phát hiện thay đổi phá vỡ của [Buf](https://buf.build) nên được sử dụng trên tất cả các gói ổn định (không phải alpha hoặc beta) để ngăn chặn sự phá vỡ như vậy.

Với điều đó, các phiên bản ổn định khác nhau (tức là `v1` hoặc `v2`) của một gói nên được coi là các gói khác nhau và đây nên là cách tiếp cận cuối cùng để nâng cấp schema protobuf. Các kịch bản mà việc tạo `v2` có thể có ý nghĩa là:

* Chúng ta muốn tạo một module mới với chức năng tương tự như module hiện có và việc thêm `v2` là cách tự nhiên nhất để làm điều này. Trong trường hợp đó, thực sự chỉ có hai module khác nhau nhưng tương tự với các API khác nhau.
* Chúng ta muốn thêm một API mới được cải tiến cho một module hiện có và việc thêm nó vào gói hiện có quá cồng kềnh, vì vậy việc đặt nó trong `v2` sẽ gọn gàng hơn cho người dùng. Trong trường hợp này, cần chú ý không xóa hỗ trợ cho `v1` nếu nó đang được sử dụng tích cực trong các hợp đồng thông minh bất biến.

#### Hướng Dẫn về Phiên Bản Gói Không Ổn Định (alpha và beta)

Các hướng dẫn sau được khuyến nghị để đánh dấu các gói là alpha hoặc beta:

* Đánh dấu thứ gì đó là `alpha` hoặc `beta` nên là biện pháp cuối cùng và chỉ đưa thứ gì đó vào gói ổn định (tức là `v1` hoặc `v2`) nên được ưu tiên hơn
* Một gói _nên_ được đánh dấu là `alpha` _chỉ khi_ có các cuộc thảo luận tích cực về việc xóa hoặc thay đổi đáng kể gói đó trong tương lai gần
* Một gói _nên_ được đánh dấu là `beta` _chỉ khi_ có cuộc thảo luận tích cực về việc tái cấu trúc/làm lại đáng kể chức năng trong tương lai gần nhưng không xóa nó
* Các module _có thể và nên_ có các kiểu trong cả gói ổn định (tức là `v1` hoặc `v2`) và không ổn định (`alpha` hoặc `beta`).

_`alpha` và `beta` không nên được sử dụng để tránh trách nhiệm duy trì khả năng tương thích._ Bất cứ khi nào code được phát hành ra thế giới, đặc biệt là trên blockchain, có chi phí cao khi thay đổi mọi thứ. Trong một số trường hợp, ví dụ với các hợp đồng thông minh bất biến, một thay đổi phá vỡ có thể không thể sửa chữa.

Khi đánh dấu thứ gì đó là `alpha` hoặc `beta`, những người bảo trì nên đặt câu hỏi sau:

* chi phí yêu cầu người khác thay đổi code của họ so với lợi ích của chúng ta trong việc duy trì khả năng thay đổi nó là gì?
* kế hoạch chuyển điều này sang `v1` là gì và điều đó sẽ ảnh hưởng đến người dùng như thế nào?

`alpha` hoặc `beta` thực sự nên được sử dụng để truyền đạt "các thay đổi đang được lên kế hoạch".

Như một nghiên cứu trường hợp, gRPC reflection nằm trong gói `grpc.reflection.v1alpha`. Nó chưa được thay đổi kể từ năm 2017 và hiện được sử dụng trong các phần mềm được sử dụng rộng rãi khác như gRPCurl. Một số người có lẽ sử dụng nó trong các dịch vụ sản xuất và vì vậy nếu họ thực sự đi và thay đổi gói thành `grpc.reflection.v1`, một số phần mềm sẽ bị hỏng và họ có lẽ không muốn làm điều đó... Vì vậy bây giờ gói `v1alpha` ít nhiều là `v1` thực tế. Chúng ta đừng làm vậy.

Sau đây là hướng dẫn để làm việc với các gói không ổn định:

* [Hậu tố phiên bản được khuyến nghị của Buf](https://buf.build/docs/lint-checkers#package_version_suffix) (vd: `v1alpha1`) _nên_ được sử dụng cho các gói không ổn định
* Các gói không ổn định thường nên được loại trừ khỏi phát hiện thay đổi phá vỡ
* Các module hợp đồng thông minh bất biến (tức là CosmWasm) _nên_ chặn các hợp đồng thông minh/script liên tục khỏi việc tương tác với các gói `alpha`/`beta`

#### Bỏ Hậu Tố v1

Thay vì sử dụng [hậu tố phiên bản được khuyến nghị của Buf](https://buf.build/docs/lint-checkers#package_version_suffix), chúng ta có thể bỏ `v1` cho các gói thực sự không có phiên bản thứ hai. Điều này cho phép các tên ngắn gọn hơn cho các trường hợp sử dụng phổ biến như `cosmos.bank.Send`. Các gói có phiên bản thứ hai hoặc thứ ba có thể chỉ ra điều đó bằng `.v2` hoặc `.v3`.

### Đặt Tên Gói

#### Áp Dụng Tên Gói Cấp Cao Ngắn và Duy Nhất

Các gói cấp cao nên áp dụng một tên ngắn được biết là không trùng với các tên khác trong việc sử dụng phổ biến trong hệ sinh thái Cosmos. Trong tương lai gần, một registry nên được tạo ra để đặt trước và lập chỉ mục các tên gói cấp cao được sử dụng trong hệ sinh thái Cosmos. Vì Cosmos SDK dự định cung cấp các kiểu cấp cao nhất cho dự án Cosmos, tên gói cấp cao `cosmos` được khuyến nghị sử dụng trong Cosmos SDK thay vì `cosmos_sdk` dài hơn. Các đặc tả [ICS](https://github.com/cosmos/ics) có thể xem xét một gói cấp cao ngắn như `ics23` dựa trên số tiêu chuẩn.

#### Giới Hạn Độ Sâu Sub-Package

Độ sâu sub-package nên được tăng lên một cách thận trọng. Nói chung, một sub-package đơn là đủ cho một module hoặc một thư viện. Mặc dù `x` hoặc `modules` được sử dụng trong code nguồn để biểu thị các module, điều này thường không cần thiết cho các file .proto vì các module là thứ chủ yếu mà sub-package được sử dụng cho. Chỉ những mục được biết là hiếm khi sử dụng mới nên có độ sâu sub-package sâu.

Với Cosmos SDK, khuyến nghị rằng chúng ta chỉ đơn giản viết `cosmos.bank`, `cosmos.gov`, v.v. thay vì `cosmos.x.bank`. Trong thực tế, hầu hết các kiểu không phải module có thể đi thẳng vào gói `cosmos` hoặc chúng ta có thể giới thiệu gói `cosmos.base` nếu cần. Lưu ý rằng việc đặt tên này _sẽ không_ thay đổi tên gói go, tức là gói protobuf `cosmos.bank` vẫn sẽ nằm trong `x/bank`.

### Đặt Tên Message

Tên kiểu message nên ngắn gọn nhất có thể mà không mất đi sự rõ ràng. Các kiểu `sdk.Msg` được sử dụng trong các giao dịch sẽ giữ nguyên tiền tố `Msg` vì nó cung cấp ngữ cảnh hữu ích.

### Đặt Tên Service và RPC

[ADR 021](adr-021-protobuf-query-encoding.md) chỉ định rằng các module nên triển khai một query service gRPC. Chúng ta nên xem xét nguyên tắc ngắn gọn cho tên query service và RPC vì chúng có thể được gọi từ các module script liên tục như CosmWasm. Ngoài ra, người dùng có thể sử dụng các đường dẫn truy vấn này từ các công cụ như [gRPCurl](https://github.com/fullstorydev/grpcurl). Ví dụ, chúng ta có thể rút ngắn `/cosmos_sdk.x.bank.v1.QueryService/QueryBalance` thành `/cosmos.bank.Query/Balance` mà không mất nhiều thông tin hữu ích.

Các kiểu yêu cầu và phản hồi RPC _nên_ tuân theo quy ước đặt tên `ServiceNameMethodNameRequest`/`ServiceNameMethodNameResponse`. Tức là với một phương thức RPC có tên `Balance` trên service `Query`, các kiểu yêu cầu và phản hồi sẽ là `QueryBalanceRequest` và `QueryBalanceResponse`. Điều này sẽ tự giải thích hơn là `BalanceRequest` và `BalanceResponse`.

#### Chỉ Dùng `Query` cho Query Service

Thay vì [khuyến nghị hậu tố service mặc định của Buf](https://github.com/cosmos/cosmos-sdk/pull/6033), chúng ta nên đơn giản sử dụng `Query` ngắn hơn cho các query service.

Với các kiểu service gRPC khác, chúng ta nên xem xét tuân theo khuyến nghị mặc định của Buf.

#### Bỏ `Get` và `Query` khỏi Tên RPC của Query Service

`Get` và `Query` nên được bỏ khỏi tên service `Query` vì chúng là dư thừa trong tên đầy đủ. Ví dụ, `/cosmos.bank.Query/QueryBalance` chỉ nói `Query` hai lần mà không có thông tin mới.

## Cải Tiến Tương Lai

Một registry tên gói cấp cao nên được tạo ra để phối hợp việc đặt tên trên toàn bộ hệ sinh thái, ngăn chặn xung đột, và cũng giúp các nhà phát triển khám phá các schema hữu ích. Một điểm khởi đầu đơn giản sẽ là một kho git với quản trị dựa trên cộng đồng.

## Hậu Quả

### Tích Cực

* Tên sẽ ngắn gọn hơn và dễ đọc và gõ hơn
* Tất cả các giao dịch sử dụng `Any` sẽ ngắn hơn (`_sdk.x` và `.v1` sẽ bị xóa)
* Các lần import file `.proto` sẽ chuẩn hơn (không có `"third_party/proto"` trong đường dẫn)
* Việc tạo code sẽ dễ dàng hơn cho các client vì các file .proto sẽ nằm trong một thư mục `proto/` duy nhất có thể được sao chép thay vì rải rác khắp Cosmos SDK

### Tiêu Cực

### Trung Lập

* Các file `.proto` sẽ cần được tổ chức lại và tái cấu trúc
* Một số module có thể cần được đánh dấu là alpha hoặc beta

## Tham Khảo
