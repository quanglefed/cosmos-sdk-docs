---
sidebar_position: 1
---

# Tài Khoản

:::note Tóm tắt
Tài liệu này mô tả hệ thống tài khoản và khóa công khai tích hợp sẵn của Cosmos SDK.
:::

:::note Tài liệu cần đọc trước

* [Cấu trúc của một ứng dụng Cosmos SDK](./00-app-anatomy.md)

:::

## Định nghĩa tài khoản

Trong Cosmos SDK, một _tài khoản_ chỉ một cặp _khóa công khai_ `PubKey` và _khóa riêng tư_ `PrivKey`. `PubKey` có thể được suy ra để tạo ra nhiều `Address` khác nhau, được dùng để xác định người dùng (trong số các bên khác) trong ứng dụng. `Address` cũng được liên kết với [`message`](../../build/building-modules/02-messages-and-queries.md#messages) để xác định người gửi `message`. `PrivKey` được dùng để tạo [chữ ký số](#signatures) nhằm chứng minh rằng một `Address` liên kết với `PrivKey` đã phê duyệt một `message` nhất định.

Để dẫn xuất khóa HD (hierarchical deterministic), Cosmos SDK sử dụng một tiêu chuẩn gọi là [BIP32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki). BIP32 cho phép người dùng tạo một HD wallet (như được quy định trong [BIP44](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki)) — một tập hợp các tài khoản được dẫn xuất từ một hạt nhân bí mật (seed) ban đầu. Seed thường được tạo từ một cụm từ ghi nhớ (mnemonic) 12 hoặc 24 từ. Một seed duy nhất có thể dẫn xuất bất kỳ số lượng `PrivKey` nào bằng cách sử dụng hàm mật mã một chiều. Sau đó, `PubKey` có thể được dẫn xuất từ `PrivKey`. Đương nhiên, mnemonic là thông tin nhạy cảm nhất, vì các khóa riêng tư luôn có thể được tái tạo nếu mnemonic được giữ nguyên.

```text
     Tài khoản 0                    Tài khoản 1                    Tài khoản 2

+------------------+            +------------------+           +------------------+
|                  |            |                  |           |                  |
|    Address 0     |            |    Address 1     |           |    Address 2     |
|        ^         |            |        ^         |           |        ^         |
|        |         |            |        |         |           |        |         |
|        |         |            |        |         |           |        |         |
|        |         |            |        |         |           |        |         |
|        +         |            |        +         |           |        +         |
|  Public key 0    |            |  Public key 1    |           |  Public key 2    |
|        ^         |            |        ^         |           |        ^         |
|        |         |            |        |         |           |        |         |
|        |         |            |        |         |           |        |         |
|        |         |            |        |         |           |        |         |
|        +         |            |        +         |           |        +         |
|  Private key 0   |            |  Private key 1   |           |  Private key 2   |
|        ^         |            |        ^         |           |        ^         |
+------------------+            +------------------+           +------------------+
         |                               |                              |
         |                               |                              |
         |                               |                              |
         +--------------------------------------------------------------+
                                         |
                                         |
                               +---------+---------+
                               |                   |
                               |  Master PrivKey   |
                               |                   |
                               +-------------------+
                                         |
                                         |
                               +---------+---------+
                               |                   |
                               |  Mnemonic (Seed)  |
                               |                   |
                               +-------------------+
```

Trong Cosmos SDK, các khóa được lưu trữ và quản lý bằng cách sử dụng một đối tượng gọi là [`Keyring`](#keyring).

## Khóa, tài khoản, địa chỉ và chữ ký

Phương thức xác thực người dùng chính là sử dụng [chữ ký số](https://en.wikipedia.org/wiki/Digital_signature). Người dùng ký giao dịch bằng khóa riêng tư của họ. Xác minh chữ ký được thực hiện bằng khóa công khai liên kết. Để xác minh chữ ký on-chain, chúng ta lưu khóa công khai trong đối tượng `Account` (cùng với các dữ liệu khác cần thiết cho việc xác thực giao dịch đúng cách).

Trong node, tất cả dữ liệu được lưu trữ bằng tuần tự hóa Protocol Buffers.

Cosmos SDK hỗ trợ các sơ đồ khóa số sau để tạo chữ ký số:

* `secp256k1`, được triển khai trong [package `crypto/keys/secp256k1` của Cosmos SDK](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/crypto/keys/secp256k1/secp256k1.go).
* `secp256r1`, được triển khai trong [package `crypto/keys/secp256r1` của Cosmos SDK](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/crypto/keys/secp256r1/pubkey.go).
* `tm-ed25519`, được triển khai trong [package `crypto/keys/ed25519` của Cosmos SDK](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/crypto/keys/ed25519/ed25519.go). Sơ đồ này chỉ được hỗ trợ cho xác thực đồng thuận.

|              | Độ dài địa chỉ (byte) | Độ dài khóa công khai (byte) | Dùng cho xác thực giao dịch | Dùng cho đồng thuận (cometbft) |
| :----------: | :-------------------: | :--------------------------: | :-------------------------: | :-----------------------------: |
| `secp256k1`  |          20           |             33               |            có               |               không             |
| `secp256r1`  |          32           |             33               |            có               |               không             |
| `tm-ed25519` |    -- không dùng --   |             32               |           không             |               có                |

## Địa chỉ

`Address` và `PubKey` đều là thông tin công khai xác định các tác nhân trong ứng dụng. `Account` được dùng để lưu thông tin xác thực. Triển khai tài khoản cơ bản được cung cấp bởi đối tượng `BaseAccount`.

Mỗi tài khoản được xác định bằng `Address` là một chuỗi byte được dẫn xuất từ một khóa công khai. Trong Cosmos SDK, chúng ta định nghĩa 3 loại địa chỉ xác định ngữ cảnh mà tài khoản được sử dụng:

* `AccAddress` xác định người dùng (người gửi của `message`).
* `ValAddress` xác định các operator validator.
* `ConsAddress` xác định các validator node đang tham gia vào đồng thuận. Validator node được dẫn xuất bằng đường cong **`ed25519`**.

Các kiểu này triển khai interface `Address`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/types/address.go#L126-L134
```

Thuật toán xây dựng địa chỉ được định nghĩa trong [ADR-28](https://github.com/cosmos/cosmos-sdk/blob/main/docs/architecture/adr-028-public-key-addresses.md). Đây là cách tiêu chuẩn để lấy địa chỉ tài khoản từ một khóa công khai `pub`:

```go
sdk.AccAddress(pub.Address().Bytes())
```

Đáng lưu ý, cả phương thức `Marshal()` và `Bytes()` đều trả về cùng dạng `[]byte` thô của địa chỉ. `Marshal()` được yêu cầu để tương thích với Protobuf.

Để tương tác với người dùng, địa chỉ được định dạng bằng [Bech32](https://en.bitcoin.it/wiki/Bech32) và được triển khai bởi phương thức `String`. Định dạng Bech32 là định dạng duy nhất được hỗ trợ khi tương tác với blockchain. Phần con người đọc được (human-readable part) của Bech32 (tiền tố Bech32) được dùng để biểu thị loại địa chỉ. Ví dụ:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/types/address.go#L299-L316
```

|                    | Tiền tố Bech32 của địa chỉ |
| ------------------ | -------------------------- |
| Tài khoản          | cosmos                     |
| Operator Validator | cosmosvaloper              |
| Node Consensus     | cosmosvalcons              |

### Khóa công khai

Khóa công khai trong Cosmos SDK được định nghĩa bởi interface `cryptotypes.PubKey`. Vì khóa công khai được lưu trong store, `cryptotypes.PubKey` mở rộng interface `proto.Message`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/crypto/types/types.go#L8-L17
```

Định dạng nén được sử dụng cho tuần tự hóa `secp256k1` và `secp256r1`.

* Byte đầu tiên là `0x02` nếu tọa độ `y` là tọa độ lớn nhất theo thứ tự từ điển trong hai tọa độ liên kết với tọa độ `x`.
* Ngược lại byte đầu tiên là `0x03`.

Tiền tố này được theo sau bởi tọa độ `x`.

Khóa công khai không được dùng để tham chiếu tài khoản (hoặc người dùng) và nói chung không được dùng khi soạn các message giao dịch (với một vài ngoại lệ: `MsgCreateValidator`, `Validator` và message `Multisig`). Để tương tác với người dùng, `PubKey` được định dạng bằng Protobufs JSON (hàm [ProtoMarshalJSON](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/codec/json.go#L14-L34)). Ví dụ:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/client/keys/output.go#L23-L39
```

## Keyring

`Keyring` là một đối tượng lưu trữ và quản lý tài khoản. Trong Cosmos SDK, một triển khai `Keyring` tuân theo interface `Keyring`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/crypto/keyring/keyring.go#L58-L106
```

Triển khai mặc định của `Keyring` đến từ thư viện bên thứ ba [`99designs/keyring`](https://github.com/99designs/keyring).

Một vài ghi chú về các phương thức `Keyring`:

* `Sign(uid string, msg []byte) ([]byte, types.PubKey, error)` xử lý nghiêm ngặt chữ ký của các byte `msg`. Bạn phải chuẩn bị và mã hóa giao dịch thành dạng `[]byte` chuẩn. Vì protobuf không xác định (non-deterministic), trong [ADR-020](../../build/architecture/adr-020-protobuf-transaction-encoding.md) đã quyết định rằng `payload` chuẩn để ký là struct `SignDoc`, được mã hóa xác định bằng [ADR-027](../../build/architecture/adr-027-deterministic-protobuf-serialization.md). Lưu ý rằng xác minh chữ ký không được triển khai mặc định trong Cosmos SDK, nó được ủy thác cho [`anteHandler`](../advanced/00-baseapp.md#antehandler).

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/proto/cosmos/tx/v1beta1/tx.proto#L50-L67
```

* `NewAccount(uid, mnemonic, bip39Passphrase, hdPath string, algo SignatureAlgo) (*Record, error)` tạo một tài khoản mới dựa trên [`bip44 path`](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki) và lưu nó vào đĩa. `PrivKey` **không bao giờ được lưu dạng không mã hóa**, thay vào đó nó được [mã hóa bằng passphrase](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/crypto/armor.go) trước khi được lưu. Trong ngữ cảnh của phương thức này, loại khóa và số sequence đề cập đến phân đoạn của đường dẫn dẫn xuất BIP44 (ví dụ: `0`, `1`, `2`, ...) được dùng để dẫn xuất khóa riêng tư và công khai từ mnemonic. Sử dụng cùng mnemonic và đường dẫn dẫn xuất, cùng `PrivKey`, `PubKey` và `Address` sẽ được tạo ra. Các loại khóa sau được keyring hỗ trợ:

* `secp256k1`
* `ed25519`

* `ExportPrivKeyArmor(uid, encryptPassphrase string) (armor string, err error)` xuất khóa riêng tư ở định dạng mã hóa ASCII-armored bằng passphrase đã cho. Sau đó bạn có thể import khóa riêng tư lại vào keyring bằng hàm `ImportPrivKey(uid, armor, passphrase string)` hoặc giải mã nó thành khóa riêng tư thô bằng hàm `UnarmorDecryptPrivKey(armorStr string, passphrase string)`.

### Tạo loại khóa mới

Để tạo một loại khóa mới dùng trong keyring, interface `keyring.SignatureAlgo` phải được thỏa mãn.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/crypto/keyring/signing_algorithms.go#L11-L16
```

Interface bao gồm ba phương thức trong đó `Name()` trả về tên của thuật toán dưới dạng `hd.PubKeyType` và `Derive()` và `Generate()` phải trả về các hàm sau tương ứng:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/crypto/hd/algo.go#L28-L31
```

Khi `keyring.SignatureAlgo` đã được triển khai, nó phải được thêm vào [danh sách các thuật toán được hỗ trợ](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/crypto/keyring/keyring.go#L209) của keyring.

Để đơn giản, việc triển khai loại khóa mới nên được thực hiện trong package `crypto/hd`. Có một ví dụ về triển khai `secp256k1` đang hoạt động trong [algo.go](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/crypto/hd/algo.go#L38).

#### Triển khai thuật toán secp256r1

Đây là ví dụ về cách secp256r1 có thể được triển khai.

Đầu tiên cần một hàm mới để tạo khóa riêng tư từ một số bí mật trong package secp256r1. Hàm này có thể trông như sau:

```go
// cosmos-sdk/crypto/keys/secp256r1/privkey.go

// NewPrivKeyFromSecret tạo khóa riêng tư được dẫn xuất cho số bí mật
// được biểu diễn dưới dạng big-endian. `secret` phải là phần tử trường ECDSA hợp lệ.
func NewPrivKeyFromSecret(secret []byte) (*PrivKey, error) {
	var d = new(big.Int).SetBytes(secret)
	if d.Cmp(secp256r1.Params().N) >= 1 {
		return nil, errorsmod.Wrap(errors.ErrInvalidRequest, "secret not in the curve base field")
	}
	sk := new(ecdsa.PrivKey)
	return &PrivKey{&ecdsaSK{*sk}}, nil
}
```

Sau đó `secp256r1Algo` có thể được triển khai.

```go
// cosmos-sdk/crypto/hd/secp256r1Algo.go

package hd

import (
	"github.com/cosmos/go-bip39"
	
	"github.com/cosmos/cosmos-sdk/crypto/keys/secp256r1"
	"github.com/cosmos/cosmos-sdk/crypto/types"
)

// Secp256r1Type sử dụng các tham số ECDSA secp256r1.
const Secp256r1Type = PubKeyType("secp256r1")

var Secp256r1 = secp256r1Algo{}

type secp256r1Algo struct{}

func (s secp256r1Algo) Name() PubKeyType {
	return Secp256r1Type
}

// Derive dẫn xuất và trả về khóa riêng tư secp256r1 cho seed và HD path đã cho.
func (s secp256r1Algo) Derive() DeriveFn {
	return func(mnemonic string, bip39Passphrase, hdPath string) ([]byte, error) {
		seed, err := bip39.NewSeedWithErrorChecking(mnemonic, bip39Passphrase)
		if err != nil {
			return nil, err
		}

		masterPriv, ch := ComputeMastersFromSeed(seed)
		if len(hdPath) == 0 {
			return masterPriv[:], nil
		}
		derivedKey, err := DerivePrivateKeyForPath(masterPriv, ch, hdPath)

		return derivedKey, err
	}
}

// Generate tạo khóa riêng tư secp256r1 từ các byte đã cho.
func (s secp256r1Algo) Generate() GenerateFn {
	return func(bz []byte) types.PrivKey {
		key, err := secp256r1.NewPrivKeyFromSecret(bz)
		if err != nil {
			panic(err)
		}
		return key
	}
}
```

Cuối cùng, thuật toán phải được thêm vào danh sách [các thuật toán được hỗ trợ](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/crypto/keyring/keyring.go#L209) bởi keyring.

```go
// cosmos-sdk/crypto/keyring/keyring.go

func newKeystore(kr keyring.Keyring, cdc codec.Codec, backend string, opts ...Option) keystore {
	// Các tùy chọn mặc định cho keybase, có thể bị ghi đè bằng
	// hàm Option
	options := Options{
		SupportedAlgos:       SigningAlgoList{hd.Secp256k1, hd.Secp256r1}, // được thêm ở đây
		SupportedAlgosLedger: SigningAlgoList{hd.Secp256k1},
	}
...
```

Sau đây để tạo khóa mới bằng thuật toán của bạn, bạn phải chỉ định nó bằng flag `--algo`:

`simd keys add myKey --algo secp256r1`
