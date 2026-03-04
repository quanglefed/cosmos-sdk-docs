---
sidebar_position: 1
---

# Query Services

:::note Tóm tắt
Một Protobuf Query service xử lý các [`query`](./02-messages-and-queries.md#queries). Query service đặc thù cho module mà chúng được định nghĩa, và chỉ xử lý các `query` được định nghĩa trong module đó. Chúng được gọi từ [`phương thức Query`](../../learn/advanced/00-baseapp.md#query) của `BaseApp`.
:::

:::note Yêu Cầu Đọc Trước

* [Module Manager](./01-module-manager.md)
* [Messages và Queries](./02-messages-and-queries.md)

:::

## Triển Khai query service của module

### gRPC Service

Khi định nghĩa một Protobuf `Query` service, một interface `QueryServer` được tạo ra cho mỗi module với tất cả các phương thức service:

```go
type QueryServer interface {
	QueryBalance(context.Context, *QueryBalanceParams) (*types.Coin, error)
	QueryAllBalances(context.Context, *QueryAllBalancesParams) (*QueryAllBalancesResponse, error)
}
```

Các phương thức query tùy chỉnh này nên được triển khai bởi keeper của module, thường trong `./keeper/grpc_query.go`. Tham số đầu tiên của các phương thức này là `context.Context` chung. Do đó, Cosmos SDK cung cấp hàm `sdk.UnwrapSDKContext` để lấy `context.Context` từ `context.Context` được cung cấp.

Dưới đây là ví dụ triển khai cho module bank:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/bank/keeper/grpc_query.go
```

### Gọi Query từ State Machine

Cosmos SDK v0.47 giới thiệu annotation Protobuf mới `cosmos.query.v1.module_query_safe`, được sử dụng để khai báo rằng một query an toàn khi được gọi từ bên trong state machine, ví dụ:

* Hàm query của một Keeper có thể được gọi từ Keeper của module khác,
* Các lời gọi query liên module theo ADR-033,
* Các contract CosmWasm cũng có thể tương tác trực tiếp với các query này.

Nếu annotation `module_query_safe` được đặt thành `true`, có nghĩa là:

* Query có tính xác định (deterministic): cho một chiều cao block nhất định, nó sẽ trả về cùng phản hồi khi được gọi nhiều lần, và không tạo ra bất kỳ thay đổi state-machine-breaking nào giữa các phiên bản SDK patch.
* Mức tiêu thụ gas không thay đổi giữa các lần gọi và giữa các phiên bản patch.

Nếu bạn là nhà phát triển module và muốn sử dụng annotation `module_query_safe` cho query của mình, bạn phải đảm bảo những điều sau:

* Query có tính xác định và sẽ không tạo ra các thay đổi state-machine-breaking mà không có nâng cấp phối hợp
* Gas của nó được theo dõi, để tránh vector tấn công nơi không có gas được tính cho các query có thể tốn nhiều tài nguyên tính toán.
