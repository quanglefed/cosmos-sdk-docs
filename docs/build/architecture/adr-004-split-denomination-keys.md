# ADR 004: Tách Khóa Denomination

## Changelog

* 2020-01-08: Phiên bản đầu tiên
* 2020-01-09: Sửa đổi để xử lý vesting account
* 2020-01-14: Cập nhật từ phản hồi review
* 2020-01-30: Cập nhật từ triển khai

### Thuật Ngữ

* denom / denomination key -- định danh token duy nhất.

## Bối Cảnh

Với IBC không cần cấp phép, bất kỳ ai cũng có thể gửi các denomination tùy ý đến bất kỳ tài khoản nào khác. Hiện tại, tất cả số dư khác không được lưu cùng với tài khoản trong một struct `sdk.Coins`, điều này tạo ra mối lo ngại denial-of-service tiềm năng, vì quá nhiều denomination sẽ ngày càng tốn kém để tải và lưu mỗi lần tài khoản được sửa đổi. Xem issues [5467](https://github.com/cosmos/cosmos-sdk/issues/5467) và [4982](https://github.com/cosmos/cosmos-sdk/issues/4982) để biết thêm bối cảnh.

Đơn giản là từ chối tiền gửi đến sau giới hạn số denomination không hoạt động, vì nó mở ra một vector griefing: ai đó có thể gửi cho người dùng nhiều coin vô nghĩa qua IBC và sau đó ngăn người dùng nhận các denomination thực (như phần thưởng staking).

## Quyết Định

Số dư sẽ được lưu trữ theo từng tài khoản và từng denomination dưới một khóa duy nhất theo denomination và tài khoản, cho phép truy cập đọc & ghi O(1) vào số dư của một tài khoản cụ thể trong một denomination cụ thể.

### Interface Tài Khoản (x/auth)

`GetCoins()` và `SetCoins()` sẽ bị xóa khỏi interface tài khoản, vì số dư coin bây giờ sẽ được lưu trữ và quản lý bởi module bank.

Interface vesting account sẽ thay thế `SpendableCoins` bằng `LockedCoins` không cần số dư tài khoản nữa. Ngoài ra, `TrackDelegation()` bây giờ sẽ chấp nhận số dư tài khoản của tất cả token được tính theo vesting balance thay vì tải toàn bộ số dư tài khoản.

Vesting account sẽ tiếp tục lưu trữ vesting gốc, delegated free và delegated vesting coin (điều này an toàn vì chúng không thể chứa denomination tùy ý).

### Bank Keeper (x/bank)

Các API sau sẽ được thêm vào keeper `x/bank`:

* `GetAllBalances(ctx Context, addr AccAddress) Coins`
* `GetBalance(ctx Context, addr AccAddress, denom string) Coin`
* `SetBalance(ctx Context, addr AccAddress, coin Coin)`
* `LockedCoins(ctx Context, addr AccAddress) Coins`
* `SpendableCoins(ctx Context, addr AccAddress) Coins`

Các API bổ sung có thể được thêm để hỗ trợ iteration và chức năng phụ trợ không thiết yếu cho chức năng core hay persistence.

Số dư sẽ được lưu trữ trước bởi địa chỉ, sau đó bởi denomination (ngược lại cũng có thể, nhưng việc lấy tất cả số dư cho một tài khoản đơn được coi là thường xuyên hơn):

```go
var BalancesPrefix = []byte("balances")

func (k Keeper) SetBalance(ctx Context, addr AccAddress, balance Coin) error {
  if !balance.IsValid() {
    return err
  }

  store := ctx.KVStore(k.storeKey)
  balancesStore := prefix.NewStore(store, BalancesPrefix)
  accountStore := prefix.NewStore(balancesStore, addr.Bytes())

  bz := Marshal(balance)
  accountStore.Set([]byte(balance.Denom), bz)

  return nil
}
```

Điều này sẽ dẫn đến việc số dư được lập chỉ mục theo biểu diễn byte của `balances/{address}/{denom}`.

`DelegateCoins()` và `UndelegateCoins()` sẽ được thay đổi để chỉ tải từng số dư tài khoản riêng lẻ theo denomination được tìm thấy trong số tiền (un)delegation. Kết quả là, bất kỳ thay đổi nào đối với số dư tài khoản sẽ được thực hiện theo denomination.

`SubtractCoins()` và `AddCoins()` sẽ được thay đổi để đọc và ghi số dư trực tiếp thay vì gọi `GetCoins()` / `SetCoins()` (không còn tồn tại nữa).

`trackDelegation()` và `trackUndelegation()` sẽ được thay đổi để không còn cập nhật số dư tài khoản nữa.

Các API bên ngoài sẽ cần scan tất cả số dư dưới một tài khoản để duy trì tương thích ngược. Khuyến nghị rằng các API này sử dụng `GetBalance` và `SetBalance` thay vì `GetAllBalances` khi có thể để không tải toàn bộ số dư tài khoản.

### Module Supply

Module supply, để triển khai invariant tổng cung, giờ sẽ cần scan tất cả tài khoản và gọi `GetAllBalances` bằng keeper `x/bank`, sau đó tổng hợp số dư và kiểm tra rằng chúng khớp với tổng cung dự kiến.

## Trạng Thái

Đã Chấp Nhận.

## Hậu Quả

### Tích Cực

* Đọc và ghi O(1) số dư (liên quan đến số denomination mà một tài khoản có số dư khác không). Lưu ý, điều này không liên quan đến chi phí I/O thực tế, mà là tổng số lần đọc trực tiếp cần thiết.

### Tiêu Cực

* Đọc/ghi kém hiệu quả hơn một chút khi đọc và ghi tất cả số dư của một tài khoản đơn trong một giao dịch.

### Trung Lập

Không có gì đặc biệt.

## Tài Liệu Tham Khảo

* Ref: https://github.com/cosmos/cosmos-sdk/issues/4982
* Ref: https://github.com/cosmos/cosmos-sdk/issues/5467
* Ref: https://github.com/cosmos/cosmos-sdk/issues/5492
