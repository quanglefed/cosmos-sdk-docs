---
sidebar_position: 1
---

# `x/evidence`

* [Khái niệm](#khái-niệm)
* [State](#state)
* [Messages](#messages)
* [Events](#events)
* [Parameters](#parameters)
* [BeginBlock](#beginblock)
* [Client](#client)
  * [CLI](#cli)
  * [REST](#rest)
  * [gRPC](#grpc)

## Tóm tắt

`x/evidence` là một triển khai module Cosmos SDK theo [ADR 009](https://github.com/cosmos/cosmos-sdk/blob/main/docs/architecture/adr-009-evidence-module.md),
cho phép gửi (submission) và xử lý các loại evidence tuỳ ý về hành vi sai trái
(misbehavior) như equivocation và counterfactual signing.

Module evidence khác với cách xử lý evidence “chuẩn” vốn thường kỳ vọng consensus
engine bên dưới (ví dụ CometBFT) tự động submit evidence khi nó được phát hiện,
bằng việc cho phép client và các chain bên ngoài submit evidence phức tạp hơn trực tiếp.

Mọi kiểu evidence cụ thể (concrete evidence type) phải triển khai interface `Evidence`.
`Evidence` được submit trước tiên sẽ được route qua `Router` của module evidence, nơi nó
cố gắng tìm `Handler` tương ứng đã được đăng ký cho kiểu `Evidence` cụ thể đó.
Mỗi kiểu `Evidence` phải có một `Handler` được đăng ký trong keeper của module evidence
để có thể được route và thực thi thành công.

Mỗi handler tương ứng cũng phải thoả mãn “hợp đồng” interface `Handler`. `Handler` cho
một kiểu `Evidence` có thể thực hiện bất kỳ chuyển đổi state nào như slashing, jailing,
và tombstoning.

## Khái niệm

### Evidence

Bất kỳ kiểu evidence cụ thể nào được submit vào module `x/evidence` phải thoả mãn
hợp đồng `Evidence` dưới đây. Không phải mọi kiểu evidence đều thoả mãn hợp đồng này
theo cùng một cách và một số dữ liệu có thể hoàn toàn không liên quan với một số kiểu
evidence nhất định. Một interface bổ sung `ValidatorEvidence` (mở rộng `Evidence`) cũng
được tạo để định nghĩa hợp đồng cho evidence chống lại validator độc hại.

```go
// Evidence defines the contract which concrete evidence types of misbehavior
// must implement.
type Evidence interface {
	proto.Message

	Route() string
	String() string
	Hash() []byte
	ValidateBasic() error

	// Height at which the infraction occurred
	GetHeight() int64
}

// ValidatorEvidence extends Evidence interface to define contract
// for evidence against malicious validators
type ValidatorEvidence interface {
	Evidence

	// The consensus address of the malicious validator at time of infraction
	GetConsensusAddress() sdk.ConsAddress

	// The total power of the malicious validator at time of infraction
	GetValidatorPower() int64

	// The total validator set power at time of infraction
	GetTotalPower() int64
}
```

### Đăng ký & xử lý

Module `x/evidence` trước hết cần “biết” tất cả loại evidence mà nó dự kiến sẽ xử lý.
Điều này được thực hiện bằng cách đăng ký phương thức `Route` trong hợp đồng `Evidence`
với một `Router` (định nghĩa bên dưới). `Router` nhận `Evidence` và cố gắng tìm `Handler`
tương ứng cho `Evidence` thông qua phương thức `Route`.

```go
type Router interface {
  AddRoute(r string, h Handler) Router
  HasRoute(r string) bool
  GetRoute(path string) Handler
  Seal()
  Sealed() bool
}
```

`Handler` (định nghĩa bên dưới) chịu trách nhiệm thực thi toàn bộ logic nghiệp vụ
để xử lý `Evidence`. Thông thường bao gồm validate evidence, cả kiểm tra stateless
qua `ValidateBasic` và kiểm tra stateful qua các keeper cung cấp cho `Handler`.
Ngoài ra, `Handler` có thể thực hiện các hành vi như slashing và jailing validator.
Mọi `Evidence` được xử lý bởi `Handler` nên được persist.

```go
// Handler defines an agnostic Evidence handler. The handler is responsible
// for executing all corresponding business logic necessary for verifying the
// evidence as valid. In addition, the Handler may execute any necessary
// slashing and potential jailing.
type Handler func(context.Context, Evidence) error
```

## State

Hiện tại module `x/evidence` chỉ lưu `Evidence` hợp lệ đã được submit vào state.
Evidence state cũng được lưu và export trong `GenesisState` của module `x/evidence`.

```protobuf
// GenesisState defines the evidence module's genesis state.
message GenesisState {
  // evidence defines all the evidence at genesis.
  repeated google.protobuf.Any evidence = 1;
}

```

Tất cả `Evidence` được lấy và lưu thông qua một prefix `KVStore` với prefix `0x00` (`KeyPrefixEvidence`).

## Messages

### MsgSubmitEvidence

Evidence được submit thông qua message `MsgSubmitEvidence`:

```protobuf
// MsgSubmitEvidence represents a message that supports submitting arbitrary
// Evidence of misbehavior such as equivocation or counterfactual signing.
message MsgSubmitEvidence {
  string              submitter = 1;
  google.protobuf.Any evidence  = 2;
}
```

Lưu ý: `Evidence` của message `MsgSubmitEvidence` phải có `Handler` tương ứng đã được đăng ký
trong `Router` của module `x/evidence` để được xử lý và route đúng.

Khi `Evidence` đã được đăng ký với `Handler` tương ứng, nó được xử lý như sau:

```go
func SubmitEvidence(ctx Context, evidence Evidence) error {
  if _, err := GetEvidence(ctx, evidence.Hash()); err == nil {
    return errorsmod.Wrap(types.ErrEvidenceExists, strings.ToUpper(hex.EncodeToString(evidence.Hash())))
  }
  if !router.HasRoute(evidence.Route()) {
    return errorsmod.Wrap(types.ErrNoEvidenceHandlerExists, evidence.Route())
  }

  handler := router.GetRoute(evidence.Route())
  if err := handler(ctx, evidence); err != nil {
    return errorsmod.Wrap(types.ErrInvalidEvidence, err.Error())
  }

  ctx.EventManager().EmitEvent(
		sdk.NewEvent(
			types.EventTypeSubmitEvidence,
			sdk.NewAttribute(types.AttributeKeyEvidenceHash, strings.ToUpper(hex.EncodeToString(evidence.Hash()))),
		),
	)

  SetEvidence(ctx, evidence)
  return nil
}
```

Trước hết, không được tồn tại `Evidence` hợp lệ đã submit trùng hoàn toàn (cùng hash).
Tiếp theo, `Evidence` được route tới `Handler` và thực thi. Cuối cùng, nếu không có lỗi
khi xử lý `Evidence`, một event được phát ra và `Evidence` được persist vào state.

## Events

Module `x/evidence` phát ra các event sau:

### Handlers

#### MsgSubmitEvidence

| Type            | Attribute Key | Attribute Value |
| --------------- | ------------- | --------------- |
| submit_evidence | evidence_hash | {evidenceHash}  |
| message         | module        | evidence        |
| message         | sender        | {senderAddress} |
| message         | action        | submit_evidence |

## Parameters

Module evidence không có tham số.

## BeginBlock

### Xử lý evidence

Block CometBFT có thể bao gồm [Evidence](https://github.com/cometbft/cometbft/blob/main/spec/abci/abci%2B%2B_basic_concepts.md#evidence)
cho biết một validator đã thực hiện hành vi độc hại. Thông tin liên quan được chuyển tới ứng dụng
dưới dạng ABCI Evidence trong `abci.RequestBeginBlock` để validator có thể bị trừng phạt tương ứng.

#### Equivocation

Cosmos SDK xử lý hai loại evidence trong ABCI `BeginBlock`:

* `DuplicateVoteEvidence`,
* `LightClientAttackEvidence`.

Module evidence xử lý hai loại evidence này theo cùng một cách. Trước hết, Cosmos SDK chuyển đổi
kiểu evidence cụ thể của CometBFT sang interface `Evidence` của SDK bằng `Equivocation` làm kiểu cụ thể.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/tree/release/v0.50.x/proto/cosmos/evidence/v1beta1/evidence.proto#L12-L32
```

Để một `Equivocation` được submit trong `block` là hợp lệ, nó phải thoả:

`Evidence.Timestamp >= block.Timestamp - MaxEvidenceAge`

Trong đó:

* `Evidence.Timestamp` là timestamp trong block tại height `Evidence.Height`
* `block.Timestamp` là timestamp của block hiện tại

Nếu evidence `Equivocation` hợp lệ được đưa vào block, stake của validator sẽ bị giảm (slashed)
bởi `SlashFractionDoubleSign` (định nghĩa bởi module `x/slashing`) dựa trên stake của họ tại thời
điểm infraction xảy ra, thay vì tại thời điểm evidence được phát hiện. Ta muốn “follow the stake”,
tức là stake đã góp phần tạo ra infraction phải bị slash, ngay cả khi stake đó đã bị redelegate
hoặc bắt đầu unbonding.

Ngoài ra, validator sẽ bị jail vĩnh viễn và tombstoned để không thể tái gia nhập validator set.

Evidence `Equivocation` được xử lý như sau:

```go reference
https://github.com/cosmos/cosmos-sdk/tree/release/v0.50.x/x/evidence/keeper/infraction.go#L26-L140
```

**Lưu ý:** Các lời gọi slashing, jailing và tombstoning được uỷ quyền qua module `x/slashing`,
module này phát ra các event mang tính thông tin và cuối cùng uỷ quyền tiếp các lời gọi tới module `x/staking`.
Xem tài liệu về slashing và jailing trong [State Transitions](../staking/README.md#state-transitions).

## Client

### CLI

Người dùng có thể truy vấn và tương tác với module `evidence` bằng CLI.

#### Query

Các lệnh `query` cho phép truy vấn state của `evidence`.

```bash
simd query evidence --help
```

#### evidence

Lệnh `evidence` cho phép liệt kê tất cả evidence hoặc evidence theo hash.

Cách dùng:

```bash
simd query evidence [flags]
```

Truy vấn evidence theo hash.

Ví dụ:

```bash
simd query evidence evidence "DF0C23E8634E480F84B9D5674A7CDC9816466DEC28A3358F73260F68D28D7660"
```

Ví dụ output:

```bash
evidence:
  consensus_address: cosmosvalcons1ntk8eualewuprz0gamh8hnvcem2nrcdsgz563h
  height: 11
  power: 100
  time: "2021-10-20T16:08:38.194017624Z"
```

Lấy tất cả evidence.

Ví dụ:

```bash
simd query evidence list
```

Ví dụ output:

```bash
evidence:
  consensus_address: cosmosvalcons1ntk8eualewuprz0gamh8hnvcem2nrcdsgz563h
  height: 11
  power: 100
  time: "2021-10-20T16:08:38.194017624Z"
pagination:
  next_key: null
  total: "1"
```

### REST

Người dùng có thể truy vấn module `evidence` qua các endpoint REST.

#### Evidence

Lấy evidence theo hash:

```bash
/cosmos/evidence/v1beta1/evidence/{hash}
```

Ví dụ:

```bash
curl -X GET "http://localhost:1317/cosmos/evidence/v1beta1/evidence/DF0C23E8634E480F84B9D5674A7CDC9816466DEC28A3358F73260F68D28D7660"
```

Ví dụ output:

```bash
{
  "evidence": {
    "consensus_address": "cosmosvalcons1ntk8eualewuprz0gamh8hnvcem2nrcdsgz563h",
    "height": "11",
    "power": "100",
    "time": "2021-10-20T16:08:38.194017624Z"
  }
}
```

#### Tất cả evidence

Lấy tất cả evidence:

```bash
/cosmos/evidence/v1beta1/evidence
```

Ví dụ:

```bash
curl -X GET "http://localhost:1317/cosmos/evidence/v1beta1/evidence"
```

Ví dụ output:

```bash
{
  "evidence": [
    {
      "consensus_address": "cosmosvalcons1ntk8eualewuprz0gamh8hnvcem2nrcdsgz563h",
      "height": "11",
      "power": "100",
      "time": "2021-10-20T16:08:38.194017624Z"
    }
  ],
  "pagination": {
    "total": "1"
  }
}
```

### gRPC

Người dùng có thể truy vấn module `evidence` qua các endpoint gRPC.

#### Evidence

Lấy evidence theo hash:

```bash
cosmos.evidence.v1beta1.Query/Evidence
```

Ví dụ:

```bash
grpcurl -plaintext -d '{"evidence_hash":"DF0C23E8634E480F84B9D5674A7CDC9816466DEC28A3358F73260F68D28D7660"}' localhost:9090 cosmos.evidence.v1beta1.Query/Evidence
```

Ví dụ output:

```bash
{
  "evidence": {
    "consensus_address": "cosmosvalcons1ntk8eualewuprz0gamh8hnvcem2nrcdsgz563h",
    "height": "11",
    "power": "100",
    "time": "2021-10-20T16:08:38.194017624Z"
  }
}
```

#### Tất cả evidence

Lấy tất cả evidence:

```bash
cosmos.evidence.v1beta1.Query/AllEvidence
```

Ví dụ:

```bash
grpcurl -plaintext localhost:9090 cosmos.evidence.v1beta1.Query/AllEvidence
```

Ví dụ output:

```bash
{
  "evidence": [
    {
      "consensus_address": "cosmosvalcons1ntk8eualewuprz0gamh8hnvcem2nrcdsgz563h",
      "height": "11",
      "power": "100",
      "time": "2021-10-20T16:08:38.194017624Z"
    }
  ],
  "pagination": {
    "total": "1"
  }
}
```

