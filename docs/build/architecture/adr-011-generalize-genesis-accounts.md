# ADR 011: Tổng Quát Hóa Genesis Account

## Changelog

* 2019-08-30: Bản nháp đầu tiên

## Bối Cảnh

Hiện tại, Cosmos SDK cho phép các kiểu tài khoản tùy chỉnh; keeper `auth` lưu trữ bất kỳ kiểu nào thỏa mãn interface `Account` của nó. Tuy nhiên `auth` không xử lý export hoặc load tài khoản đến/từ genesis file, điều này được thực hiện bởi `genaccounts`, chỉ xử lý một trong 4 kiểu tài khoản cụ thể (`BaseAccount`, `ContinuousVestingAccount`, `DelayedVestingAccount` và `ModuleAccount`).

Các dự án muốn sử dụng tài khoản tùy chỉnh (chẳng hạn như vesting account tùy chỉnh) cần fork và sửa đổi `genaccounts`.

## Quyết Định

Tóm lại, chúng ta sẽ (un)marshal tất cả tài khoản (kiểu interface) trực tiếp bằng amino, thay vì chuyển đổi sang kiểu `GenesisAccount` của `genaccounts`. Vì làm điều này xóa phần lớn code của `genaccounts`, chúng ta sẽ merge `genaccounts` vào `auth`. Các tài khoản đã marshal sẽ được lưu trữ trong trạng thái genesis của `auth`.

Các thay đổi chi tiết:

### 1) (Un)Marshal tài khoản trực tiếp bằng amino

`GenesisState` của module `auth` nhận thêm một trường `Accounts`. Lưu ý rằng đây không phải kiểu `exported.Account` vì những lý do được nêu trong phần 3.

```go
// GenesisState - tất cả trạng thái auth phải được cung cấp khi genesis
type GenesisState struct {
    Params   Params           `json:"params" yaml:"params"`
    Accounts []GenesisAccount `json:"accounts" yaml:"accounts"`
}
```

Bây giờ `InitGenesis` và `ExportGenesis` của `auth` cũng (un)marshal tài khoản cũng như các params đã được định nghĩa.

```go
// InitGenesis - Khởi tạo trạng thái store từ dữ liệu genesis
func InitGenesis(ctx sdk.Context, ak AccountKeeper, data GenesisState) {
    ak.SetParams(ctx, data.Params)
    // tải tài khoản
    for _, a := range data.Accounts {
        acc := ak.NewAccount(ctx, a) // đặt account number
        ak.SetAccount(ctx, acc)
    }
}

// ExportGenesis trả về GenesisState cho context và keeper nhất định
func ExportGenesis(ctx sdk.Context, ak AccountKeeper) GenesisState {
    params := ak.GetParams(ctx)

    var genAccounts []exported.GenesisAccount
    ak.IterateAccounts(ctx, func(account exported.Account) bool {
        genAccount := account.(exported.GenesisAccount)
        genAccounts = append(genAccounts, genAccount)
        return false
    })

    return NewGenesisState(params, genAccounts)
}
```

### 2) Đăng ký các kiểu tài khoản tùy chỉnh trên codec của `auth`

Codec của `auth` phải có tất cả kiểu tài khoản tùy chỉnh được đăng ký để marshal chúng. Chúng ta sẽ theo mẫu được thiết lập trong `gov` cho các đề xuất.

Ví dụ định nghĩa tài khoản tùy chỉnh:

```go
import authtypes "github.com/cosmos/cosmos-sdk/x/auth/types"

// Đăng ký kiểu module account với codec của module auth để nó có thể giải mã module account được lưu trong genesis file
func init() {
    authtypes.RegisterAccountTypeCodec(ModuleAccount{}, "cosmos-sdk/ModuleAccount")
}

type ModuleAccount struct {
    ...
```

Định nghĩa codec `auth`:

```go
var ModuleCdc *codec.LegacyAmino

func init() {
    ModuleCdc = codec.NewLegacyAmino()
    // đăng ký msg của module và interface Account
    ...
    // để codec không được seal
}

// RegisterAccountTypeCodec đăng ký một kiểu tài khoản bên ngoài được định nghĩa trong module khác cho ModuleCdc nội bộ.
func RegisterAccountTypeCodec(o interface{}, name string) {
    ModuleCdc.RegisterConcrete(o, name, nil)
}
```

### 3) Xác thực Genesis cho các kiểu tài khoản tùy chỉnh

Các module triển khai phương thức `ValidateGenesis`. Vì `auth` không biết về các triển khai tài khoản, các tài khoản sẽ cần tự xác thực.

Chúng ta sẽ unmarshal tài khoản vào interface `GenesisAccount` bao gồm phương thức `Validate`.

```go
type GenesisAccount interface {
    exported.Account
    Validate() error
}
```

Sau đó hàm `ValidateGenesis` của `auth` trở thành:

```go
// ValidateGenesis thực hiện xác thực cơ bản dữ liệu genesis auth trả về
// lỗi cho bất kỳ tiêu chí xác thực thất bại.
func ValidateGenesis(data GenesisState) error {
    // Xác thực params
    ...

    // Xác thực tài khoản
    addrMap := make(map[string]bool, len(data.Accounts))
    for _, acc := range data.Accounts {

        // kiểm tra tài khoản trùng lặp
        addrStr := acc.GetAddress().String()
        if _, ok := addrMap[addrStr]; ok {
            return fmt.Errorf("duplicate account found in genesis state; address: %s", addrStr)
        }
        addrMap[addrStr] = true

        // kiểm tra xác thực cụ thể của tài khoản
        if err := acc.Validate(); err != nil {
            return fmt.Errorf("invalid account found in genesis state; address: %s, error: %s", addrStr, err.Error())
        }

    }
    return nil
}
```

### 4) Chuyển lệnh CLI add-genesis-account sang `auth`

Module `genaccounts` chứa lệnh CLI để thêm base hoặc vesting account vào genesis file. Điều này sẽ được chuyển sang `auth`. Chúng ta sẽ để cho các dự án tự viết các lệnh của riêng họ để thêm tài khoản tùy chỉnh. Một handler CLI có thể mở rộng, tương tự như `gov`, có thể được tạo nhưng không đáng phức tạp cho use case nhỏ này.

### 5) Cập nhật module và vesting account

Theo phương thức mới, các kiểu module và vesting account cần một số cập nhật nhỏ:

* Đăng ký kiểu trên codec của `auth` (được hiển thị ở trên)
* Phương thức `Validate` cho mỗi kiểu `Account` cụ thể

## Trạng Thái

Đề Xuất

## Hậu Quả

### Tích Cực

* tài khoản tùy chỉnh có thể được sử dụng mà không cần fork `genaccounts`
* giảm số dòng code

### Tiêu Cực

### Trung Lập

* Module `genaccounts` không còn tồn tại
* Tài khoản trong genesis file được lưu dưới `accounts` trong `auth` thay vì trong module `genaccounts`.
* Lệnh CLI `add-genesis-account` giờ trong `auth`

## Tài Liệu Tham Khảo
