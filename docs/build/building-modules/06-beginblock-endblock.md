---
sidebar_position: 1
---

# BeginBlocker và EndBlocker

:::note Tóm tắt
`BeginBlocker` và `EndBlocker` là các phương thức tùy chọn mà nhà phát triển module có thể triển khai trong module của họ. Chúng sẽ được kích hoạt vào đầu và cuối của mỗi block tương ứng, khi các thông điệp ABCI [`BeginBlock`](../../learn/advanced/00-baseapp.md#beginblock) và [`EndBlock`](../../learn/advanced/00-baseapp.md#endblock) được nhận từ consensus engine bên dưới.
:::

:::note Yêu Cầu Đọc Trước

* [Module Manager](./01-module-manager.md)

:::

## BeginBlocker và EndBlocker

`BeginBlocker` và `EndBlocker` là cách cho nhà phát triển module thêm việc thực thi logic tự động vào module của họ. Đây là một công cụ mạnh mẽ cần được sử dụng cẩn thận, vì các hàm tự động phức tạp có thể làm chậm hoặc thậm chí dừng chain.

Trong phiên bản 0.47.0, Prepare và Process Proposal đã được thêm vào cho phép nhà phát triển ứng dụng thực hiện công việc tùy ý ở các giai đoạn đó, nhưng chúng không ảnh hưởng đến công việc sẽ được thực hiện trong BeginBlock. Nếu ứng dụng yêu cầu `BeginBlock` phải thực thi trước khi bất kỳ loại công việc nào được thực hiện thì điều này hiện không thể thực hiện được (0.50.0).

Khi cần, `BeginBlocker` và `EndBlocker` được triển khai như một phần của các [`interface HasBeginBlocker`, `HasABCIEndBlocker` và `EndBlocker`](./01-module-manager.md#appmodule). Điều này có nghĩa là cả hai đều có thể bị bỏ qua nếu không cần thiết. Các phương thức `BeginBlock` và `EndBlock` của interface được triển khai trong `module.go` thường ủy quyền cho các phương thức `BeginBlocker` và `EndBlocker` tương ứng, thường được triển khai trong `abci.go`.

Việc triển khai thực tế của `BeginBlocker` và `EndBlocker` trong `abci.go` rất tương tự với [`Msg` service](./03-msg-services.md):

* Chúng thường sử dụng [`keeper`](./06-keeper.md) và [`ctx`](../../learn/advanced/02-context.md) để lấy thông tin về trạng thái mới nhất.
* Nếu cần, chúng sử dụng `keeper` và `ctx` để kích hoạt các chuyển đổi trạng thái.
* Nếu cần, chúng có thể phát ra các [`sự kiện`](../../learn/advanced/08-events.md) thông qua `EventManager` của `ctx`.

Một loại `EndBlocker` đặc biệt có sẵn để trả về các bản cập nhật validator cho consensus engine bên dưới dưới dạng [`[]abci.ValidatorUpdates`](https://docs.cometbft.com/v0.37/spec/abci/abci++_methods#endblock). Đây là cách ưu tiên để triển khai các thay đổi validator tùy chỉnh.

Nhà phát triển có thể định nghĩa thứ tự thực thi giữa các hàm `BeginBlocker`/`EndBlocker` của từng module trong ứng dụng của họ thông qua các phương thức `SetOrderBeginBlocker`/`SetOrderEndBlocker` của module manager. Để biết thêm về module manager, nhấn vào [đây](./01-module-manager.md#manager).

Xem ví dụ triển khai `BeginBlocker` từ module `distribution`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/distribution/abci.go#L14-L38
```

và ví dụ triển khai `EndBlocker` từ module `staking`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/staking/keeper/abci.go#L22-L27
```
