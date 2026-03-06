# Đánh giá rủi ro Transaction Malleability của Cosmos SDK và khuyến nghị

## Changelog

* 2025-03-10: Bản nháp ban đầu (@aaronc)

## Trạng thái

ĐỀ XUẤT (Chưa triển khai)

## Tóm tắt

Trong lịch sử, một số vấn đề liên quan tới encoding và sign mode đã dẫn tới khả
năng giao dịch Cosmos SDK có thể được mã hoá lại (re-encoded) theo cách làm thay
đổi hash của giao dịch (và trong một số trường hợp hiếm, cả ý nghĩa), mà không
làm chữ ký trở nên không hợp lệ.

Tài liệu này mô tả chi tiết các trường hợp đó, các rủi ro tiềm ẩn, mức độ mà chúng
đã hoặc chưa được giải quyết, và đưa ra khuyến nghị cho các cải tiến trong tương lai.

## Rà soát

Một giả định ngây thơ về giao dịch Cosmos SDK là: hash các bytes thô của giao dịch
đã được gửi lên mạng tạo ra một định danh duy nhất an toàn cho giao dịch đó. Trong
thực tế, có nhiều cách mà giao dịch có thể bị thao túng để tạo ra bytes giao dịch
khác (và do đó hash khác) nhưng vẫn vượt qua được việc xác minh chữ ký.

Tài liệu này cố gắng liệt kê (enumerate) các rủi ro tiềm ẩn về “malleability” của
giao dịch mà chúng tôi đã xác định, và mức độ mà chúng đã hoặc chưa được giải quyết
trong các sign mode khác nhau. Chúng tôi cũng xác định các lỗ hổng có thể bị đưa
vào nếu developer thay đổi trong tương lai mà không cân nhắc kỹ các độ phức tạp
liên quan tới transaction encoding, sign modes và chữ ký.

### Rủi ro gắn với malleability

Malleability của giao dịch gây ra các rủi ro tiềm năng sau cho người dùng cuối:

* dữ liệu không được ký có thể được thêm vào giao dịch và được state machine xử lý
* client thường dựa vào hash giao dịch để kiểm tra trạng thái giao dịch, nhưng việc
  hash giao dịch đã submit có khớp với hash giao dịch đã được xử lý hay không chủ yếu
  phụ thuộc vào các tác nhân tốt trong mạng hơn là đảm bảo nền tảng ở mức giao thức
* giao dịch có thể bị thực thi nhiều hơn một lần (cơ chế replay protection lỗi)

Nếu một client tạo một giao dịch, lưu lại hash của nó rồi cố gắng truy vấn các node
để kiểm tra trạng thái giao dịch, quá trình này có thể kết luận sai rằng giao dịch
chưa được xử lý nếu một bộ xử lý trung gian decode và re-encode giao dịch với các
quy tắc encoding khác (dù ác ý hay vô tình).
Miễn là không có malleability trong chính bytes chữ ký, client _nên_ truy vấn giao
dịch theo chữ ký thay vì theo hash.

Không nhận thức được rủi ro này có thể khiến client gửi lại cùng một giao dịch
nhiều lần nếu họ tin rằng các giao dịch trước đó thất bại hoặc bị “lạc” trong xử lý.
Điều này có thể là một vector tấn công chống lại người dùng nếu ví (wallet) chủ yếu
truy vấn giao dịch theo hash.

Nếu state machine tự dựa vào hash giao dịch làm cơ chế replay, điều đó sẽ lỗi và
không cung cấp replay protection như kỳ vọng. Thay vào đó, state machine nên dựa
vào các biểu diễn (representation) mang tính quyết định của giao dịch, thay vì
encoding thô, hoặc các nonce khác, nếu muốn cung cấp replay protection mà không
dựa vào số sequence tài khoản tăng đơn điệu.

### Nguồn gốc của malleability

#### Protobuf encoding không quyết định (non-deterministic)

