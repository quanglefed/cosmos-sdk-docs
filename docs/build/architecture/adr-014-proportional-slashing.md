# ADR 14: Slashing Tỷ Lệ

## Changelog

* 2019-10-15: Bản nháp đầu tiên
* 2020-05-25: Xóa correlation root slashing
* 2020-07-01: Cập nhật để bao gồm hàm S-curve thay vì tuyến tính

## Bối Cảnh

Trong các chain dựa trên Proof of Stake, việc tập trung hóa quyền lực đồng thuận vào một tập hợp nhỏ validator có thể gây hại cho mạng do tăng nguy cơ kiểm duyệt, thất bại liveness, tấn công fork, v.v. Tuy nhiên, mặc dù sự tập trung hóa này gây ra ngoại tác tiêu cực cho mạng, nó không được cảm nhận trực tiếp bởi những người ủy quyền đang đóng góp vào việc ủy quyền cho các validator đã lớn. Chúng ta muốn có cách chuyển chi phí ngoại tác tiêu cực của tập trung hóa lên các validator lớn và những người ủy quyền của họ.

## Quyết Định

### Thiết Kế

Để giải quyết vấn đề này, chúng ta sẽ triển khai một quy trình gọi là Proportional Slashing. Mong muốn là validator càng lớn thì họ càng bị slash nhiều hơn. Nỗ lực đầu tiên ngây thơ là làm cho tỷ lệ phần trăm slash của validator tỷ lệ với phần chia sẻ quyền bỏ phiếu đồng thuận của họ.

```text
slash_amount = k * power // power là sức mạnh bỏ phiếu của validator vi phạm và k là hằng số on-chain nào đó
```

Tuy nhiên, điều này sẽ khuyến khích các validator với số lượng stake lớn chia quyền bỏ phiếu của họ giữa các tài khoản (tấn công sybil), để nếu họ vi phạm, tất cả bị slash ở tỷ lệ thấp hơn. Giải pháp cho điều này là tính không chỉ phần trăm bỏ phiếu của validator mà còn phần trăm bỏ phiếu của tất cả các validator khác bị slash trong khung thời gian được chỉ định.

```text
slash_amount = k * (power_1 + power_2 + ... + power_n) // trong đó power_i là sức mạnh bỏ phiếu của validator thứ i vi phạm trong khung thời gian được chỉ định và k là hằng số on-chain nào đó
```

Bây giờ, nếu ai đó chia một validator 10% thành hai validator 5% mỗi cái đều vi phạm, và cả hai vi phạm trong cùng khung thời gian, cả hai sẽ bị slash ở tổng mức 10%.

Tuy nhiên trong thực tế, chúng ta có thể không muốn quan hệ tuyến tính giữa lượng stake bị vi phạm và tỷ lệ phần trăm stake bị slash. Cụ thể, chỉ 5% stake ký kép thực sự không làm gì để đe dọa đáng kể bảo mật, trong khi 30% stake bị vi phạm rõ ràng xứng đáng với hệ số slashing lớn, do rất gần điểm mà bảo mật Tendermint bị đe dọa. Quan hệ tuyến tính sẽ yêu cầu hệ số gap 6 giữa hai trường hợp này, trong khi sự khác biệt về rủi ro đối với mạng lớn hơn nhiều. Chúng tôi đề xuất sử dụng S-curve (chính thức là [hàm logistic](https://en.wikipedia.org/wiki/Logistic_function) để giải quyết điều này. S-Curve nắm bắt tiêu chí mong muốn khá tốt. Chúng cho phép hệ số slashing tối thiểu đối với các giá trị nhỏ, sau đó tăng rất nhanh gần một điểm ngưỡng nào đó trong đó rủi ro đặt ra trở nên đáng chú ý.

#### Tham Số Hóa

Điều này yêu cầu tham số hóa một hàm logistic. Cách tham số hóa điều này được hiểu rất rõ. Nó có bốn tham số:

1) Hệ số slashing tối thiểu
2) Hệ số slashing tối đa
3) Điểm uốn của S-curve (về cơ bản là nơi bạn muốn căn giữa S)
4) Tốc độ tăng trưởng của S-curve (S được kéo dài như thế nào)

