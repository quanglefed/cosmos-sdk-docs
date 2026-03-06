# `x/genutil`

## Khái niệm

Gói `genutil` chứa nhiều chức năng tiện ích liên quan đến genesis để dùng trong
ứng dụng blockchain. Cụ thể:

* Các giao dịch genesis (gentx)
* Lệnh thu thập và tạo gentx
* Xử lý gentx trong `InitChain`
* Tạo file genesis
* Xác thực file genesis
* Migration file genesis
* Khởi tạo liên quan CometBFT
  * Chuyển đổi genesis của ứng dụng sang genesis CometBFT

## Genesis

Genutil chứa cấu trúc dữ liệu định nghĩa genesis của ứng dụng.
Genesis của ứng dụng gồm một consensus genesis (ví dụ: genesis CometBFT) và dữ
liệu genesis liên quan ứng dụng.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-rc.0/x/genutil/types/genesis.go#L24-L34
```

Genesis của ứng dụng sau đó có thể được chuyển đổi sang đúng định dạng cho
consensus engine:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-rc.0/x/genutil/types/genesis.go#L126-L136
```

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-rc.0/server/start.go#L397-L407
```

## Client

### CLI

Các lệnh genutil có trong nhóm lệnh con `genesis`.

#### add-genesis-account

Thêm một genesis account vào `genesis.json`. Xem thêm [tại đây](https://docs.cosmos.network/main/run-node/run-node#adding-genesis-accounts).

#### collect-gentxs

Thu thập các genesis tx và xuất ra file `genesis.json`.

```shell
simd genesis collect-gentxs
```

Lệnh này sẽ tạo một file `genesis.json` mới bao gồm dữ liệu từ tất cả validator
đôi khi được gọi là “super genesis file” để phân biệt với genesis file của một
validator đơn lẻ.

#### gentx

Tạo một genesis tx mang theo self-delegation.

```shell
simd genesis gentx [key_name] [amount] --chain-id [chain-id]
```

Lệnh này sẽ tạo genesis transaction cho chain mới. `amount` ở đây nên ít nhất là
`1000000000stake`. Nếu bạn cung cấp quá nhiều hoặc quá ít, bạn sẽ gặp lỗi khi
khởi chạy node.

#### migrate

Migrate genesis sang một phiên bản (SDK) đích.

```shell
simd genesis migrate [target-version]
```

::::tip
Lệnh `migrate` có thể mở rộng và nhận một `MigrationMap`. Map này ánh xạ từ phiên
bản đích sang các hàm migration genesis.
Khi không dùng `MigrationMap` mặc định, khuyến nghị vẫn gọi `MigrationMap` mặc
định tương ứng với phiên bản SDK của chain và prepend/append các migration genesis
của riêng bạn.
::::

#### validate-genesis

Xác thực file genesis tại vị trí mặc định hoặc tại đường dẫn được truyền vào.

```shell
simd genesis validate-genesis
```

::::warning
`validate-genesis` chỉ xác thực genesis có hợp lệ với **binary ứng dụng hiện tại**
hay không. Để xác thực một genesis từ phiên bản ứng dụng trước đó, hãy dùng lệnh
`migrate` để migrate genesis sang phiên bản hiện tại.
::::

