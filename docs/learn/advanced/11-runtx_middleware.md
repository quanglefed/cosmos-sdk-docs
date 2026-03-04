---
sidebar_position: 1
---

# Middleware Phục Hồi RunTx

Hàm `BaseApp.runTx()` xử lý các Go panic có thể xảy ra trong quá trình thực thi giao dịch — ví dụ, khi keeper gặp trạng thái không hợp lệ và gây ra panic. Tùy thuộc vào loại panic, các handler khác nhau sẽ được sử dụng; chẳng hạn, handler mặc định in ra thông báo lỗi vào log. Recovery middleware được dùng để thêm xử lý phục hồi panic tùy chỉnh cho các nhà phát triển ứng dụng Cosmos SDK.

Thêm ngữ cảnh có thể tìm thấy trong [ADR-022](../../build/architecture/adr-022-custom-panic-handling.md) và phần triển khai trong [recovery.go](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/baseapp/recovery.go).

## Giao diện (Interface)

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/baseapp/recovery.go#L14-L17
```

`recoveryObj` là giá trị trả về của hàm `recover()` từ package `builtin` của Go.

**Hợp đồng (Contract):**

* RecoveryHandler trả về `nil` nếu `recoveryObj` chưa được xử lý và cần được chuyển đến middleware phục hồi tiếp theo.
* RecoveryHandler trả về một `error` khác nil nếu `recoveryObj` đã được xử lý.

## Đăng ký RecoveryHandler tùy chỉnh

`BaseApp.AddRunTxRecoveryHandler(handlers ...RecoveryHandler)`

Phương thức BaseApp này thêm recovery middleware vào chuỗi phục hồi mặc định.

## Ví dụ

Giả sử chúng ta muốn phát ra trạng thái chuỗi "Consensus failure" nếu một lỗi cụ thể xảy ra.

Chúng ta có một module keeper gây ra panic:

```go
func (k FooKeeper) Do(obj interface{}) {
    if obj == nil {
        // điều đó không nên xảy ra, cần crash ứng dụng
        err := errorsmod.Wrap(fooTypes.InternalError, "obj is nil")
        panic(err)
    }
}
```

Theo mặc định, panic đó sẽ được phục hồi và thông báo lỗi sẽ được in vào log. Để ghi đè hành vi đó, chúng ta nên đăng ký một RecoveryHandler tùy chỉnh:

```go
// Hàm khởi tạo ứng dụng Cosmos SDK
customHandler := func(recoveryObj interface{}) error {
    err, ok := recoveryObj.(error)
    if !ok {
        return nil
    }

    if fooTypes.InternalError.Is(err) {
        panic(fmt.Errorf("FooKeeper đã panic với lỗi: %w", err))
    }

    return nil
}

baseApp := baseapp.NewBaseApp(...)
baseApp.AddRunTxRecoveryHandler(customHandler)
```