Giao dịch Cosmos SDK được mã hoá bằng protobuf binary encoding khi chúng được gửi
lên mạng. Protobuf binary vốn không phải là một encoding quyết định, nghĩa là cùng
một payload logic có thể có nhiều biểu diễn bytes hợp lệ. Ở mức cơ bản, điều này
nghĩa là protobuf nói chung có thể được decode và re-encode để tạo ra một luồng
byte khác (và do đó hash khác) mà không thay đổi ý nghĩa logic của bytes.
[ADR 027: Deterministic Protobuf Serialization](https://github.com/cosmos/cosmos-sdk/blob/main/docs/architecture/adr-027-deterministic-protobuf-serialization.md)
mô tả chi tiết những gì cần làm để tạo ra cái mà chúng ta coi là một protobuf
serialization “chuẩn” (canonical), mang tính quyết định. Tóm lại, các nguồn gốc
malleability ở mức encoding sau đã được xác định và được giải quyết bởi đặc tả này:

* các field có thể được phát ra theo bất kỳ thứ tự nào
* giá trị mặc định của field có thể được include hoặc omit, và điều này không đổi nghĩa trừ khi dùng `optional`
* field `repeated` dạng scalar có thể dùng packed hoặc “regular” encoding
* `varint` có thể chứa thêm các bit bị bỏ qua
* có thể thêm các field thừa và thường bị decoder bỏ qua. [ADR 020](https://github.com/cosmos/cosmos-sdk/blob/main/docs/architecture/adr-020-protobuf-transaction-encoding.md#unknown-field-filtering)
  quy định rằng nhìn chung các field thừa như vậy nên khiến message và giao dịch bị từ chối

Khi dùng `SIGN_MODE_DIRECT`, không nguồn malleability nào ở trên được chấp nhận vì:

* chữ ký của message và extension phải được thực hiện trên bytes đã mã hoá thô của các field đó
* lớp vỏ tx ngoài cùng (`TxRaw`) phải tuân theo quy tắc ADR 027 hoặc bị từ chối

Tuy nhiên, các giao dịch được ký với `SIGN_MODE_LEGACY_AMINO_JSON` không có cách
nào để bảo vệ khỏi các malleability ở trên vì thứ được ký là biểu diễn JSON của
nội dung logic của giao dịch. Nội dung logic này có thể có rất nhiều protobuf
binary encoding hợp lệ, nên nói chung không có đảm bảo nào về hash giao dịch khi
ký kiểu Amino JSON.

Ngoài việc nhận thức tính không quyết định chung của protobuf binary, developer
cần đặc biệt chú ý để đảm bảo các unknown protobuf field bị từ chối khi phát triển
các khả năng mới liên quan tới giao dịch protobuf. Định dạng protobuf được thiết
kế với giả định rằng dữ liệu unknown (đối với decoder) mà encoder biết có thể được
decoder bỏ qua một cách an toàn. Giả định này có thể tương đối an toàn trong môi
trường tập trung nội bộ của Google. Nhưng trong hệ thống blockchain phân tán, giả
định này nhìn chung là không an toàn. Nếu một client mới encode một message protobuf
với dữ liệu dành cho server mới, thì không an toàn để một server cũ chỉ đơn giản
bỏ qua và loại bỏ các chỉ dẫn mà nó không hiểu. Những chỉ dẫn đó có thể bao gồm
thông tin quan trọng mà người ký giao dịch đang dựa vào, và việc coi chúng là không
quan trọng là không an toàn.

[ADR 020](https://github.com/cosmos/cosmos-sdk/blob/main/docs/architecture/adr-020-protobuf-transaction-encoding.md#unknown-field-filtering)
quy định một số điều khoản cho các field “không quan trọng” (non-critical) có thể
được bỏ qua an toàn bởi các server cũ. Trên thực tế, tôi chưa thấy trường hợp sử
dụng hợp lệ nào cho điều này. Đây là điều mà maintainer nên biết khi thiết kế, nhưng
có thể không cần thiết hoặc thậm chí không an toàn 100%.

#### Value encoding không quyết định

Ngoài tính không quyết định của protobuf binary, một số dữ liệu trong field protobuf
được encode bằng một micro-format mà bản thân nó có thể không quyết định. Ví dụ: encode
số nguyên hoặc số thập phân. Một số decoder có thể cho phép số 0 ở đầu hoặc cuối mà
không đổi ý nghĩa logic, ví dụ `00100` vs `100` hoặc `100.00` vs `100`. Vì vậy nếu một
sign mode encode số theo cách quyết định, nhưng decoder chấp nhận nhiều biểu diễn,
người dùng có thể ký trên giá trị `100` trong khi `0100` lại được encode. Điều này có
thể xảy ra với Amino JSON nếu integer decoder chấp nhận số 0 ở đầu. Tôi tin rằng triển
khai `Int` hiện tại sẽ từ chối điều này; tuy nhiên, có lẽ vẫn có thể encode biểu diễn
bát phân hoặc thập lục phân trong giao dịch trong khi người dùng ký trên số nguyên
thập phân.

#### Signature encoding

Chữ ký bản thân chúng cũng được encode bằng một micro-format đặc thù cho thuật toán chữ
ký đang dùng; đôi khi các micro-format này có thể cho phép không quyết định (nhiều
bytes hợp lệ cho cùng một chữ ký).
Phần lớn thuật toán chữ ký được SDK hỗ trợ nên từ chối bytes không canonical trong triển
khai hiện tại. Tuy nhiên, kiểu protobuf `Multisignature` dùng encoding protobuf thông
thường và không có kiểm tra xem bytes đã decode có tuân theo quy tắc canonical ADR 027
hay không. Vì vậy, giao dịch multisig có thể có malleability trong chữ ký.
Bất kỳ thuật toán chữ ký mới hay tuỳ biến nào cũng phải đảm bảo từ chối bytes chữ ký
không canonical, nếu không ngay cả với `SIGN_MODE_DIRECT` vẫn có thể xảy ra malleability
hash giao dịch bằng cách re-encode chữ ký theo một biểu diễn không canonical.

#### Các field không được Amino JSON bao phủ

Một khu vực khác cần được xử lý cẩn thận là sự khác biệt giữa `AminoSignDoc`
(xem [`aminojson.proto`](../../x/tx/signing/aminojson/internal/aminojsonpb/aminojson.proto))
dùng cho `SIGN_MODE_LEGACY_AMINO_JSON` và nội dung thực tế của `TxBody` và `AuthInfo`
(xem [`tx.proto`](../../proto/cosmos/tx/v1beta1/tx.proto)).
Nếu có field được thêm vào `TxBody` hoặc `AuthInfo`, chúng phải hoặc có biểu diễn tương
ứng trong `AminoSignDoc`, hoặc chữ ký Amino JSON phải bị từ chối khi các field mới đó
được thiết lập. Việc đảm bảo điều này là một quy trình rất thủ công, và developer có
thể dễ dàng mắc lỗi cập nhật `TxBody` hoặc `AuthInfo` mà không để ý tới triển khai
`GetSignBytes` cho Amino JSON. Đây là một lỗ hổng nghiêm trọng: nội dung không được ký
có thể đi vào giao dịch và xác minh chữ ký vẫn pass.

## Tóm tắt sign mode và khuyến nghị

Các sign mode được SDK chính thức hỗ trợ là `SIGN_MODE_DIRECT`, `SIGN_MODE_TEXTUAL`,
`SIGN_MODE_DIRECT_AUX`, và `SIGN_MODE_LEGACY_AMINO_JSON`.
`SIGN_MODE_LEGACY_AMINO_JSON` được ví sử dụng phổ biến và hiện là sign mode duy nhất
được hỗ trợ trên thiết bị phần cứng Nano Ledger (dù `SIGN_MODE_TEXTUAL` được thiết kế
để cũng hỗ trợ thiết bị phần cứng).
`SIGN_MODE_DIRECT` là sign mode đơn giản nhất và cũng khá phổ biến.
`SIGN_MODE_DIRECT_AUX` là một biến thể của `SIGN_MODE_DIRECT` có thể được dùng bởi các
auxiliary signer trong giao dịch multi-signer cho các signer không trả gas.
`SIGN_MODE_TEXTUAL` được dự định là sự thay thế cho `SIGN_MODE_LEGACY_AMINO_JSON`,
nhưng theo những gì chúng ta biết nó chưa được client nào áp dụng và do đó chưa dùng
trong thực tế.

Tất cả các mối quan ngại malleability đã biết đều đã được giải quyết trong triển khai
hiện tại của `SIGN_MODE_DIRECT`.
Malleability duy nhất được biết có thể xảy ra với giao dịch ký bằng `SIGN_MODE_DIRECT`
sẽ phải nằm trong chính bytes chữ ký.
Vì chữ ký không được ký “đè lên” (signatures are not signed over), không sign mode nào
có thể giải quyết điều này trực tiếp; thay vào đó, các thuật toán chữ ký cần cẩn thận
từ chối bytes chữ ký không canonical để ngăn malleability.
Đối với malleability đã biết của kiểu `Multisignature`, chúng ta nên đảm bảo mọi chữ ký
hợp lệ đã được encode theo quy tắc canonical ADR 027 khi thực hiện xác minh chữ ký.

`SIGN_MODE_DIRECT_AUX` cung cấp mức độ an toàn tương tự `SIGN_MODE_DIRECT` vì:

* bytes `TxBody` đã mã hoá thô được ký trong `SignDocDirectAux`, và
* một giao dịch dùng `SIGN_MODE_DIRECT_AUX` vẫn yêu cầu primary signer ký giao dịch bằng `SIGN_MODE_DIRECT`

`SIGN_MODE_TEXTUAL` cũng cung cấp mức độ an toàn tương tự `SIGN_MODE_DIRECT` vì hash của
bytes `TxBody` và `AuthInfo` đã mã hoá thô được ký.

Thật không may, phần lớn rủi ro malleability chưa được giải quyết ảnh hưởng đến
`SIGN_MODE_LEGACY_AMINO_JSON`, và sign mode này vẫn được dùng phổ biến.
Khuyến nghị thực hiện các cải tiến sau cho việc ký Amino JSON:

* thêm hash của `TxBody` và `AuthInfo` vào `AminoSignDoc` để giải quyết malleability ở mức encoding
* khi xây dựng `AminoSignDoc`, nên dùng API [protoreflect](https://pkg.go.dev/google.golang.org/protobuf/reflect/protoreflect)
  để đảm bảo không có field nào trong `TxBody` hoặc `AuthInfo` đã được thiết lập mà không có mapping trong `AminoSignDoc`
* các field có trong `TxBody` hoặc `AuthInfo` nhưng không có trong `AminoSignDoc` (như extension options) nên được thêm
  vào `AminoSignDoc` nếu có thể

## Kiểm thử

Để kiểm thử rằng giao dịch có khả năng chống malleability,
chúng ta có thể phát triển một test suite chạy trên mọi sign mode, cố gắng thao túng
bytes giao dịch theo các cách sau:

* thay đổi protobuf encoding bằng cách
  * đổi thứ tự field
  * thiết lập giá trị mặc định
  * thêm bit thừa vào varint, hoặc
  * thiết lập unknown field mới
* sửa đổi giá trị số nguyên và số thập phân được encode dạng chuỗi bằng số 0 ở đầu hoặc cuối

Bất cứ khi nào thực hiện một trong các thao túng này, chúng ta nên quan sát rằng bytes sign doc
cho sign mode đang test cũng thay đổi, nghĩa là chữ ký tương ứng cũng phải thay đổi.

Trong trường hợp Amino JSON, chúng ta cũng nên phát triển các bài test đảm bảo rằng nếu bất kỳ field
nào của `TxBody` hoặc `AuthInfo` không được `AminoSignDoc` hỗ trợ được thiết lập, thì việc ký sẽ thất bại.

Trong trường hợp chung của việc decode giao dịch, chúng ta nên có unit test để đảm bảo rằng:

* bất kỳ bytes `TxRaw` nào không tuân theo encoding canonical ADR 027 đều khiến việc decode thất bại, và
* bất kỳ phần tử giao dịch top-level nào bao gồm `TxBody`, `AuthInfo`, public key, và message có unknown field
  được thiết lập đều khiến giao dịch bị từ chối
(điều này đảm bảo ADR 020 unknown field filtering được áp dụng đúng)

Với mỗi thuật toán chữ ký được hỗ trợ, cũng nên có unit test để đảm bảo chữ ký phải được encode canonical
hoặc sẽ bị từ chối.

## Tham khảo

* [ADR 027: Deterministic Protobuf Serialization](https://github.com/cosmos/cosmos-sdk/blob/main/docs/architecture/adr-027-deterministic-protobuf-serialization.md)
* [ADR 020](https://github.com/cosmos/cosmos-sdk/blob/main/docs/architecture/adr-020-protobuf-transaction-encoding.md#unknown-field-filtering)
* [`aminojson.proto`](../../x/tx/signing/aminojson/internal/aminojsonpb/aminojson.proto)
* [`tx.proto`](../../proto/cosmos/tx/v1beta1/tx.proto)

