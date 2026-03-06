# ADR 068: Preblock

## Changelog

* Sept 13, 2023: Bản nháp ban đầu

## Trạng thái

BẢN NHÁP

## Tóm tắt

Giới thiệu `PreBlock`, chạy trước begin blocker của các module khác, và cho phép
chỉnh sửa tham số đồng thuận (consensus parameters); các thay đổi này sẽ hiển thị
(visible) cho các logic state machine thực thi sau đó.

## Bối cảnh

Khi nâng cấp lên sdk 0.47, định dạng lưu trữ của consensus parameters đã thay đổi,
nhưng trong block migration, `ctx.ConsensusParams()` luôn là `nil`, vì nó không thể
tải định dạng cũ bằng code mới. Đáng lẽ nó phải được migrate bởi module `x/upgrade`
trước, nhưng không may, migration xảy ra trong handler `BeginBlocker`, vốn chạy
sau khi `ctx` đã được khởi tạo.

Khi cố gắng giải quyết điều này, ta thấy module `x/upgrade` không thể sửa context
để khiến consensus parameters hiển thị cho các module khác, vì context được truyền
bằng giá trị (passed by value), và đội sdk muốn giữ như vậy — điều này tốt cho
tính cô lập giữa các module.

## Phương án thay thế

Giải pháp thay thế đầu tiên giới thiệu `MigrateModuleManager`, hiện chỉ bao gồm
module `x/upgrade`. Baseapp sẽ chạy `BeginBlocker` của chúng trước các module khác,
và tải lại consensus parameters của context ở giữa.

## Quyết định

Đề xuất phương thức vòng đời (lifecycle) mới này.

### `PreBlocker`

Có hai ngữ nghĩa xung quanh phương thức vòng đời mới:

* Nó chạy trước `BeginBlocker` của tất cả module
* Nó có thể sửa consensus parameters trong storage, và báo hiệu cho caller thông
  qua giá trị trả về.

Khi nó trả về `ConsensusParamsChanged=true`, caller phải refresh consensus parameters
trong finalize context:

```
app.finalizeBlockState.ctx = app.finalizeBlockState.ctx.WithConsensusParams(app.GetConsensusParams())
```

Context mới phải được truyền tới tất cả các phương thức vòng đời còn lại.

## Hệ quả

### Tương thích ngược

### Tích cực

### Tiêu cực

### Trung tính

## Thảo luận thêm

## Test Cases

## Tham khảo

* [1] https://github.com/cosmos/cosmos-sdk/issues/16494
* [2] https://github.com/cosmos/cosmos-sdk/pull/16583
* [3] https://github.com/cosmos/cosmos-sdk/pull/17421
* [4] https://github.com/cosmos/cosmos-sdk/pull/17713

