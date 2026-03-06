# ADR 020: Mã Hóa Giao Dịch Bằng Protocol Buffer

## Changelog

* 2020 06 tháng 3: Bản nháp đầu tiên
* 2020 12 tháng 3: Cập nhật API
* 2020 13 tháng 4: Thêm chi tiết về xử lý interface `oneof`
* 2020 30 tháng 4: Chuyển sang `Any`
* 2020 14 tháng 5: Mô tả mã hóa khóa công khai
* 2020 08 tháng 6: Lưu trữ `TxBody` và `AuthInfo` dưới dạng byte trong `SignDoc`; Tài liệu hóa `TxRaw` là kiểu broadcast và lưu trữ.
* 2020 07 tháng 8: Sử dụng ADR 027 để tuần tự hóa `SignDoc`.
* 2020 19 tháng 8: Di chuyển trường sequence từ `SignDoc` sang `SignerInfo`, như đã thảo luận trong [#6966](https://github.com/cosmos/cosmos-sdk/issues/6966).
* 2020 25 tháng 9: Xóa kiểu `PublicKey` thay bằng `secp256k1.PubKey`, `ed25519.PubKey` và `multisig.LegacyAminoPubKey`.
* 2020 15 tháng 10: Thêm các phương thức `GetAccount` và `GetAccountWithHeight` vào interface `AccountRetriever`.
* 2021 24 tháng 2: Cosmos SDK không còn sử dụng interface `PubKey` của Tendermint nữa mà dùng `cryptotypes.PubKey` riêng của mình. Cập nhật để phản ánh điều này.
* 2021 03 tháng 5: Đổi tên `clientCtx.JSONMarshaler` thành `clientCtx.JSONCodec`.
* 2021 10 tháng 6: Thêm `clientCtx.Codec: codec.Codec`.

## Trạng Thái

Đã Chấp Nhận

## Bối Cảnh

ADR này là sự tiếp nối của động lực, thiết kế và bối cảnh được thiết lập trong [ADR 019](./adr-019-protobuf-state-encoding.md), cụ thể là chúng ta nhằm thiết kế con đường di chuyển Protocol Buffer cho phía client của Cosmos SDK.

Cụ thể, con đường di chuyển phía client chủ yếu bao gồm tạo và ký giao dịch, xây dựng và định tuyến message, ngoài ra còn có các handler CLI & REST và logic nghiệp vụ (tức là querier).

Với điều này trong tâm trí, chúng ta sẽ giải quyết con đường di chuyển qua hai lĩnh vực chính: tx và truy vấn. Tuy nhiên, ADR này chỉ tập trung vào giao dịch. Truy vấn nên được đề cập trong một ADR tương lai, nhưng nó nên được xây dựng dựa trên các đề xuất này.

Dựa trên các cuộc thảo luận chi tiết ([\\#6030](https://github.com/cosmos/cosmos-sdk/issues/6030) và [\\#6078](https://github.com/cosmos/cosmos-sdk/issues/6078)), thiết kế ban đầu cho giao dịch đã được thay đổi đáng kể từ cách tiếp cận `oneof`/JSON-signing sang cách tiếp cận được mô tả dưới đây.

## Quyết Định

### Giao Dịch

Vì các giá trị interface được mã hóa với `google.protobuf.Any` trong trạng thái (xem [ADR 019](adr-019-protobuf-state-encoding.md)), `sdk.Msg` được mã hóa với `Any` trong giao dịch.

Một trong những mục tiêu chính của việc sử dụng `Any` để mã hóa các giá trị interface là có một tập core các kiểu được tái sử dụng bởi các app để các client có thể tương thích an toàn với nhiều chain nhất có thể.

Đây là một trong những mục tiêu của đặc tả này để cung cấp định dạng giao dịch cross-chain linh hoạt phục vụ nhiều trường hợp sử dụng mà không phá vỡ tương thích client.

Để tạo điều kiện ký, giao dịch được tách thành `TxBody`, sẽ được tái sử dụng bởi `SignDoc` bên dưới, và `signatures`:

```protobuf
// types/types.proto
package cosmos_sdk.v1;

message Tx {
    TxBody body = 1;
    AuthInfo auth_info = 2;
    // Danh sách chữ ký khớp với độ dài và thứ tự của signer_infos trong AuthInfo để
    // cho phép kết nối thông tin meta chữ ký như khóa công khai và chế độ ký theo vị trí.
    repeated bytes signatures = 3;
}

// Biến thể của Tx ghim biểu diễn nhị phân chính xác của body và
// auth_info của người ký. Điều này được dùng để ký, broadcast và xác minh. Nhị phân
// `serialize(tx: TxRaw)` được lưu trữ trong Tendermint và hash `sha256(serialize(tx: TxRaw))`
// trở thành "txhash", thường được sử dụng làm ID giao dịch.
message TxRaw {
    // Tuần tự hóa protobuf của TxBody khớp với biểu diễn trong SignDoc.
    bytes body = 1;
    // Tuần tự hóa protobuf của AuthInfo khớp với biểu diễn trong SignDoc.
    bytes auth_info = 2;
    // Danh sách chữ ký khớp với độ dài và thứ tự của signer_infos trong AuthInfo để
    // cho phép kết nối thông tin meta chữ ký như khóa công khai và chế độ ký theo vị trí.
    repeated bytes signatures = 3;
}

message TxBody {
    // Danh sách các message cần thực thi. Những người ký bắt buộc của các message đó định nghĩa
    // số lượng và thứ tự của các phần tử trong signer_infos của AuthInfo và signatures của Tx.
    // Mỗi địa chỉ người ký bắt buộc được thêm vào danh sách chỉ lần đầu tiên nó xuất hiện.
    //
    // Theo quy ước, người ký đầu tiên bắt buộc (thường từ message đầu tiên) được gọi là
    // người ký chính và trả phí cho toàn bộ giao dịch.
    repeated google.protobuf.Any messages = 1;
    string memo = 2;
    int64 timeout_height = 3;
    repeated google.protobuf.Any extension_options = 1023;
}

message AuthInfo {
    // Danh sách này định nghĩa các chế độ ký cho những người ký bắt buộc. Số lượng
    // và thứ tự của các phần tử phải khớp với những người ký bắt buộc từ các message của TxBody.
    // Phần tử đầu tiên là người ký chính và là người trả phí.
    repeated SignerInfo signer_infos = 1;
    // Phí có thể được tính toán dựa trên chi phí đánh giá body và xác minh chữ ký của người ký.
    // Điều này có thể được ước tính qua mô phỏng.
    Fee fee = 2;
}

message SignerInfo {
    // Khóa công khai là tùy chọn cho các tài khoản đã tồn tại trong trạng thái. Nếu không được đặt,
    // người xác minh có thể sử dụng địa chỉ người ký bắt buộc cho vị trí này và tra cứu khóa công khai.
    google.protobuf.Any public_key = 1;
    // ModeInfo mô tả chế độ ký của người ký và là cấu trúc lồng nhau
    // để hỗ trợ khóa công khai multisig lồng nhau.
    ModeInfo mode_info = 2;
    // sequence là sequence của tài khoản, mô tả
    // số lượng giao dịch đã cam kết được ký bởi một địa chỉ nhất định. Nó được dùng để ngăn chặn
    // tấn công replay.
    uint64 sequence = 3;
}

message ModeInfo {
    oneof sum {
        Single single = 1;
        Multi multi = 2;
    }

    // Single là thông tin chế độ cho một người ký đơn. Nó được cấu trúc như một message
    // để cho phép các trường bổ sung như locale cho SIGN_MODE_TEXTUAL trong tương lai
    message Single {
        SignMode mode = 1;
    }

    // Multi là thông tin chế độ cho khóa công khai multisig
    message Multi {
        // bitarray chỉ định các khóa nào trong multisig đang ký
        CompactBitArray bitarray = 1;
        // mode_infos là các chế độ tương ứng của những người ký của multisig
        // có thể bao gồm các khóa công khai multisig lồng nhau
        repeated ModeInfo mode_infos = 2;
    }
}

enum SignMode {
    SIGN_MODE_UNSPECIFIED = 0;

    SIGN_MODE_DIRECT = 1;

    SIGN_MODE_TEXTUAL = 2;

    SIGN_MODE_LEGACY_AMINO_JSON = 127;
}
```

Như sẽ được thảo luận bên dưới, để bao gồm càng nhiều `Tx` càng tốt trong `SignDoc`, `SignerInfo` được tách khỏi chữ ký để chỉ các chữ ký thô sống ngoài những gì được ký.

Vì chúng ta hướng đến định dạng giao dịch cross-chain linh hoạt, có thể mở rộng, các tùy chọn xử lý giao dịch mới nên được thêm vào `TxBody` ngay khi các trường hợp sử dụng đó được phát hiện, ngay cả khi chúng chưa thể được triển khai.

Vì có overhead phối hợp trong điều này, `TxBody` bao gồm một trường `extension_options` có thể được sử dụng cho bất kỳ tùy chọn xử lý giao dịch nào chưa được bao gồm. Tuy nhiên, các nhà phát triển app nên cố gắng đưa các cải tiến quan trọng lên `Tx`.

### Ký

Tất cả các chế độ ký dưới đây nhằm cung cấp các đảm bảo sau:

* **Không Thể Biến Dạng (No Malleability)**: `TxBody` và `AuthInfo` không thể thay đổi khi giao dịch đã được ký.
* **Gas Có Thể Dự Đoán (Predictable Gas)**: nếu tôi đang ký một giao dịch trong đó tôi trả phí, gas cuối cùng phụ thuộc hoàn toàn vào những gì tôi đang ký.

Những đảm bảo này cung cấp mức độ tin tưởng tối đa cho những người ký message rằng việc thao túng `Tx` bởi các trung gian không thể dẫn đến bất kỳ thay đổi có ý nghĩa nào.

#### `SIGN_MODE_DIRECT`

Hành vi ký "direct" là ký các byte `TxBody` thô được broadcast qua mạng. Điều này có những ưu điểm:

* yêu cầu khả năng client bổ sung tối thiểu ngoài triển khai protocol buffer tiêu chuẩn
* để lại hầu như không có lỗ hổng nào cho transaction malleability (tức là không có sự khác biệt tinh tế nào giữa định dạng ký và mã hóa có thể bị khai thác bởi kẻ tấn công)

Chữ ký được cấu trúc bằng `SignDoc` bên dưới, tái sử dụng tuần tự hóa của `TxBody` và `AuthInfo` và chỉ thêm các trường cần thiết cho chữ ký:

```protobuf
// types/types.proto
message SignDoc {
    // Tuần tự hóa protobuf của TxBody khớp với biểu diễn trong TxRaw.
    bytes body = 1;
    // Tuần tự hóa protobuf của AuthInfo khớp với biểu diễn trong TxRaw.
    bytes auth_info = 2;
    string chain_id = 3;
    uint64 account_number = 4;
}
```

Để ký theo chế độ mặc định, client thực hiện các bước sau:

1. Tuần tự hóa `TxBody` và `AuthInfo` bằng bất kỳ triển khai protobuf hợp lệ nào.
2. Tạo `SignDoc` và tuần tự hóa nó bằng [ADR 027](./adr-027-deterministic-protobuf-serialization.md).
3. Ký các byte `SignDoc` đã mã hóa.
4. Xây dựng `TxRaw` và tuần tự hóa nó để broadcast.

Xác minh chữ ký dựa trên việc so sánh các byte `TxBody` và `AuthInfo` thô được mã hóa trong `TxRaw`, không dựa trên bất kỳ thuật toán ["canonical hóa"](https://github.com/regen-network/canonical-proto3) nào tạo thêm độ phức tạp cho client ngoài việc ngăn chặn một số hình thức khả năng nâng cấp (sẽ được giải quyết sau trong tài liệu này).

Những người xác minh chữ ký thực hiện:

1. Deserialize `TxRaw` và lấy ra `body` và `auth_info`.
2. Tạo danh sách địa chỉ người ký bắt buộc từ các message.
3. Đối với mỗi người ký bắt buộc:
   * Lấy account number và sequence từ trạng thái.
   * Lấy khóa công khai từ trạng thái hoặc `signer_infos` của `AuthInfo`.
   * Tạo `SignDoc` và tuần tự hóa nó bằng [ADR 027](./adr-027-deterministic-protobuf-serialization.md).
   * Xác minh chữ ký ở cùng vị trí danh sách so với `SignDoc` đã tuần tự hóa.

#### `SIGN_MODE_LEGACY_AMINO`

Để hỗ trợ các ví và sàn giao dịch cũ, Amino JSON sẽ tạm thời được hỗ trợ ký giao dịch. Khi các ví và sàn giao dịch đã có cơ hội nâng cấp sang ký dựa trên protobuf, tùy chọn này sẽ bị vô hiệu hóa. Trong thời gian đó, việc vô hiệu hóa ký Amino hiện tại được dự kiến sẽ gây ra quá nhiều hỏng hóc để thực hiện được. Lưu ý rằng đây chủ yếu là yêu cầu của Cosmos Hub và các chain khác có thể chọn vô hiệu hóa ký Amino ngay lập tức.

Các client cũ sẽ có thể ký một giao dịch bằng định dạng Amino JSON hiện tại và mã hóa nó thành protobuf bằng cách sử dụng REST endpoint `/tx/encode` trước khi broadcast.

#### `SIGN_MODE_TEXTUAL`

Như đã được thảo luận rộng rãi trong [\\#6078](https://github.com/cosmos/cosmos-sdk/issues/6078), có mong muốn về mã hóa ký có thể đọc được bởi người dùng, đặc biệt là cho các ví phần cứng như [Ledger](https://www.ledger.com) hiển thị nội dung giao dịch cho người dùng trước khi ký. JSON là một nỗ lực theo hướng này nhưng không đáp ứng đầy đủ.

`SIGN_MODE_TEXTUAL` được dự định là một placeholder cho mã hóa có thể đọc được bởi người dùng sẽ thay thế Amino JSON. Mã hóa mới này nên tập trung hơn nữa vào khả năng đọc so với JSON, có thể dựa trên các chuỗi định dạng như [MessageFormat](http://userguide.icu-project.org/formatparse/messages).

Để đảm bảo rằng định dạng có thể đọc mới không bị transaction malleability, `SIGN_MODE_TEXTUAL` yêu cầu _các byte có thể đọc được nối với `SignDoc` thô_ để tạo ra các byte ký.

Nhiều định dạng có thể đọc (thậm chí có thể là các message được bản địa hóa) có thể được hỗ trợ bởi `SIGN_MODE_TEXTUAL` khi nó được triển khai.

### Lọc Trường Không Xác Định

Các trường không xác định trong các message protobuf nói chung nên bị từ chối bởi các bộ xử lý giao dịch vì:

* dữ liệu quan trọng có thể có mặt trong các trường không xác định mà nếu bị bỏ qua, sẽ gây ra hành vi không mong muốn cho client
* chúng tạo ra lỗ hổng malleability nơi kẻ tấn công có thể làm phình kích thước tx bằng cách thêm dữ liệu ngẫu nhiên không được giải thích vào nội dung không được ký (tức là `Tx` master, không phải `TxBody`)

Cũng có các tình huống mà chúng ta có thể chọn bỏ qua an toàn các trường không xác định (https://github.com/cosmos/cosmos-sdk/issues/6078#issuecomment-624400188) để cung cấp tương thích xuôi graceful với các client mới hơn.

Chúng ta đề xuất rằng các số trường có bit 11 được đặt (đối với hầu hết các trường hợp sử dụng, đây là phạm vi 1024-2047) được coi là các trường không quan trọng có thể được bỏ qua an toàn nếu không xác định.

Để xử lý điều này, chúng ta sẽ cần một bộ lọc trường không xác định:

* luôn từ chối các trường không xác định trong nội dung không được ký (tức là `Tx` cấp cao nhất và các phần không được ký của `AuthInfo` nếu có dựa trên chế độ ký)
* từ chối các trường không xác định trong tất cả các message (bao gồm các `Any` lồng nhau) ngoại trừ các trường có bit 11 được đặt

Điều này có thể cần là một lần chạy phân tích protobuf tùy chỉnh nhận các byte message và `FileDescriptor` rồi trả về một kết quả boolean.

### Mã Hóa Khóa Công Khai

Các khóa công khai trong Cosmos SDK triển khai interface `cryptotypes.PubKey`. Chúng ta đề xuất sử dụng `Any` cho mã hóa protobuf như chúng ta đang làm với các interface khác (ví dụ: trong `BaseAccount.PubKey` và `SignerInfo.PublicKey`). Các khóa công khai sau được triển khai: secp256k1, secp256r1, ed25519 và legacy-multisignature.

Ví dụ:

```protobuf
message PubKey {
    bytes key = 1;
}
```

`multisig.LegacyAminoPubKey` có một mảng các thành viên `Any` để hỗ trợ bất kỳ kiểu khóa công khai protobuf nào.

Các app chỉ nên cố gắng xử lý một tập hợp khóa công khai đã đăng ký mà họ đã kiểm thử. Các decorator ante handler xác minh chữ ký được cung cấp sẽ thực thi điều này.

### CLI & REST

Hiện tại, các handler REST và CLI mã hóa và giải mã các kiểu và tx qua mã hóa Amino JSON bằng một Amino codec cụ thể. Vì một số kiểu được xử lý trong client có thể là interface, tương tự như cách chúng ta mô tả trong [ADR 019](./adr-019-protobuf-state-encoding.md), logic client bây giờ sẽ cần nhận một codec interface không chỉ biết cách xử lý tất cả các kiểu, mà còn biết cách tạo giao dịch, chữ ký và message.

```go
type AccountRetriever interface {
  GetAccount(clientCtx Context, addr sdk.AccAddress) (client.Account, error)
  GetAccountWithHeight(clientCtx Context, addr sdk.AccAddress) (client.Account, int64, error)
  EnsureExists(clientCtx client.Context, addr sdk.AccAddress) error
  GetAccountNumberSequence(clientCtx client.Context, addr sdk.AccAddress) (uint64, uint64, error)
}

type Generator interface {
  NewTx() TxBuilder
  NewFee() ClientFee
  NewSignature() ClientSignature
  MarshalTx(tx types.Tx) ([]byte, error)
}

type TxBuilder interface {
  GetTx() sdk.Tx

  SetMsgs(...sdk.Msg) error
  GetSignatures() []sdk.Signature
  SetSignatures(...sdk.Signature)
  GetFee() sdk.Fee
  SetFee(sdk.Fee)
  GetMemo() string
  SetMemo(string)
}
```

Sau đó chúng ta cập nhật `Context` để có các trường mới: `Codec`, `TxGenerator` và `AccountRetriever`, và chúng ta cập nhật `AppModuleBasic.GetTxCmd` để nhận một `Context` nên có tất cả các trường này được điền trước.

Mỗi phương thức client sau đó nên sử dụng một trong các phương thức `Init` để khởi tạo lại `Context` đã được điền trước. `tx.GenerateOrBroadcastTx` có thể được dùng để tạo hoặc broadcast một giao dịch. Ví dụ:

```go
import "github.com/spf13/cobra"
import "github.com/cosmos/cosmos-sdk/client"
import "github.com/cosmos/cosmos-sdk/client/tx"

func NewCmdDoSomething(clientCtx client.Context) *cobra.Command {
	return &cobra.Command{
		RunE: func(cmd *cobra.Command, args []string) error {
			clientCtx := ctx.InitWithInput(cmd.InOrStdin())
			msg := NewSomeMsg{...}
			tx.GenerateOrBroadcastTx(clientCtx, msg)
		},
	}
}
```

## Cải Tiến Tương Lai

### Đặc Tả `SIGN_MODE_TEXTUAL`

Đặc tả và triển khai cụ thể của `SIGN_MODE_TEXTUAL` được dự kiến là cải tiến tương lai gần để ứng dụng ledger và các ví khác có thể chuyển đổi dần dần khỏi Amino JSON.

### `SIGN_MODE_DIRECT_AUX`

(\\*Được tài liệu hóa là lựa chọn (3) trong https://github.com/cosmos/cosmos-sdk/issues/6078#issuecomment-628026933)

Chúng ta có thể thêm một chế độ `SIGN_MODE_DIRECT_AUX` để hỗ trợ các tình huống trong đó nhiều chữ ký đang được thu thập vào một giao dịch duy nhất nhưng người soạn message chưa biết chữ ký nào sẽ được bao gồm trong giao dịch cuối cùng. Ví dụ, tôi có thể có một ví multisig 3/5 và muốn gửi `TxBody` đến tất cả 5 người ký để xem ai ký trước. Ngay khi tôi có 3 chữ ký thì tôi sẽ tiến hành xây dựng giao dịch đầy đủ.

Với `SIGN_MODE_DIRECT`, mỗi người ký cần ký `AuthInfo` đầy đủ bao gồm danh sách đầy đủ của tất cả người ký và các chế độ ký của họ, khiến tình huống trên trở nên rất khó khăn.

`SIGN_MODE_DIRECT_AUX` sẽ cho phép các người ký "phụ" tạo chữ ký của họ chỉ sử dụng `TxBody` và `PublicKey` của chính họ. Điều này cho phép danh sách đầy đủ người ký trong `AuthInfo` được trì hoãn cho đến khi chữ ký đã được thu thập.

Một người ký "phụ" là bất kỳ người ký nào ngoài người ký chính đang trả phí. Đối với người ký chính, `AuthInfo` đầy đủ thực sự cần thiết để tính toán gas và phí vì điều đó phụ thuộc vào số lượng người ký và các kiểu khóa nào cùng chế độ ký họ đang sử dụng. Tuy nhiên, những người ký phụ không cần lo lắng về phí hoặc gas và do đó có thể chỉ ký `TxBody`.

Để tạo chữ ký trong `SIGN_MODE_DIRECT_AUX`, các bước sau sẽ được thực hiện:

1. Mã hóa `SignDocAux` (với yêu cầu tương tự rằng các trường phải được tuần tự hóa theo thứ tự):

    ```protobuf
    // types/types.proto
    message SignDocAux {
        bytes body_bytes = 1;
        // PublicKey được bao gồm trong SignDocAux:
        // 1. như một trường hợp đặc biệt cho khóa công khai multisig. Đối với khóa công khai multisig,
        // người ký nên sử dụng khóa công khai multisig cấp cao nhất họ đang ký,
        // không phải khóa công khai của riêng họ. Điều này để ngăn chặn một hình thức
        // malleability trong đó chữ ký có thể bị lấy ra ngoài bối cảnh của
        // khóa multisig được dự định ký cho
        // 2. để bảo vệ chống lại tình huống trong đó thông tin cấu hình được mã hóa
        // trong khóa công khai (đã được đề xuất) sao cho hai khóa có thể tạo ra
        // cùng một chữ ký nhưng có các thuộc tính bảo mật khác nhau
        //
        // Bằng cách bao gồm nó ở đây, người soạn AuthInfo không thể tham chiếu đến
        // biến thể khóa công khai mà người ký không có ý định sử dụng
        PublicKey public_key = 2;
        string chain_id = 3;
        uint64 account_number = 4;
    }
    ```

2. Ký các byte `SignDocAux` đã mã hóa
3. Gửi chữ ký và `SignerInfo` của họ đến người ký chính, người sau đó sẽ ký và broadcast giao dịch cuối cùng (với `SIGN_MODE_DIRECT` và `AuthInfo` được thêm) khi đã thu thập đủ chữ ký

### `SIGN_MODE_DIRECT_RELAXED`

(_Được tài liệu hóa là lựa chọn (1)(a) trong https://github.com/cosmos/cosmos-sdk/issues/6078#issuecomment-628026933_)

Đây là biến thể của `SIGN_MODE_DIRECT` trong đó nhiều người ký không cần phải phối hợp khóa công khai và chế độ ký trước. Nó sẽ liên quan đến một `SignDoc` thay thế tương tự như `SignDocAux` ở trên với phí. Điều này có thể được thêm trong tương lai nếu các nhà phát triển client thấy gánh nặng thu thập khóa công khai và chế độ trước là quá nặng nề.

## Hậu Quả

### Tích Cực

* Cải thiện hiệu suất đáng kể.
* Hỗ trợ tương thích kiểu ngược và xuôi.
* Hỗ trợ tốt hơn cho các client đa ngôn ngữ.
* Nhiều chế độ ký cho phép phát triển giao thức mạnh mẽ hơn.

### Tiêu Cực

* Các URL kiểu `google.protobuf.Any` làm tăng kích thước giao dịch mặc dù tác động có thể không đáng kể hoặc nén có thể giảm thiểu điều đó.

### Trung Lập

## Tài Liệu Tham Khảo
