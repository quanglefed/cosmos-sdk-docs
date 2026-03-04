---
sidebar_position: 1
---

# Invariants

:::note Tóm tắt
Invariant là một thuộc tính của ứng dụng phải luôn luôn đúng. Trong ngữ cảnh của Cosmos SDK, `Invariant` là một hàm kiểm tra một invariant cụ thể. Các hàm này hữu ích để phát hiện bug sớm và hành động dựa trên chúng để hạn chế hậu quả tiềm tàng (ví dụ: bằng cách dừng chain). Chúng cũng hữu ích trong quá trình phát triển ứng dụng để phát hiện bug thông qua simulation.
:::

:::note Yêu Cầu Đọc Trước

* [Keepers](./06-keeper.md)

:::

## Triển Khai `Invariant`

Một `Invariant` là một hàm kiểm tra một invariant cụ thể trong một module. Các `Invariant` của module phải tuân theo kiểu `Invariant`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/types/invariant.go#L9
```

Giá trị trả về `string` là thông điệp invariant, có thể được sử dụng khi in log, và giá trị trả về `bool` là kết quả thực tế của kiểm tra invariant.

Trong thực tế, mỗi module triển khai các `Invariant` trong file `keeper/invariants.go` trong thư mục của module. Tiêu chuẩn là triển khai một hàm `Invariant` cho mỗi nhóm logic của các invariant với mô hình sau:

```go
// Ví dụ cho Invariant kiểm tra các invariant liên quan đến số dư

func BalanceInvariants(k Keeper) sdk.Invariant {
	return func(ctx context.Context) (string, bool) {
        // Triển khai kiểm tra cho các invariant liên quan đến số dư
    }
}
```

Ngoài ra, nhà phát triển module thường nên triển khai một hàm `AllInvariants` chạy tất cả các hàm `Invariant` của module:

```go
// AllInvariants chạy tất cả invariants của module.
// Trong ví dụ này, module triển khai hai Invariant: BalanceInvariants và DepositsInvariants

func AllInvariants(k Keeper) sdk.Invariant {

	return func(ctx context.Context) (string, bool) {
		res, stop := BalanceInvariants(k)(ctx)
		if stop {
			return res, stop
		}

		return DepositsInvariant(k)(ctx)
	}
}
```

Cuối cùng, nhà phát triển module cần triển khai phương thức `RegisterInvariants` như một phần của [`AppModule` interface](./01-module-manager.md#appmodule). Thật vậy, phương thức `RegisterInvariants` của module, được triển khai trong file `module/module.go`, thường chỉ ủy quyền lời gọi đến phương thức `RegisterInvariants` được triển khai trong file `keeper/invariants.go`. Phương thức `RegisterInvariants` đăng ký một route cho mỗi hàm `Invariant` trong [`InvariantRegistry`](#invariant-registry):

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/staking/keeper/invariants.go#L12-L22
```

Để biết thêm, xem ví dụ về [triển khai `Invariant` từ module `staking`](https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/staking/keeper/invariants.go).

## Invariant Registry

`InvariantRegistry` là một registry nơi các `Invariant` của tất cả module trong một ứng dụng được đăng ký. Chỉ có một `InvariantRegistry` cho mỗi **ứng dụng**, có nghĩa là nhà phát triển module không cần triển khai `InvariantRegistry` của riêng họ khi xây dựng module. **Tất cả những gì nhà phát triển module cần làm là đăng ký các invariant của module trong `InvariantRegistry`, như đã giải thích ở phần trên**. Phần còn lại của phần này cung cấp thêm thông tin về bản thân `InvariantRegistry`, và không chứa bất cứ điều gì trực tiếp liên quan đến nhà phát triển module.

Về cốt lõi, `InvariantRegistry` được định nghĩa trong Cosmos SDK như một interface:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/types/invariant.go#L14-L17
```

Thông thường, interface này được triển khai trong `keeper` của một module cụ thể. Triển khai được sử dụng nhiều nhất của `InvariantRegistry` có thể tìm thấy trong module `crisis`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/crisis/keeper/keeper.go#L48-L50
```

Do đó `InvariantRegistry` thường được khởi tạo bằng cách khởi tạo `keeper` của module `crisis` trong [hàm constructor của ứng dụng](../../learn/beginner/00-app-anatomy.md#constructor-function).

Các `Invariant` có thể được kiểm tra thủ công qua [`message`](./02-messages-and-queries.md), nhưng thường xuyên nhất chúng được kiểm tra tự động vào cuối mỗi block. Đây là ví dụ từ module `crisis`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/crisis/abci.go#L13-L23
```

Trong cả hai trường hợp, nếu một trong các `Invariant` trả về false, `InvariantRegistry` có thể kích hoạt logic đặc biệt (ví dụ: làm cho ứng dụng panic và in thông điệp `Invariant` trong log).
