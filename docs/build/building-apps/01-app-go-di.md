---
sidebar_position: 1
---

# Tổng Quan Về `app_di.go`

:::note Tóm tắt

Cosmos SDK giúp việc wiring một `app.go` trở nên dễ dàng hơn nhiều nhờ [runtime](./00-runtime.md) và app wiring.
Tìm hiểu thêm về lý do của App Wiring trong [ADR-057](../../../architecture/adr-057-app-wiring.md).

:::

:::note Đọc Trước

* [`runtime` Là Gì?](./00-runtime.md)
* [Tài liệu Depinject](../packages/01-depinject.md)
* [Các module hỗ trợ depinject](../building-modules/15-depinject.md)
* [ADR 057: App Wiring](../../../architecture/adr-057-app-wiring.md)

:::

Phần này nhằm cung cấp tổng quan về file `app_di.go` của `SimApp` với App Wiring.

## `app_config.go`

File `app_config.go` là nơi duy nhất để cấu hình tất cả các tham số module.

1. Tạo biến `AppConfig`:

    ```go reference
    https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/simapp/app_config.go#L289-L303
    ```

    Trong đó `appConfig` kết hợp cấu hình [runtime](./00-runtime.md) và cấu hình các module (bổ sung).

    ```go reference
    https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/simapp/app_di.go#L113-L161
    ```

2. Cấu hình module `runtime`:

    Trong cấu hình này, thứ tự mà các module được định nghĩa trong PreBlockers, BeginBlocks và EndBlockers là quan trọng. Chúng được đặt tên theo thứ tự chúng nên được thực thi bởi module manager.

    ```go reference
    https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/simapp/app_config.go#L103-L188
    ```

3. Wiring các module khác:

    Bên cạnh runtime, các module khác (hỗ trợ depinject) được wired trong `AppConfig`:

    ```go reference
    https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/simapp/app_config.go#L103-L286
    ```

    Lưu ý: `tx` không phải là module mà là cấu hình. Nó cũng nên được wired trong `AppConfig`.

    ```go reference
    https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/simapp/app_config.go#L222-L227
    ```

Xem file `app_config.go` đầy đủ cho `SimApp` [tại đây](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/simapp/app_config.go).

### Các Định Dạng Thay Thế

:::tip
Ví dụ trên cho thấy cách tạo `AppConfig` bằng Go. Tuy nhiên, cũng có thể tạo `AppConfig` bằng YAML hoặc JSON.
Cấu hình sau đó có thể được nhúng với `go:embed` và đọc bằng [`appconfig.LoadYAML`](https://pkg.go.dev/cosmossdk.io/core/appconfig#LoadYAML) hoặc [`appconfig.LoadJSON`](https://pkg.go.dev/cosmossdk.io/core/appconfig#LoadJSON) trong `app_di.go`.

```go
//go:embed app_config.yaml
var (
    appConfigYaml []byte
    appConfig = appconfig.LoadYAML(appConfigYaml)
)
```

:::

```yaml
modules:
  - name: runtime
    config:
      "@type": cosmos.app.runtime.v1alpha1.Module
      app_name: SimApp
      begin_blockers: [staking, auth, bank]
      end_blockers: [bank, auth, staking]
      init_genesis: [bank, auth, staking]
  - name: auth
    config:
      "@type": cosmos.auth.module.v1.Module
      bech32_prefix: cosmos
  - name: bank
    config:
      "@type": cosmos.bank.module.v1.Module
  - name: staking
    config:
      "@type": cosmos.staking.module.v1.Module
  - name: tx
    config:
      "@type": cosmos.tx.module.v1.Module
```

Có thể tìm thấy ví dụ đầy đủ hơn về `app.yaml` [tại đây](https://github.com/cosmos/cosmos-sdk/blob/release/v0.53.x/simapp/example_app.yaml).

## `app_di.go`

`app_di.go` là nơi `SimApp` được xây dựng. `depinject.Inject` tự động wire các module và keeper của ứng dụng khi được cung cấp cấu hình ứng dụng (`AppConfig`). `SimApp` được xây dựng khi gọi `*runtime.AppBuilder` được inject với `appBuilder.Build(...)`.
Tóm lại, `depinject` và [package `runtime`](./00-runtime.md) trừu tượng hóa việc wiring của ứng dụng, và `AppBuilder` là nơi ứng dụng được xây dựng. [`runtime`](./00-runtime.md) đảm nhận việc đăng ký các codec, KV store, subspace và khởi tạo `baseapp`.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/simapp/app_di.go#L100-L270
```

:::warning
Khi sử dụng `depinject.Inject`, các kiểu được inject phải là con trỏ.
:::

### Cấu Hình Nâng Cao

Trong các trường hợp nâng cao, có thể inject thêm cấu hình (module) theo cách mà `AppConfig` chưa hỗ trợ. Trong trường hợp này, sử dụng `depinject.Configs` để kết hợp cấu hình bổ sung, và `AppConfig` cùng `depinject.Supply` để cung cấp cấu hình bổ sung. Thông tin thêm về cách `depinject.Configs` và `depinject.Supply` hoạt động có thể tìm thấy trong [tài liệu `depinject`](https://pkg.go.dev/cosmossdk.io/depinject).

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/simapp/app_di.go#L114-L162
```

### Đăng Ký Các Module Không Dùng App Wiring

Có thể kết hợp các module hỗ trợ app wiring / depinject với các module không dùng app wiring. Để làm vậy, sử dụng phương thức `app.RegisterModules` để đăng ký các module trên ứng dụng, cũng như `app.RegisterStores` để đăng ký các store bổ sung cần thiết.

```go
// ....
app.App = appBuilder.Build(db, traceStore, baseAppOptions...)

// đăng ký module thủ công
app.RegisterStores(storetypes.NewKVStoreKey(example.ModuleName))
app.ExampleKeeper = examplekeeper.NewKeeper(app.appCodec, app.AccountKeeper.AddressCodec(), runtime.NewKVStoreService(app.GetKey(example.ModuleName)), authtypes.NewModuleAddress(govtypes.ModuleName).String())
exampleAppModule := examplemodule.NewAppModule(app.ExampleKeeper)
if err := app.RegisterModules(&exampleAppModule); err != nil {
	panic(err)
}

// ....
```

:::warning
Khi sử dụng AutoCLI và kết hợp các module app wiring và non-app wiring, các tùy chọn AutoCLI nên được xây dựng thủ công thay vì inject. Nếu không, nó sẽ bỏ lỡ các module không dùng depinject và không đăng ký CLI của chúng.
:::

### `app_di.go` Đầy Đủ

:::tip
Lưu ý rằng trong file `app_di.go` đầy đủ của `SimApp`, các tiện ích kiểm thử cũng được định nghĩa, nhưng chúng cũng có thể được định nghĩa trong một file riêng biệt.
:::

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/simapp/app_di.go
```
