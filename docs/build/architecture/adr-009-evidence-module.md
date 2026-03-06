# ADR 009: Module Bằng Chứng

## Changelog

* 31 tháng 7 năm 2019: Bản nháp đầu tiên
* 24 tháng 10 năm 2019: Triển khai đầu tiên

## Trạng Thái

Đã Chấp Nhận

## Bối Cảnh

Để hỗ trợ xây dựng các ứng dụng blockchain có độ bảo mật cao, mạnh mẽ và có khả năng tương tác, điều quan trọng đối với Cosmos SDK là phải cung cấp một cơ chế trong đó bằng chứng tùy ý có thể được gửi, đánh giá và xác minh dẫn đến một hình phạt đã được thỏa thuận cho bất kỳ hành vi sai trái nào được thực hiện bởi validator, chẳng hạn như equivocation (bỏ phiếu kép), ký khi unbonded, ký một chuyển đổi trạng thái không đúng (trong tương lai), v.v. Hơn nữa, một cơ chế như vậy là tối quan trọng cho bất kỳ triển khai giao thức [IBC](https://github.com/cosmos/ics/blob/master/ibc/2_IBC_ARCHITECTURE.md) hoặc cross-chain validation nào để hỗ trợ khả năng cho bất kỳ hành vi sai trái nào được chuyển tiếp từ một chain có thế chấp về chain chính để validator vi phạm có thể bị slash.

## Quyết Định

Chúng ta sẽ triển khai một module evidence trong Cosmos SDK hỗ trợ chức năng sau:

* Cung cấp cho nhà phát triển các trừu tượng và interface cần thiết để định nghĩa các message evidence tùy chỉnh, message handler và các phương thức để slash và phạt phù hợp cho hành vi sai trái.
* Hỗ trợ khả năng định tuyến evidence message đến handler trong bất kỳ module nào để xác định tính hợp lệ của hành vi sai trái được gửi.
* Hỗ trợ khả năng, thông qua quản trị, để sửa đổi các hình phạt slashing của bất kỳ loại evidence nào.
* Triển khai Querier để hỗ trợ truy vấn params, loại evidence, params và tất cả hành vi sai trái hợp lệ đã gửi.

### Kiểu Dữ Liệu

Trước tiên, chúng ta định nghĩa kiểu interface `Evidence`. Module `x/evidence` có thể triển khai các kiểu riêng có thể được sử dụng bởi nhiều chain (ví dụ: `CounterFactualEvidence`). Ngoài ra, các module khác có thể triển khai các kiểu `Evidence` riêng theo cách tương tự như cách quản trị có thể mở rộng. Điều quan trọng cần lưu ý là bất kỳ kiểu cụ thể nào triển khai interface `Evidence` có thể bao gồm các trường tùy ý như thời gian vi phạm. Chúng ta muốn kiểu `Evidence` duy trì linh hoạt nhất có thể.

Khi gửi evidence đến module `x/evidence`, kiểu cụ thể phải cung cấp địa chỉ consensus của validator, nên được module `x/slashing` biết đến (giả sử vi phạm là hợp lệ), chiều cao mà vi phạm xảy ra và sức mạnh của validator tại cùng chiều cao mà vi phạm xảy ra.

```go
type Evidence interface {
  Route() string
  Type() string
  String() string
  Hash() HexBytes
  ValidateBasic() error

  // Địa chỉ consensus của validator độc hại tại thời điểm vi phạm
  GetConsensusAddress() ConsAddress

  // Chiều cao mà vi phạm xảy ra
  GetHeight() int64

  // Tổng sức mạnh của validator độc hại tại thời điểm vi phạm
  GetValidatorPower() int64

  // Tổng sức mạnh validator set tại thời điểm vi phạm
  GetTotalPower() int64
}
```

### Định Tuyến & Xử Lý

Mỗi kiểu `Evidence` phải ánh xạ đến một route duy nhất cụ thể và được đăng ký với module `x/evidence`. Nó thực hiện điều này thông qua triển khai `Router`.

```go
type Router interface {
  AddRoute(r string, h Handler) Router
  HasRoute(r string) bool
  GetRoute(path string) Handler
  Seal()
}
```

Sau khi định tuyến thành công qua module `x/evidence`, kiểu `Evidence` được truyền qua một `Handler`. `Handler` này chịu trách nhiệm thực thi tất cả logic nghiệp vụ tương ứng cần thiết để xác minh evidence là hợp lệ. Ngoài ra, `Handler` có thể thực thi bất kỳ slashing và jailing tiềm năng nào cần thiết. Vì tỷ lệ slashing thường sẽ là kết quả của một số hàm tĩnh, cho phép `Handler` làm điều này cung cấp sự linh hoạt tối đa. Ví dụ có thể là `k * evidence.GetValidatorPower()` trong đó `k` là tham số on-chain được kiểm soát bởi quản trị. Kiểu `Evidence` nên cung cấp tất cả thông tin bên ngoài cần thiết để `Handler` thực hiện các chuyển đổi trạng thái cần thiết. Nếu không có lỗi nào được trả về, `Evidence` được coi là hợp lệ.

```go
type Handler func(Context, Evidence) error
```

### Gửi

`Evidence` được gửi qua kiểu message `MsgSubmitEvidence` được xử lý nội bộ bởi `SubmitEvidence` của module `x/evidence`.

```go
type MsgSubmitEvidence struct {
  Evidence
}

func handleMsgSubmitEvidence(ctx Context, keeper Keeper, msg MsgSubmitEvidence) Result {
  if err := keeper.SubmitEvidence(ctx, msg.Evidence); err != nil {
    return err.Result()
  }

  // phát sự kiện...

  return Result{
    // ...
  }
}
```

Keeper của module `x/evidence` chịu trách nhiệm khớp `Evidence` với router của module và gọi `Handler` tương ứng, có thể bao gồm slashing và jailing validator. Khi thành công, evidence đã gửi được lưu trữ.

```go
func (k Keeper) SubmitEvidence(ctx Context, evidence Evidence) error {
  handler := keeper.router.GetRoute(evidence.Route())
  if err := handler(ctx, evidence); err != nil {
    return ErrInvalidEvidence(keeper.codespace, err)
  }

  keeper.setEvidence(ctx, evidence)
  return nil
}
```

### Genesis

Cuối cùng, chúng ta cần biểu diễn trạng thái genesis của module `x/evidence`. Module chỉ cần một danh sách tất cả vi phạm hợp lệ đã gửi và bất kỳ params cần thiết nào mà module cần để xử lý evidence đã gửi. Module `x/evidence` sẽ tự nhiên định nghĩa và định tuyến các kiểu evidence gốc mà nhiều khả năng nó sẽ cần các hằng số hình phạt slashing.

```go
type GenesisState struct {
  Params       Params
  Infractions  []Evidence
}
```

## Hậu Quả

### Tích Cực

* Cho phép state machine xử lý hành vi sai trái được gửi on-chain và phạt validator dựa trên các tham số slashing đã được thỏa thuận.
* Cho phép các kiểu evidence được định nghĩa và xử lý bởi bất kỳ module nào. Điều này cho phép slashing và jailing được định nghĩa bởi các cơ chế phức tạp hơn.
* Không chỉ phụ thuộc vào Tendermint để gửi evidence.

### Tiêu Cực

* Không có cách dễ dàng để giới thiệu các kiểu evidence mới thông qua quản trị trên chain đang hoạt động do không thể giới thiệu handler tương ứng của kiểu evidence mới.

### Trung Lập

* Chúng ta có nên lưu trữ vi phạm vô thời hạn không? Hay nên dựa vào các sự kiện?

## Tài Liệu Tham Khảo

* [ICS](https://github.com/cosmos/ics)
* [Kiến Trúc IBC](https://github.com/cosmos/ics/blob/master/ibc/1_IBC_ARCHITECTURE.md)
* [Trách Nhiệm Fork Tendermint](https://github.com/tendermint/spec/blob/7b3138e69490f410768d9b1ffc7a17abc23ea397/spec/consensus/fork-accountability.md)
