---
sidebar_position: 1
---

# `x/auth`

## Tóm tắt

Tài liệu này mô tả module auth của Cosmos SDK.

Module auth chịu trách nhiệm xác định các kiểu giao dịch (transaction) và tài khoản
(account) cơ sở cho một ứng dụng, vì bản thân SDK là “bất khả tri” (agnostic) với
các chi tiết này. Module chứa các middleware, nơi thực hiện mọi kiểm tra hợp lệ cơ
bản của giao dịch (chữ ký, nonce, các trường phụ trợ), và expose account keeper,
cho phép các module khác đọc, ghi, và chỉnh sửa tài khoản.

Module này được sử dụng trong Cosmos Hub.

## Nội dung

* [Khái niệm](#khái-niệm)
  * [Gas & Phí](#gas--phí)
* [State](#state)
  * [Tài khoản](#tài-khoản)
* [AnteHandler](#antehandler)
* [Keeper](#keeper)
  * [Account Keeper](#account-keeper)
* [Tham số](#tham-số)
* [Client](#client)
  * [CLI](#cli)
  * [gRPC](#grpc)
  * [REST](#rest)

## Khái niệm

**Lưu ý:** module auth khác với [module authz](../authz/).

Sự khác biệt:

* `auth` - xác thực tài khoản và giao dịch cho các ứng dụng Cosmos SDK, và chịu trách nhiệm xác định các kiểu giao dịch và tài khoản cơ sở.
* `authz` - uỷ quyền để tài khoản thực hiện hành động thay mặt tài khoản khác, và cho phép bên cấp quyền (granter) cấp quyền cho bên nhận quyền (grantee) để grantee có thể thực thi message thay mặt granter.

### Gas & Phí

Phí phục vụ hai mục đích đối với operator của mạng.

Phí giới hạn mức tăng trưởng của state được lưu bởi mọi full node và cho phép
kiểm duyệt giao dịch theo cách “mục đích chung” đối với các giao dịch có giá trị
kinh tế thấp. Phí phù hợp nhất như một cơ chế chống spam trong bối cảnh validator
không quan tâm tới việc sử dụng mạng và danh tính người dùng.

Phí được xác định bởi giới hạn gas và giá gas mà giao dịch cung cấp, theo công thức
`fees = ceil(gasLimit * gasPrices)`. Tx phát sinh chi phí gas cho mọi thao tác đọc/ghi state,
xác minh chữ ký, cũng như chi phí tỷ lệ theo kích thước tx. Operator nên thiết lập
giá gas tối thiểu khi khởi chạy node. Họ phải thiết lập chi phí đơn vị của gas theo
mỗi mệnh giá token mà họ muốn hỗ trợ:

`simd start ... --minimum-gas-prices=0.00001stake;0.05photinos`

Khi thêm giao dịch vào mempool hoặc gossip giao dịch, validator kiểm tra xem giá gas
của giao dịch (được suy ra từ phí cung cấp) có thoả mãn bất kỳ mức giá gas tối thiểu
nào của validator hay không. Nói cách khác, một giao dịch phải cung cấp ít nhất một
mệnh giá phí khớp với giá gas tối thiểu của validator.

CometBFT hiện chưa cung cấp cơ chế ưu tiên mempool theo phí, và lọc mempool theo phí
là cục bộ theo node và không thuộc về đồng thuận. Tuy nhiên, khi đã đặt giá gas tối
thiểu, operator có thể triển khai cơ chế như vậy ở cấp node.

Do giá thị trường của token biến động, kỳ vọng validator sẽ điều chỉnh động (dynamically)
giá gas tối thiểu tới một mức khuyến khích việc sử dụng mạng.

## State

### Tài khoản

Tài khoản chứa thông tin xác thực cho một người dùng bên ngoài được định danh duy nhất
trong một blockchain SDK, bao gồm public key, địa chỉ, và account number / sequence number
để bảo vệ chống replay. Vì lý do hiệu năng, do số dư tài khoản cũng cần được lấy để trả phí,
cấu trúc tài khoản cũng lưu số dư của người dùng dưới dạng `sdk.Coins`.

Tài khoản được expose ra ngoài dưới dạng interface, và được lưu nội bộ dưới dạng
base account hoặc vesting account. Client của module muốn thêm các kiểu tài khoản
khác có thể làm như vậy.

* `0x01 | Address -> ProtocolBuffer(account)`

#### Account Interface

Account interface expose các phương thức để đọc và ghi thông tin tài khoản tiêu chuẩn.
Lưu ý rằng mọi phương thức này thao tác trên một struct tài khoản tuân theo interface —
để ghi tài khoản vào store, cần dùng account keeper.

```go
// AccountI is an interface used to store coins at a given address within state.
// It presumes a notion of sequence numbers for replay protection,
// a notion of account numbers for replay protection for previously pruned accounts,
// and a pubkey for authentication purposes.
//
// Many complex conditions can be used in the concrete struct which implements AccountI.
type AccountI interface {
	proto.Message

	GetAddress() sdk.AccAddress
	SetAddress(sdk.AccAddress) error // errors if already set.

	GetPubKey() crypto.PubKey // can return nil.
	SetPubKey(crypto.PubKey) error

	GetAccountNumber() uint64
	SetAccountNumber(uint64) error

	GetSequence() uint64
	SetSequence(uint64) error

	// Ensure that account implements stringer
	String() string
}
```

##### Base Account

Base account là kiểu tài khoản đơn giản nhất và phổ biến nhất, chỉ lưu mọi field
cần thiết trực tiếp trong một struct.

```protobuf
// BaseAccount defines a base account type. It contains all the necessary fields
// for basic account functionality. Any custom account type should extend this
// type for additional functionality (e.g. vesting).
message BaseAccount {
  string address = 1;
  google.protobuf.Any pub_key = 2;
  uint64 account_number = 3;
  uint64 sequence       = 4;
}
```

### Vesting Account

Xem [Vesting](https://docs.cosmos.network/main/modules/auth/vesting/).

## AnteHandler

Module `x/auth` hiện không có transaction handler riêng, nhưng có expose `AnteHandler`
đặc biệt, dùng để thực hiện các kiểm tra hợp lệ cơ bản trên giao dịch, để có thể loại
nó khỏi mempool.
`AnteHandler` có thể xem như một tập các decorator kiểm tra giao dịch trong context
hiện tại, theo [ADR 010](https://github.com/cosmos/cosmos-sdk/blob/main/docs/architecture/adr-010-modular-antehandler.md).

Lưu ý `AnteHandler` được gọi trong cả `CheckTx` và `DeliverTx`, vì proposer CometBFT
hiện có thể đưa vào khối đề xuất các giao dịch không vượt qua `CheckTx`.

### Decorator

Module auth cung cấp các `AnteDecorator` được nối (chain) đệ quy thành một `AnteHandler`
duy nhất theo thứ tự sau:

* `SetUpContextDecorator`: Thiết lập `GasMeter` trong `Context` và bọc `AnteHandler`
  kế tiếp với một mệnh đề defer để recover từ mọi panic `OutOfGas` ở phía sau trong
  chuỗi `AnteHandler`, nhằm trả về error kèm thông tin gas được cung cấp và gas đã dùng.
* `RejectExtensionOptionsDecorator`: Từ chối mọi extension option có thể tuỳ chọn
  được thêm vào trong giao dịch protobuf.
* `MempoolFeeDecorator`: Kiểm tra phí `tx` có lớn hơn tham số `minFee` cục bộ của mempool
  trong `CheckTx` hay không.
* `ValidateBasicDecorator`: Gọi `tx.ValidateBasic` và trả về error khác-nil nếu có.
* `TxTimeoutHeightDecorator`: Kiểm tra `tx` có timeout theo height hay không.
* `ValidateMemoDecorator`: Xác thực memo của `tx` theo tham số ứng dụng và trả về error khác-nil nếu có.
* `ConsumeGasTxSizeDecorator`: Tiêu thụ gas tỷ lệ với kích thước `tx` dựa trên tham số ứng dụng.
* `DeductFeeDecorator`: Trừ `FeeAmount` từ signer đầu tiên của `tx`. Nếu module `x/feegrant`
  được bật và có fee granter, nó trừ phí từ tài khoản fee granter.
* `SetPubKeyDecorator`: Thiết lập pubkey từ các signer của `tx` mà pubkey tương ứng chưa được
  lưu trong state machine và trong context hiện tại.
* `ValidateSigCountDecorator`: Xác thực số lượng chữ ký trong `tx` dựa trên tham số ứng dụng.
* `SigGasConsumeDecorator`: Tiêu thụ lượng gas được tham số hoá cho mỗi chữ ký. Điều này yêu
  cầu pubkey phải được đặt trong context cho mọi signer như một phần của `SetPubKeyDecorator`.
* `SigVerificationDecorator`: Xác minh mọi chữ ký là hợp lệ. Điều này yêu cầu pubkey phải được
  đặt trong context cho mọi signer như một phần của `SetPubKeyDecorator`.
* `IncrementSequenceDecorator`: Tăng sequence tài khoản cho mỗi signer để ngăn replay attack.

## Keeper

Module auth chỉ expose một keeper: account keeper, có thể dùng để đọc và ghi tài khoản.

### Account Keeper

Hiện tại chỉ expose một account keeper có đầy đủ quyền (fully-permissioned), có khả năng
cả đọc và ghi mọi field của mọi tài khoản, và iterate qua mọi tài khoản đã lưu.

```go
// AccountKeeperI is the interface contract that x/auth's keeper implements.
type AccountKeeperI interface {
	// Return a new account with the next account number and the specified address. Does not save the new account to the store.
	NewAccountWithAddress(sdk.Context, sdk.AccAddress) types.AccountI

	// Return a new account with the next account number. Does not save the new account to the store.
	NewAccount(sdk.Context, types.AccountI) types.AccountI

	// Check if an account exists in the store.
	HasAccount(sdk.Context, sdk.AccAddress) bool

	// Retrieve an account from the store.
	GetAccount(sdk.Context, sdk.AccAddress) types.AccountI

	// Set an account in the store.
	SetAccount(sdk.Context, types.AccountI)

	// Remove an account from the store.
	RemoveAccount(sdk.Context, types.AccountI)

	// Iterate over all accounts, calling the provided function. Stop iteration when it returns true.
	IterateAccounts(sdk.Context, func(types.AccountI) bool)

	// Fetch the public key of an account at a specified address
	GetPubKey(sdk.Context, sdk.AccAddress) (crypto.PubKey, error)

	// Fetch the sequence of an account at a specified address.
	GetSequence(sdk.Context, sdk.AccAddress) (uint64, error)

	// Fetch the next account number, and increment the internal counter.
	NextAccountNumber(sdk.Context) uint64
}
```

## Tham số

Module auth có các tham số sau:

| Key                    | Type        | Ví dụ |
| ---------------------- | ----------- | ----- |
| MaxMemoCharacters      | uint64      | 256   |
| TxSigLimit             | uint64      | 7     |
| TxSizeCostPerByte      | uint64      | 10    |
| SigVerifyCostED25519   | uint64      | 590   |
| SigVerifyCostSecp256k1 | uint64      | 1000  |

## Client

### CLI

Người dùng có thể truy vấn và tương tác với module `auth` bằng CLI.

### Query

Các lệnh `query` cho phép người dùng truy vấn state của `auth`.

```bash
simd query auth --help
```

#### account

Lệnh `account` cho phép người dùng truy vấn một tài khoản theo địa chỉ.

```bash
simd query auth account [address] [flags]
```

Ví dụ:

```bash
simd query auth account cosmos1...
```

Ví dụ output:

```bash
'@type': /cosmos.auth.v1beta1.BaseAccount
account_number: "0"
address: cosmos1zwg6tpl8aw4rawv8sgag9086lpw5hv33u5ctr2
pub_key:
  '@type': /cosmos.crypto.secp256k1.PubKey
  key: ApDrE38zZdd7wLmFS9YmqO684y5DG6fjZ4rVeihF/AQD
sequence: "1"
```

#### accounts

Lệnh `accounts` cho phép người dùng truy vấn tất cả tài khoản hiện có.

```bash
simd query auth accounts [flags]
```

Ví dụ:

```bash
simd query auth accounts
```

Ví dụ output:

```bash
accounts:
- '@type': /cosmos.auth.v1beta1.BaseAccount
  account_number: "0"
  address: cosmos1zwg6tpl8aw4rawv8sgag9086lpw5hv33u5ctr2
  pub_key:
    '@type': /cosmos.crypto.secp256k1.PubKey
    key: ApDrE38zZdd7wLmFS9YmqO684y5DG6fjZ4rVeihF/AQD
  sequence: "1"
- '@type': /cosmos.auth.v1beta1.ModuleAccount
  base_account:
    account_number: "8"
    address: cosmos1yl6hdjhmkf37639730gffanpzndzdpmhwlkfhr
    pub_key: null
    sequence: "0"
  name: transfer
  permissions:
  - minter
  - burner
- '@type': /cosmos.auth.v1beta1.ModuleAccount
  base_account:
    account_number: "4"
    address: cosmos1fl48vsnmsdzcv85q5d2q4z5ajdha8yu34mf0eh
    pub_key: null
    sequence: "0"
  name: bonded_tokens_pool
  permissions:
  - burner
  - staking
- '@type': /cosmos.auth.v1beta1.ModuleAccount
  base_account:
    account_number: "5"
    address: cosmos1tygms3xhhs3yv487phx3dw4a95jn7t7lpm470r
    pub_key: null
    sequence: "0"
  name: not_bonded_tokens_pool
  permissions:
  - burner
  - staking
- '@type': /cosmos.auth.v1beta1.ModuleAccount
  base_account:
    account_number: "6"
    address: cosmos10d07y265gmmuvt4z0w9aw880jnsr700j6zn9kn
    pub_key: null
    sequence: "0"
  name: gov
  permissions:
  - burner
- '@type': /cosmos.auth.v1beta1.ModuleAccount
  base_account:
    account_number: "3"
    address: cosmos1jv65s3grqf6v6jl3dp4t6c9t9rk99cd88lyufl
    pub_key: null
    sequence: "0"
  name: distribution
  permissions: []
- '@type': /cosmos.auth.v1beta1.BaseAccount
  account_number: "1"
  address: cosmos147k3r7v2tvwqhcmaxcfql7j8rmkrlsemxshd3j
  pub_key: null
  sequence: "0"
- '@type': /cosmos.auth.v1beta1.ModuleAccount
  base_account:
    account_number: "7"
    address: cosmos1m3h30wlvsf8llruxtpukdvsy0km2kum8g38c8q
    pub_key: null
    sequence: "0"
  name: mint
  permissions:
  - minter
- '@type': /cosmos.auth.v1beta1.ModuleAccount
  base_account:
    account_number: "2"
    address: cosmos17xpfvakm2amg962yls6f84z3kell8c5lserqta
    pub_key: null
    sequence: "0"
  name: fee_collector
  permissions: []
pagination:
  next_key: null
  total: "0"
```

#### params

Lệnh `params` cho phép người dùng truy vấn các tham số auth hiện tại.

```bash
simd query auth params [flags]
```

Ví dụ:

```bash
simd query auth params
```

Ví dụ output:

```bash
max_memo_characters: "256"
sig_verify_cost_ed25519: "590"
sig_verify_cost_secp256k1: "1000"
tx_sig_limit: "7"
tx_size_cost_per_byte: "10"
```

### Transactions

Module `auth` hỗ trợ các lệnh giao dịch để giúp bạn ký (sign) và nhiều thứ khác.
So với các module khác, bạn có thể truy cập trực tiếp các lệnh giao dịch của module `auth`
chỉ bằng lệnh `tx`.

Dùng trực tiếp cờ `--help` để xem thêm thông tin về lệnh `tx`.

```bash
simd tx --help
```

#### `sign`

Lệnh `sign` cho phép người dùng ký các giao dịch được tạo offline.

```bash
simd tx sign tx.json --from $ALICE > tx.signed.json
```

Kết quả là một giao dịch đã ký có thể được broadcast lên mạng bằng lệnh broadcast.

Thông tin thêm về lệnh `sign` có thể xem bằng `simd tx sign --help`.

#### `sign-batch`

Lệnh `sign-batch` cho phép người dùng ký nhiều giao dịch được tạo offline.
Các giao dịch có thể nằm trong một file, mỗi dòng một tx, hoặc nằm trong nhiều file.

```bash
simd tx sign txs.json --from $ALICE > tx.signed.json
```

hoặc

```bash 
simd tx sign tx1.json tx2.json tx3.json --from $ALICE > tx.signed.json
```

Kết quả là nhiều giao dịch đã ký. Để gộp các giao dịch đã ký thành một giao dịch,
dùng cờ `--append`.

Thông tin thêm về lệnh `sign-batch` có thể xem bằng `simd tx sign-batch --help`.

#### `multi-sign`

Lệnh `multi-sign` cho phép người dùng ký các giao dịch được tạo offline bởi một
tài khoản multisig.

```bash
simd tx multisign transaction.json k1k2k3 k1sig.json k2sig.json k3sig.json
```

Trong đó `k1k2k3` là địa chỉ tài khoản multisig, `k1sig.json` là chữ ký của signer thứ nhất,
`k2sig.json` là chữ ký của signer thứ hai, và `k3sig.json` là chữ ký của signer thứ ba.

##### Giao dịch multisig lồng nhau

Để cho phép giao dịch được ký bởi multisig lồng nhau, nghĩa là một thành viên của
một tài khoản multisig có thể là một tài khoản multisig khác, cần dùng cờ
`--skip-signature-verification`.

```bash
# First aggregate signatures of the multisig participant
simd tx multi-sign transaction.json ms1 ms1p1sig.json ms1p2sig.json --signature-only --skip-signature-verification > ms1sig.json

# Then use the aggregated signatures and the other signatures to sign the final transaction
simd tx multi-sign transaction.json k1ms1 k1sig.json ms1sig.json --skip-signature-verification
```

Trong đó `ms1` là địa chỉ tài khoản multisig lồng nhau, `ms1p1sig.json` là chữ ký của
người tham gia thứ nhất, `ms1p2sig.json` là chữ ký của người tham gia thứ hai, và
`ms1sig.json` là chữ ký đã tổng hợp (aggregated) của multisig lồng nhau.

`k1ms1` là một tài khoản multisig gồm một signer cá nhân và một tài khoản multisig lồng nhau (`ms1`).
`k1sig.json` là chữ ký của signer cá nhân.

Thông tin thêm về lệnh `multi-sign` có thể xem bằng `simd tx multi-sign --help`.

#### `multisign-batch`

`multisign-batch` hoạt động tương tự `sign-batch`, nhưng dành cho tài khoản multisig.
Khác biệt là `multisign-batch` yêu cầu tất cả giao dịch nằm trong một file, và không có cờ `--append`.

Thông tin thêm về lệnh `multisign-batch` có thể xem bằng `simd tx multisign-batch --help`.

#### `validate-signatures`

Lệnh `validate-signatures` cho phép người dùng xác thực chữ ký của một giao dịch đã ký.

```bash
$ simd tx validate-signatures tx.signed.json
Signers:
  0: cosmos1l6vsqhh7rnwsyr2kyz3jjg3qduaz8gwgyl8275

Signatures:
  0: cosmos1l6vsqhh7rnwsyr2kyz3jjg3qduaz8gwgyl8275                      [OK]
```

Thông tin thêm về lệnh `validate-signatures` có thể xem bằng `simd tx validate-signatures --help`.

#### `broadcast`

Lệnh `broadcast` cho phép người dùng broadcast một giao dịch đã ký lên mạng.

```bash
simd tx broadcast tx.signed.json
```

Thông tin thêm về lệnh `broadcast` có thể xem bằng `simd tx broadcast --help`.

### gRPC

Người dùng có thể truy vấn module `auth` qua các endpoint gRPC.

#### Account

Endpoint `account` cho phép người dùng truy vấn một tài khoản theo địa chỉ.

```bash
cosmos.auth.v1beta1.Query/Account
```

Ví dụ:

```bash
grpcurl -plaintext \
    -d '{"address":"cosmos1.."}' \
    localhost:9090 \
    cosmos.auth.v1beta1.Query/Account
```

Ví dụ output:

```bash
{
  "account":{
    "@type":"/cosmos.auth.v1beta1.BaseAccount",
    "address":"cosmos1zwg6tpl8aw4rawv8sgag9086lpw5hv33u5ctr2",
    "pubKey":{
      "@type":"/cosmos.crypto.secp256k1.PubKey",
      "key":"ApDrE38zZdd7wLmFS9YmqO684y5DG6fjZ4rVeihF/AQD"
    },
    "sequence":"1"
  }
}
```

#### Accounts

Endpoint `accounts` cho phép người dùng truy vấn tất cả tài khoản hiện có.

```bash
cosmos.auth.v1beta1.Query/Accounts
```

Ví dụ:

```bash
grpcurl -plaintext \
    localhost:9090 \
    cosmos.auth.v1beta1.Query/Accounts
```

Ví dụ output:

```bash
{
   "accounts":[
      {
         "@type":"/cosmos.auth.v1beta1.BaseAccount",
         "address":"cosmos1zwg6tpl8aw4rawv8sgag9086lpw5hv33u5ctr2",
         "pubKey":{
            "@type":"/cosmos.crypto.secp256k1.PubKey",
            "key":"ApDrE38zZdd7wLmFS9YmqO684y5DG6fjZ4rVeihF/AQD"
         },
         "sequence":"1"
      },
      {
         "@type":"/cosmos.auth.v1beta1.ModuleAccount",
         "baseAccount":{
            "address":"cosmos1yl6hdjhmkf37639730gffanpzndzdpmhwlkfhr",
            "accountNumber":"8"
         },
         "name":"transfer",
         "permissions":[
            "minter",
            "burner"
         ]
      },
      {
         "@type":"/cosmos.auth.v1beta1.ModuleAccount",
         "baseAccount":{
            "address":"cosmos1fl48vsnmsdzcv85q5d2q4z5ajdha8yu34mf0eh",
            "accountNumber":"4"
         },
         "name":"bonded_tokens_pool",
         "permissions":[
            "burner",
            "staking"
         ]
      },
      {
         "@type":"/cosmos.auth.v1beta1.ModuleAccount",
         "baseAccount":{
            "address":"cosmos1tygms3xhhs3yv487phx3dw4a95jn7t7lpm470r",
            "accountNumber":"5"
         },
         "name":"not_bonded_tokens_pool",
         "permissions":[
            "burner",
            "staking"
         ]
      },
      {
         "@type":"/cosmos.auth.v1beta1.ModuleAccount",
         "baseAccount":{
            "address":"cosmos10d07y265gmmuvt4z0w9aw880jnsr700j6zn9kn",
            "accountNumber":"6"
         },
         "name":"gov",
         "permissions":[
            "burner"
         ]
      },
      {
         "@type":"/cosmos.auth.v1beta1.ModuleAccount",
         "baseAccount":{
            "address":"cosmos1jv65s3grqf6v6jl3dp4t6c9t9rk99cd88lyufl",
            "accountNumber":"3"
         },
         "name":"distribution"
      },
      {
         "@type":"/cosmos.auth.v1beta1.BaseAccount",
         "accountNumber":"1",
         "address":"cosmos147k3r7v2tvwqhcmaxcfql7j8rmkrlsemxshd3j"
      },
      {
         "@type":"/cosmos.auth.v1beta1.ModuleAccount",
         "baseAccount":{
            "address":"cosmos1m3h30wlvsf8llruxtpukdvsy0km2kum8g38c8q",
            "accountNumber":"7"
         },
         "name":"mint",
         "permissions":[
            "minter"
         ]
      },
      {
         "@type":"/cosmos.auth.v1beta1.ModuleAccount",
         "baseAccount":{
            "address":"cosmos17xpfvakm2amg962yls6f84z3kell8c5lserqta",
            "accountNumber":"2"
         },
         "name":"fee_collector"
      }
   ],
   "pagination":{
      "total":"9"
   }
}
```

#### Params

Endpoint `params` cho phép người dùng truy vấn các tham số auth hiện tại.

```bash
cosmos.auth.v1beta1.Query/Params
```

Ví dụ:

```bash
grpcurl -plaintext \
    localhost:9090 \
    cosmos.auth.v1beta1.Query/Params
```

Ví dụ output:

```bash
{
  "params": {
    "maxMemoCharacters": "256",
    "txSigLimit": "7",
    "txSizeCostPerByte": "10",
    "sigVerifyCostEd25519": "590",
    "sigVerifyCostSecp256k1": "1000"
  }
}
```

### REST

Người dùng có thể truy vấn module `auth` qua các endpoint REST.

#### Account

Endpoint `account` cho phép người dùng truy vấn một tài khoản theo địa chỉ.

```bash
/cosmos/auth/v1beta1/account?address={address}
```

#### Accounts

Endpoint `accounts` cho phép người dùng truy vấn tất cả tài khoản hiện có.

```bash
/cosmos/auth/v1beta1/accounts
```

#### Params

Endpoint `params` cho phép người dùng truy vấn các tham số auth hiện tại.

```bash
/cosmos/auth/v1beta1/params
```

