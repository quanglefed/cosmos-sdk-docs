---
sidebar_position: 1
---

# Giao dịch (Transactions)

:::note Tóm tắt
`Transactions` (Giao dịch) là các đối tượng được người dùng cuối tạo ra để kích hoạt các thay đổi trạng thái trong ứng dụng.
:::

:::note Tài liệu cần đọc trước

* [Cấu trúc của một ứng dụng Cosmos SDK](../beginner/00-app-anatomy.md)

:::

## Giao dịch

Giao dịch bao gồm các metadata được giữ trong [context](./02-context.md) và các [`sdk.Msg`](../../build/building-modules/02-messages-and-queries.md) kích hoạt các thay đổi trạng thái bên trong một module thông qua Protobuf [`Msg` service](../../build/building-modules/03-msg-services.md) của module đó.

Khi người dùng muốn tương tác với ứng dụng và thực hiện thay đổi trạng thái (ví dụ: gửi coin), họ tạo ra các giao dịch. Mỗi `sdk.Msg` trong một giao dịch phải được ký bằng khóa riêng tư liên kết với (các) tài khoản tương ứng, trước khi giao dịch được phát sóng lên mạng. Giao dịch sau đó phải được đưa vào một block, được xác thực và chấp thuận bởi mạng thông qua quá trình đồng thuận. Để đọc thêm về vòng đời của một giao dịch, nhấp vào [đây](../beginner/01-tx-lifecycle.md).

## Định nghĩa kiểu

Các đối tượng giao dịch là các kiểu Cosmos SDK triển khai interface `Tx`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/types/tx_msg.go#L53-L58
```

Nó chứa các phương thức sau:

* **GetMsgs:** giải nén giao dịch và trả về danh sách các `sdk.Msg` chứa trong đó — một giao dịch có thể có một hoặc nhiều message, được định nghĩa bởi các nhà phát triển module.

Là nhà phát triển, bạn hiếm khi cần thao tác trực tiếp với `Tx`, vì `Tx` là kiểu trung gian được dùng để tạo giao dịch. Thay vào đó, các nhà phát triển nên ưu tiên dùng interface `TxBuilder`, bạn có thể tìm hiểu thêm về nó [bên dưới](#tạo-giao-dịch).

### Ký giao dịch

Mỗi message trong một giao dịch phải được ký bởi các địa chỉ được chỉ định trong `GetSigners` của nó. Cosmos SDK hiện cho phép ký giao dịch theo hai cách khác nhau.

#### `SIGN_MODE_DIRECT` (được khuyến nghị)

Triển khai phổ biến nhất của interface `Tx` là Protobuf message `Tx`, được dùng trong `SIGN_MODE_DIRECT`:

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/proto/cosmos/tx/v1beta1/tx.proto#L15-L28
```

Vì quá trình tuần tự hóa Protobuf không xác định (non-deterministic), Cosmos SDK sử dụng thêm kiểu `TxRaw` để biểu thị các byte cố định mà giao dịch được ký lên. Bất kỳ người dùng nào cũng có thể tạo `body` và `auth_info` hợp lệ cho một giao dịch, và tuần tự hóa hai message này bằng Protobuf. `TxRaw` sau đó cố định biểu diễn nhị phân chính xác của người dùng cho `body` và `auth_info`, được gọi lần lượt là `body_bytes` và `auth_info_bytes`. Tài liệu được ký bởi tất cả các bên ký của giao dịch là `SignDoc` (được tuần tự hóa xác định bằng [ADR-027](../../build/architecture/adr-027-deterministic-protobuf-serialization.md)):

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/proto/cosmos/tx/v1beta1/tx.proto#L50-L67
```

Sau khi được ký bởi tất cả các bên ký, `body_bytes`, `auth_info_bytes` và `signatures` được tập hợp vào `TxRaw`, các byte tuần tự hóa của nó được phát sóng lên mạng.

#### `SIGN_MODE_LEGACY_AMINO_JSON`

Triển khai kế thừa (legacy) của interface `Tx` là struct `StdTx` từ `x/auth`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/x/auth/migrations/legacytx/stdtx.go#L82-L89
```

