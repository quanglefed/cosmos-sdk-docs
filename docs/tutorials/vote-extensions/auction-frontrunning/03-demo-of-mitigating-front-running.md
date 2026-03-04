# Demo Giảm Thiểu Front-Running Với Vote Extensions

Mục đích của demo này là kiểm tra việc triển khai `VoteExtensionHandler` và `PrepareProposalHandler` mà chúng ta vừa thêm vào codebase. Các handler này được thiết kế để giảm thiểu front-running bằng cách đảm bảo tất cả validator có cùng góc nhìn nhất quán về mempool khi chuẩn bị proposal.

Trong demo này, chúng ta đang sử dụng mạng lưới 3 validator. Validator Beacon đặc biệt vì nó có một custom transaction provider được bật. Điều này có nghĩa là nó có thể thao túng thứ tự giao dịch trong một proposal để có lợi (tức là front-running).

1. Khởi động mạng lưới validator: Bước này thiết lập mạng lưới với 3 validator. Script `./scripts/configure.sh` được dùng để cấu hình mạng lưới và các validator.

```shell
cd scripts
configure.sh
```

Nếu điều này không hoạt động, hãy đảm bảo bạn đã chạy `make build` trong thư mục `tutorials/nameservice/base`.

<!-- nolint:all -->
2. Cho alice thử đặt chỗ `bob.cosmos`: Đây là một giao dịch thông thường mà alice muốn thực hiện. Script `./scripts/reserve.sh "bob.cosmos"` được dùng để gửi giao dịch này.

```shell
reserve.sh "bob.cosmos"
```
<!-- //nolint:all -->
3. Truy vấn để xác minh tên đã được đặt chỗ: Bước này để kiểm tra kết quả giao dịch. Script `./scripts/whois.sh "bob.cosmos"` được dùng để truy vấn trạng thái blockchain.

```shell
whois.sh "bob.cosmos"
```

Kết quả sẽ trả về:

```{
  "name":  {
    "name":  "bob.cosmos",
    "owner":  "cosmos1nq9wuvuju4jdmpmzvxmg8zhhu2ma2y2l2pnu6w",
    "resolve_address":  "cosmos1h6zy2kn9efxtw5z22rc5k9qu7twl70z24kr3ht",
    "amount":  [
      {
        "denom":  "uatom",
        "amount":  "1000"
      }
    ]
  }
}
```

Để phát hiện các nỗ lực front-running của beacon, hãy kiểm tra kỹ log trong giai đoạn `ProcessProposal`. Mở log của mỗi validator, bao gồm beacon, `val1` và `val2`, để quan sát hành vi sau. Mở file log của validator node. Vị trí của file này có thể thay đổi tùy theo cài đặt của bạn, nhưng thường nằm trong thư mục như `$HOME/cosmos/nodes/#{validator}/logs`. Thư mục trong trường hợp này sẽ nằm dưới validator, vậy là `beacon`, `val1` hoặc `val2`. Chạy lệnh sau để theo dõi log của validator hoặc beacon:

```shell
tail -f $HOME/cosmos/nodes/#{validator}/logs
```

```shell
2:47PM ERR ❌️:: Detected invalid proposal bid :: name:"bob.cosmos" resolveAddress:"cosmos1wmuwv38pdur63zw04t0c78r2a8dyt08hf9tpvd" owner:"cosmos1wmuwv38pdur63zw04t0c78r2a8dyt08hf9tpvd" amount:<denom:"uatom" amount:"2000" >  module=server
2:47PM ERR ❌️:: Unable to validate bids in Process Proposal :: <nil> module=server
2:47PM ERR prevote step: state machine rejected a proposed block; this should not happen:the proposer may be misbehaving; prevoting nil err=null height=142 module=consensus round=0
```

<!-- //nolint:all -->
4. Liệt kê các khóa của Beacon: Bước này để xác minh địa chỉ của các validator. Script `./scripts/list-beacon-keys.sh` được dùng để liệt kê các khóa.

```shell
list-beacon-keys.sh
```

Chúng ta sẽ nhận được kết quả tương tự như sau:

```shell
[
  {
    "name": "alice",
    "type": "local",
    "address": "cosmos1h6zy2kn9efxtw5z22rc5k9qu7twl70z24kr3ht",
    "pubkey": "{\"@type\":\"/cosmos.crypto.secp256k1.PubKey\",\"key\":\"A32cvBUkNJz+h2vld4A5BxvU5Rd+HyqpR3aGtvEhlm4C\"}"
  },
  {
    "name": "barbara",
    "type": "local",
    "address": "cosmos1nq9wuvuju4jdmpmzvxmg8zhhu2ma2y2l2pnu6w",
    "pubkey": "{\"@type\":\"/cosmos.crypto.secp256k1.PubKey\",\"key\":\"Ag9PFsNyTQPoJdbyCWia5rZH9CrvSrjMsk7Oz4L3rXQ5\"}"
  },
  {
    "name": "beacon-key",
    "type": "local",
    "address": "cosmos1ez9a6x7lz4gvn27zr368muw8jeyas7sv84lfup",
    "pubkey": "{\"@type\":\"/cosmos.crypto.secp256k1.PubKey\",\"key\":\"AlzJZMWyN7lass710TnAhyuFKAFIaANJyw5ad5P2kpcH\"}"
  },
  {
    "name": "cindy",
    "type": "local",
    "address": "cosmos1m5j6za9w4qc2c5ljzxmm2v7a63mhjeag34pa3g",
    "pubkey": "{\"@type\":\"/cosmos.crypto.secp256k1.PubKey\",\"key\":\"A6F1/3yot5OpyXoSkBbkyl+3rqBkxzRVSJfvSpm/AvW5\"}"
  }
]
```

Điều này cho phép chúng ta khớp các địa chỉ và thấy rằng bid không bị front-run bởi beacon, vì địa chỉ resolve là địa chỉ của Alice chứ không phải địa chỉ của beacon.

Bằng cách chạy demo này, chúng ta có thể xác minh rằng `VoteExtensionHandler` và `PrepareProposalHandler` đang hoạt động như mong đợi và có khả năng ngăn chặn front-running.

## Kết Luận

Trong tutorial này, chúng ta đã giải quyết front-running và MEV, tập trung vào điểm dễ bị tổn thương của đấu giá nameservice với những vấn đề này. Chúng ta đã khám phá vote extension — một tính năng quan trọng của ABCI 2.0 — và tích hợp chúng vào một ứng dụng Cosmos SDK.

Qua các bài tập thực tế, bạn đã triển khai vote extension và kiểm tra hiệu quả của chúng trong việc tạo ra một hệ thống đấu giá công bằng. Bạn đã có được những hiểu biết thực tế bằng cách cấu hình mạng lưới validator và phân tích log blockchain.

Hãy tiếp tục thử nghiệm với các khái niệm này, tham gia cộng đồng và cập nhật các tiến bộ mới. Kiến thức bạn đã có được ở đây rất quan trọng để phát triển các ứng dụng blockchain bảo mật và công bằng.
