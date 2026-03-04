---
sidebar_position: 1
---

# Vòng Đời Giao Dịch

:::note Tóm tắt
Tài liệu này mô tả vòng đời của một giao dịch từ khi tạo cho đến khi các thay đổi trạng thái được commit. Định nghĩa giao dịch được mô tả trong [tài liệu khác](../advanced/01-transactions.md). Giao dịch được gọi là `Tx`.
:::

:::note Tài liệu cần đọc trước

* [Cấu trúc của một ứng dụng Cosmos SDK](./00-app-anatomy.md)
:::

## Tạo giao dịch

### Tạo giao dịch

Một trong những giao diện ứng dụng chính là command-line interface. Giao dịch `Tx` có thể được tạo bởi người dùng bằng cách nhập một lệnh theo định dạng sau từ [command-line](../advanced/07-cli.md), cung cấp loại giao dịch trong `[command]`, các đối số trong `[args]`, và cấu hình như gas price trong `[flags]`:

```bash
[appname] tx [command] [args] [flags]
```

Lệnh này tự động **tạo** giao dịch, **ký** nó bằng private key của tài khoản, và **broadcast** nó đến peer node được chỉ định.

Có một số flag bắt buộc và tùy chọn để tạo giao dịch. Flag `--from` chỉ định [tài khoản](./03-accounts.md) nào là nguồn gốc của giao dịch. Ví dụ, nếu giao dịch là gửi coin, tiền sẽ được lấy từ địa chỉ `from` được chỉ định.

#### Gas và Phí

Ngoài ra, có một số [flag](../advanced/07-cli.md) mà người dùng có thể dùng để chỉ định mức sẵn lòng thanh toán trong [phí](./04-gas-fees.md):

* `--gas` chỉ số [gas](./04-gas-fees.md), đại diện cho tài nguyên tính toán, mà `Tx` tiêu thụ. Gas phụ thuộc vào giao dịch và không được tính chính xác cho đến khi thực thi, nhưng có thể được ước tính bằng cách cung cấp `auto` làm giá trị cho `--gas`.
* `--gas-adjustment` (tùy chọn) có thể được dùng để tăng `gas` nhằm tránh ước tính thiếu. Ví dụ, người dùng có thể chỉ định hệ số điều chỉnh gas là 1.5 để dùng 1.5 lần gas ước tính.
* `--gas-prices` chỉ định mức người dùng sẵn lòng trả cho mỗi đơn vị gas, có thể theo một hoặc nhiều đơn vị token. Ví dụ, `--gas-prices=0.025uatom, 0.025upho` có nghĩa người dùng sẵn lòng trả 0.025uatom VÀ 0.025upho cho mỗi đơn vị gas.
* `--fees` chỉ định tổng số phí người dùng sẵn lòng trả.
* `--timeout-height` chỉ định chiều cao block timeout để ngăn giao dịch được commit sau một chiều cao nhất định.

Giá trị cuối cùng của phí được trả bằng gas nhân với gas price. Nói cách khác, `fees = ceil(gas * gasPrices)`. Do đó, vì phí có thể được tính từ gas price và ngược lại, người dùng chỉ cần chỉ định một trong hai.

Sau đó, các validator quyết định có đưa giao dịch vào block không bằng cách so sánh `gas-prices` được cung cấp hoặc tính toán với `min-gas-prices` cục bộ của họ. `Tx` bị từ chối nếu `gas-prices` của nó không đủ cao, do đó người dùng được khuyến khích trả nhiều hơn.

#### Giao Dịch Không Có Thứ Tự (Unordered Transactions)

Với Cosmos SDK v0.53.0, người dùng có thể gửi giao dịch không có thứ tự đến các chuỗi đã bật tính năng này. Các flag sau cho phép người dùng tạo giao dịch không có thứ tự từ CLI.

* `--unordered` chỉ định rằng giao dịch này là không có thứ tự. (sequence của giao dịch phải không được đặt)
* `--timeout-duration` chỉ định khoảng thời gian mà giao dịch không có thứ tự có hiệu lực trong mempool. Nonce không có thứ tự của giao dịch sẽ được đặt thành thời gian tạo giao dịch + timeout duration.

:::warning

Giao dịch không có thứ tự PHẢI để các giá trị sequence không được đặt. Khi một giao dịch vừa không có thứ tự vừa chứa giá trị sequence khác không, giao dịch sẽ bị từ chối. Các dịch vụ bên ngoài hoạt động dựa trên các giả định trước đây về giá trị sequence của giao dịch nên được cập nhật để xử lý giao dịch không có thứ tự. Các dịch vụ cần biết rằng khi giao dịch là không có thứ tự, sequence của giao dịch sẽ luôn bằng không.

