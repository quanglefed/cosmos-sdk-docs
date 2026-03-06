---
sidebar_position: 1
---

# `x/epochs`

## Tóm tắt

Trong SDK, ta thường muốn chạy một số đoạn code theo chu kỳ. Mục đích của module
`epochs` là cho phép các module khác đăng ký rằng chúng muốn được “báo hiệu”
(signaled) mỗi khi hết một chu kỳ. Vì vậy một module khác có thể chỉ định rằng
nó muốn thực thi code mỗi tuần, bắt đầu tại thời điểm UTC = x. `epochs` tạo một
interface epoch tổng quát cho các module khác để chúng có thể dễ dàng nhận tín
hiệu khi các sự kiện như vậy xảy ra.

## Nội dung

1. **[Khái niệm](#khái-niệm)**
2. **[State](#state)**
3. **[Events](#events)**
4. **[Keeper](#keeper)**
5. **[Hooks](#hooks)**
6. **[Query](#query)**

## Khái niệm

Module epochs định nghĩa các “timer” on-chain chạy theo các khoảng thời gian cố định.
Các module SDK khác có thể đăng ký logic để được thực thi tại mỗi nhịp (tick) của timer.
Ta gọi khoảng thời gian giữa hai nhịp tick là một “epoch”.

Mỗi timer có một định danh (identifier) duy nhất.
Mỗi epoch có thời điểm bắt đầu và kết thúc, trong đó `end time = start time + timer interval`.
Trên mainnet, ta chỉ dùng một identifier với khoảng thời gian `một ngày`.

Timer sẽ tick ở block đầu tiên có block time lớn hơn end time của timer, và đặt start
time mới bằng end time của timer trước đó. (Đáng chú ý: nó không đặt bằng block time!)
Điều này nghĩa là nếu chain bị down một thời gian, bạn sẽ nhận được một timer tick
mỗi block cho tới khi timer “bắt kịp”.

## State

Module Epochs giữ một `EpochInfo` cho mỗi identifier.
Đối tượng này chứa trạng thái hiện tại của timer ứng với identifier đó.
Các field của nó được sửa ở mỗi lần timer tick.
EpochInfo được khởi tạo như một phần của khởi tạo genesis hoặc logic upgrade, và
chỉ được sửa trong begin blocker.

## Events

Module `epochs` phát ra các event sau:

### BeginBlocker

| Type        | Attribute Key | Attribute Value |
| ----------- | ------------- | --------------- |
| epoch_start | epoch_number  | {epoch_number}  |
| epoch_start | start_time    | {start_time}    |

### EndBlocker

| Type      | Attribute Key | Attribute Value |
| --------- | ------------- | --------------- |
| epoch_end | epoch_number  | {epoch_number}  |

## Keeper

### Hàm của keeper

Keeper của module epochs cung cấp các hàm tiện ích để quản lý epoch.

## Hooks

```go
  // the first block whose timestamp is after the duration is counted as the end of the epoch
  AfterEpochEnd(ctx sdk.Context, epochIdentifier string, epochNumber int64)
  // new epoch is next block of epoch end block
  BeforeEpochStart(ctx sdk.Context, epochIdentifier string, epochNumber int64)
```

### Cách module nhận hook

Trong hàm nhận hook của module khác, họ cần lọc `epochIdentifier` và chỉ thực thi
logic cho identifier cụ thể. Việc lọc epochIdentifier có thể đặt trong `Params` của
module khác để có thể bị sửa bởi governance.

Đây là UX dev tiêu chuẩn:

```golang
func (k MyModuleKeeper) AfterEpochEnd(ctx sdk.Context, epochIdentifier string, epochNumber int64) {
    params := k.GetParams(ctx)
    if epochIdentifier == params.DistrEpochIdentifier {
    // my logic
  }
}
```

### Cô lập panic

Nếu một epoch hook panic, cập nhật state của hook đó sẽ bị revert, nhưng ta vẫn
tiếp tục chạy các hook còn lại. Điều này cho phép dùng logic epoch nâng cao mà
không lo state machine bị halt, hoặc làm halt các module chạy sau.

Điều này cũng nghĩa là nếu hook của bạn kỳ vọng một hành vi từ hook trước đó, và
hook trước đó đã bị revert, hook của bạn cũng có thể gặp vấn đề. Vì vậy hãy luôn
tính tới tình huống “nếu hook trước đó không chạy” trong các kiểm tra an toàn
khi viết epoch hook mới.

## Query

Module Epochs cung cấp các truy vấn sau để kiểm tra state của module.

```protobuf
service Query {
  // EpochInfos provide running epochInfos
  rpc EpochInfos(QueryEpochsInfoRequest) returns (QueryEpochsInfoResponse) {}
  // CurrentEpoch provide current epoch of specified identifier
  rpc CurrentEpoch(QueryCurrentEpochRequest) returns (QueryCurrentEpochResponse) {}
}
```

### Epoch Infos

Truy vấn danh sách epochInfos đang chạy:

```sh
<appd> query epochs epoch-infos
```

::::details Ví dụ

Ví dụ output:

```sh
epochs:
- current_epoch: "183"
  current_epoch_start_height: "2438409"
  current_epoch_start_time: "2021-12-18T17:16:09.898160996Z"
  duration: 86400s
  epoch_counting_started: true
  identifier: day
  start_time: "2021-06-18T17:00:00Z"
- current_epoch: "26"
  current_epoch_start_height: "2424854"
  current_epoch_start_time: "2021-12-17T17:02:07.229632445Z"
  duration: 604800s
  epoch_counting_started: true
  identifier: week
  start_time: "2021-06-18T17:00:00Z"
```

::::

### Current Epoch

Truy vấn epoch hiện tại theo identifier chỉ định:

```sh
<appd> query epochs current-epoch [identifier]
```

::::details Ví dụ

Truy vấn epoch `day` hiện tại:

```sh
<appd> query epochs current-epoch day
```

Trong ví dụ này sẽ trả về:

```sh
current_epoch: "183"
```

::::

