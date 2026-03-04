---
sidebar_position: 1
---

# Module Simulation (Mô Phỏng Module)

:::note Yêu Cầu Đọc Trước

* [Cosmos Blockchain Simulator](../../learn/advanced/12-simulation.md)

:::

## Tóm Tắt

Tài liệu này hướng dẫn nhà phát triển tích hợp các module tùy chỉnh của họ với `Simulations` của Cosmos SDK. Simulation hữu ích để kiểm thử các trường hợp biên (edge cases) trong các triển khai module.

* [Gói Simulation](#simulation-package)
* [Simulation App Module](#simulation-app-module)
* [SimsX](#simsx)
    * [Ví dụ triển khai](#example-implementations)
* [Store decoders](#store-decoders)
* [Randomized genesis](#randomized-genesis)
* [Random weighted operations](#random-weighted-operations)
    * [Sử dụng Simsx](#using-simsx)
* [App Simulator manager](#app-simulator-manager)
* [Chạy Simulation](#running-simulations)



## Gói Simulation

Cosmos SDK đề xuất tổ chức mã liên quan đến simulation của bạn trong gói `x/<module>/simulation`.

## Simulation App Module

Để tích hợp với `SimulationManager` của Cosmos SDK, các app module phải triển khai interface `AppModuleSimulation`.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/3c6deab626648e47de752c33dac5d06af83e3ee3/types/module/simulation.go#L16-L27
```

Xem ví dụ triển khai các phương thức này từ `x/distribution` [tại đây](https://github.com/cosmos/cosmos-sdk/blob/b55b9e14fb792cc8075effb373be9d26327fddea/x/distribution/module.go#L170-L194).

## SimsX

Cosmos SDK v0.53.0 giới thiệu gói mới `simsx`, cung cấp DevX được cải thiện để viết mã simulation.

Nó hiển thị các extension interface sau mà các module có thể triển khai để tích hợp với runner `simsx` mới.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/main/testutil/simsx/runner.go#L223-L234
```

Các phương thức này cho phép xây dựng các message ngẫu nhiên và/hoặc các proposal message.

:::tip
Lưu ý rằng các module **không nên** triển khai cả `HasWeightedOperationsX` và `HasWeightedOperationsXWithProposals`.
Xem mã runner [tại đây](https://github.com/cosmos/cosmos-sdk/blob/main/testutil/simsx/runner.go#L330-L339) để biết chi tiết.

Nếu module **không** có các message handler hoặc governance proposal handler, các phương thức interface này **không cần** được triển khai.
:::

### Ví Dụ Triển Khai

* `HasWeightedOperationsXWithProposals`: [x/gov](https://github.com/cosmos/cosmos-sdk/blob/main/x/gov/module.go#L242-L261)
* `HasWeightedOperationsX`: [x/bank](https://github.com/cosmos/cosmos-sdk/blob/main/x/bank/module.go#L199-L203)
* `HasProposalMsgsX`: [x/bank](https://github.com/cosmos/cosmos-sdk/blob/main/x/bank/module.go#L194-L197)

## Store Decoders

Đăng ký các store decoder là bắt buộc cho simulation `AppImportExport`. Điều này cho phép các cặp khóa-giá trị từ các store được giải mã sang các kiểu tương ứng. Cụ thể, nó khớp khóa với một kiểu cụ thể và sau đó unmarshal giá trị từ `KVPair` sang kiểu được cung cấp.

Các module sử dụng [collections](https://github.com/cosmos/cosmos-sdk/blob/main/collections/README.md) có thể sử dụng hàm `NewStoreDecoderFuncFromCollectionsSchema` tự động xây dựng decoder:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/main/x/bank/module.go#L181-L184
```

Các module không sử dụng collections phải xây dựng store decoder thủ công. Xem triển khai [tại đây](https://github.com/cosmos/cosmos-sdk/blob/main/x/distribution/simulation/decoder.go) từ module distribution làm ví dụ.

## Randomized Genesis (Genesis Ngẫu Nhiên)

Simulator kiểm thử các tình huống và giá trị khác nhau cho các tham số genesis. Các app module phải triển khai phương thức `GenerateGenesisState` để tạo ra `GenesisState` ngẫu nhiên ban đầu từ một seed nhất định.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/main/types/module/simulation.go#L20
```

Xem ví dụ từ `x/auth` [tại đây](https://github.com/cosmos/cosmos-sdk/blob/main/x/auth/module.go#L169-L172).

Khi các tham số genesis của module được tạo ngẫu nhiên (hoặc với các khóa và giá trị được định nghĩa trong file `params`), chúng được marshal sang định dạng JSON và thêm vào app genesis JSON cho simulation.

## Random Weighted Operations (Các Thao Tác Có Trọng Số Ngẫu Nhiên)

Operations là một trong những phần quan trọng của Cosmos SDK simulation. Chúng là các giao dịch (`Msg`) được mô phỏng với các giá trị trường ngẫu nhiên. Người gửi của operation cũng được gán ngẫu nhiên.

Các operation trong simulation được mô phỏng bằng cách sử dụng toàn bộ [vòng đời giao dịch](../../learn/advanced/01-transactions.md) của một ứng dụng `ABCI` hiển thị `BaseApp`.

### Sử Dụng Simsx

Simsx giới thiệu khả năng định nghĩa một `MsgFactory` cho mỗi message của module.

Các factory này được đăng ký trong `WeightedOperationsX` và/hoặc `ProposalMsgsX`.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/main/x/distribution/module.go#L196-L206
```

Lưu ý rằng tên được truyền vào `weights.Get` phải khớp với tên của operation được đặt trong `WeightedOperations`.

Ví dụ, nếu module chứa operation `op_weight_msg_set_withdraw_address`, tên được truyền vào `weights.Get` phải là `msg_set_withdraw_address`.

Xem `x/distribution` để biết ví dụ triển khai các message factory [tại đây](https://github.com/cosmos/cosmos-sdk/blob/main/x/distribution/simulation/msg_factory.go).

## App Simulator Manager

Bước tiếp theo là thiết lập `SimulatorManager` ở cấp độ app. Điều này cần thiết cho các file kiểm thử simulation ở bước tiếp theo.

```go
type CoolApp struct {
...
sm *module.SimulationManager
}
```

Trong constructor của ứng dụng, xây dựng simulation manager bằng các module từ `ModuleManager` và gọi phương thức `RegisterStoreDecoders`.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/main/simapp/app.go#L650-L660
```

Lưu ý rằng bạn có thể ghi đè một số module. Điều này hữu ích nếu cấu hình module hiện có trong `ModuleManager` phải khác trong `SimulationManager`.

Cuối cùng, ứng dụng nên hiển thị `SimulationManager` thông qua phương thức sau được định nghĩa trong interface `Runtime`:

```go
// SimulationManager triển khai interface SimulationApp
func (app *SimApp) SimulationManager() *module.SimulationManager {
return app.sm
}
```

## Chạy Simulation

Để chạy simulation, sử dụng runner `simsx`.

Gọi hàm sau từ gói `simsx` để bắt đầu mô phỏng với seed mặc định:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/main/testutil/simsx/runner.go#L69-L88
```

Nếu muốn seed tùy chỉnh, các test nên sử dụng `RunWithSeed`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/b55b9e14fb792cc8075effb373be9d26327fddea/testutil/simsx/runner.go#L151-L168
```

Các hàm này nên được gọi trong các test (tức là app_test.go, app_sim_test.go, v.v.)

Ví dụ:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/main/simapp/sim_test.go#L53-L65
```
