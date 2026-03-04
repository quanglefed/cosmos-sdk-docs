---
sidebar_position: 1
---

# Chạy Một Node

:::note Tóm tắt
Bây giờ ứng dụng đã sẵn sàng và keyring đã được điền, đã đến lúc xem cách chạy blockchain node. Trong phần này, ứng dụng chúng ta đang chạy được gọi là [`simapp`](https://github.com/cosmos/cosmos-sdk/tree/main/simapp), và binary CLI tương ứng là `simd`.
:::

:::note Đọc Trước

* [Cấu trúc của một ứng dụng Cosmos SDK](../../learn/beginner/00-app-anatomy.md)
* [Thiết lập keyring](./00-keyring.md)

:::

## Khởi Tạo Chain

:::warning
Hãy đảm bảo bạn có thể build binary của riêng mình, và thay `simd` bằng tên binary của bạn trong các đoạn code.
:::

Trước khi thực sự chạy node, chúng ta cần khởi tạo chain, và quan trọng nhất là genesis file của nó. Điều này được thực hiện bằng lệnh con `init`:

```bash
# Đối số <moniker> là tên người dùng tùy chỉnh của node, nên dễ đọc.
simd init <moniker> --chain-id my-test-chain
```

Lệnh trên tạo ra tất cả các file cấu hình cần thiết để node của bạn chạy, cũng như một genesis file mặc định, định nghĩa trạng thái ban đầu của mạng lưới.

:::tip
Tất cả các file cấu hình này nằm trong `~/.simapp` theo mặc định, nhưng bạn có thể ghi đè vị trí của thư mục này bằng cách truyền flag `--home` vào mỗi lệnh, hoặc đặt biến môi trường `$APPD_HOME` (trong đó `APPD` là tên của binary).
:::

Thư mục `~/.simapp` có cấu trúc như sau:

```bash
.                                   # ~/.simapp
  |- data                           # Chứa các database được node sử dụng.
  |- config/
      |- app.toml                   # File cấu hình liên quan đến ứng dụng.
      |- config.toml                # File cấu hình liên quan đến CometBFT.
      |- genesis.json               # Genesis file.
      |- node_key.json              # Khóa private để xác thực node trong giao thức p2p.
      |- priv_validator_key.json    # Khóa private để dùng làm validator trong giao thức đồng thuận.
```

## Cập Nhật Một Số Cài Đặt Mặc Định

Nếu bạn muốn thay đổi bất kỳ giá trị trường nào trong file cấu hình (ví dụ: genesis.json), bạn có thể dùng lệnh `jq` ([cài đặt](https://stedolan.github.io/jq/download/) & [tài liệu](https://stedolan.github.io/jq/manual/#Assignment)) & `sed` để làm điều đó. Một vài ví dụ được liệt kê ở đây.

```bash
# để thay đổi chain-id
jq '.chain_id = "testing"' genesis.json > temp.json && mv temp.json genesis.json

# để bật api server
sed -i '/\[api\]/,+3 s/enable = false/enable = true/' app.toml

# để thay đổi voting_period
jq '.app_state.gov.voting_params.voting_period = "600s"' genesis.json > temp.json && mv temp.json genesis.json

# để thay đổi inflation
jq '.app_state.mint.minter.inflation = "0.300000000000000000"' genesis.json > temp.json && mv temp.json genesis.json
```

### Tương Tác Client

Khi khởi tạo một node, GRPC và REST mặc định là localhost để tránh việc node của bạn bị lộ ra công cộng. Khuyến nghị không nên để lộ các endpoint này mà không có một proxy xử lý cân bằng tải hoặc xác thực được thiết lập giữa node và công cộng.

:::tip
Một công cụ thường được dùng cho mục đích này là [nginx](https://nginx.org).
:::

## Thêm Tài Khoản Genesis

Trước khi khởi động chain, bạn cần điền trạng thái với ít nhất một tài khoản. Để làm vậy, trước tiên [tạo một tài khoản mới trong keyring](./00-keyring.md#adding-keys-to-the-keyring) tên là `my_validator` trong backend keyring `test` (bạn có thể chọn tên khác và backend khác).

Bây giờ bạn đã tạo một tài khoản cục bộ, hãy cấp cho nó một số token `stake` trong genesis file của chain. Làm như vậy cũng đảm bảo rằng chain biết đến sự tồn tại của tài khoản này:

```bash
simd genesis add-genesis-account $MY_VALIDATOR_ADDRESS 100000000000stake
```

Nhớ rằng `$MY_VALIDATOR_ADDRESS` là biến lưu địa chỉ của khóa `my_validator` trong [keyring](./00-keyring.md#adding-keys-to-the-keyring). Cũng lưu ý rằng các token trong Cosmos SDK có định dạng `{amount}{denom}`: `amount` là số thập phân có độ chính xác 18 chữ số, và `denom` là định danh token duy nhất với khóa mệnh giá (ví dụ: `atom` hoặc `uatom`). Ở đây, chúng ta đang cấp token `stake`, vì `stake` là định danh token được dùng để staking trong [`simapp`](https://github.com/cosmos/cosmos-sdk/tree/main/simapp). Với chain của riêng bạn có denom staking riêng, định danh token đó nên được sử dụng thay thế.

Bây giờ tài khoản của bạn đã có một số token, bạn cần thêm một validator vào chain. Validator là các full-node đặc biệt tham gia vào quá trình đồng thuận (được triển khai trong [consensus engine bên dưới](../../learn/intro/02-sdk-app-architecture.md#cometbft)) để thêm các block mới vào chain. Bất kỳ tài khoản nào cũng có thể tuyên bố ý định trở thành validator operator, nhưng chỉ những người có đủ delegation mới được vào tập hợp hoạt động (ví dụ, chỉ 125 ứng viên validator hàng đầu có nhiều delegation nhất được làm validator ở Cosmos Hub). Đối với hướng dẫn này, bạn sẽ thêm node cục bộ của mình (được tạo qua lệnh `init` ở trên) làm validator của chain. Validator có thể được khai báo trước khi chain được khởi động lần đầu qua một giao dịch đặc biệt được đưa vào genesis file gọi là `gentx`:

```bash
# Tạo một gentx.
simd genesis gentx my_validator 100000000stake --chain-id my-test-chain --keyring-backend test

# Thêm gentx vào genesis file.
simd genesis collect-gentxs
```

Một `gentx` làm ba việc:

1. Đăng ký tài khoản `validator` bạn đã tạo làm tài khoản validator operator (tức là tài khoản kiểm soát validator).
2. Tự ủy thác `amount` token staking được cung cấp.
3. Liên kết tài khoản operator với CometBFT node pubkey sẽ được dùng để ký block. Nếu không có flag `--pubkey` nào được cung cấp, nó mặc định là pubkey của node cục bộ được tạo qua lệnh `simd init` ở trên.

Để biết thêm thông tin về `gentx`, sử dụng lệnh sau:

```bash
simd genesis gentx --help
```

## Cấu Hình Node Sử Dụng `app.toml` và `config.toml`

Cosmos SDK tự động tạo ra hai file cấu hình bên trong `~/.simapp/config`:

* `config.toml`: dùng để cấu hình CometBFT, tìm hiểu thêm trên [tài liệu của CometBFT](https://docs.cometbft.com/v0.37/core/configuration),
* `app.toml`: được tạo bởi Cosmos SDK, dùng để cấu hình ứng dụng của bạn, chẳng hạn như chiến lược pruning trạng thái, telemetry, cấu hình gRPC và REST server, state sync...

Cả hai file đều có nhiều comment, vui lòng tham khảo trực tiếp chúng để điều chỉnh node của bạn.

Một ví dụ cấu hình cần điều chỉnh là trường `minimum-gas-prices` bên trong `app.toml`, định nghĩa mức giá gas tối thiểu mà validator node sẵn sàng chấp nhận để xử lý giao dịch. Tùy thuộc vào chain, nó có thể là chuỗi rỗng hoặc không. Nếu nó rỗng, hãy đảm bảo chỉnh sửa trường với một giá trị nào đó, ví dụ `10token`, nếu không node sẽ dừng khi khởi động. Để phục vụ mục đích của tutorial này, hãy đặt giá gas tối thiểu là 0:

```toml
 # Giá gas tối thiểu mà validator sẵn sàng chấp nhận để xử lý giao dịch.
 # Phí của giao dịch phải đáp ứng mức tối thiểu của bất kỳ mệnh giá nào
 # được chỉ định trong cấu hình này (vd: 0.25token1;0.0001token2).
 minimum-gas-prices = "0stake"
```

:::tip
Khi chạy một node (không phải validator!) và không muốn chạy application mempool, đặt trường `max-txs` thành `-1`.

```toml
[mempool]
# Đặt max-txs thành 0 sẽ cho phép số lượng giao dịch không giới hạn trong mempool.
# Đặt max_txs thành âm 1 (-1) sẽ vô hiệu hóa việc chèn giao dịch vào mempool.
# Đặt max_txs thành số dương (> 0) sẽ giới hạn số lượng giao dịch trong mempool theo số lượng được chỉ định.
#
# Lưu ý, cấu hình này chỉ áp dụng cho các triển khai mempool phía ứng dụng
# được tích hợp sẵn trong SDK.
max-txs = "-1"
```

:::

## Chạy Localnet

Bây giờ mọi thứ đã được thiết lập, bạn cuối cùng có thể khởi động node:

```bash
simd start
```

Bạn sẽ thấy các block xuất hiện.

Lệnh trên cho phép bạn chạy một node đơn. Điều này đủ cho phần tiếp theo về tương tác với node này, nhưng bạn có thể muốn chạy nhiều node cùng lúc và xem đồng thuận diễn ra giữa chúng như thế nào.

Cách đơn giản nhất là chạy các lệnh tương tự trong các cửa sổ terminal riêng biệt. Điều này có thể thực hiện được, tuy nhiên, trong Cosmos SDK, chúng ta tận dụng sức mạnh của [Docker Compose](https://docs.docker.com/compose/) để chạy một localnet. Nếu bạn cần nguồn cảm hứng về cách thiết lập localnet của riêng mình với Docker Compose, bạn có thể xem [`docker-compose.yml`](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/docker-compose.yml) của Cosmos SDK.

### Ứng Dụng Độc Lập/CometBFT

Theo mặc định, Cosmos SDK chạy CometBFT trong cùng tiến trình với ứng dụng. Nếu bạn muốn chạy ứng dụng và CometBFT trong các tiến trình riêng biệt, hãy khởi động ứng dụng với flag `--with-comet=false` và đặt `rpc.laddr` trong `config.toml` thành địa chỉ RPC của CometBFT node.

## Ghi Log

Ghi log cung cấp cách để xem những gì đang xảy ra với một node. Mức log mặc định là info. Đây là mức toàn cục và tất cả log info sẽ được xuất ra terminal. Nếu bạn muốn lọc các log cụ thể ra terminal thay vì tất cả, thì việc đặt `module:log_level` là cách để làm điều đó.

Ví dụ:

Trong config.toml:

```toml
log_level: "state:info,p2p:info,consensus:info,x/staking:info,x/ibc:info,*error"
```

## State Sync

State sync là hành động mà một node đồng bộ trạng thái mới nhất hoặc gần mới nhất của blockchain. Điều này hữu ích cho người dùng không muốn đồng bộ tất cả các block trong lịch sử. Đọc thêm trong [tài liệu CometBFT](https://docs.cometbft.com/v0.37/core/state-sync).

State sync hoạt động nhờ snapshot. Đọc cách SDK xử lý snapshot [tại đây](https://github.com/cosmos/cosmos-sdk/blob/825245d/store/snapshots/README.md).

### State Sync Cục Bộ

State sync cục bộ hoạt động tương tự như state sync thông thường ngoại trừ nó hoạt động từ snapshot cục bộ của trạng thái thay vì snapshot được cung cấp qua mạng p2p. Các bước để bắt đầu state sync cục bộ tương tự như state sync thông thường với một số thiết kế khác nhau.

1. Như đã đề cập trong https://docs.cometbft.com/v0.37/core/state-sync, bạn phải đặt chiều cao và hash trong config.toml cùng với một vài rpc server (link đã đề cập có hướng dẫn cách làm điều này).
2. Chạy `<appd snapshot restore <height> <format>` để khôi phục snapshot cục bộ (lưu ý: trước tiên tải nó từ file bằng lệnh *load*).
3. Bootstrapping Comet state để khởi động node sau khi snapshot đã được nhập. Điều này có thể được thực hiện với lệnh bootstrap `<app> comet bootstrap-state`.

### Lệnh Snapshots

Cosmos SDK cung cấp các lệnh để quản lý snapshot. Các lệnh này có thể được thêm vào ứng dụng với đoạn code sau trong `cmd/<app>/root.go`:

```go
import (
  "github.com/cosmos/cosmos-sdk/client/snapshot"
)

func initRootCmd(/* ... */) {
  // ...
  rootCmd.AddCommand(
    snapshot.Cmd(appCreator),
  )
}
```

Sau đó các lệnh sau có sẵn tại `<appd> snapshots [command]`:

* **list**: liệt kê các snapshot cục bộ
* **load**: Tải một file snapshot vào snapshot store
* **restore**: Khôi phục trạng thái ứng dụng từ snapshot cục bộ
* **export**: Xuất trạng thái ứng dụng vào snapshot store
* **dump**: Dump snapshot dưới dạng định dạng lưu trữ portable
* **delete**: Xóa một snapshot cục bộ
