---
sidebar_position: 1
---

# Nâng Cấp Modules

:::note Tóm tắt
[In-Place Store Migrations](../../learn/advanced/15-upgrade.md) cho phép các module của bạn nâng cấp lên các phiên bản mới có các thay đổi phá vỡ (breaking changes). Tài liệu này trình bày cách xây dựng các module để tận dụng chức năng này.
:::

:::note Yêu Cầu Đọc Trước

* [In-Place Store Migration](../../learn/advanced/15-upgrade.md)

:::

## Consensus Version (Phiên Bản Đồng Thuận)

Nâng cấp thành công các module hiện có yêu cầu mỗi `AppModule` phải triển khai hàm `ConsensusVersion() uint64`.

* Các phiên bản phải được hard-code bởi nhà phát triển module.
* Phiên bản ban đầu **phải** được đặt thành 1.

Consensus version đóng vai trò là phiên bản state-breaking của các app module và phải được tăng lên khi module giới thiệu các thay đổi phá vỡ.

## Đăng Ký Migration

Để đăng ký chức năng diễn ra trong quá trình nâng cấp module, bạn phải đăng ký migration nào bạn muốn thực hiện.

Đăng ký migration diễn ra trong `Configurator` bằng cách sử dụng phương thức `RegisterMigration`. Tham chiếu `AppModule` đến configurator nằm trong phương thức `RegisterServices`.

Bạn có thể đăng ký một hoặc nhiều migration. Nếu bạn đăng ký nhiều hơn một migration script, liệt kê các migration theo thứ tự tăng dần và đảm bảo có đủ migration dẫn đến consensus version mong muốn. Ví dụ, để migrate đến phiên bản 3 của một module, đăng ký các migration riêng biệt cho phiên bản 1 và 2 như trong ví dụ sau:

```go
func (am AppModule) RegisterServices(cfg module.Configurator) {
    // --snip--
    cfg.RegisterMigration(types.ModuleName, 1, func(ctx sdk.Context) error {
        // Thực hiện in-place store migrations từ ConsensusVersion 1 sang 2.
    })
     cfg.RegisterMigration(types.ModuleName, 2, func(ctx sdk.Context) error {
        // Thực hiện in-place store migrations từ ConsensusVersion 2 sang 3.
    })
}
```

Vì các migration này là các hàm cần truy cập store của Keeper, hãy sử dụng một wrapper xung quanh các keeper gọi là `Migrator` như trong ví dụ này:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/bank/keeper/migrations.go
```

## Viết Migration Scripts

Để định nghĩa chức năng diễn ra trong quá trình nâng cấp, hãy viết migration script và đặt các hàm vào thư mục `migrations/`. Ví dụ, để viết migration script cho module bank, đặt các hàm trong `x/bank/migrations/`. Sử dụng quy ước đặt tên được khuyến nghị cho các hàm này. Ví dụ, `v2bank` là script migrate gói `x/bank/migrations/v2`:

```go
// Migrating bank module từ phiên bản 1 sang 2
func (m Migrator) Migrate1to2(ctx sdk.Context) error {
	return v2bank.MigrateStore(ctx, m.keeper.storeKey) // v2bank là gói `x/bank/migrations/v2`.
}
```

Để xem mã ví dụ về các thay đổi được triển khai trong một migration của các balance keys, hãy xem [migrateBalanceKeys](https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/bank/migrations/v2/store.go#L55-L76). Để biết ngữ cảnh, mã này giới thiệu các migration của bank store cập nhật các địa chỉ được tiền tố bởi độ dài của chúng theo byte như được trình bày trong [ADR-028](../../../architecture/adr-028-public-key-addresses.md).
