---
sidebar_position: 1
---

# PreBlocker

:::note Tóm tắt
`PreBlocker` là một phương thức tùy chọn mà nhà phát triển module có thể triển khai trong module của họ. Chúng sẽ được kích hoạt trước [`BeginBlock`](../../learn/advanced/00-baseapp.md#beginblock).
:::

:::note Yêu Cầu Đọc Trước

* [Module Manager](./01-module-manager.md)

:::

## PreBlocker

Có hai ngữ nghĩa xung quanh phương thức vòng đời mới này:

* Nó chạy trước `BeginBlocker` của tất cả các module
* Nó có thể sửa đổi các consensus parameter trong storage, và báo hiệu cho người gọi thông qua giá trị trả về.

Khi nó trả về `ConsensusParamsChanged=true`, người gọi phải làm mới các consensus parameter trong deliver context:

```
app.finalizeBlockState.ctx = app.finalizeBlockState.ctx.WithConsensusParams(app.GetConsensusParams())
```

Context mới phải được truyền cho tất cả các phương thức vòng đời khác.
