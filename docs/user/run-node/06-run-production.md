---
sidebar_position: 1
---

# Chạy Trong Môi Trường Production

:::note Tóm tắt
Phần này mô tả cách chạy an toàn một node trong môi trường công cộng và/hoặc trên mainnet của một trong nhiều blockchain công khai của Cosmos SDK.
:::

Khi vận hành một node, full node hoặc validator, trong production, điều quan trọng là phải thiết lập server của bạn một cách bảo mật.

:::note
Có nhiều cách khác nhau để bảo mật server và node của bạn, các bước được mô tả ở đây chỉ là một cách. Để xem một cách khác để thiết lập server, xem [tutorial chạy trong production](https://tutorials.cosmos.network/hands-on-exercise/4-run-in-prod).
:::

:::note
Hướng dẫn này giả định hệ điều hành bên dưới là Ubuntu.
:::

## Thiết Lập Server

### Người Dùng

Khi tạo server, hầu hết các lần nó được tạo với user `root`. User này có quyền cao hơn trên server. Khi vận hành một node, khuyến nghị không chạy node của bạn với tư cách user root.

1. Tạo user mới

```bash
sudo adduser change_me
```

2. Chúng ta muốn cho phép user này thực hiện các tác vụ sudo

```bash
sudo usermod -aG sudo change_me
```

Bây giờ khi đăng nhập vào server, có thể sử dụng user không phải `root`.

### Go

1. Cài đặt phiên bản [Go](https://go.dev/doc/install) được ứng dụng khuyến nghị.

:::warning
Trong quá khứ, các validator [đã gặp vấn đề](https://github.com/cosmos/cosmos-sdk/issues/13976) khi sử dụng các phiên bản Go khác nhau. Khuyến nghị toàn bộ tập hợp validator sử dụng phiên bản Go được ứng dụng khuyến nghị.
:::

### Firewall

Các node không nên có tất cả các cổng mở ra công cộng — đây là cách đơn giản để bị DDOS. Thứ hai, [CometBFT](https://github.com/cometbft/cometbft) khuyến nghị không bao giờ expose các cổng không cần thiết để vận hành node.

Khi thiết lập firewall, có một số cổng có thể mở khi vận hành Cosmos SDK node. Có CometBFT json-RPC, prometheus, p2p, remote signer và Cosmos SDK GRPC và REST. Nếu node được vận hành như một node không cung cấp endpoint để gửi hoặc truy vấn, thì tối đa ba endpoint là cần thiết.

Hầu hết, nếu không phải tất cả server đều được trang bị [ufw](https://help.ubuntu.com/community/UFW). Ufw sẽ được dùng trong tutorial này.

1. Reset UFW để không cho phép tất cả kết nối đến và cho phép kết nối đi

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

2. Hãy đảm bảo rằng cổng 22 (ssh) vẫn mở.

```bash
sudo ufw allow ssh
```

hoặc

```bash
sudo ufw allow 22
```

Cả hai lệnh trên đều tương đương nhau.

3. Cho phép Cổng 26656 (cometbft p2p port). Nếu node có p2p port đã được sửa đổi thì phải sử dụng cổng đó ở đây.

```bash
sudo ufw allow 26656/tcp
```

4. Cho phép cổng 26660 (cometbft [prometheus](https://prometheus.io)). Đây cũng là cổng theo dõi của ứng dụng.

```bash
sudo ufw allow 26660/tcp
```

5. NẾU node được thiết lập muốn expose jsonRPC của CometBFT và GRPC và REST của Cosmos SDK thì làm theo bước này. (Tùy chọn)

##### CometBFT JsonRPC

```bash
sudo ufw allow 26657/tcp
```

##### Cosmos SDK GRPC

```bash
sudo ufw allow 9090/tcp
```

##### Cosmos SDK REST

```bash
sudo ufw allow 1317/tcp
```

6. Cuối cùng, bật ufw

```bash
sudo ufw enable
```

### Ký

Nếu node đang được khởi động là validator, có nhiều cách validator có thể ký block.

#### File

Ký dựa trên file là cách tiếp cận đơn giản và mặc định. Cách này hoạt động bằng cách lưu trữ khóa đồng thuận được tạo khi khởi tạo để ký block. Cách này chỉ an toàn như thiết lập server của bạn vì nếu server bị xâm phạm thì khóa của bạn cũng vậy. Khóa này nằm trong thư mục `config/priv_val_key.json` được tạo khi khởi tạo.

Một file thứ hai tồn tại mà người dùng cần biết, file nằm trong thư mục data `data/priv_val_state.json`. File này bảo vệ node của bạn khỏi việc ký đôi. Nó theo dõi chiều cao ký cuối cùng, vòng và chữ ký mới nhất của khóa đồng thuận. Nếu node gặp sự cố và cần được khôi phục, file này phải được giữ lại để đảm bảo rằng khóa đồng thuận sẽ không được sử dụng để ký một block đã được ký trước đó.

#### Remote Signer

Remote signer là một server thứ cấp tách biệt với node đang chạy dùng để ký block bằng khóa đồng thuận. Điều này có nghĩa là khóa đồng thuận không nằm trên chính node. Điều này tăng bảo mật vì full node kết nối với remote signer có thể được thay thế mà không bỏ lỡ block.

Hai remote signer được sử dụng nhiều nhất là [tmkms](https://github.com/iqlusioninc/tmkms) từ [Iqlusion](https://www.iqlusion.io) và [horcrux](https://github.com/strangelove-ventures/horcrux) từ [Strangelove](https://strange.love).

##### TMKMS

###### Các Dependency

1. Cập nhật dependency server và cài đặt thêm những thứ cần thiết.

```sh
sudo apt update -y && sudo apt install build-essential curl jq -y
```

2. Cài đặt Rust:

```sh
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

3. Cài đặt Libusb:

```sh
sudo apt install libusb-1.0-0-dev
```

###### Thiết Lập

Có hai cách để cài đặt tmkms, từ source hoặc `cargo install`. Trong các ví dụ, chúng ta sẽ đề cập đến việc tải xuống hoặc build từ source và sử dụng softsign. Softsign là viết tắt của software signing (ký bằng phần mềm), nhưng bạn có thể sử dụng [yubihsm](https://www.yubico.com/products/hardware-security-module/) làm khóa ký nếu muốn.

1. Build:

Từ source:

```bash
cd $HOME
git clone https://github.com/iqlusioninc/tmkms.git
cd $HOME/tmkms
cargo install tmkms --features=softsign
tmkms init config
tmkms softsign keygen ./config/secrets/secret_connection_key
```

hoặc

Cargo install:

```bash
cargo install tmkms --features=softsign
tmkms init config
tmkms softsign keygen ./config/secrets/secret_connection_key
```

:::note
Để sử dụng tmkms với yubikey, cài đặt binary với `--features=yubihsm`.
:::

2. Di chuyển khóa validator từ full node đến instance tmkms mới.

```bash
scp user@123.456.32.123:~/.simd/config/priv_validator_key.json ~/tmkms/config/secrets
```

3. Import khóa validator vào tmkms.

```bash
tmkms softsign import $HOME/tmkms/config/secrets/priv_validator_key.json $HOME/tmkms/config/secrets/priv_validator_key
```

Tại đây, cần xóa `priv_validator_key.json` khỏi cả validator node và tmkms node. Vì khóa đã được import vào tmkms (ở trên), nó không còn cần thiết trên các node nữa. Khóa có thể được lưu trữ an toàn offline.

4. Sửa đổi `tmkms.toml`.

```bash
vim $HOME/tmkms/config/tmkms.toml
```

Ví dụ này cho thấy cấu hình có thể được dùng cho soft signing. Ví dụ có IP là `123.456.12.345` với cổng `26659` và chain_id là `test-chain-waSDSe`. Đây là các mục phải được sửa đổi cho usecase của tmkms và mạng lưới.

```toml
# File cấu hình CometBFT KMS

## Cấu Hình Chain

[[chain]]
id = "osmosis-1"
key_format = { type = "bech32", account_key_prefix = "cosmospub", consensus_key_prefix = "cosmosvalconspub" }
state_file = "/root/tmkms/config/state/priv_validator_state.json"

## Cấu Hình Signing Provider

### Cấu Hình Software-based Signer

[[providers.softsign]]
chain_ids = ["test-chain-waSDSe"]
key_type = "consensus"
path = "/root/tmkms/config/secrets/priv_validator_key"

## Cấu Hình Validator

[[validator]]
chain_id = "test-chain-waSDSe"
addr = "tcp://123.456.12.345:26659"
secret_key = "/root/tmkms/config/secrets/secret_connection_key"
protocol_version = "v0.34"
reconnect = true
```

5. Đặt địa chỉ của instance tmkms.

```bash
vim $HOME/.simd/config/config.toml

priv_validator_laddr = "tcp://0.0.0.0:26659"
```

:::tip
Địa chỉ trên được đặt thành `0.0.0.0` nhưng khuyến nghị đặt tmkms server để bảo mật khi khởi động.
:::

:::tip
Khuyến nghị comment hoặc xóa các dòng chỉ định đường dẫn của khóa validator và validator:

```toml
# Đường dẫn đến file JSON chứa khóa private để dùng làm validator trong giao thức đồng thuận
# priv_validator_key_file = "config/priv_validator_key.json"

# Đường dẫn đến file JSON chứa trạng thái ký cuối cùng của validator
# priv_validator_state_file = "data/priv_validator_state.json"
```

:::

6. Khởi động hai tiến trình.

```bash
tmkms start -c $HOME/tmkms/config/tmkms.toml
```

```bash
simd start
```
