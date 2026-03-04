---
sidebar_position: 1
---

# Chạy Testnet

:::note Tóm tắt
Lệnh con `simd testnet` giúp dễ dàng khởi tạo và khởi động một mạng thử nghiệm mô phỏng cho mục đích kiểm thử.
:::

Ngoài các lệnh để [chạy một node](./01-run-node.md), binary `simd` còn bao gồm lệnh `testnet` cho phép bạn khởi động một mạng thử nghiệm mô phỏng trong tiến trình (in-process) hoặc khởi tạo các file cho một mạng thử nghiệm mô phỏng chạy trong tiến trình riêng.

## Khởi Tạo Files

Trước tiên, hãy xem lệnh con `init-files`.

Lệnh này tương tự như lệnh `init` khi khởi tạo một node đơn, nhưng trong trường hợp này chúng ta đang khởi tạo nhiều node, tạo các genesis transaction cho từng node, và sau đó thu thập các transaction đó.

Lệnh con `init-files` khởi tạo các file cần thiết để chạy một mạng thử nghiệm trong một tiến trình riêng (tức là sử dụng Docker container). Chạy lệnh này không phải là điều kiện tiên quyết cho lệnh con `start` ([xem bên dưới](#start-testnet)).

Để khởi tạo các file cho một mạng thử nghiệm, chạy lệnh sau:

```bash
simd testnet init-files
```

Bạn sẽ thấy kết quả sau trong terminal:

```bash
Successfully initialized 4 node directories
```

Thư mục output mặc định là thư mục `.testnets` tương đối. Hãy xem các file được tạo trong thư mục `.testnets`.

### gentxs

Thư mục `gentxs` bao gồm một genesis transaction cho từng node validator. Mỗi file chứa một genesis transaction được mã hóa JSON dùng để đăng ký một node validator tại thời điểm genesis. Các genesis transaction được thêm vào file `genesis.json` trong mỗi thư mục node trong quá trình khởi tạo.

### nodes

Một thư mục node được tạo cho mỗi node validator. Trong mỗi thư mục node là một thư mục `simd`. Thư mục `simd` là thư mục home của mỗi node, chứa các file cấu hình và dữ liệu cho node đó (tức là các file tương tự được bao gồm trong thư mục mặc định `~/.simapp` khi chạy một node đơn).

## Khởi Động Testnet

Bây giờ, hãy xem lệnh con `start`.

Lệnh con `start` vừa khởi tạo vừa khởi động một mạng thử nghiệm trong tiến trình. Đây là cách nhanh nhất để khởi động một mạng thử nghiệm cục bộ cho mục đích kiểm thử.

Bạn có thể khởi động mạng thử nghiệm cục bộ bằng cách chạy lệnh sau:

```bash
simd testnet start
```

Bạn sẽ thấy kết quả tương tự như sau:

```bash
acquiring test network lock
preparing test network with chain-id "chain-mtoD9v"


+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
++       CỤMC TỪ GỢI NHỚ NÀY CHỈ DÙNG CHO MỤC ĐÍCH THỬ NGHIỆM        ++
++                KHÔNG SỬ DỤNG TRONG MÔI TRƯỜNG SẢN XUẤT                 ++
++                                                         ++
++  sustain know debris minute gate hybrid stereo custom   ++
++  divorce cross spoon machine latin vibrant term oblige  ++
++   moment beauty laundry repeat grab game bronze truly   ++
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++


starting test network...
started test network
press the Enter Key to terminate
```

Node validator đầu tiên hiện đang chạy trong tiến trình, có nghĩa là mạng thử nghiệm sẽ kết thúc khi bạn đóng cửa sổ terminal hoặc nhấn phím Enter. Trong output, cụm từ gợi nhớ (mnemonic) cho node validator đầu tiên được cung cấp cho mục đích kiểm thử. Node validator đang sử dụng cùng các địa chỉ mặc định được sử dụng khi khởi tạo và khởi động một node đơn (không cần cung cấp cờ `--node`).

Kiểm tra trạng thái của node validator đầu tiên:

```shell
simd status
```

Import khóa từ cụm từ gợi nhớ được cung cấp:

```shell
simd keys add test --recover --keyring-backend test
```

Kiểm tra số dư của địa chỉ tài khoản:

```shell
simd q bank balances [address]
```

Sử dụng tài khoản thử nghiệm này để kiểm thử thủ công với mạng thử nghiệm.

## Tùy Chọn Testnet

Bạn có thể tùy chỉnh cấu hình của mạng thử nghiệm bằng các cờ (flag). Để xem tất cả tùy chọn cờ, hãy thêm cờ `--help` vào sau mỗi lệnh.
