---
sidebar_position: 1
---

# Giao diện Dòng lệnh (CLI)

:::note Tóm tắt
Tài liệu này mô tả cách giao diện dòng lệnh (CLI) hoạt động ở cấp độ cao, cho một [**ứng dụng**](../beginner/00-app-anatomy.md). Tài liệu riêng về triển khai CLI cho một [**module**](../../build/building-modules/00-intro.md) Cosmos SDK có thể tìm thấy [ở đây](../../build/building-modules/09-module-interfaces.md#cli).
:::

## Giao diện Dòng lệnh

### Lệnh mẫu

Không có cách duy nhất để tạo CLI, nhưng các module Cosmos SDK thường dùng [Thư viện Cobra](https://github.com/spf13/cobra). Xây dựng CLI với Cobra đòi hỏi định nghĩa các lệnh, đối số và flag. [**Lệnh**](#root-command) hiểu các hành động mà người dùng muốn thực hiện, chẳng hạn như `tx` để tạo giao dịch và `query` để query ứng dụng. Mỗi lệnh cũng có thể có các subcommand lồng nhau, cần thiết để đặt tên cho kiểu giao dịch cụ thể. Người dùng cũng cung cấp **Đối số (Arguments)**, chẳng hạn như số tài khoản để gửi coin đến, và [**Flags**](#flags) để sửa đổi các khía cạnh khác nhau của lệnh, chẳng hạn như giá gas hoặc node nào để phát sóng đến.

Đây là ví dụ về lệnh mà người dùng có thể nhập để tương tác với CLI `simd` của simapp để gửi một số token:

```bash
simd tx bank send $MY_VALIDATOR_ADDRESS $RECIPIENT 1000stake --gas auto --gas-prices <gasPrices>
```

Bốn chuỗi đầu tiên xác định lệnh:

* Lệnh gốc của toàn bộ ứng dụng `simd`.
* Subcommand `tx`, chứa tất cả lệnh cho phép người dùng tạo giao dịch.
* Subcommand `bank` để chỉ ra module cần định tuyến đến (module [`x/bank`](../../build/modules/bank/README.md) trong trường hợp này).
* Kiểu giao dịch `send`.

Hai chuỗi tiếp theo là đối số: `from_address` mà người dùng muốn gửi từ, `to_address` của người nhận, và `amount` họ muốn gửi. Cuối cùng, các chuỗi cuối của lệnh là flag tùy chọn để chỉ định mức phí người dùng sẵn sàng trả (được tính bằng lượng gas dùng để thực thi giao dịch và giá gas do người dùng cung cấp).

CLI tương tác với một [node](./03-node.md) để xử lý lệnh này. Bản thân interface được định nghĩa trong một file `main.go`.

### Xây dựng CLI

File `main.go` cần có một hàm `main()` tạo một lệnh gốc, mà tất cả các lệnh ứng dụng sẽ được thêm vào đó như các subcommand. Lệnh gốc còn xử lý:

* **thiết lập cấu hình** bằng cách đọc các file cấu hình (ví dụ: file cấu hình Cosmos SDK).
* **thêm các flag** như `--chain-id`.
* **khởi tạo `codec`** bằng cách inject các codec ứng dụng. [`codec`](./05-encoding.md) được dùng để mã hóa và giải mã các cấu trúc dữ liệu cho ứng dụng - store chỉ có thể lưu trữ `[]byte` nên developer phải định nghĩa định dạng tuần tự hóa cho cấu trúc dữ liệu hoặc dùng mặc định là Protobuf.
* **thêm subcommand** cho tất cả các tương tác người dùng có thể có, bao gồm [lệnh giao dịch](#transaction-commands) và [lệnh query](#query-commands).

Hàm `main()` cuối cùng tạo một executor và thực thi lệnh gốc. Xem ví dụ về hàm `main()` từ ứng dụng `simapp`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/simapp/simd/main.go#L14-L24
```

Phần còn lại của tài liệu sẽ chi tiết hóa những gì cần triển khai cho mỗi bước và bao gồm các đoạn code nhỏ từ các file CLI của `simapp`.

## Thêm Lệnh vào CLI

Mọi CLI ứng dụng đầu tiên xây dựng một lệnh gốc, sau đó thêm chức năng bằng cách tổng hợp các subcommand sử dụng `rootCmd.AddCommand()`. Phần lớn khả năng độc đáo của ứng dụng nằm trong các lệnh giao dịch và query, được gọi lần lượt là `TxCmd` và `QueryCmd`.

### Root Command (Lệnh gốc)

Lệnh gốc (gọi là `rootCmd`) là thứ người dùng nhập đầu tiên vào dòng lệnh. Chuỗi dùng để gọi lệnh thường là tên ứng dụng có hậu tố `-d`, ví dụ: `simd` hoặc `gaiad`. Lệnh gốc thường bao gồm các lệnh sau:

* Lệnh **Status** từ các công cụ rpc client Cosmos SDK, in thông tin về trạng thái của [`Node`](./03-node.md) đã kết nối. Trạng thái của node bao gồm `NodeInfo`, `SyncInfo` và `ValidatorInfo`.
* Các lệnh **Keys** [commands](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/client/keys) từ các công cụ client Cosmos SDK, bao gồm tập hợp subcommand để sử dụng các chức năng key trong công cụ crypto Cosmos SDK, gồm thêm key mới và lưu vào keyring, liệt kê tất cả khóa công khai trong keyring, và xóa key. Ví dụ: người dùng có thể gõ `simd keys add ` để thêm key mới và lưu bản sao đã mã hóa vào keyring, dùng flag `--recover` để khôi phục private key từ seed phrase hoặc flag `--multisig` để nhóm nhiều key lại tạo multisig key. Xem chi tiết đầy đủ về lệnh `add` key [tại đây](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/client/keys/add.go). Để biết thêm về cách dùng `--keyring-backend` cho việc lưu trữ thông tin xác thực key, xem [tài liệu keyring](../../user/run-node/00-keyring.md).
* Lệnh **Server** từ package server Cosmos SDK. Các lệnh này chịu trách nhiệm cung cấp các cơ chế cần thiết để khởi động ứng dụng ABCI CometBFT và cung cấp framework CLI (dựa trên [cobra](https://github.com/spf13/cobra)) cần thiết để bootstrap đầy đủ một ứng dụng. Package cung cấp hai hàm chính: `StartCmd` và `ExportCmd` tạo các lệnh để khởi động ứng dụng và export state tương ứng. Tìm hiểu thêm [tại đây](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/server).
* Lệnh [**Transaction**](#transaction-commands).
* Lệnh [**Query**](#query-commands).

Tiếp theo là ví dụ hàm `rootCmd` từ ứng dụng `simapp`. Nó khởi tạo lệnh gốc, thêm [*persistent* flag](#flags) và hàm `PreRun` chạy trước mỗi lần thực thi, và thêm tất cả subcommand cần thiết.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/simapp/simd/cmd/root_v2.go#L47-L130
```

:::tip
Dùng `EnhanceRootCommand()` từ các tùy chọn AutoCLI để tự động thêm các lệnh được tạo từ các module vào lệnh gốc.
Ngoài ra nó cũng thêm tất cả lệnh module được định nghĩa thủ công (`tx` và `query`).
Đọc thêm về [AutoCLI](https://docs.cosmos.network/main/core/autocli) trong phần chuyên biệt.
:::

`rootCmd` có một hàm gọi là `initAppConfig()` hữu ích cho việc đặt các cấu hình tùy chỉnh của ứng dụng.
Theo mặc định ứng dụng dùng template cấu hình CometBFT từ Cosmos SDK, có thể được ghi đè qua `initAppConfig()`.
Đây là ví dụ code để ghi đè template `app.toml` mặc định:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/simapp/simd/cmd/root_v2.go#L144-L199
```

`initAppConfig()` cũng cho phép ghi đè [cấu hình server](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/server/config/config.go#L231) mặc định của Cosmos SDK. Một ví dụ là cấu hình `min-gas-prices`, xác định mức giá gas tối thiểu mà validator sẵn sàng chấp nhận để xử lý giao dịch. Theo mặc định, Cosmos SDK đặt tham số này thành `""` (chuỗi rỗng), buộc tất cả validator phải chỉnh sửa `app.toml` của họ và đặt giá trị khác rỗng, nếu không node sẽ dừng khi khởi động. Điều này có thể không phải là trải nghiệm tốt nhất cho validator, nên chain developer có thể đặt giá trị `app.toml` mặc định cho validator trong hàm `initAppConfig()` này.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/simapp/simd/cmd/root_v2.go#L164-L180
```

Các subcommand `status` và `keys` ở cấp root phổ biến trên hầu hết ứng dụng và không tương tác với application state. Phần lớn chức năng của ứng dụng - những gì người dùng thực sự có thể *làm* với nó - được bật bởi các lệnh `tx` và `query`.

### Transaction Commands (Lệnh giao dịch)

[Giao dịch](./01-transactions.md) là các đối tượng bọc [`Msg`](../../build/building-modules/02-messages-and-queries.md#messages) kích hoạt các thay đổi trạng thái. Để bật tạo giao dịch qua giao diện CLI, hàm `txCommand` thường được thêm vào `rootCmd`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/simapp/simd/cmd/root_v2.go#L222-L229
```

Hàm `txCommand` này thêm tất cả giao dịch có sẵn cho người dùng cuối của ứng dụng. Thường bao gồm:

* Lệnh **Sign** từ module [`auth`](../../build/modules/auth/README.md) ký các message trong giao dịch. Để bật multisig, thêm lệnh `MultiSign` của module `auth`. Vì mọi giao dịch đều cần chữ ký để hợp lệ, lệnh ký là cần thiết cho mọi ứng dụng.
* Lệnh **Broadcast** từ các công cụ client Cosmos SDK, để phát sóng giao dịch.
* Tất cả [lệnh giao dịch module](../../build/building-modules/09-module-interfaces.md#transaction-commands) mà ứng dụng phụ thuộc vào, lấy bằng hàm `AddTxCommands()` của [basic module manager](../../build/building-modules/01-module-manager.md#basic-manager), hoặc nâng cao bởi [AutoCLI](https://docs.cosmos.network/main/core/autocli).

Đây là ví dụ `txCommand` tổng hợp các subcommand này từ ứng dụng `simapp`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/simapp/simd/cmd/root_v2.go#L270-L292
```

:::tip
Khi dùng AutoCLI để tạo lệnh giao dịch module, `EnhanceRootCommand()` tự động thêm lệnh `tx` của module vào lệnh gốc.
Đọc thêm về [AutoCLI](https://docs.cosmos.network/main/core/autocli) trong phần chuyên biệt.
:::

### Query Commands (Lệnh query)

[**Query**](../../build/building-modules/02-messages-and-queries.md#queries) là các đối tượng cho phép người dùng truy xuất thông tin về application state. Để bật tạo query qua giao diện CLI, hàm `queryCommand` thường được thêm vào `rootCmd`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/simapp/simd/cmd/root_v2.go#L222-L229
```

Hàm `queryCommand` này thêm tất cả query có sẵn cho người dùng cuối của ứng dụng. Thường bao gồm:

* **QueryTx** và/hoặc các lệnh query giao dịch khác từ module `auth` cho phép người dùng tìm giao dịch bằng hash, danh sách tag, hoặc block height. Các query này cho phép người dùng xem giao dịch đã được đưa vào block chưa.
* Lệnh **Account** từ module `auth`, hiển thị trạng thái (ví dụ: số dư tài khoản) của tài khoản theo địa chỉ.
* Lệnh **Validator** từ các công cụ rpc client Cosmos SDK, hiển thị validator set của một height cho trước.
* Lệnh **Block** từ các công cụ RPC client Cosmos SDK, hiển thị dữ liệu block của một height cho trước.
* Tất cả [lệnh query module](../../build/building-modules/09-module-interfaces.md#query-commands) mà ứng dụng phụ thuộc vào, lấy bằng hàm `AddQueryCommands()` của [basic module manager](../../build/building-modules/01-module-manager.md#basic-manager), hoặc nâng cao bởi [AutoCLI](https://docs.cosmos.network/main/core/autocli).

Đây là ví dụ `queryCommand` tổng hợp các subcommand từ ứng dụng `simapp`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/simapp/simd/cmd/root_v2.go#L249-L268
```

:::tip
Khi dùng AutoCLI để tạo lệnh query module, `EnhanceRootCommand()` tự động thêm lệnh `query` của module vào lệnh gốc.
Đọc thêm về [AutoCLI](https://docs.cosmos.network/main/core/autocli) trong phần chuyên biệt.
:::

## Flags

Flag được dùng để sửa đổi lệnh; developer có thể đưa chúng vào file `flags.go` cùng với CLI. Người dùng có thể đưa chúng một cách tường minh vào lệnh hoặc cấu hình trước trong [`app.toml`](../../user/run-node/01-run-node.md#configuring-the-node-using-apptoml-and-configtoml). Các flag được cấu hình trước phổ biến bao gồm `--node` để kết nối đến và `--chain-id` của blockchain mà người dùng muốn tương tác.

Một *persistent* flag (trái với *local* flag) được thêm vào một lệnh sẽ áp dụng cho tất cả các subcommand con: chúng sẽ kế thừa các giá trị được cấu hình cho các flag này. Ngoài ra, tất cả flag có giá trị mặc định khi được thêm vào lệnh; một số bật/tắt tùy chọn nhưng số khác là giá trị rỗng mà người dùng cần ghi đè để tạo lệnh hợp lệ. Một flag có thể được đánh dấu *required* để lỗi tự động được ném ra nếu người dùng không cung cấp giá trị, nhưng cũng có thể xử lý flag thiếu theo cách khác.

Flag được thêm trực tiếp vào lệnh (thường trong [file CLI của module](../../build/building-modules/09-module-interfaces.md#flags) nơi định nghĩa lệnh module) và không có flag nào ngoài persistent flag của `rootCmd` phải được thêm ở cấp ứng dụng. Thông thường ta thêm *persistent* flag cho `--chain-id`, định danh duy nhất của blockchain mà ứng dụng thuộc về, vào lệnh gốc. Việc thêm flag này có thể thực hiện trong hàm `main()`. Thêm flag này hợp lý vì chain ID không nên thay đổi giữa các lệnh trong CLI ứng dụng này.

## Biến môi trường

Mỗi flag được liên kết với biến môi trường có tên tương ứng. Tên của biến môi trường gồm hai phần - `basename` viết hoa theo sau là tên flag. `-` phải được thay bằng `_`. Ví dụ, flag `--node` cho ứng dụng có basename `GAIA` được liên kết với `GAIA_NODE`. Điều này giúp giảm số lượng flag cần nhập cho các thao tác thường xuyên. Ví dụ thay vì:

```shell
gaia --home=./ --node=<node address> --chain-id="testchain-1" --keyring-backend=test tx ... --from=<key name>
```

cách sau sẽ tiện hơn:

```shell
# định nghĩa biến môi trường trong .env, .envrc, v.v.
GAIA_HOME=<path to home>
GAIA_NODE=<node address>
GAIA_CHAIN_ID="testchain-1"
GAIA_KEYRING_BACKEND="test"

# và sau đó chỉ cần dùng
gaia tx ... --from=<key name>
```

## Cấu hình

Điều quan trọng là lệnh gốc của ứng dụng phải dùng thuộc tính `PersistentPreRun()` của cobra command để thực thi lệnh, để tất cả lệnh con đều có quyền truy cập vào các context server và client. Các context này được đặt giá trị mặc định ban đầu và có thể được sửa đổi, phạm vi theo lệnh, trong các hàm `PersistentPreRun()` tương ứng của chúng. Lưu ý rằng `client.Context` thường được điền sẵn các giá trị "mặc định" hữu ích cho tất cả lệnh kế thừa và ghi đè nếu cần.

Đây là ví dụ hàm `PersistentPreRun()` từ `simapp`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/simapp/simd/cmd/root_v2.go#L81-L120
```

Lời gọi `SetCmdClientContextHandler` đọc các persistent flag qua `ReadPersistentCommandFlags`, tạo `client.Context` và đặt nó vào `Context` của lệnh gốc.

Lời gọi `InterceptConfigsPreRunHandler` tạo viper literal, `server.Context` mặc định và logger, đặt chúng vào `Context` của lệnh gốc. `server.Context` sẽ được sửa đổi và lưu vào disk. Hàm nội bộ `interceptConfigs` đọc hoặc tạo cấu hình CometBFT dựa trên home path được cung cấp. Ngoài ra, `interceptConfigs` cũng đọc và tải cấu hình ứng dụng `app.toml`, và bind vào viper literal của `server.Context`. Điều này quan trọng để ứng dụng có thể truy cập không chỉ CLI flags mà còn các giá trị cấu hình ứng dụng từ file này.

:::tip
Khi muốn cấu hình logger được sử dụng, không dùng `InterceptConfigsPreRunHandler` (đặt logger mặc định của SDK), mà dùng `InterceptConfigsAndCreateContext` và đặt server context cùng logger thủ công:

```diff
-return server.InterceptConfigsPreRunHandler(cmd, customAppTemplate, customAppConfig, customCMTConfig)

+serverCtx, err := server.InterceptConfigsAndCreateContext(cmd, customAppTemplate, customAppConfig, customCMTConfig)
+if err != nil {
+	return err
+}
+
+// ghi đè logger server mặc định
+logger, err := server.CreateSDKLogger(serverCtx, cmd.OutOrStdout())
+if err != nil {
+	return err
+}
+serverCtx.Logger = logger.With(log.ModuleKey, "server")
+
+// đặt server context
+return server.SetCmdServerContext(cmd, serverCtx)
```

:::
