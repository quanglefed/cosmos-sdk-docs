---
sidebar_position: 1
---

# Context (Ngữ cảnh)

:::note Tóm tắt
`context` là một cấu trúc dữ liệu được thiết kế để truyền từ hàm này sang hàm khác, mang theo thông tin về trạng thái hiện tại của ứng dụng. Nó cung cấp quyền truy cập vào một branched storage (một nhánh an toàn của toàn bộ trạng thái) cũng như các đối tượng và thông tin hữu ích như `gasMeter`, `block height` (chiều cao block), `consensus parameters` (tham số đồng thuận) và nhiều hơn nữa.
:::

:::note Tài liệu cần đọc trước

* [Cấu trúc của một ứng dụng Cosmos SDK](../beginner/00-app-anatomy.md)
* [Vòng đời của một giao dịch](../beginner/01-tx-lifecycle.md)

:::

## Định nghĩa Context

`Context` của Cosmos SDK là một cấu trúc dữ liệu tùy chỉnh chứa [`context`](https://pkg.go.dev/context) của stdlib Go làm nền tảng, và có nhiều kiểu bổ sung trong định nghĩa của nó dành riêng cho Cosmos SDK. `Context` là thành phần không thể thiếu trong quá trình xử lý giao dịch vì nó cho phép các module dễ dàng truy cập [store](./04-store.md#base-layer-kvstores) tương ứng của chúng trong [`multistore`](./04-store.md#multistore) và lấy ngữ cảnh giao dịch như block header và gas meter.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/types/context.go#L40-L67
```

* **Base Context:** Kiểu cơ sở là một [Context](https://pkg.go.dev/context) của Go, được giải thích thêm trong phần [Go Context Package](#go-context-package) bên dưới.
* **Multistore:** `BaseApp` của mỗi ứng dụng chứa một [`CommitMultiStore`](./04-store.md#multistore) được cung cấp khi tạo `Context`. Gọi các phương thức `KVStore()` và `TransientStore()` cho phép các module lấy [`KVStore`](./04-store.md#base-layer-kvstores) tương ứng của chúng bằng `StoreKey` duy nhất.
* **Header:** [header](https://docs.cometbft.com/v0.37/spec/core/data_structures#header) là một kiểu Blockchain. Nó mang thông tin quan trọng về trạng thái của blockchain, chẳng hạn như chiều cao block và proposer của block hiện tại.
* **Header Hash:** Hash header của block hiện tại, được lấy trong `abci.FinalizeBlock`.
* **Chain ID:** Số định danh duy nhất của blockchain mà block thuộc về.
* **Transaction Bytes:** Biểu diễn `[]byte` của giao dịch đang được xử lý bằng context. Mỗi giao dịch được xử lý bởi nhiều phần khác nhau của Cosmos SDK và consensus engine (ví dụ: CometBFT) xuyên suốt [vòng đời](../beginner/01-tx-lifecycle.md) của nó, một số phần không hiểu các kiểu giao dịch. Do đó, giao dịch được marshal thành kiểu `[]byte` chung bằng một số [định dạng mã hóa](./05-encoding.md) như [Amino](./05-encoding.md).
* **Logger:** Một `logger` từ các thư viện CometBFT. Tìm hiểu thêm về log [ở đây](https://docs.cometbft.com/v0.37/core/configuration). Các module gọi phương thức này để tạo logger riêng dành cho module của họ.
* **VoteInfo:** Danh sách kiểu ABCI [`VoteInfo`](https://docs.cometbft.com/main/spec/abci/abci++_methods.html#voteinfo), bao gồm tên của validator và boolean cho biết họ có ký block không.
* **Gas Meters:** Cụ thể là [`gasMeter`](../beginner/04-gas-fees.md#main-gas-meter) cho giao dịch hiện đang được xử lý bằng context và [`blockGasMeter`](../beginner/04-gas-fees.md#block-gas-meter) cho toàn bộ block mà nó thuộc về. Người dùng chỉ định số tiền phí họ muốn trả cho việc thực thi giao dịch; các gas meter này theo dõi lượng [gas](../beginner/04-gas-fees.md) đã sử dụng trong giao dịch hoặc block cho đến nay. Nếu gas meter cạn kiệt, quá trình thực thi dừng lại.
* **CheckTx Mode:** Giá trị boolean cho biết giao dịch có nên được xử lý trong chế độ `CheckTx` hay `DeliverTx`.
* **Min Gas Price:** Giá [gas](../beginner/04-gas-fees.md) tối thiểu mà một node sẵn sàng chấp nhận để đưa giao dịch vào block của nó. Giá này là giá trị cục bộ được cấu hình bởi từng node riêng lẻ, do đó **không nên được sử dụng trong bất kỳ hàm nào dẫn đến chuyển đổi trạng thái**.
* **Consensus Params:** Kiểu ABCI [Consensus Parameters](https://docs.cometbft.com/v0.37/spec/abci/abci++_app_requirements#consensus-parameters), chỉ định một số giới hạn nhất định cho blockchain, chẳng hạn như gas tối đa cho một block.
* **Event Manager:** Event manager cho phép bất kỳ caller nào có quyền truy cập vào `Context` để phát ra [`Events`](./08-events.md). Các module có thể định nghĩa `Events` dành riêng cho module bằng cách định nghĩa nhiều `Types` và `Attributes` hoặc dùng các định nghĩa chung trong `types/`. Các client có thể đăng ký hoặc truy vấn các `Events` này. Các `Events` này được thu thập trong suốt `FinalizeBlock` và được trả về cho CometBFT để lập chỉ mục.
* **Priority:** Mức độ ưu tiên của giao dịch, chỉ có liên quan trong `CheckTx`.
* **KV `GasConfig`:** Cho phép ứng dụng đặt `GasConfig` tùy chỉnh cho `KVStore`.
* **Transient KV `GasConfig`:** Cho phép ứng dụng đặt `GasConfig` tùy chỉnh cho `KVStore` tạm thời.
* **StreamingManager:** Trường `streamingManager` cung cấp quyền truy cập vào streaming manager, cho phép các module đăng ký các thay đổi trạng thái được phát ra bởi blockchain. Streaming manager được sử dụng bởi API lắng nghe trạng thái (state listening API), được mô tả trong [ADR 038](https://docs.cosmos.network/main/architecture/adr-038-state-listening).
* **CometInfo:** Trường nhẹ chứa thông tin về block hiện tại, chẳng hạn như chiều cao block, thời gian và hash. Thông tin này có thể được dùng để xác thực bằng chứng, cung cấp dữ liệu lịch sử và cải thiện trải nghiệm người dùng. Xem thêm chi tiết [ở đây](https://github.com/cosmos/cosmos-sdk/blob/main/core/comet/service.go#L14).
* **HeaderInfo:** Trường `headerInfo` chứa thông tin về header của block hiện tại, chẳng hạn như chain ID, giới hạn gas và timestamp. Xem thêm chi tiết [ở đây](https://github.com/cosmos/cosmos-sdk/blob/main/core/header/service.go#L14).

## Go Context Package

Một `Context` cơ bản được định nghĩa trong [Golang Context Package](https://pkg.go.dev/context). Một `Context` là cấu trúc dữ liệu bất biến (immutable) mang dữ liệu có phạm vi yêu cầu (request-scoped) qua các API và tiến trình. Context cũng được thiết kế để hỗ trợ concurrency và sử dụng trong goroutine.

Context được thiết kế để **bất biến**; chúng không bao giờ nên được chỉnh sửa. Thay vào đó, quy ước là tạo một context con từ context cha bằng hàm `With`. Ví dụ:

```go
childCtx = parentCtx.WithBlockHeader(header)
```

Tài liệu [Golang Context Package](https://pkg.go.dev/context) hướng dẫn các nhà phát triển luôn truyền context `ctx` một cách tường minh như tham số đầu tiên của một tiến trình.

## Store branching

`Context` chứa một `MultiStore`, cho phép chức năng branching và caching bằng `CacheMultiStore` (các query trong `CacheMultiStore` được cache để tránh các round trip trong tương lai). Mỗi `KVStore` được branch trong một vùng lưu trữ tạm thời (ephemeral storage) an toàn và riêng biệt. Các tiến trình có thể tự do ghi các thay đổi vào `CacheMultiStore`. Nếu một chuỗi chuyển đổi trạng thái được thực hiện mà không gặp vấn đề, store branch có thể được commit vào store bên dưới (underlying store) ở cuối chuỗi, hoặc bị hủy bỏ nếu có gì đó sai xảy ra. Mẫu sử dụng cho Context như sau:

1. Một tiến trình nhận Context `ctx` từ tiến trình cha của nó, cung cấp thông tin cần thiết để thực hiện tiến trình.
2. `ctx.ms` là một **branched store**, tức là một nhánh của [multistore](./04-store.md#multistore) được tạo ra để tiến trình có thể thực hiện các thay đổi trạng thái trong khi thực thi mà không ảnh hưởng đến `ctx.ms` gốc. Điều này hữu ích để bảo vệ multistore bên dưới trong trường hợp các thay đổi cần được hoàn nguyên tại một thời điểm nào đó trong quá trình thực thi.
3. Tiến trình có thể đọc và ghi từ `ctx` trong khi thực thi. Nó có thể gọi tiến trình con và truyền `ctx` cho nó khi cần.
4. Khi một tiến trình con trả về, nó kiểm tra xem kết quả là thành công hay thất bại. Nếu thất bại, không cần làm gì — `ctx` branch đơn giản bị hủy bỏ. Nếu thành công, các thay đổi được thực hiện với `CacheMultiStore` có thể được commit vào `ctx.ms` gốc thông qua `Write()`.

Ví dụ, đây là đoạn code từ hàm [`runTx`](./00-baseapp.md#runtx-antehandler-runmsgs-posthandler) trong [`baseapp`](./00-baseapp.md):

```go
runMsgCtx, msCache := app.cacheTxContext(ctx, txBytes)
result = app.runMsgs(runMsgCtx, msgs, mode)
result.GasWanted = gasWanted
if mode != runTxModeDeliver {
  return result
}
if result.IsOK() {
  msCache.Write()
}
```

Đây là quy trình:

1. Trước khi gọi `runMsgs` trên (các) message trong giao dịch, nó dùng `app.cacheTxContext()` để branch và cache context và multistore.
2. `runMsgCtx` — context với branched store, được dùng trong `runMsgs` để trả về kết quả.
3. Nếu tiến trình đang chạy trong [`checkTxMode`](./00-baseapp.md#checktx), không cần ghi các thay đổi — kết quả được trả về ngay lập tức.
4. Nếu tiến trình đang chạy trong [`deliverTxMode`](./00-baseapp.md#delivertx) và kết quả cho thấy đã chạy thành công trên tất cả các message, thì branched multistore được ghi lại vào bản gốc.