#### Tương Quan Giữa Các Validator Không Phải Sybil

Người ta sẽ lưu ý rằng mô hình này không phân biệt giữa nhiều validator chạy bởi cùng operator so với validator chạy bởi các operator khác nhau. Điều này thực ra có thể được coi là lợi ích bổ sung. Nó khuyến khích validator phân biệt thiết lập của họ với các validator khác, để tránh có các lỗi tương quan với họ hoặc sẽ có nguy cơ slash cao hơn. Vì vậy ví dụ, operator nên tránh sử dụng các nền tảng cloud hosting phổ biến hoặc sử dụng cùng nhà cung cấp Staking-as-a-Service. Điều này sẽ dẫn đến một mạng resilient và phi tập trung hơn.

#### Griefing

Griefing, hành động cố tình tự làm cho mình bị slash để làm cho slash của người khác tệ hơn, có thể là mối lo ngại ở đây. Tuy nhiên, sử dụng giao thức được mô tả ở đây, kẻ tấn công cũng bị ảnh hưởng tương đương bởi grief như nạn nhân, vì vậy nó sẽ không mang lại nhiều lợi ích cho griefer.

### Triển Khai

Trong module slashing, chúng ta sẽ thêm hai hàng đợi theo dõi tất cả sự kiện slash gần đây. Đối với lỗi ký kép, chúng ta sẽ định nghĩa "slashes gần đây" là những slash đã xảy ra trong `unbonding period` cuối cùng. Đối với lỗi liveness, chúng ta sẽ định nghĩa "slashes gần đây" là những slash đã xảy ra trong `jail period` cuối cùng.

```go
type SlashEvent struct {
    Address                     sdk.ValAddress
    ValidatorVotingPercent      sdk.Dec
    SlashedSoFar                sdk.Dec
}
```

Các sự kiện slash này sẽ được pruning khỏi hàng đợi một khi chúng cũ hơn "giai đoạn slash gần đây" tương ứng.

Bất cứ khi nào một slash mới xảy ra, một struct `SlashEvent` được tạo với phần trăm bỏ phiếu của validator vi phạm và `SlashedSoFar` là 0. Vì các sự kiện slash gần đây được pruning trước khi unbonding period và unjail period hết hạn, không thể có cùng validator có nhiều SlashEvent trong cùng Queue cùng một lúc.

Sau đó chúng ta sẽ iterate qua tất cả SlashEvent trong hàng đợi, cộng `ValidatorVotingPercent` của chúng để tính phần trăm mới để slash tất cả validator trong hàng đợi, sử dụng công thức "Square of Sum of Roots" được giới thiệu ở trên.

Khi chúng ta có `NewSlashPercent`, chúng ta sau đó iterate qua tất cả `SlashEvent` trong hàng đợi một lần nữa, và nếu `NewSlashPercent > SlashedSoFar` cho SlashEvent đó, chúng ta gọi `staking.Slash(slashEvent.Address, slashEvent.Power, Math.Min(Math.Max(minSlashPercent, NewSlashPercent - SlashedSoFar), maxSlashPercent)` (chúng ta truyền vào sức mạnh của validator trước khi bất kỳ slash nào xảy ra để slash đúng số lượng token). Sau đó chúng ta đặt lượng `SlashEvent.SlashedSoFar` thành `NewSlashPercent`.

## Trạng Thái

Đề Xuất

## Hậu Quả

### Tích Cực

* Tăng phi tập trung hóa bằng cách không khuyến khích ủy quyền cho các validator lớn
* Khuyến khích phi tương quan của Validator
* Phạt nặng hơn các cuộc tấn công so với lỗi ngẫu nhiên
* Linh hoạt hơn trong tham số hóa tỷ lệ slashing

### Tiêu Cực

* Tốn kém tính toán hơn so với triển khai hiện tại. Sẽ yêu cầu lưu trữ thêm dữ liệu về "các sự kiện slashing gần đây" trên chain.