:::

#### Ví dụ CLI

Người dùng ứng dụng `app` có thể nhập lệnh sau vào CLI để tạo giao dịch gửi 1000uatom từ `senderAddress` đến `recipientAddress`. Lệnh chỉ định mức gas sẵn lòng trả: ước tính tự động nhân lên 1.5 lần, với gas price là 0.025uatom mỗi đơn vị gas.

```bash
appd tx send <recipientAddress> 1000uatom --from <senderAddress> --gas auto --gas-adjustment 1.5 --gas-prices 0.025uatom
```

#### Các phương thức tạo giao dịch khác

Command-line là cách dễ dàng để tương tác với ứng dụng, nhưng `Tx` cũng có thể được tạo bằng [giao diện gRPC hoặc REST](../advanced/06-grpc_rest.md) hoặc một số điểm vào khác được định nghĩa bởi nhà phát triển ứng dụng.

## Thêm vào Mempool

Mỗi full-node (chạy CometBFT) nhận được một `Tx` gửi một [thông báo ABCI](https://docs.cometbft.com/v0.37/spec/p2p/legacy-docs/messages/) `CheckTx` đến lớp ứng dụng để kiểm tra tính hợp lệ, và nhận một `abci.CheckTxResponse`. Nếu `Tx` qua được các kiểm tra, nó được giữ trong [**Mempool**](https://docs.cometbft.com/v0.37/spec/p2p/legacy-docs/messages/mempool) của node — một pool giao dịch trong bộ nhớ duy nhất cho mỗi node, chờ được đưa vào block. Các node trung thực loại bỏ `Tx` nếu nó được phát hiện là không hợp lệ. Trước khi đồng thuận, các node liên tục kiểm tra các giao dịch đến và gossip chúng đến các peer.

### Các loại kiểm tra

Các full-node thực hiện kiểm tra stateless rồi stateful trên `Tx` trong quá trình `CheckTx`, với mục tiêu xác định và từ chối giao dịch không hợp lệ càng sớm càng tốt để tránh lãng phí tính toán.

**Kiểm tra _stateless_** không yêu cầu node truy cập trạng thái — light client hoặc node offline cũng có thể thực hiện — và do đó ít tốn kém về mặt tính toán hơn. Kiểm tra stateless bao gồm đảm bảo địa chỉ không rỗng, thực thi các số không âm, và các logic khác được chỉ định trong các định nghĩa.

**Kiểm tra _stateful_** xác thực giao dịch và message dựa trên trạng thái đã commit. Ví dụ bao gồm kiểm tra rằng các giá trị liên quan tồn tại và có thể giao dịch được, địa chỉ có đủ tiền, và người gửi được ủy quyền hoặc có quyền sở hữu đúng để giao dịch. Tại bất kỳ thời điểm nào, các full-node thường có [nhiều phiên bản](../advanced/00-baseapp.md#state-updates) trạng thái nội bộ của ứng dụng cho các mục đích khác nhau.

Để xác minh `Tx`, các full-node gọi `CheckTx`, bao gồm cả kiểm tra _stateless_ và _stateful_. Việc xác thực thêm xảy ra sau đó trong giai đoạn [`DeliverTx`](#delivertx). `CheckTx` đi qua nhiều bước, bắt đầu bằng giải mã `Tx`.

### Giải mã (Decoding)

Khi `Tx` được ứng dụng nhận từ consensus engine bên dưới (ví dụ: CometBFT), nó vẫn ở dạng `[]byte` đã [mã hóa](../advanced/05-encoding.md) và cần được unmarshal để được xử lý. Sau đó, hàm [`runTx`](../advanced/00-baseapp.md#runtx-antehandler-runmsgs-posthandler) được gọi để chạy ở chế độ `runTxModeCheck`, nghĩa là hàm chạy tất cả các kiểm tra nhưng thoát trước khi thực thi các message và ghi các thay đổi trạng thái.

### ValidateBasic (đã lỗi thời)

Các message ([`sdk.Msg`](../advanced/01-transactions.md#messages)) được trích xuất từ giao dịch (`Tx`). Phương thức `ValidateBasic` của interface `sdk.Msg` được triển khai bởi nhà phát triển module sẽ được chạy cho mỗi giao dịch. Để loại bỏ các message rõ ràng không hợp lệ, kiểu `BaseApp` gọi phương thức `ValidateBasic` rất sớm trong quá trình xử lý message.

`ValidateBasic` chỉ có thể bao gồm các kiểm tra **stateless** (các kiểm tra không yêu cầu truy cập trạng thái).

:::warning
Phương thức `ValidateBasic` trên message đã bị lỗi thời thay bằng cách xác thực message trực tiếp trong các [`Msg` service](../../build/building-modules/03-msg-services.md#Validation) tương ứng.

Đọc [RFC 001](https://docs.cosmos.network/main/rfc/rfc-001-tx-validation) để biết thêm chi tiết.
:::

:::note
`BaseApp` vẫn gọi `ValidateBasic` trên các message triển khai phương thức đó để đảm bảo tương thích ngược.
:::

#### Hướng dẫn

`ValidateBasic` không nên được dùng nữa. Việc xác thực message nên được thực hiện trong `Msg` service khi [xử lý một message](../../build/building-modules/msg-services#Validation) trong module Msg Server.

### AnteHandler

`AnteHandler`, dù tùy chọn, nhưng trong thực tế thường được dùng để thực hiện xác minh chữ ký, tính toán gas, khấu trừ phí, và các hoạt động cốt lõi khác liên quan đến giao dịch blockchain.

Một bản sao của cached context được cung cấp cho `AnteHandler`, thực hiện các kiểm tra giới hạn được chỉ định cho loại giao dịch. Sử dụng bản sao cho phép `AnteHandler` thực hiện các kiểm tra stateful cho `Tx` mà không sửa đổi trạng thái đã commit cuối cùng, và hoàn tác về trạng thái ban đầu nếu thực thi thất bại.

Ví dụ, `AnteHandler` của module [`auth`](https://github.com/cosmos/cosmos-sdk/blob/main/x/auth/README.md) kiểm tra và tăng số sequence, kiểm tra chữ ký và số tài khoản, và khấu trừ phí từ người ký đầu tiên của giao dịch.

:::warning
Ante handler chỉ chạy trên một giao dịch. Nếu một giao dịch nhúng nhiều message (như một số giao dịch x/authz, x/gov chẳng hạn), ante handler chỉ biết về message ngoài. Các message bên trong chủ yếu được định tuyến trực tiếp đến [message router](https://docs.cosmos.network/main/learn/advanced/baseapp#msg-service-router) và sẽ bỏ qua chuỗi ante handler. Hãy ghi nhớ điều này khi thiết kế ante handler của bạn.
:::

### Gas

[`Context`](../advanced/02-context.md), giữ một `GasMeter` theo dõi lượng gas đã sử dụng trong quá trình thực thi `Tx`, được khởi tạo. Lượng gas mà người dùng cung cấp cho `Tx` được gọi là `GasWanted`. Nếu `GasConsumed` — lượng gas tiêu thụ trong quá trình thực thi — vượt quá `GasWanted`, việc thực thi dừng lại và các thay đổi đối với bản sao cached của trạng thái không được commit. Ngược lại, `CheckTx` đặt `GasUsed` bằng `GasConsumed` và trả về trong kết quả. Sau khi tính toán giá trị gas và phí, validator node kiểm tra rằng `gas-prices` do người dùng chỉ định lớn hơn `min-gas-prices` được định nghĩa cục bộ của họ.

### Loại bỏ hoặc Thêm vào Mempool

Nếu tại bất kỳ thời điểm nào trong `CheckTx` mà `Tx` thất bại, nó bị loại bỏ và vòng đời giao dịch kết thúc ở đó. Nếu không, nếu nó qua `CheckTx` thành công, giao thức mặc định là chuyển tiếp nó đến các peer node và thêm vào Mempool để `Tx` trở thành ứng cử viên được đưa vào block tiếp theo.

**Mempool** phục vụ mục đích theo dõi các giao dịch được tất cả full-node thấy. Full-node giữ một **bộ nhớ cache mempool** của `mempool.cache_size` giao dịch cuối cùng mà chúng đã thấy, như lớp phòng thủ đầu tiên để ngăn chặn tấn công replay. Lý tưởng nhất, `mempool.cache_size` đủ lớn để bao gồm tất cả giao dịch trong mempool đầy đủ. Nếu bộ nhớ cache mempool quá nhỏ, `CheckTx` chịu trách nhiệm xác định và từ chối các giao dịch replay.

Các biện pháp phòng ngừa hiện có bao gồm phí và bộ đếm `sequence` (nonce) để phân biệt giao dịch replay với các giao dịch giống hệt nhưng hợp lệ. Nếu kẻ tấn công cố spam các node với nhiều bản sao của `Tx`, các full-node giữ bộ nhớ cache mempool sẽ từ chối tất cả bản sao giống nhau thay vì chạy `CheckTx` trên chúng. Ngay cả khi các bản sao có số `sequence` tăng dần, kẻ tấn công bị cản trở bởi nhu cầu trả phí.

Các validator node giữ mempool để ngăn chặn tấn công replay, giống như full-node, nhưng cũng dùng nó như một pool giao dịch chưa xác nhận để chuẩn bị đưa vào block. Lưu ý rằng ngay cả khi `Tx` vượt qua tất cả kiểm tra ở giai đoạn này, nó vẫn có thể bị phát hiện là không hợp lệ sau này, vì `CheckTx` không xác thực đầy đủ giao dịch (tức là không thực sự thực thi các message).

## Đưa vào Block

Đồng thuận, quá trình mà các validator node đồng ý về giao dịch nào cần chấp nhận, xảy ra theo **vòng**. Mỗi vòng bắt đầu với một proposer tạo một block gồm các giao dịch gần đây nhất và kết thúc với **validator** — các full-node đặc biệt có voting power chịu trách nhiệm đồng thuận — đồng ý chấp nhận block hoặc chọn block `nil` thay thế. Các validator node thực thi thuật toán đồng thuận như [CometBFT](https://docs.cometbft.com/v0.37/spec/consensus/) để đi đến thỏa thuận này.

Bước đầu tiên của đồng thuận là **đề xuất block**. Một proposer trong số các validator được thuật toán đồng thuận chọn để tạo và đề xuất block — để `Tx` được đưa vào, nó phải có trong mempool của proposer này.

## Thay Đổi Trạng Thái

Bước tiếp theo của đồng thuận là thực thi các giao dịch để xác thực đầy đủ chúng. Tất cả full-node nhận được đề xuất block từ proposer đúng sẽ thực thi các giao dịch bằng cách gọi hàm ABCI `FinalizeBlock`. Như đã đề cập xuyên suốt tài liệu, `BeginBlock`, `ExecuteTx` và `EndBlock` được gọi bên trong `FinalizeBlock`. Mặc dù mỗi full-node hoạt động độc lập và cục bộ, kết quả luôn nhất quán và rõ ràng, vì các thay đổi trạng thái do message mang lại là có thể dự đoán được và các giao dịch được sắp xếp cụ thể trong block được đề xuất.

```text
        --------------------------
        | Nhận đề xuất block     |
        --------------------------
                    |
                    v
        -------------------------
        |     FinalizeBlock      |
        -------------------------
                    |
                    v
            -------------------
            |   BeginBlock     | 
            -------------------
                    |
                    v
            --------------------
            | ExecuteTx(tx0)   |
            | ExecuteTx(tx1)   |
            | ExecuteTx(tx2)   |
            | ExecuteTx(tx3)   |
            |       .          |
            |       .          |
            |       .          |
            -------------------
                    |
                    v
            --------------------
            |    EndBlock      |
            --------------------
                    |
                    v
        -------------------------
        |       Consensus        |
        -------------------------
                    |
                    v
        -------------------------
        |         Commit         |
        -------------------------
```

### Thực Thi Giao Dịch

Hàm ABCI `FinalizeBlock` được định nghĩa trong [`BaseApp`](../advanced/00-baseapp.md) thực hiện phần lớn các chuyển đổi trạng thái: nó được chạy cho mỗi giao dịch trong block theo thứ tự tuần tự như đã commit trong quá trình đồng thuận. Về cơ bản, việc thực thi giao dịch gần giống với `CheckTx` nhưng gọi hàm [`runTx`](../advanced/00-baseapp.md#runtx) ở chế độ deliver thay vì check mode. Thay vì dùng `checkState`, các full-node dùng `finalizeblock`:

* **Giải mã:** Vì `FinalizeBlock` là một lời gọi ABCI, `Tx` được nhận ở dạng `[]byte` đã mã hóa. Các node trước tiên unmarshal giao dịch bằng [`TxConfig`](./00-app-anatomy.md#register-codec) được định nghĩa trong ứng dụng, sau đó gọi `runTx` trong `execModeFinalize`, rất giống với `CheckTx` nhưng cũng thực thi và ghi các thay đổi trạng thái.

* **Kiểm tra và `AnteHandler`:** Full-node gọi `validateBasicMsgs` và `AnteHandler` một lần nữa. Lần kiểm tra thứ hai này xảy ra vì chúng có thể không đã thấy các giao dịch tương tự trong giai đoạn thêm vào Mempool, và một proposer độc hại có thể đã bao gồm các giao dịch không hợp lệ. Một điểm khác biệt là `AnteHandler` không so sánh `gas-prices` với `min-gas-prices` của node vì giá trị đó là cục bộ cho mỗi node.

* **`MsgServiceRouter`:** Sau khi `CheckTx` thoát, `FinalizeBlock` tiếp tục chạy [`runMsgs`](../advanced/00-baseapp.md#runtx-antehandler-runmsgs-posthandler) để thực thi đầy đủ mỗi `Msg` trong giao dịch. Vì giao dịch có thể có message từ các module khác nhau, `BaseApp` cần biết module nào để tìm handler phù hợp bằng cách dùng `MsgServiceRouter` của `BaseApp`.

* **`Msg` service:** Protobuf `Msg` service chịu trách nhiệm thực thi mỗi message trong `Tx` và gây ra các chuyển đổi trạng thái được lưu trong `finalizeBlockState`.

* **PostHandlers:** [`PostHandler`](../advanced/00-baseapp.md#posthandler) chạy sau khi thực thi message. Nếu chúng thất bại, thay đổi trạng thái của `runMsgs` cũng như `PostHandlers` đều bị hoàn tác.

* **Gas:** Trong khi `Tx` đang được xử lý, `GasMeter` được dùng để theo dõi lượng gas đang được sử dụng; nếu thực thi hoàn tất, `GasUsed` được đặt và trả về trong `abci.ExecTxResult`. Nếu thực thi dừng vì `BlockGasMeter` hoặc `GasMeter` đã cạn hoặc có gì đó sai, một hàm deferred ở cuối sẽ xử lý lỗi hoặc panic phù hợp.

Nếu có bất kỳ thay đổi trạng thái thất bại nào do `Tx` không hợp lệ hoặc `GasMeter` cạn kiệt, quá trình xử lý giao dịch kết thúc và mọi thay đổi trạng thái đều bị hoàn tác. Giao dịch không hợp lệ trong đề xuất block khiến validator node từ chối block và bỏ phiếu cho block `nil` thay thế.

### Commit

Bước cuối cùng là các node commit block và thay đổi trạng thái. Các validator node thực hiện bước thực thi chuyển đổi trạng thái trước đó để xác thực các giao dịch, sau đó ký block để xác nhận nó. Các full-node không phải validator không tham gia vào đồng thuận — tức là họ không thể bỏ phiếu — nhưng lắng nghe các phiếu bầu để hiểu liệu họ có nên commit các thay đổi trạng thái hay không.

Khi nhận đủ phiếu validator (2/3+ _precommit_ được tính theo voting power), các full-node commit vào một block mới để thêm vào blockchain và hoàn thiện các chuyển đổi trạng thái trong lớp ứng dụng. Một state root mới được tạo ra để phục vụ như bằng chứng merkle cho các chuyển đổi trạng thái. Ứng dụng sử dụng phương thức ABCI [`Commit`](../advanced/00-baseapp.md#commit) kế thừa từ [Baseapp](../advanced/00-baseapp.md); nó đồng bộ tất cả các chuyển đổi trạng thái bằng cách ghi `deliverState` vào trạng thái nội bộ của ứng dụng. Ngay khi các thay đổi trạng thái được commit, `checkState` bắt đầu lại từ trạng thái đã commit gần đây nhất và `deliverState` được reset về `nil`.

Lưu ý rằng không phải tất cả block đều có cùng số lượng giao dịch và đồng thuận có thể dẫn đến block `nil` hoặc block không có giao dịch. Trong một mạng blockchain công khai, các validator cũng có thể **byzantine** — tức là độc hại — điều này có thể ngăn `Tx` được commit vào blockchain.

Tại đây, vòng đời giao dịch của `Tx` đã hoàn tất: các node đã xác minh tính hợp lệ của nó, thực thi bằng cách thực hiện các thay đổi trạng thái của nó, và commit các thay đổi đó. Bản thân `Tx`, ở dạng `[]byte`, được lưu trong một block và được nối thêm vào blockchain.
