---
sidebar_position: 1
---

# `x/upgrade`

## Tóm tắt

`x/upgrade` là một triển khai module Cosmos SDK giúp nâng cấp một chain Cosmos đang chạy
“mượt” (smoothly) sang một phiên bản phần mềm mới (có thể breaking). Module làm điều
này bằng cách cung cấp hook `PreBlocker` để ngăn state machine của blockchain tiếp tục
chạy khi đã đạt tới một block height nâng cấp được định nghĩa trước.

Module không áp đặt cách governance quyết định nâng cấp, mà chỉ cung cấp cơ chế
để điều phối nâng cấp một cách an toàn. Nếu không có hỗ trợ nâng cấp từ phần mềm,
việc nâng cấp một chain đang chạy là rủi ro vì tất cả validator cần dừng state machine
đúng tại cùng một điểm trong quá trình. Nếu không làm đúng, có thể xuất hiện
không nhất quán state và rất khó khôi phục.

* [Khái niệm](#khái-niệm)
* [State](#state)
* [Events](#events)
* [Client](#client)
  * [CLI](#cli)
  * [REST](#rest)
  * [gRPC](#grpc)
* [Tài nguyên](#tài-nguyên)

## Khái niệm

### Plan

Module `x/upgrade` định nghĩa kiểu `Plan` để lên lịch một lần nâng cấp “live”.
Một `Plan` có thể được lên lịch tại một block height cụ thể.
Một `Plan` được tạo khi một release candidate (được “đóng băng”) cùng một `Handler`
phù hợp (xem bên dưới) đã được đồng thuận. `Name` của `Plan` tương ứng với một
`Handler` cụ thể. Thông thường, `Plan` được tạo thông qua một proposal governance;
nếu được thông qua, nó sẽ được lên lịch. `Info` của `Plan` có thể chứa metadata
về nâng cấp, thường là thông tin nâng cấp đặc thù ứng dụng được đưa on-chain, ví dụ
một git commit mà validator có thể tự động nâng cấp tới.

```go
type Plan struct {
  Name   string
  Height int64
  Info   string
}
```

#### Sidecar process

Nếu operator chạy binary ứng dụng đồng thời chạy thêm một tiến trình sidecar hỗ trợ
tự động tải xuống và nâng cấp binary, trường `Info` cho phép tiến trình này hoạt động
liền mạch. Công cụ đó là [Cosmovisor](https://github.com/cosmos/cosmos-sdk/tree/main/tools/cosmovisor#readme).

### Handler

Module `x/upgrade` hỗ trợ nâng cấp từ major version X sang major version Y. Để làm
được điều này, operator node trước hết phải nâng cấp binary hiện tại lên một binary
mới có chứa `Handler` tương ứng cho version Y mới. Giả định rằng version này đã
được test đầy đủ và được cộng đồng phê duyệt. `Handler` định nghĩa những migration
state nào cần xảy ra trước khi binary Y mới có thể chạy chain thành công. Tự nhiên,
`Handler` phụ thuộc vào ứng dụng (application specific) và không được định nghĩa theo
từng module. Đăng ký `Handler` được thực hiện qua `Keeper#SetUpgradeHandler` trong ứng dụng.

```go
type UpgradeHandler func(Context, Plan, VersionMap) (VersionMap, error)
```

Trong mỗi lần thực thi `EndBlock`, module `x/upgrade` kiểm tra xem có `Plan` nào
cần chạy không (được lên lịch tại height đó). Nếu có, `Handler` tương ứng sẽ được
thực thi. Nếu `Plan` dự kiến chạy nhưng không có `Handler` nào được đăng ký, hoặc
nếu binary được nâng cấp quá sớm, node sẽ panic “gracefully” và thoát.

### StoreLoader

Module `x/upgrade` cũng hỗ trợ store migration như một phần của nâng cấp.
`StoreLoader` thiết lập các migration cần xảy ra trước khi binary mới có thể chạy
chain thành công. `StoreLoader` cũng phụ thuộc vào ứng dụng và không được định nghĩa
theo từng module. Đăng ký `StoreLoader` được thực hiện qua `app#SetStoreLoader` trong ứng dụng.

```go
func UpgradeStoreLoader (upgradeHeight int64, storeUpgrades *store.StoreUpgrades) baseapp.StoreLoader
```

Nếu có một nâng cấp đã lên kế hoạch và đạt upgrade height, binary cũ sẽ ghi `Plan`
ra disk trước khi panic.

Thông tin này rất quan trọng để đảm bảo `StoreUpgrades` diễn ra trơn tru tại đúng
height và đúng nâng cấp mong đợi. Nó loại bỏ khả năng binary mới thực thi
`StoreUpgrades` nhiều lần khi restart. Ngoài ra nếu có nhiều nâng cấp được lên lịch
cùng một height, `Name` sẽ đảm bảo `StoreUpgrades` chỉ diễn ra trong planned upgrade handler.

### Proposal

Thông thường, một `Plan` được đề xuất và gửi qua governance bằng một proposal chứa
message `MsgSoftwareUpgrade`. Proposal này tuân theo quy trình governance tiêu chuẩn.
Nếu proposal được thông qua, `Plan` (nhắm tới một `Handler` cụ thể) sẽ được lưu và lên lịch.
Nâng cấp có thể bị trì hoãn hoặc đẩy nhanh bằng cách cập nhật `Plan.Height` trong một proposal mới.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/upgrade/v1beta1/tx.proto#L29-L41
```

#### Huỷ proposal nâng cấp

Proposal nâng cấp có thể bị huỷ. Có một loại message `MsgCancelUpgrade` được bật qua gov,
có thể được nhúng trong một proposal, bỏ phiếu và nếu thông qua sẽ xoá `Plan` nâng cấp
đã lên lịch.
Tất nhiên điều này yêu cầu phải biết nâng cấp đó là “ý tưởng tồi” từ khá lâu trước khi
đến thời điểm nâng cấp, để kịp thời gian bỏ phiếu.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/upgrade/v1beta1/tx.proto#L48-L57
```

Nếu muốn khả năng này, upgrade height nên là
`2 * (VotingPeriod + DepositPeriod) + (SafetyDelta)` tính từ lúc bắt đầu proposal nâng cấp.
`SafetyDelta` là khoảng thời gian sẵn có từ khi proposal nâng cấp được thông qua đến khi
nhận ra đó là ý tưởng tồi (do đồng thuận xã hội bên ngoài).

Một proposal `MsgCancelUpgrade` cũng có thể được tạo khi proposal `MsgSoftwareUpgrade`
ban đầu vẫn đang được bỏ phiếu, miễn là `VotingPeriod` kết thúc sau proposal `MsgSoftwareUpgrade`.

## State

State nội bộ của module `x/upgrade` tương đối tối giản. State chứa `Plan` nâng cấp
đang hoạt động (nếu có) tại key `0x0`, và nếu `Plan` được đánh dấu là “done” tại key `0x1`.
State chứa consensus version của tất cả app module trong ứng dụng. Các version được
luu dưới dạng big endian `uint64`, và truy cập bằng prefix `0x2` kèm tên module (kiểu `string`).
State duy trì một `Protocol Version` tại key `0x3`.

* Plan: `0x0 -> Plan`
* Done: `0x1 | byte(plan name)  -> BigEndian(Block Height)`
* ConsensusVersion: `0x2 | byte(module name)  -> BigEndian(Module Consensus Version)`
* ProtocolVersion: `0x3 -> BigEndian(Protocol Version)`

Module `x/upgrade` không có genesis state.

## Events

`x/upgrade` không tự phát ra event. Mọi event liên quan proposal được phát ra qua module `x/gov`.

## Client

### CLI

Người dùng có thể truy vấn và tương tác với module `upgrade` qua CLI.

#### Query

Các lệnh `query` cho phép truy vấn state của `upgrade`.

```bash
simd query upgrade --help
```

##### applied

Lệnh `applied` cho phép truy vấn block header tại height mà một nâng cấp đã hoàn tất được áp dụng.

```bash
simd query upgrade applied [upgrade-name] [flags]
```

Nếu upgrade-name trước đó đã được thực thi trên chain, lệnh sẽ trả về header của block tại height áp dụng.
Điều này giúp client xác định binary nào hợp lệ trong một khoảng block nhất định, và có thêm ngữ cảnh để hiểu các migration trong quá khứ.

Ví dụ:

```bash
simd query upgrade applied "test-upgrade"
```

Ví dụ output:

```bash
"block_id": {
    "hash": "A769136351786B9034A5F196DC53F7E50FCEB53B48FA0786E1BFC45A0BB646B5",
    "parts": {
      "total": 1,
      "hash": "B13CBD23011C7480E6F11BE4594EE316548648E6A666B3575409F8F16EC6939E"
    }
  },
  "block_size": "7213",
  "header": {
    "version": {
      "block": "11"
    },
    "chain_id": "testnet-2",
    "height": "455200",
    "time": "2021-04-10T04:37:57.085493838Z",
    "last_block_id": {
      "hash": "0E8AD9309C2DC411DF98217AF59E044A0E1CCEAE7C0338417A70338DF50F4783",
      "parts": {
        "total": 1,
        "hash": "8FE572A48CD10BC2CBB02653CA04CA247A0F6830FF19DC972F64D339A355E77D"
      }
    },
    "last_commit_hash": "DE890239416A19E6164C2076B837CC1D7F7822FC214F305616725F11D2533140",
    "data_hash": "E3B0C44298FC1C149AFBF4C8996FB92427AE41E4649B934CA495991B7852B855",
    "validators_hash": "A31047ADE54AE9072EE2A12FF260A8990BA4C39F903EAF5636B50D58DBA72582",
    "next_validators_hash": "A31047ADE54AE9072EE2A12FF260A8990BA4C39F903EAF5636B50D58DBA72582",
    "consensus_hash": "048091BC7DDC283F77BFBF91D73C44DA58C3DF8A9CBC867405D8B7F3DAADA22F",
    "app_hash": "28ECC486AFC332BA6CC976706DBDE87E7D32441375E3F10FD084CD4BAF0DA021",
    "last_results_hash": "E3B0C44298FC1C149AFBF4C8996FB92427AE41E4649B934CA495991B7852B855",
    "evidence_hash": "E3B0C44298FC1C149AFBF4C8996FB92427AE41E4649B934CA495991B7852B855",
    "proposer_address": "2ABC4854B1A1C5AA8403C4EA853A81ACA901CC76"
  },
  "num_txs": "0"
}
```

##### module versions

Lệnh `module_versions` lấy danh sách tên module và consensus version tương ứng.
Nếu thêm tên module cụ thể, chỉ trả về thông tin của module đó.

```bash
simd query upgrade module_versions [optional module_name] [flags]
```

Ví dụ:

```bash
simd query upgrade module_versions
```

Ví dụ output:

```bash
module_versions:
- name: auth
  version: "2"
- name: authz
  version: "1"
- name: bank
  version: "2"
- name: distribution
  version: "2"
- name: evidence
  version: "1"
- name: feegrant
  version: "1"
- name: genutil
  version: "1"
- name: gov
  version: "2"
- name: ibc
  version: "2"
- name: mint
  version: "1"
- name: params
  version: "1"
- name: slashing
  version: "2"
- name: staking
  version: "2"
- name: transfer
  version: "1"
- name: upgrade
  version: "1"
- name: vesting
  version: "1"
```

Ví dụ:

```bash
regen query upgrade module_versions ibc
```

Ví dụ output:

```bash
module_versions:
- name: ibc
  version: "2"
```

##### plan

Lệnh `plan` lấy upgrade plan đang được lên lịch (nếu có).

```bash
regen query upgrade plan [flags]
```

Ví dụ:

```bash
simd query upgrade plan
```

Ví dụ output:

```bash
height: "130"
info: ""
name: test-upgrade
time: "0001-01-01T00:00:00Z"
upgraded_client_state: null
```

#### Transactions

Module upgrade hỗ trợ các giao dịch sau:

* `software-proposal` - gửi proposal nâng cấp:

```bash
simd tx upgrade software-upgrade v2 --title="Test Proposal" --summary="testing" --deposit="100000000stake" --upgrade-height 1000000 \
--upgrade-info '{ "binaries": { "linux/amd64":"https://example.com/simd.zip?checksum=sha256:aec070645fe53ee3b3763059376134f058cc337247c978add178b6ccdfb0019f" } }' --from cosmos1..
```

* `cancel-software-upgrade` - huỷ proposal nâng cấp đã gửi:

```bash
simd tx upgrade cancel-software-upgrade --title="Test Proposal" --summary="testing" --deposit="100000000stake" --from cosmos1..
```

### REST

Người dùng có thể truy vấn module `upgrade` qua các endpoint REST.

#### Applied Plan

`AppliedPlan` truy vấn một upgrade plan đã áp dụng theo tên.

```bash
/cosmos/upgrade/v1beta1/applied_plan/{name}
```

Ví dụ:

```bash
curl -X GET "http://localhost:1317/cosmos/upgrade/v1beta1/applied_plan/v2.0-upgrade" -H "accept: application/json"
```

Ví dụ output:

```bash
{
  "height": "30"
}
```

#### Current Plan

`CurrentPlan` truy vấn upgrade plan hiện tại.

```bash
/cosmos/upgrade/v1beta1/current_plan
```

Ví dụ:

```bash
curl -X GET "http://localhost:1317/cosmos/upgrade/v1beta1/current_plan" -H "accept: application/json"
```

Ví dụ output:

```bash
{
  "plan": "v2.1-upgrade"
}
```

#### Module versions

`ModuleVersions` truy vấn danh sách module version từ state.

```bash
/cosmos/upgrade/v1beta1/module_versions
```

Ví dụ:

```bash
curl -X GET "http://localhost:1317/cosmos/upgrade/v1beta1/module_versions" -H "accept: application/json"
```

Ví dụ output:

```bash
{
  "module_versions": [
    {
      "name": "auth",
      "version": "2"
    },
    {
      "name": "authz",
      "version": "1"
    },
    {
      "name": "bank",
      "version": "2"
    },
    {
      "name": "distribution",
      "version": "2"
    },
    {
      "name": "evidence",
      "version": "1"
    },
    {
      "name": "feegrant",
      "version": "1"
    },
    {
      "name": "genutil",
      "version": "1"
    },
    {
      "name": "gov",
      "version": "2"
    },
    {
      "name": "ibc",
      "version": "2"
    },
    {
      "name": "mint",
      "version": "1"
    },
    {
      "name": "params",
      "version": "1"
    },
    {
      "name": "slashing",
      "version": "2"
    },
    {
      "name": "staking",
      "version": "2"
    },
    {
      "name": "transfer",
      "version": "1"
    },
    {
      "name": "upgrade",
      "version": "1"
    },
    {
      "name": "vesting",
      "version": "1"
    }
  ]
}
```

### gRPC

Người dùng có thể truy vấn module `upgrade` qua các endpoint gRPC.

#### Applied Plan

`AppliedPlan` truy vấn một upgrade plan đã áp dụng theo tên.

```bash
cosmos.upgrade.v1beta1.Query/AppliedPlan
```

Ví dụ:

```bash
grpcurl -plaintext \
    -d '{"name":"v2.0-upgrade"}' \
    localhost:9090 \
    cosmos.upgrade.v1beta1.Query/AppliedPlan
```

Ví dụ output:

```bash
{
  "height": "30"
}
```

#### Current Plan

`CurrentPlan` truy vấn upgrade plan hiện tại.

```bash
cosmos.upgrade.v1beta1.Query/CurrentPlan
```

Ví dụ:

```bash
grpcurl -plaintext localhost:9090 cosmos.upgrade.v1beta1.Query/CurrentPlan
```

Ví dụ output:

```bash
{
  "plan": "v2.1-upgrade"
}
```

#### Module versions

`ModuleVersions` truy vấn danh sách module version từ state.

```bash
cosmos.upgrade.v1beta1.Query/ModuleVersions
```

Ví dụ:

```bash
grpcurl -plaintext localhost:9090 cosmos.upgrade.v1beta1.Query/ModuleVersions
```

Ví dụ output:

```bash
{
  "module_versions": [
    {
      "name": "auth",
      "version": "2"
    },
    {
      "name": "authz",
      "version": "1"
    },
    {
      "name": "bank",
      "version": "2"
    },
    {
      "name": "distribution",
      "version": "2"
    },
    {
      "name": "evidence",
      "version": "1"
    },
    {
      "name": "feegrant",
      "version": "1"
    },
    {
      "name": "genutil",
      "version": "1"
    },
    {
      "name": "gov",
      "version": "2"
    },
    {
      "name": "ibc",
      "version": "2"
    },
    {
      "name": "mint",
      "version": "1"
    },
    {
      "name": "params",
      "version": "1"
    },
    {
      "name": "slashing",
      "version": "2"
    },
    {
      "name": "staking",
      "version": "2"
    },
    {
      "name": "transfer",
      "version": "1"
    },
    {
      "name": "upgrade",
      "version": "1"
    },
    {
      "name": "vesting",
      "version": "1"
    }
  ]
}
```

## Tài nguyên

Danh sách tài nguyên (bên ngoài) để tìm hiểu thêm về module `x/upgrade`.

* [Cosmos Dev Series: Cosmos Blockchain Upgrade](https://medium.com/web3-surfers/cosmos-dev-series-cosmos-sdk-based-blockchain-upgrade-b5e99181554c) - Bài viết giải thích chi tiết cách software upgrade hoạt động.