Tài liệu được ký bởi tất cả các bên ký là `StdSignDoc`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/x/auth/migrations/legacytx/stdsign.go#L30-L43
```

được mã hóa thành bytes bằng Amino JSON. Sau khi tất cả chữ ký được tập hợp vào `StdTx`, `StdTx` được tuần tự hóa bằng Amino JSON và các byte này được phát sóng lên mạng.

#### Các chế độ ký khác

Cosmos SDK cũng cung cấp một số chế độ ký khác cho các trường hợp sử dụng đặc biệt.

#### `SIGN_MODE_DIRECT_AUX`

`SIGN_MODE_DIRECT_AUX` là chế độ ký được phát hành trong Cosmos SDK v0.46, nhắm đến các giao dịch có nhiều bên ký. Trong khi `SIGN_MODE_DIRECT` yêu cầu mỗi bên ký phải ký lên cả `TxBody` và `AuthInfo` (bao gồm thông tin của tất cả các bên ký khác, tức là sequence tài khoản, khóa công khai và thông tin chế độ của họ), thì `SIGN_MODE_DIRECT_AUX` cho phép N-1 bên ký chỉ ký lên `TxBody` và thông tin bên ký _của chính họ_. Hơn nữa, mỗi bên ký phụ trợ (auxiliary signer — tức là bên ký dùng `SIGN_MODE_DIRECT_AUX`) không cần ký lên phí:

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/proto/cosmos/tx/v1beta1/tx.proto#L68-L93
```

Trường hợp sử dụng là giao dịch đa chữ ký (multi-signer), trong đó một trong các bên ký được chỉ định để thu thập tất cả chữ ký, phát sóng chữ ký và trả phí, còn những người khác chỉ quan tâm đến nội dung giao dịch. Điều này nhìn chung cho phép UX đa chữ ký tốt hơn. Nếu Alice, Bob và Charlie là một phần của giao dịch 3 bên ký, thì Alice và Bob đều có thể dùng `SIGN_MODE_DIRECT_AUX` để ký lên `TxBody` và thông tin bên ký của riêng họ (không cần bước bổ sung để thu thập thông tin của các bên ký khác, như trong `SIGN_MODE_DIRECT`), mà không cần chỉ định phí trong SignDoc. Charlie sau đó có thể thu thập cả hai chữ ký từ Alice và Bob, và tạo giao dịch cuối cùng bằng cách thêm phí vào. Lưu ý rằng người trả phí của giao dịch (trong trường hợp này là Charlie) phải ký lên phí, do đó phải dùng `SIGN_MODE_DIRECT` hoặc `SIGN_MODE_LEGACY_AMINO_JSON`.

#### `SIGN_MODE_TEXTUAL`

`SIGN_MODE_TEXTUAL` là chế độ ký mới nhằm cung cấp trải nghiệm ký tốt hơn trên ví phần cứng (hardware wallets) và được đưa vào phiên bản v0.50. Trong chế độ này, bên ký ký lên biểu diễn chuỗi dạng con người đọc được (human-readable) của giao dịch (CBOR) và làm cho tất cả dữ liệu được hiển thị dễ đọc hơn. Dữ liệu được định dạng dưới dạng màn hình, và mỗi màn hình được thiết kế để hiển thị toàn bộ ngay cả trên các thiết bị nhỏ như Ledger Nano.

Ngoài ra còn có các màn hình _chuyên gia_ (expert), chỉ hiển thị nếu người dùng đã chọn tùy chọn đó trên thiết bị phần cứng của họ. Các màn hình này chứa thông tin như số tài khoản, sequence tài khoản và hash dữ liệu ký.

Dữ liệu được định dạng bằng một tập hợp các `ValueRenderer` mà SDK cung cấp mặc định cho tất cả các message và kiểu giá trị đã biết. Các nhà phát triển chain cũng có thể tùy chọn triển khai `ValueRenderer` riêng cho một kiểu/message nếu muốn hiển thị thông tin theo cách khác.

Nếu bạn muốn tìm hiểu thêm, hãy tham khảo [ADR-050](../../build/architecture/adr-050-sign-mode-textual.md).

#### Chế độ ký tùy chỉnh

