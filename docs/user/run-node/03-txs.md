---
sidebar_position: 1
---

# Tạo, Ký và Phát Sóng Giao Dịch

:::note Tóm tắt
Tài liệu này mô tả cách tạo một giao dịch (chưa ký), ký giao dịch đó (với một hoặc nhiều khóa), và phát sóng nó lên mạng.
:::

## Sử Dụng CLI

Cách đơn giản nhất để gửi giao dịch là sử dụng CLI, như chúng ta đã thấy ở trang trước khi [tương tác với một node](./02-interact-node.md#using-the-cli). Ví dụ, chạy lệnh sau:

```bash
simd tx bank send $MY_VALIDATOR_ADDRESS $RECIPIENT 1000stake --chain-id my-test-chain --keyring-backend test
```

sẽ thực hiện các bước sau:

* tạo một giao dịch với một `Msg` (`MsgSend` của `x/bank`), và in giao dịch đã tạo ra console.
* yêu cầu người dùng xác nhận gửi giao dịch từ tài khoản `$MY_VALIDATOR_ADDRESS`.
* lấy `$MY_VALIDATOR_ADDRESS` từ keyring. Điều này khả thi vì chúng ta đã [thiết lập keyring cho CLI](./00-keyring.md) ở bước trước.
* ký giao dịch đã tạo bằng tài khoản trong keyring.
* phát sóng giao dịch đã ký lên mạng. Điều này khả thi vì CLI kết nối đến endpoint CometBFT RPC của node.

CLI gộp tất cả các bước cần thiết vào một trải nghiệm người dùng đơn giản. Tuy nhiên, cũng có thể thực hiện từng bước một cách riêng lẻ.

### Tạo Giao Dịch

Tạo giao dịch có thể được thực hiện đơn giản bằng cách thêm cờ `--generate-only` vào bất kỳ lệnh `tx` nào, ví dụ:

```bash
simd tx bank send $MY_VALIDATOR_ADDRESS $RECIPIENT 1000stake --chain-id my-test-chain --generate-only
```

Lệnh này sẽ in giao dịch chưa ký dưới dạng JSON ra console. Chúng ta cũng có thể lưu giao dịch chưa ký vào một file (để dễ dàng chuyển giữa các người ký) bằng cách thêm `> unsigned_tx.json` vào lệnh trên.

### Ký Giao Dịch

Ký giao dịch bằng CLI yêu cầu giao dịch chưa ký phải được lưu trong một file. Giả sử giao dịch chưa ký nằm trong file `unsigned_tx.json` ở thư mục hiện tại (xem đoạn trước về cách thực hiện). Sau đó, đơn giản chạy lệnh sau:

```bash
simd tx sign unsigned_tx.json --chain-id my-test-chain --keyring-backend test --from $MY_VALIDATOR_ADDRESS
```

Lệnh này sẽ giải mã giao dịch chưa ký và ký nó với `SIGN_MODE_DIRECT` bằng khóa của `$MY_VALIDATOR_ADDRESS`, mà chúng ta đã thiết lập trong keyring. Giao dịch đã ký sẽ được xuất ra dưới dạng JSON ra console, và tương tự như trên, chúng ta có thể lưu vào file bằng cách thêm `--output-document signed_tx.json`.

Một số cờ hữu ích cần xem xét trong lệnh `tx sign`:

* `--sign-mode`: bạn có thể sử dụng `amino-json` để ký giao dịch bằng `SIGN_MODE_LEGACY_AMINO_JSON`,
* `--offline`: ký ở chế độ ngoại tuyến (offline). Điều này có nghĩa là lệnh `tx sign` không kết nối đến node để lấy số tài khoản và số thứ tự (sequence) của người ký, vốn cần thiết để ký. Trong trường hợp này, bạn phải cung cấp thủ công các cờ `--account-number` và `--sequence`. Điều này hữu ích cho việc ký ngoại tuyến, tức là ký trong môi trường bảo mật không có quyền truy cập internet.

#### Ký với Nhiều Người Ký

:::warning
Xin lưu ý rằng việc ký giao dịch với nhiều người ký hoặc với tài khoản multisig, trong đó có ít nhất một người ký sử dụng `SIGN_MODE_DIRECT`, hiện chưa khả dụng. Bạn có thể theo dõi [issue Github này](https://github.com/cosmos/cosmos-sdk/issues/8141) để biết thêm thông tin.
:::

Ký với nhiều người ký được thực hiện bằng lệnh `tx multisign`. Lệnh này giả định rằng tất cả người ký đều sử dụng `SIGN_MODE_LEGACY_AMINO_JSON`. Quy trình tương tự như quy trình lệnh `tx sign`, nhưng thay vì ký một file giao dịch chưa ký, mỗi người ký ký file đã được ký bởi những người ký trước đó. Lệnh `tx multisign` sẽ bổ sung chữ ký vào các giao dịch hiện có. Điều quan trọng là các người ký phải ký giao dịch **theo cùng thứ tự** như được chỉ định trong giao dịch, có thể truy xuất bằng phương thức `GetSigners()`.

Ví dụ, bắt đầu với `unsigned_tx.json`, và giả sử giao dịch có 4 người ký, chúng ta sẽ chạy:

```bash
# Để signer1 ký giao dịch chưa ký.
simd tx multisign unsigned_tx.json signer_key_1 --chain-id my-test-chain --keyring-backend test > partial_tx_1.json
# Bây giờ signer1 sẽ gửi partial_tx_1.json cho signer2.
# Signer2 thêm chữ ký của mình:
simd tx multisign partial_tx_1.json signer_key_2 --chain-id my-test-chain --keyring-backend test > partial_tx_2.json
# Signer2 gửi file partial_tx_2.json cho signer3, và signer3 có thể thêm chữ ký của mình:
simd tx multisign partial_tx_2.json signer_key_3 --chain-id my-test-chain --keyring-backend test > partial_tx_3.json
```

### Phát Sóng Giao Dịch

Phát sóng giao dịch được thực hiện bằng lệnh sau:

```bash
simd tx broadcast tx_signed.json
```

Bạn có thể tùy chọn truyền cờ `--broadcast-mode` để chỉ định phản hồi nào sẽ nhận từ node:

* `sync`: CLI chờ phản hồi thực thi CheckTx.
* `async`: CLI trả về ngay lập tức (giao dịch có thể thất bại).

### Mã Hóa Giao Dịch

Để phát sóng giao dịch sử dụng các endpoint gRPC hoặc REST, giao dịch cần được mã hóa trước. Điều này có thể được thực hiện bằng CLI.

Mã hóa giao dịch được thực hiện bằng lệnh sau:

```bash
simd tx encode tx_signed.json
```

Lệnh này sẽ đọc giao dịch từ file, tuần tự hóa (serialize) nó bằng Protobuf, và in các byte giao dịch dưới dạng base64 ra console.

### Giải Mã Giao Dịch

CLI cũng có thể được sử dụng để giải mã các byte giao dịch.

Giải mã giao dịch được thực hiện bằng lệnh sau:

```bash
simd tx decode [protobuf-byte-string]
```

Lệnh này sẽ giải mã các byte giao dịch và in giao dịch dưới dạng JSON ra console. Bạn cũng có thể lưu giao dịch vào file bằng cách thêm `> tx.json` vào lệnh trên.

## Lập Trình Với Go

Có thể thao tác giao dịch theo chương trình thông qua Go bằng cách sử dụng giao diện `TxBuilder` của Cosmos SDK.

### Tạo Giao Dịch

Trước khi tạo giao dịch, cần tạo một phiên bản mới của `TxBuilder`. Vì Cosmos SDK hỗ trợ cả giao dịch Amino và Protobuf, bước đầu tiên là quyết định sử dụng phương thức mã hóa nào. Tất cả các bước tiếp theo vẫn giữ nguyên, cho dù bạn sử dụng Amino hay Protobuf, vì `TxBuilder` trừu tượng hóa các cơ chế mã hóa. Trong đoạn mã dưới đây, chúng ta sẽ sử dụng Protobuf.

```go
import (
	"github.com/cosmos/cosmos-sdk/simapp"
)

func sendTx() error {
    // Chọn codec của bạn: Amino hoặc Protobuf. Ở đây, chúng ta sử dụng Protobuf, được cung cấp bởi hàm sau.
    app := simapp.NewSimApp(...)

    // Tạo một TxBuilder mới.
    txBuilder := app.TxConfig().NewTxBuilder()

    // --snip--
}
```

Chúng ta cũng có thể thiết lập một số khóa và địa chỉ sẽ gửi và nhận các giao dịch. Ở đây, cho mục đích hướng dẫn, chúng ta sẽ sử dụng một số dữ liệu giả để tạo khóa.

```go
import (
	"github.com/cosmos/cosmos-sdk/testutil/testdata"
)

priv1, _, addr1 := testdata.KeyTestPubAddr()
priv2, _, addr2 := testdata.KeyTestPubAddr()
priv3, _, addr3 := testdata.KeyTestPubAddr()
```

Có thể điền vào `TxBuilder` thông qua các phương thức của nó:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/client/tx_config.go#L39-L57
```

```go
import (
	banktypes "github.com/cosmos/cosmos-sdk/x/bank/types"
)

func sendTx() error {
    // --snip--

    // Định nghĩa hai thông điệp MsgSend của x/bank:
    // - từ addr1 đến addr3,
    // - từ addr2 đến addr3.
    // Điều này có nghĩa là giao dịch cần hai người ký: addr1 và addr2.
    msg1 := banktypes.NewMsgSend(addr1, addr3, types.NewCoins(types.NewInt64Coin("atom", 12)))
    msg2 := banktypes.NewMsgSend(addr2, addr3, types.NewCoins(types.NewInt64Coin("atom", 34)))

    err := txBuilder.SetMsgs(msg1, msg2)
    if err != nil {
        return err
    }

    txBuilder.SetGasLimit(...)
    txBuilder.SetFeeAmount(...)
    txBuilder.SetMemo(...)
    txBuilder.SetTimeoutHeight(...)
}
```

Tại thời điểm này, giao dịch bên trong `TxBuilder` đã sẵn sàng để được ký.

#### Tạo Giao Dịch Không Theo Thứ Tự (Unordered Transaction)

Bắt đầu từ Cosmos SDK v0.53.0, người dùng có thể gửi các giao dịch không theo thứ tự đến các chuỗi đã bật tính năng này.

:::warning

Giao dịch không theo thứ tự PHẢI để trống giá trị sequence. Khi một giao dịch vừa không theo thứ tự vừa chứa giá trị sequence khác không,
giao dịch đó sẽ bị từ chối. Các dịch vụ bên ngoài hoạt động dựa trên các giả định trước đây về giá trị sequence của giao dịch nên được cập nhật để xử lý các giao dịch không theo thứ tự.
Các dịch vụ cần biết rằng khi giao dịch không theo thứ tự, sequence của giao dịch sẽ luôn bằng không.

:::

Sử dụng ví dụ ở trên, chúng ta có thể đặt các trường bắt buộc để đánh dấu giao dịch là không theo thứ tự.
Theo mặc định, các giao dịch không theo thứ tự tính thêm 2240 đơn vị gas để bù đắp chi phí lưu trữ bổ sung hỗ trợ chức năng của chúng.
Các đơn vị gas bổ sung có thể tùy chỉnh và do đó khác nhau tùy theo chuỗi, vì vậy hãy kiểm tra ante handler của chuỗi để biết giá trị gas được đặt, nếu có.

```go
func sendTx() error {
    // --snip--
    expiration := 5 * time.Minute
    txBuilder.SetUnordered(true)
    txBuilder.SetTimeoutTimestamp(time.Now().Add(expiration + (1 * time.Nanosecond)))
}
```

Để ký giao dịch, cần thực hiện hai bước:

* với mỗi người ký, điền vào `SignerInfo` của người ký đó trong `TxBuilder`,
* một khi tất cả `SignerInfo` đã được điền, với mỗi người ký, ký `SignDoc` (phần dữ liệu cần được ký).

Trong API hiện tại của `TxBuilder`, cả hai bước đều được thực hiện bằng cùng một phương thức: `SetSignatures()`. API hiện tại yêu cầu chúng ta trước tiên thực hiện một lần `SetSignatures()` _với chữ ký rỗng_, chỉ để điền `SignerInfo`, và lần thứ hai `SetSignatures()` để thực sự ký đúng payload.

```go
import (
    cryptotypes "github.com/cosmos/cosmos-sdk/crypto/types"
	"github.com/cosmos/cosmos-sdk/types/tx/signing"
	xauthsigning "github.com/cosmos/cosmos-sdk/x/auth/signing"
)

func sendTx() error {
    // --snip--

    privs := []cryptotypes.PrivKey{priv1, priv2}
    accNums:= []uint64{..., ...} // Số tài khoản của các tài khoản
    accSeqs:= []uint64{..., ...} // Số thứ tự (sequence) của các tài khoản

    // Lần đầu: chúng ta thu thập tất cả thông tin người ký. Chúng ta sử dụng thủ thuật
    // "đặt chữ ký rỗng" để làm điều đó.
    var sigsV2 []signing.SignatureV2
    for i, priv := range privs {
        sigV2 := signing.SignatureV2{
            PubKey: priv.PubKey(),
            Data: &signing.SingleSignatureData{
                SignMode:  encCfg.TxConfig.SignModeHandler().DefaultMode(),
                Signature: nil,
            },
            Sequence: accSeqs[i],
        }

        sigsV2 = append(sigsV2, sigV2)
    }
    err := txBuilder.SetSignatures(sigsV2...)
    if err != nil {
        return err
    }

    // Lần hai: tất cả thông tin người ký đã được đặt, vì vậy mỗi người ký có thể ký.
    sigsV2 = []signing.SignatureV2{}
    for i, priv := range privs {
        signerData := xauthsigning.SignerData{
            ChainID:       chainID,
            AccountNumber: accNums[i],
            Sequence:      accSeqs[i],
        }
        sigV2, err := tx.SignWithPrivKey(
            encCfg.TxConfig.SignModeHandler().DefaultMode(), signerData,
            txBuilder, priv, encCfg.TxConfig, accSeqs[i])
        if err != nil {
            return nil, err
        }

        sigsV2 = append(sigsV2, sigV2)
    }
    err = txBuilder.SetSignatures(sigsV2...)
    if err != nil {
        return err
    }
}
```

`TxBuilder` hiện đã được điền đầy đủ. Để in nó, bạn có thể sử dụng giao diện `TxConfig` từ cấu hình mã hóa ban đầu `encCfg`:

```go
func sendTx() error {
    // --snip--

    // Tạo các byte được mã hóa Protobuf.
    txBytes, err := encCfg.TxConfig.TxEncoder()(txBuilder.GetTx())
    if err != nil {
        return err
    }

    // Tạo một chuỗi JSON.
    txJSONBytes, err := encCfg.TxConfig.TxJSONEncoder()(txBuilder.GetTx())
    if err != nil {
        return err
    }
    txJSON := string(txJSONBytes)
}
```

### Phát Sóng Giao Dịch

Cách phát sóng giao dịch được ưu tiên là sử dụng gRPC, mặc dù việc sử dụng REST (thông qua `gRPC-gateway`) hoặc CometBFT RPC cũng khả thi. Tổng quan về sự khác biệt giữa các phương pháp này được trình bày [ở đây](../../learn/advanced/06-grpc_rest.md). Trong hướng dẫn này, chúng ta chỉ mô tả phương pháp gRPC.

```go
import (
    "context"
    "fmt"

	"google.golang.org/grpc"

	"github.com/cosmos/cosmos-sdk/types/tx"
)

func sendTx(ctx context.Context) error {
    // --snip--

    // Tạo kết nối đến máy chủ gRPC.
    grpcConn := grpc.Dial(
        "127.0.0.1:9090", // Hoặc địa chỉ máy chủ gRPC của bạn.
        grpc.WithInsecure(), // Cosmos SDK không hỗ trợ bất kỳ cơ chế bảo mật truyền tải nào.
    )
    defer grpcConn.Close()

    // Phát sóng giao dịch qua gRPC. Chúng ta tạo một client mới cho dịch vụ Protobuf Tx.
    txClient := tx.NewServiceClient(grpcConn)
    // Sau đó chúng ta gọi phương thức BroadcastTx trên client này.
    grpcRes, err := txClient.BroadcastTx(
        ctx,
        &tx.BroadcastTxRequest{
            Mode:    tx.BroadcastMode_BROADCAST_MODE_SYNC,
            TxBytes: txBytes, // Dữ liệu nhị phân Proto của giao dịch đã ký, xem bước trước.
        },
    )
    if err != nil {
        return err
    }

    fmt.Println(grpcRes.TxResponse.Code) // Nên là `0` nếu giao dịch thành công

    return nil
}
```

#### Mô Phỏng Giao Dịch

Trước khi phát sóng giao dịch, đôi khi chúng ta muốn chạy thử (dry-run) giao dịch để ước tính một số thông tin về giao dịch mà không thực sự commit nó. Đây được gọi là mô phỏng giao dịch, và có thể thực hiện như sau:

```go
import (
	"context"
	"fmt"
	"testing"

	"github.com/cosmos/cosmos-sdk/client"
	"github.com/cosmos/cosmos-sdk/types/tx"
	authtx "github.com/cosmos/cosmos-sdk/x/auth/tx"
)

func simulateTx() error {
    // --snip--

    // Mô phỏng giao dịch qua gRPC. Chúng ta tạo một client mới cho dịch vụ Protobuf Tx.
    txClient := tx.NewServiceClient(grpcConn)
    txBytes := /* Điền vào với các byte giao dịch đã ký của bạn. */

    // Sau đó chúng ta gọi phương thức Simulate trên client này.
    grpcRes, err := txClient.Simulate(
        context.Background(),
        &tx.SimulateRequest{
            TxBytes: txBytes,
        },
    )
    if err != nil {
        return err
    }

    fmt.Println(grpcRes.GasInfo) // In ra lượng gas ước tính đã sử dụng.

    return nil
}
```

## Sử Dụng gRPC

Không thể tạo hoặc ký giao dịch bằng gRPC, chỉ có thể phát sóng. Để phát sóng giao dịch bằng gRPC, bạn cần tạo, ký và mã hóa giao dịch bằng CLI hoặc lập trình với Go.

### Phát Sóng Giao Dịch

Phát sóng giao dịch bằng endpoint gRPC có thể được thực hiện bằng cách gửi yêu cầu `BroadcastTx` như sau, trong đó `txBytes` là các byte được mã hóa Protobuf của giao dịch đã ký:

```bash
grpcurl -plaintext \
    -d '{"tx_bytes":"{{txBytes}}","mode":"BROADCAST_MODE_SYNC"}' \
    localhost:9090 \
    cosmos.tx.v1beta1.Service/BroadcastTx
```

## Sử Dụng REST

Không thể tạo hoặc ký giao dịch bằng REST, chỉ có thể phát sóng. Để phát sóng giao dịch bằng REST, bạn cần tạo, ký và mã hóa giao dịch bằng CLI hoặc lập trình với Go.

### Phát Sóng Giao Dịch

Phát sóng giao dịch bằng endpoint REST (được phục vụ bởi `gRPC-gateway`) có thể được thực hiện bằng cách gửi yêu cầu POST như sau, trong đó `txBytes` là các byte được mã hóa Protobuf của giao dịch đã ký:

```bash
curl -X POST \
    -H "Content-Type: application/json" \
    -d' {"tx_bytes":"{{txBytes}}","mode":"BROADCAST_MODE_SYNC"}' \
    localhost:1317/cosmos/tx/v1beta1/txs
```

## Sử Dụng CosmJS (JavaScript & TypeScript)

CosmJS hướng đến việc xây dựng các thư viện client bằng JavaScript có thể được nhúng vào các ứng dụng web. Vui lòng xem [https://cosmos.github.io/cosmjs](https://cosmos.github.io/cosmjs) để biết thêm thông tin. Tính đến tháng 1 năm 2021, tài liệu CosmJS vẫn đang trong quá trình phát triển.
