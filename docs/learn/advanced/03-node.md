---
sidebar_position: 1
---

# Node Client (Daemon)

:::note Tóm tắt
Điểm cuối chính của một ứng dụng Cosmos SDK là daemon client, còn được gọi là full-node client. Full-node chạy state-machine, bắt đầu từ một genesis file. Nó kết nối với các peer chạy cùng client để nhận và chuyển tiếp các giao dịch, block proposal và chữ ký. Full-node được cấu thành từ ứng dụng (được định nghĩa bằng Cosmos SDK) và một consensus engine kết nối với ứng dụng thông qua ABCI.
:::

:::note Tài liệu cần đọc trước

* [Cấu trúc của một ứng dụng SDK](../beginner/00-app-anatomy.md)

:::

## Hàm `main`

Full-node client của bất kỳ ứng dụng Cosmos SDK nào được xây dựng bằng cách chạy hàm `main`. Client thường được đặt tên bằng cách thêm hậu tố `-d` vào tên ứng dụng (ví dụ: `appd` cho ứng dụng tên là `app`), và hàm `main` được định nghĩa trong file `./appd/cmd/main.go`. Chạy hàm này tạo ra một executable `appd` đi kèm với một tập hợp các lệnh. Đối với ứng dụng tên là `app`, lệnh chính là [`appd start`](#lệnh-start), để khởi động full-node.

Nhìn chung, các nhà phát triển sẽ triển khai hàm `main.go` theo cấu trúc sau:

* Đầu tiên, một [`encodingCodec`](./05-encoding.md) được khởi tạo cho ứng dụng.
* Sau đó, `config` được lấy ra và các tham số cấu hình được đặt. Điều này chủ yếu liên quan đến việc đặt các tiền tố Bech32 cho [địa chỉ](../beginner/03-accounts.md#addresses).

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/types/config.go#L14-L29
```

* Sử dụng [cobra](https://github.com/spf13/cobra), lệnh gốc (root command) của full-node client được tạo. Sau đó, tất cả các lệnh tùy chỉnh của ứng dụng được thêm vào bằng phương thức `AddCommand()` của `rootCmd`.
* Thêm các lệnh server mặc định vào `rootCmd` bằng phương thức `server.AddCommands()`. Các lệnh này tách biệt với các lệnh được thêm ở trên vì chúng là tiêu chuẩn và được định nghĩa ở cấp Cosmos SDK. Chúng nên được chia sẻ bởi tất cả các ứng dụng dựa trên Cosmos SDK. Chúng bao gồm lệnh quan trọng nhất: [lệnh `start`](#lệnh-start).
* Chuẩn bị và thực thi `executor`.

```go reference
https://github.com/cometbft/cometbft/blob/v0.37.0/libs/cli/setup.go#L74-L78
```

Xem ví dụ về hàm `main` từ ứng dụng `simapp`, ứng dụng demo của Cosmos SDK:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/simapp/simd/main.go
```

## Lệnh `start`

Lệnh `start` được định nghĩa trong thư mục `/server` của Cosmos SDK. Nó được thêm vào lệnh gốc của full-node client trong [hàm `main`](#hàm-main) và được người dùng cuối gọi để khởi động node của họ:

```bash
# Đối với ứng dụng ví dụ tên "app", lệnh sau khởi động full-node.
appd start

# Sử dụng simapp của Cosmos SDK, các lệnh sau khởi động simapp node.
simd start
```

Nhắc lại, full-node bao gồm ba lớp khái niệm: lớp mạng, lớp đồng thuận và lớp ứng dụng. Hai lớp đầu thường được gói lại trong một thực thể gọi là consensus engine (CometBFT theo mặc định), trong khi lớp thứ ba là state-machine được định nghĩa với sự hỗ trợ của Cosmos SDK. Hiện tại, Cosmos SDK sử dụng CometBFT làm consensus engine mặc định, nghĩa là lệnh start được triển khai để khởi động một CometBFT node.

Luồng của lệnh `start` khá đơn giản. Đầu tiên, nó lấy `config` từ `context` để mở `db` (một instance [`leveldb`](https://github.com/syndtr/goleveldb) theo mặc định). `db` này chứa trạng thái mới nhất đã biết của ứng dụng (rỗng nếu ứng dụng được khởi động lần đầu).

Với `db`, lệnh `start` tạo một instance mới của ứng dụng bằng hàm `appCreator`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/server/start.go#L1007
```

Lưu ý rằng `appCreator` là một hàm thỏa mãn chữ ký `AppCreator`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/server/types/app.go#L69
```

Trong thực tế, [constructor của ứng dụng](../beginner/00-app-anatomy.md#constructor-function) được truyền vào như `appCreator`.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/simapp/simd/cmd/root_v2.go#L294-L308
```

Sau đó, instance của `app` được dùng để khởi tạo một CometBFT node mới:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/server/start.go#L361-L400
```

CometBFT node có thể được tạo với `app` vì ứng dụng đó thỏa mãn [`abci.Application` interface](https://github.com/cometbft/cometbft/blob/v0.37.0/abci/types/application.go#L9-L35) (khi `app` mở rộng [`baseapp`](./00-baseapp.md)). Trong phương thức `node.New`, CometBFT đảm bảo rằng chiều cao của ứng dụng (tức là số block từ genesis) bằng chiều cao của CometBFT node. Sự chênh lệch giữa hai chiều cao này phải luôn âm hoặc bằng không. Nếu nó âm tuyệt đối, `node.New` sẽ replay lại các block cho đến khi chiều cao của ứng dụng đạt chiều cao của CometBFT node. Cuối cùng, nếu chiều cao của ứng dụng là `0`, CometBFT node sẽ gọi [`InitChain`](./00-baseapp.md#initchain) trên ứng dụng để khởi tạo trạng thái từ genesis file.

Khi CometBFT node đã được khởi tạo và đồng bộ với ứng dụng, node có thể được khởi động:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/server/start.go#L373-L374
```

Khi khởi động, node sẽ bootstrap RPC và P2P server của nó và bắt đầu kết nối với các peer. Trong quá trình bắt tay với các peer, nếu node nhận ra rằng các peer đang đi trước, nó sẽ query tất cả các block theo thứ tự để bắt kịp. Sau đó, nó sẽ chờ các block proposal mới và chữ ký block từ các validator để tiến lên.

## Các lệnh khác

Để khám phá cách khởi động một node và tương tác với nó một cách cụ thể, hãy tham khảo hướng dẫn [Chạy Node, API và CLI](../../user/run-node/01-run-node.md).
