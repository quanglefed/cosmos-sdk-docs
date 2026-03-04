---
sidebar_position: 1
---

# Telemetry (Đo từ xa)

:::note Tóm tắt
Thu thập các thông tin chi tiết có giá trị về ứng dụng và module của bạn với các metric và telemetry tùy chỉnh.
:::

Cosmos SDK cho phép các operator và nhà phát triển có cái nhìn sâu sắc về hiệu suất và hành vi của ứng dụng thông qua việc sử dụng package `telemetry`. Để bật telemetry, đặt `telemetry.enabled = true` trong file cấu hình app.toml.

Cosmos SDK hiện hỗ trợ bật in-memory và prometheus làm telemetry sink. In-memory sink luôn được gắn (khi telemetry được bật) với khoảng thời gian 10 giây và thời gian lưu giữ 1 phút. Điều này có nghĩa là các metric sẽ được tổng hợp trong 10 giây, và metric sẽ được giữ lại trong 1 phút.

Để query các metric đang hoạt động (xem ghi chú về lưu giữ ở trên), bạn phải bật API server (`api.enabled = true` trong app.toml). Một endpoint API duy nhất được cung cấp: `http://localhost:1317/metrics?format={text|prometheus}`, mặc định là `text`.

## Phát ra metric

Nếu telemetry được bật thông qua cấu hình, một bộ thu metric toàn cục duy nhất được đăng ký thông qua thư viện [go-metrics](https://github.com/hashicorp/go-metrics). Điều này cho phép phát ra và thu thập metric thông qua [API](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/telemetry/wrapper.go) đơn giản. Ví dụ:

```go
func EndBlocker(ctx sdk.Context, k keeper.Keeper) {
  defer telemetry.ModuleMeasureSince(types.ModuleName, time.Now(), telemetry.MetricKeyEndBlocker)

  // ...
}
```

Các nhà phát triển có thể dùng trực tiếp package `telemetry`, cung cấp các wrapper xung quanh API metric có thêm các label hữu ích, hoặc phải dùng trực tiếp thư viện `go-metrics`. Nên thêm càng nhiều ngữ cảnh và chiều (dimensionality) phù hợp vào metric càng tốt, vì vậy package `telemetry` được khuyến nghị. Bất kể package hoặc phương thức nào được sử dụng, Cosmos SDK hỗ trợ các kiểu metric sau:

* gauges (đồng hồ đo)
* summaries (tóm tắt)
* counters (bộ đếm)

## Labels

Một số thành phần của module sẽ có tên của chúng tự động được thêm làm label (ví dụ: `BeginBlock`). Các operator cũng có thể cung cấp cho ứng dụng một tập hợp label toàn cục sẽ được áp dụng cho tất cả metric được phát ra bằng package `telemetry` (ví dụ: chain-id). Label toàn cục được cung cấp dưới dạng danh sách các tuple [name, value].

Ví dụ:

```toml
global-labels = [
  ["chain_id", "chain-OfXo4V"],
]
```

## Cardinality (Lực lượng)

Cardinality là yếu tố then chốt, đặc biệt là cardinality của label và key. Cardinality là số lượng giá trị duy nhất của một thứ gì đó. Do đó, tự nhiên có sự đánh đổi giữa độ chi tiết và áp lực đặt lên telemetry sink về hiệu suất lập chỉ mục, scrape và query.

Các nhà phát triển nên chú ý hỗ trợ metric với đủ chiều và độ chi tiết để hữu ích, nhưng không tăng cardinality vượt quá giới hạn của sink. Quy tắc chung là không vượt quá cardinality là 10.

Hãy xem xét các ví dụ sau có đủ độ chi tiết và cardinality phù hợp:

* thời gian begin/end blocker
* gas giao dịch đã sử dụng
* gas block đã sử dụng
* lượng token đã mint
* số lượng tài khoản đã tạo

Các ví dụ sau tiết lộ quá nhiều cardinality và thậm chí có thể không hữu ích:

* chuyển khoản giữa các tài khoản với số lượng
* lượng bỏ phiếu/ký gửi từ các địa chỉ duy nhất

## Các Metric được Hỗ trợ

| Metric                          | Mô tả                                                                                              | Đơn vị          | Kiểu    |
|:--------------------------------|:---------------------------------------------------------------------------------------------------|:----------------|:--------|
| `tx_count`                      | Tổng số giao dịch được xử lý qua `DeliverTx`                                                      | tx              | counter |
| `tx_successful`                 | Tổng số giao dịch thành công được xử lý qua `DeliverTx`                                           | tx              | counter |
| `tx_failed`                     | Tổng số giao dịch thất bại được xử lý qua `DeliverTx`                                             | tx              | counter |
| `tx_gas_used`                   | Tổng lượng gas được sử dụng bởi giao dịch                                                         | gas             | gauge   |
| `tx_gas_wanted`                 | Tổng lượng gas được yêu cầu bởi giao dịch                                                         | gas             | gauge   |
| `tx_msg_send`                   | Tổng lượng token được gửi trong `MsgSend` (theo denom)                                            | token           | gauge   |
| `tx_msg_withdraw_reward`        | Tổng lượng token được rút trong `MsgWithdrawDelegatorReward` (theo denom)                         | token           | gauge   |
| `tx_msg_withdraw_commission`    | Tổng lượng token được rút trong `MsgWithdrawValidatorCommission` (theo denom)                     | token           | gauge   |
| `tx_msg_delegate`               | Tổng lượng token được ủy quyền trong `MsgDelegate`                                                | token           | gauge   |
| `tx_msg_begin_unbonding`        | Tổng lượng token bị hủy ủy quyền trong `MsgUndelegate`                                            | token           | gauge   |
| `tx_msg_begin_redelegate`       | Tổng lượng token được tái ủy quyền trong `MsgBeginRedelegate`                                     | token           | gauge   |
| `tx_msg_ibc_transfer`           | Tổng lượng token được chuyển qua IBC trong `MsgTransfer` (source hoặc sink chain)                 | token           | gauge   |
| `ibc_transfer_packet_receive`   | Tổng lượng token nhận được trong `FungibleTokenPacketData` (source hoặc sink chain)               | token           | gauge   |
| `new_account`                   | Tổng số tài khoản mới được tạo                                                                    | account         | counter |
| `gov_proposal`                  | Tổng số đề xuất quản trị                                                                           | proposal        | counter |
| `gov_vote`                      | Tổng số phiếu bầu quản trị cho một đề xuất                                                        | vote            | counter |
| `gov_deposit`                   | Tổng số ký gửi quản trị cho một đề xuất                                                           | deposit         | counter |
| `staking_delegate`              | Tổng số ủy quyền                                                                                   | delegation      | counter |
| `staking_undelegate`            | Tổng số hủy ủy quyền                                                                               | undelegation    | counter |
| `staking_redelegate`            | Tổng số tái ủy quyền                                                                               | redelegation    | counter |
| `ibc_transfer_send`             | Tổng số chuyển khoản IBC được gửi từ chain (source hoặc sink)                                     | transfer        | counter |
| `ibc_transfer_receive`          | Tổng số chuyển khoản IBC nhận được tại chain (source hoặc sink)                                   | transfer        | counter |
| `ibc_client_create`             | Tổng số client được tạo                                                                            | create          | counter |
| `ibc_client_update`             | Tổng số cập nhật client                                                                            | update          | counter |
| `ibc_client_upgrade`            | Tổng số nâng cấp client                                                                            | upgrade         | counter |
| `ibc_client_misbehaviour`       | Tổng số misbehaviour của client                                                                    | misbehaviour    | counter |
| `ibc_connection_open-init`      | Tổng số handshake `OpenInit` kết nối                                                               | handshake       | counter |
| `ibc_connection_open-try`       | Tổng số handshake `OpenTry` kết nối                                                                | handshake       | counter |
| `ibc_connection_open-ack`       | Tổng số handshake `OpenAck` kết nối                                                                | handshake       | counter |
| `ibc_connection_open-confirm`   | Tổng số handshake `OpenConfirm` kết nối                                                            | handshake       | counter |
| `ibc_channel_open-init`         | Tổng số handshake `OpenInit` kênh                                                                  | handshake       | counter |
| `ibc_channel_open-try`          | Tổng số handshake `OpenTry` kênh                                                                   | handshake       | counter |
| `ibc_channel_open-ack`          | Tổng số handshake `OpenAck` kênh                                                                   | handshake       | counter |
| `ibc_channel_open-confirm`      | Tổng số handshake `OpenConfirm` kênh                                                               | handshake       | counter |
| `ibc_channel_close-init`        | Tổng số handshake `CloseInit` kênh                                                                 | handshake       | counter |
| `ibc_channel_close-confirm`     | Tổng số handshake `CloseConfirm` kênh                                                              | handshake       | counter |
| `tx_msg_ibc_recv_packet`        | Tổng số gói IBC nhận được                                                                         | packet          | counter |
| `tx_msg_ibc_acknowledge_packet` | Tổng số gói IBC được xác nhận                                                                     | acknowledgement | counter |
| `ibc_timeout_packet`            | Tổng số gói IBC hết hạn                                                                           | timeout         | counter |
| `store_iavl_get`                | Thời gian của lời gọi IAVL `Store#Get`                                                            | ms              | summary |
| `store_iavl_set`                | Thời gian của lời gọi IAVL `Store#Set`                                                            | ms              | summary |
| `store_iavl_has`                | Thời gian của lời gọi IAVL `Store#Has`                                                            | ms              | summary |
| `store_iavl_delete`             | Thời gian của lời gọi IAVL `Store#Delete`                                                         | ms              | summary |
| `store_iavl_commit`             | Thời gian của lời gọi IAVL `Store#Commit`                                                         | ms              | summary |
| `store_iavl_query`              | Thời gian của lời gọi IAVL `Store#Query`                                                          | ms              | summary |