Có cơ hội để thêm chế độ ký tùy chỉnh của riêng bạn vào Cosmos SDK. Mặc dù chúng tôi không thể chấp nhận triển khai chế độ ký vào repository, chúng tôi có thể chấp nhận pull request để thêm signmode tùy chỉnh vào enum SignMode tại [đây](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/proto/cosmos/tx/signing/v1beta1/signing.proto#L17).

## Quy trình giao dịch

Quy trình để người dùng cuối gửi giao dịch là:

* quyết định các message cần đưa vào giao dịch,
* tạo giao dịch bằng `TxBuilder` của Cosmos SDK,
* phát sóng giao dịch thông qua một trong các interface có sẵn.

Các đoạn tiếp theo sẽ mô tả từng thành phần này theo thứ tự.

### Messages

:::tip
Module `sdk.Msg` không được nhầm lẫn với [ABCI Messages](https://docs.cometbft.com/v0.37/spec/abci/) — định nghĩa các tương tác giữa lớp CometBFT và lớp ứng dụng.
:::

**Messages** (hay `sdk.Msg`) là các đối tượng dành riêng cho từng module, kích hoạt các chuyển đổi trạng thái trong phạm vi module mà chúng thuộc về. Các nhà phát triển module định nghĩa các message cho module của họ bằng cách thêm các phương thức vào Protobuf [`Msg` service](../../build/building-modules/03-msg-services.md) và cũng triển khai `MsgServer` tương ứng.

Mỗi `sdk.Msg` liên kết với đúng một Protobuf [`Msg` service](../../build/building-modules/03-msg-services.md) RPC, được định nghĩa bên trong file `tx.proto` của mỗi module. Một app router của SDK tự động ánh xạ mỗi `sdk.Msg` với một RPC tương ứng. Protobuf tạo ra interface `MsgServer` cho mỗi module `Msg` service, và nhà phát triển module cần triển khai interface này. Thiết kế này đặt thêm trách nhiệm lên các nhà phát triển module, cho phép các nhà phát triển ứng dụng tái sử dụng các chức năng chung mà không cần triển khai lặp đi lặp lại logic chuyển đổi trạng thái.

Để tìm hiểu thêm về Protobuf `Msg` service và cách triển khai `MsgServer`, nhấp vào [đây](../../build/building-modules/03-msg-services.md).

Trong khi các message chứa thông tin cho logic chuyển đổi trạng thái, các metadata và thông tin liên quan khác của giao dịch được lưu trữ trong `TxBuilder` và `Context`.

### Tạo giao dịch

Interface `TxBuilder` chứa dữ liệu liên quan chặt chẽ đến việc tạo giao dịch, mà người dùng cuối có thể đặt để tạo giao dịch mong muốn:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/client/tx_config.go#L39-L57
```

* `Msg`, mảng các [message](#messages) được đưa vào giao dịch.
* `GasLimit`, tùy chọn do người dùng chọn để tính lượng gas cần trả.
* `Memo`, ghi chú hoặc bình luận kèm theo giao dịch.
* `FeeAmount`, số tiền tối đa người dùng sẵn sàng trả phí.
* `TimeoutHeight`, chiều cao block mà đến đó giao dịch vẫn còn hiệu lực.
* `Unordered`, tùy chọn cho biết giao dịch này có thể được thực thi theo bất kỳ thứ tự nào (yêu cầu Sequence không được đặt).
* `TimeoutTimestamp`, timestamp hết hạn (nonce không thứ tự) của giao dịch (bắt buộc dùng cùng với Unordered).
* `Signatures`, mảng chữ ký từ tất cả các bên ký của giao dịch.

Vì hiện có hai chế độ ký, cũng có hai triển khai của `TxBuilder`:

* [wrapper](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/x/auth/tx/builder.go#L27-L44) để tạo giao dịch cho `SIGN_MODE_DIRECT`,
* [StdTxBuilder](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/x/auth/migrations/legacytx/stdtx_builder.go#L14-L17) cho `SIGN_MODE_LEGACY_AMINO_JSON`.

Tuy nhiên, hai triển khai của `TxBuilder` nên được ẩn khỏi người dùng cuối, vì họ nên ưu tiên dùng interface `TxConfig` bao quát hơn:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/client/tx_config.go#L27-L37
```

`TxConfig` là cấu hình toàn ứng dụng để quản lý giao dịch. Quan trọng nhất, nó lưu thông tin về việc ký mỗi giao dịch bằng `SIGN_MODE_DIRECT` hay `SIGN_MODE_LEGACY_AMINO_JSON`. Bằng cách gọi `txBuilder := txConfig.NewTxBuilder()`, một `TxBuilder` mới sẽ được tạo với chế độ ký phù hợp.

Khi `TxBuilder` đã được điền đúng các giá trị qua các setter ở trên, `TxConfig` cũng sẽ đảm nhận việc mã hóa đúng các byte (lại, dùng `SIGN_MODE_DIRECT` hoặc `SIGN_MODE_LEGACY_AMINO_JSON`). Đây là đoạn mã giả minh họa cách tạo và mã hóa giao dịch, sử dụng phương thức `TxEncoder()`:

```go
txBuilder := txConfig.NewTxBuilder()
txBuilder.SetMsgs(...) // và các setter khác trên txBuilder

bz, err := txConfig.TxEncoder()(txBuilder.GetTx())
// bz là các byte để phát sóng lên mạng
```

### Phát sóng giao dịch

Sau khi các byte giao dịch được tạo ra, hiện có ba cách để phát sóng nó.

#### CLI

Các nhà phát triển ứng dụng tạo các điểm vào ứng dụng bằng cách tạo [giao diện dòng lệnh (CLI)](./07-cli.md), [gRPC và/hoặc REST interface](./06-grpc_rest.md), thường nằm trong thư mục `./cmd` của ứng dụng. Các interface này cho phép người dùng tương tác với ứng dụng qua dòng lệnh.

Đối với [giao diện dòng lệnh](../../build/building-modules/09-module-interfaces.md#cli), các nhà phát triển module tạo các subcommand để thêm vào lệnh giao dịch cấp cao nhất của ứng dụng `TxCmd`. Các lệnh CLI thực ra gói gọn tất cả các bước xử lý giao dịch thành một lệnh đơn giản: tạo message, tạo giao dịch và phát sóng. Ví dụ cụ thể xem phần [Tương tác với một Node](../../user/run-node/02-interact-node.md). Một giao dịch mẫu dùng CLI trông như sau:

```bash
simd tx send $MY_VALIDATOR_ADDRESS $RECIPIENT 1000stake
```

#### gRPC

[gRPC](https://grpc.io) là thành phần chính cho lớp RPC của Cosmos SDK. Cách sử dụng chính của nó là trong ngữ cảnh các [`Query` service](../../build/building-modules/04-query-services.md) của các module. Tuy nhiên, Cosmos SDK cũng cung cấp một số gRPC service không phụ thuộc vào module, một trong số đó là service `Tx`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/proto/cosmos/tx/v1beta1/service.proto
```

Service `Tx` cung cấp một số hàm tiện ích, chẳng hạn như mô phỏng giao dịch hoặc truy vấn giao dịch, cũng như một phương thức để phát sóng giao dịch.

Ví dụ về phát sóng và mô phỏng giao dịch được hiển thị [ở đây](../../user/run-node/03-txs.md#programmatically-with-go).

#### REST

Mỗi phương thức gRPC có REST endpoint tương ứng, được tạo ra bằng [gRPC-gateway](https://github.com/grpc-ecosystem/grpc-gateway). Do đó, thay vì dùng gRPC, bạn cũng có thể dùng HTTP để phát sóng cùng một giao dịch, trên endpoint `POST /cosmos/tx/v1beta1/txs`.

Ví dụ có thể xem [ở đây](../../user/run-node/03-txs.md#using-rest).

#### CometBFT RPC

Ba phương thức trình bày ở trên thực chất là các lớp trừu tượng cấp cao hơn trên các endpoint `/broadcast_tx_{async,sync,commit}` của CometBFT RPC, được ghi lại [ở đây](https://docs.cometbft.com/v0.37/core/rpc). Điều này có nghĩa là bạn có thể dùng trực tiếp các endpoint CometBFT RPC để phát sóng giao dịch nếu muốn.

### Giao dịch không theo thứ tự (Unordered Transactions)

:::tip
Muốn bật giao dịch không theo thứ tự trên chain của bạn?
Xem [Hướng dẫn Nâng cấp v0.53.0](https://docs.cosmos.network/v0.53/build/migrations/upgrade-guide#enable-unordered-transactions-optional)
:::

:::warning
Giao dịch không theo thứ tự BẮT BUỘC để trống giá trị sequence. Khi một giao dịch vừa là không theo thứ tự vừa chứa giá trị sequence khác không, giao dịch sẽ bị từ chối. Các service hoạt động dựa trên các giả định trước đây về giá trị sequence của giao dịch nên được cập nhật để xử lý giao dịch không theo thứ tự. Các service nên biết rằng khi giao dịch là không theo thứ tự, sequence của giao dịch sẽ luôn bằng không.
:::

Bắt đầu từ Cosmos SDK v0.53.0, các chain có thể bật hỗ trợ giao dịch không theo thứ tự. Giao dịch không theo thứ tự hoạt động bằng cách sử dụng timestamp làm giá trị nonce của giao dịch. Giá trị sequence KHÔNG được đặt trong (các) chữ ký của giao dịch. Timestamp phải lớn hơn thời gian block hiện tại và không vượt quá thời lượng timeout timestamp không thứ tự tối đa được cấu hình trên chain. Người gửi phải dùng timestamp duy nhất cho mỗi giao dịch riêng biệt. Sự khác biệt có thể nhỏ đến một nanosecond.

Các timestamp duy nhất này đóng vai trò là nonce một lần dùng (one-shot nonce), và thời gian tồn tại của chúng trong trạng thái rất ngắn. Khi giao dịch được đưa vào block, một mục gồm timeout timestamp và địa chỉ tài khoản sẽ được ghi vào trạng thái. Khi thời gian block vượt qua giá trị timeout timestamp, mục đó sẽ bị xóa. Điều này đảm bảo rằng các nonce không theo thứ tự không lấp đầy bộ nhớ của chain một cách vô hạn.
