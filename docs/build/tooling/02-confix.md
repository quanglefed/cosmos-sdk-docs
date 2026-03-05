---
sidebar_position: 1
---

# Confix

`Confix` là một công cụ quản lý cấu hình cho phép bạn quản lý cấu hình của mình thông qua CLI.

Nó dựa trên [CometBFT RFC 019](https://github.com/cometbft/cometbft/blob/5013bc3f4a6d64dcc2bf02ccc002ebc9881c62e4/docs/rfc/rfc-019-config-version.md).

## Cài Đặt

### Thêm Config Command

Để thêm công cụ confix, cần thêm `ConfigCommand` vào file root command của ứng dụng (ví dụ: `<appd>/cmd/root.go`).

Import package `confixCmd`:

```go
import "cosmossdk.io/tools/confix/cmd"
```

Tìm dòng sau:

```go
initRootCmd(rootCmd, moduleManager)
```

Sau dòng đó, thêm phần sau:

```go
rootCmd.AddCommand(
    confixcmd.ConfigCommand(),
)
```

Hàm `ConfixCommand` xây dựng lệnh gốc `config` và được định nghĩa trong package `confixCmd` (`cosmossdk.io/tools/confix/cmd`).
Ví dụ triển khai có thể tìm thấy trong `simapp`.

Lệnh sẽ có sẵn là `simd config`.

:::tip
Sử dụng confix trực tiếp trong ứng dụng có thể có ít tính năng hơn so với sử dụng độc lập.
Điều này là vì confix được versioned với SDK, trong khi `latest` là phiên bản độc lập.
:::

### Sử Dụng Confix Độc Lập

Để sử dụng Confix độc lập, mà không cần thêm nó vào ứng dụng của bạn, cài đặt nó với lệnh sau:

```bash
go install cosmossdk.io/tools/confix/cmd/confix@latest
```

Ngoài ra, để build từ source, đơn giản chạy `make confix`. Binary sẽ nằm trong `tools/confix`.

## Sử Dụng

Sử dụng độc lập:

```shell
confix --help
```

Sử dụng trong simd:

```shell
simd config fix --help
```

### Get (Lấy)

Lấy giá trị cấu hình, ví dụ:

```shell
simd config get app pruning # lấy giá trị pruning từ app.toml
simd config get client chain-id # lấy giá trị chain-id từ client.toml
```

```shell
confix get ~/.simapp/config/app.toml pruning # lấy giá trị pruning từ app.toml
confix get ~/.simapp/config/client.toml chain-id # lấy giá trị chain-id từ client.toml
```

### Set (Đặt)

Đặt giá trị cấu hình, ví dụ:

```shell
simd config set app pruning "enabled" # đặt giá trị pruning trong app.toml
simd config set client chain-id "foo-1" # đặt giá trị chain-id trong client.toml
```

```shell
confix set ~/.simapp/config/app.toml pruning "enabled" # đặt giá trị pruning trong app.toml
confix set ~/.simapp/config/client.toml chain-id "foo-1" # đặt giá trị chain-id trong client.toml
```

### Migrate (Di Chuyển)

Di chuyển file cấu hình sang phiên bản mới, loại config mặc định là `app.toml`, nếu bạn muốn thay đổi sang `client.toml`, hãy chỉ ra bằng cách thêm tham số tùy chọn, ví dụ:

```shell
simd config migrate v0.50 # di chuyển defaultHome/config/app.toml sang cấu hình v0.50 mới nhất
simd config migrate v0.50 --client # di chuyển defaultHome/config/client.toml sang cấu hình v0.50 mới nhất
```

```shell
confix migrate v0.50 ~/.simapp/config/app.toml # di chuyển ~/.simapp/config/app.toml sang cấu hình v0.50 mới nhất
confix migrate v0.50 ~/.simapp/config/client.toml --client # di chuyển ~/.simapp/config/client.toml sang cấu hình v0.50 mới nhất
```

### Diff (So Sánh)

Lấy sự khác biệt giữa file cấu hình đã cho và file cấu hình mặc định, ví dụ:

```shell
simd config diff v0.47 # lấy sự khác biệt giữa defaultHome/config/app.toml và cấu hình v0.47 mới nhất
simd config diff v0.47 --client # lấy sự khác biệt giữa defaultHome/config/client.toml và cấu hình v0.47 mới nhất
```

```shell
confix diff v0.47 ~/.simapp/config/app.toml # lấy sự khác biệt giữa ~/.simapp/config/app.toml và cấu hình v0.47 mới nhất
confix diff v0.47 ~/.simapp/config/client.toml --client # lấy sự khác biệt giữa ~/.simapp/config/client.toml và cấu hình v0.47 mới nhất
```

### View (Xem)

Xem file cấu hình, ví dụ:

```shell
simd config view client # xem cấu hình app client hiện tại
```

```shell
confix view ~/.simapp/config/client.toml # xem cấu hình app client hiện tại
```

### Maintainer (Người Bảo Trì)

Tại mỗi lần sửa đổi cấu hình mặc định của SDK, thêm cấu hình SDK mặc định trong `data/vXX-app.toml`.
Điều này cho phép người dùng sử dụng công cụ độc lập.

### Compatibility (Tương Thích)

Phiên bản độc lập được khuyến nghị là `latest`, sử dụng phiên bản phát triển mới nhất của Confix.

| SDK Version | Confix Version |
| ----------- | -------------- |
| v0.50       | v0.1.x         |
| v0.52       | v0.2.x         |
| v2          | v0.2.x         |

## Credits (Ghi Công)

Dự án này dựa trên [CometBFT RFC 019](https://github.com/cometbft/cometbft/blob/5013bc3f4a6d64dcc2bf02ccc002ebc9881c62e4/docs/rfc/rfc-019-config-version.md) và triển khai [confix](https://github.com/cometbft/cometbft/blob/v0.36.x/scripts/confix/confix.go) riêng của họ chưa bao giờ được phát hành.
