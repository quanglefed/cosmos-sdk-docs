---
sidebar_position: 1
---

# Giao diện Dòng lệnh (CLI)

:::note Tóm tắt
Tài liệu này mô tả cách giao diện dòng lệnh (CLI) hoạt động ở cấp độ cao, cho một [**ứng dụng**](../beginner/00-app-anatomy.md). Tài liệu riêng về triển khai CLI cho một [**module**](../../build/building-modules/00-intro.md) Cosmos SDK có thể tìm thấy [ở đây](../../build/building-modules/09-module-interfaces.md#cli).
:::

## Giao diện Dòng lệnh

### Lệnh mẫu

Không có cách duy nhất để tạo CLI, nhưng các module Cosmos SDK thường dùng [Thư viện Cobra](https://github.com/spf13/cobra). Xây dựng CLI với Cobra đòi hỏi định nghĩa các lệnh, đối số và flag. **Lệnh (Commands)** hiểu các hành động mà người dùng muốn thực hiện, chẳng hạn như `tx` để tạo giao dịch và `query` để query ứng dụng. Mỗi lệnh cũng có thể có các subcommand lồng nhau, cần thiết để đặt tên cho kiểu giao dịch cụ thể. Người dùng cũng cung cấp **Đối số (Arguments)**, chẳng hạn như số tài khoản để gửi coin đến, và **Flag** để sửa đổi các khía cạnh khác nhau của lệnh, chẳng hạn như giá gas hoặc node nào để phát sóng đến.

Đây là ví dụ về lệnh mà người dùng có thể nhập để tương tác với CLI `simd` của simapp để gửi một số token:

```bash
simd tx bank send $MY_VALIDATOR_ADDRESS $RECIPIENT 1000stake --gas auto --gas-prices <gasPrices>
```

Bốn chuỗi đầu tiên xác định lệnh: lệnh gốc `simd`, subcommand `tx`, subcommand `bank` chỉ ra module cần định tuyến đến, và kiểu giao dịch `send`. Hai chuỗi tiếp theo là đối số, và các chuỗi cuối là flag tùy chọn chỉ định phí.

CLI tương tác với một [node](./03-node.md) để xử lý lệnh này. Bản thân interface được định nghĩa trong một file `main.go`.

### Xây dựng CLI

File `main.go` cần có một hàm `main()` tạo một lệnh gốc (root command), mà tất cả các lệnh ứng dụng sẽ được thêm vào đó như các subcommand. Lệnh gốc còn xử lý: thiết lập cấu hình bằng cách đọc các file cấu hình; thêm các flag như `--chain-id`; khởi tạo `codec` bằng cách inject các codec ứng dụng; và thêm subcommand cho tất cả các tương tác người dùng có thể có, bao gồm transaction commands và query commands.

Hàm `main()` cuối cùng tạo một executor và thực thi lệnh gốc. Xem ví dụ về hàm `main()` từ ứng dụng `simapp`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/simapp/simd/main.go#L14-L24
```

## Thêm Lệnh vào CLI

Mọi CLI ứng dụng đầu tiên xây dựng một lệnh gốc, sau đó thêm chức năng bằng cách tổng hợp các subcommand sử dụng `rootCmd.AddCommand()`. Phần lớn khả năng độc đáo của ứng dụng nằm trong các lệnh giao dịch và query, được gọi lần lượt là `TxCmd` và `QueryCmd`.

### Root Command (Lệnh gốc)

Lệnh gốc (gọi là `rootCmd`) là thứ người dùng nhập đầu tiên vào dòng lệnh. Chuỗi dùng để gọi lệnh thường là tên ứng dụng có hậu tố `-d`, ví dụ: `simd` hoặc `gaiad`. Lệnh gốc thường bao gồm các lệnh sau:

* Lệnh **Status** từ các công cụ rpc client Cosmos SDK, in thông tin về trạng thái của Node đã kết nối, bao gồm `NodeInfo`, `SyncInfo` và `ValidatorInfo`.
* Các lệnh **Keys** từ các công cụ client Cosmos SDK, bao gồm các subcommand để thêm key mới vào keyring, liệt kê tất cả khóa công khai, và xóa key.
* Lệnh **Server** từ package server của Cosmos SDK, cung cấp các cơ chế để khởi động ứng dụng ABCI CometBFT.
* Lệnh **Transaction**.
* Lệnh **Query**.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/simapp/simd/cmd/root_v2.go#L47-L130
```

:::tip
Dùng `EnhanceRootCommand()` từ các tùy chọn AutoCLI để tự động thêm các lệnh được tạo từ các module vào lệnh gốc.
:::

`rootCmd` có một hàm gọi là `initAppConfig()` hữu ích cho việc đặt các cấu hình tùy chỉnh của ứng dụng. Theo mặc định ứng dụng dùng template cấu hình CometBFT từ Cosmos SDK, có thể được ghi đè qua `initAppConfig()`.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/simapp/simd/cmd/root_v2.go#L144-L199
```

`initAppConfig()` cũng cho phép ghi đè cấu hình server mặc định. Ví dụ là cấu hình `min-gas-prices`, xác định mức giá gas tối thiểu mà validator sẵn sàng chấp nhận. Theo mặc định, Cosmos SDK đặt tham số này thành chuỗi rỗng, buộc tất cả validator phải thiết lập giá trị khác rỗng trong `app.toml`, nếu không node sẽ dừng khi khởi động.

### Transaction Commands (Lệnh giao dịch)

Giao dịch là các đối tượng bọc `Msg` kích hoạt các thay đổi trạng thái. Hàm `txCommand` thêm tất cả các giao dịch có sẵn, bao gồm lệnh Sign, lệnh Broadcast, và tất cả lệnh giao dịch của module:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/simapp/simd/cmd/root_v2.go#L270-L292
```

### Query Commands (Lệnh query)

Hàm `queryCommand` thêm tất cả các query có sẵn, bao gồm QueryTx, lệnh Account, lệnh Validator, lệnh Block, và tất cả lệnh query của module:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/simapp/simd/cmd/root_v2.go#L249-L268
```

## Flags

Flag được dùng để sửa đổi lệnh. Người dùng có thể đưa chúng một cách tường minh vào lệnh hoặc cấu hình trước trong `app.toml`. Các flag được cấu hình trước phổ biến bao gồm `--node` để kết nối đến và `--chain-id` của blockchain.

Một *persistent* flag được thêm vào một lệnh sẽ áp dụng cho tất cả các subcommand con: chúng sẽ kế thừa các giá trị được cấu hình. Tất cả flag có giá trị mặc định khi được thêm vào lệnh. Một flag có thể được đánh dấu *required* để lỗi tự động được ném ra nếu người dùng không cung cấp giá trị.

Flag được thêm trực tiếp vào lệnh, và không có flag nào ngoài persistent flag của `rootCmd` phải được thêm ở cấp ứng dụng. Thông thường ta thêm persistent flag cho `--chain-id` vào lệnh gốc trong hàm `main()`.

## Biến môi trường

Mỗi flag được liên kết với biến môi trường có tên tương ứng. Tên của biến môi trường gồm `basename` viết hoa theo sau là tên flag, với `-` được thay bằng `_`. Ví dụ, flag `--node` cho ứng dụng có basename `GAIA` được liên kết với `GAIA_NODE`. Điều này giúp giảm số lượng flag cần nhập cho các thao tác thường xuyên.

## Cấu hình

Điều quan trọng là lệnh gốc của ứng dụng phải dùng thuộc tính `PersistentPreRun()` để tất cả lệnh con đều có quyền truy cập vào các context server và client.

Lời gọi `SetCmdClientContextHandler` đọc các persistent flag, tạo một `client.Context` và đặt nó vào `Context` của lệnh gốc. Lời gọi `InterceptConfigsPreRunHandler` tạo viper literal, `server.Context` mặc định và logger, đọc hoặc tạo cấu hình CometBFT, đọc và tải cấu hình ứng dụng `app.toml`.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/simapp/simd/cmd/root_v2.go#L81-L120
```
