---
sidebar_position: 1
---

# `x/auth/vesting`

* [Giới thiệu và yêu cầu](#giới-thiệu-và-yêu-cầu)
* [Lưu ý](#lưu-ý)
* [Các loại Vesting Account](#các-loại-vesting-account)
  * [BaseVestingAccount](#basevestingaccount)
  * [ContinuousVestingAccount](#continuousvestingaccount)
  * [DelayedVestingAccount](#delayedvestingaccount)
  * [Period](#period)
  * [PeriodicVestingAccount](#periodicvestingaccount)
  * [PermanentLockedAccount](#permanentlockedaccount)
* [Đặc tả Vesting Account](#đặc-tả-vesting-account)
  * [Xác định lượng đang vesting & đã vested](#xác-định-lượng-đang-vesting--đã-vested)
  * [Periodic Vesting Account](#periodic-vesting-account)
  * [Chuyển/Gửi](#chuyểngửi)
  * [Uỷ quyền stake (Delegating)](#uỷ-quyền-stake-delegating)
  * [Huỷ uỷ quyền (Undelegating)](#huỷ-uỷ-quyền-undelegating)
* [Keeper & Handler](#keeper--handler)
* [Khởi tạo Genesis](#khởi-tạo-genesis)
* [Ví dụ](#ví-dụ)
  * [Đơn giản](#đơn-giản)
  * [Slashing](#slashing)
  * [Periodic vesting](#periodic-vesting)
* [Thuật ngữ](#thuật-ngữ)

## Giới thiệu và yêu cầu

Đặc tả này định nghĩa hiện thực vesting account được dùng bởi Cosmos Hub. Yêu cầu
đối với vesting account là: nó được khởi tạo trong genesis với số dư ban đầu `X`
và thời điểm vesting kết thúc `ET`. Một vesting account có thể được khởi tạo thêm
với thời điểm bắt đầu vesting `ST` và số lượng kỳ vesting `P`. Nếu có thời điểm bắt
đầu vesting, kỳ vesting sẽ không bắt đầu cho tới khi chạm `ST`. Nếu có các kỳ vesting,
việc vesting diễn ra theo số kỳ được chỉ định.

Với mọi vesting account, chủ sở hữu vesting account có thể delegate và undelegate
với validator; tuy nhiên họ không thể chuyển coin sang tài khoản khác cho tới khi
coin đó đã vested. Đặc tả này cho phép bốn kiểu vesting khác nhau:

* Delayed vesting: toàn bộ coin sẽ vested khi đạt `ET`.
* Continuous vesting: coin bắt đầu vest tại `ST` và vest tuyến tính theo thời gian cho tới `ET`.
* Periodic vesting: coin bắt đầu vest tại `ST` và vest theo chu kỳ dựa trên số kỳ và lượng vest mỗi kỳ.
  Số kỳ, độ dài mỗi kỳ và lượng mỗi kỳ đều có thể cấu hình. Periodic vesting account khác continuous
  vesting account ở chỗ coin có thể được mở khoá theo các “đợt” (tranche) so le. Ví dụ: periodic vesting
  account có thể dùng cho lịch vesting theo quý, theo năm, hoặc bất kỳ hàm nào của token theo thời gian.
* Permanent locked vesting: coin bị khoá vĩnh viễn. Coin trong tài khoản này vẫn có thể dùng để delegate
  và bỏ phiếu governance ngay cả khi bị khoá.

## Lưu ý

Vesting account có thể được khởi tạo với cả coin vesting và coin không vesting.
Coin không vesting sẽ có thể chuyển ngay lập tức. Các tài khoản DelayedVesting,
ContinuousVesting, PeriodicVesting và PermanentVesting có thể được tạo bằng các
message thông thường sau genesis. Các loại vesting account khác phải được tạo ở
genesis hoặc là một phần của nâng cấp mạng thủ công. Đặc tả hiện tại chỉ cho phép
vesting _vô điều kiện_ (tức là không có khả năng đạt `ET` nhưng coin lại không vest).

## Các loại Vesting Account

```go
// VestingAccount defines an interface that any vesting account type must
// implement.
type VestingAccount interface {
  Account

  GetVestedCoins(Time)  Coins
  GetVestingCoins(Time) Coins

  // TrackDelegation performs internal vesting accounting necessary when
  // delegating from a vesting account. It accepts the current block time, the
  // delegation amount and balance of all coins whose denomination exists in
  // the account's original vesting balance.
  TrackDelegation(Time, Coins, Coins)

  // TrackUndelegation performs internal vesting accounting necessary when a
  // vesting account performs an undelegation.
  TrackUndelegation(Coins)

  GetStartTime() int64
  GetEndTime()   int64
}
```

### BaseVestingAccount

```protobuf reference
https://github.com/cosmos/cosmos-sdk/tree/release/v0.50.x/proto/cosmos/vesting/v1beta1/vesting.proto#L11-L35
```

### ContinuousVestingAccount

```protobuf reference
https://github.com/cosmos/cosmos-sdk/tree/release/v0.50.x/proto/cosmos/vesting/v1beta1/vesting.proto#L37-L46
```

### DelayedVestingAccount

```protobuf reference
https://github.com/cosmos/cosmos-sdk/tree/release/v0.50.x/proto/cosmos/vesting/v1beta1/vesting.proto#L48-L57
```

### Period

```protobuf reference
https://github.com/cosmos/cosmos-sdk/tree/release/v0.50.x/proto/cosmos/vesting/v1beta1/vesting.proto#L59-L69
```

```go
// Stores all vesting periods passed as part of a PeriodicVestingAccount
type Periods []Period

```

### PeriodicVestingAccount

```protobuf reference
https://github.com/cosmos/cosmos-sdk/tree/release/v0.50.x/proto/cosmos/vesting/v1beta1/vesting.proto#L71-L81
```

Để giảm việc type checking/assertion mang tính ad-hoc và hỗ trợ tính linh hoạt
trong việc sử dụng số dư tài khoản, interface `x/bank` `ViewKeeper` hiện có được
cập nhật để chứa các hàm sau:

```go
type ViewKeeper interface {
  // ...

  // Calculates the total locked account balance.
  LockedCoins(ctx sdk.Context, addr sdk.AccAddress) sdk.Coins

  // Calculates the total spendable balance that can be sent to other accounts.
  SpendableCoins(ctx sdk.Context, addr sdk.AccAddress) sdk.Coins
}
```

### PermanentLockedAccount

```protobuf reference
https://github.com/cosmos/cosmos-sdk/tree/release/v0.50.x/proto/cosmos/vesting/v1beta1/vesting.proto#L83-L94
```

## Đặc tả Vesting Account

Với một vesting account, ta định nghĩa các ký hiệu sau trong các thao tác tiếp theo:

* `OV`: Lượng coin vesting ban đầu (original vesting). Đây là hằng số.
* `V`: Số coin `OV` vẫn _đang vesting_. Nó được suy ra từ `OV`, `StartTime` và `EndTime`.
  Giá trị này được tính khi cần (on demand) chứ không phải theo từng block.
* `V'`: Số coin `OV` đã _vested_ (mở khoá). Giá trị này được tính khi cần chứ không theo từng block.
* `DV`: Số coin _vesting_ đã được delegate. Đây là giá trị biến và được lưu/sửa trực tiếp trong vesting account.
* `DF`: Số coin _vested_ (mở khoá) đã được delegate. Đây là giá trị biến và được lưu/sửa trực tiếp trong vesting account.
* `BC`: Số coin `OV` trừ đi các coin đã chuyển (có thể âm hoặc đã delegate). Đây được xem như số dư của base account nhúng.
  Nó được lưu/sửa trực tiếp trong vesting account.

### Xác định lượng đang vesting & đã vested

Cần lưu ý rằng các giá trị này được tính khi cần và không bắt buộc tính theo từng block
(ví dụ trong `BeginBlocker` hay `EndBlocker`).

#### Continuous vesting account

Để xác định lượng coin đã vested tại thời điểm block `T`, thực hiện:

1. Tính `X := T - StartTime`
2. Tính `Y := EndTime - StartTime`
3. Tính `V' := OV * (X / Y)`
4. Tính `V := OV - V'`

Vì vậy tổng lượng coin _đã vested_ là `V'` và lượng còn lại `V` là _đang vesting_.

```go
func (cva ContinuousVestingAccount) GetVestedCoins(t Time) Coins {
    if t <= cva.StartTime {
        // We must handle the case where the start time for a vesting account has
        // been set into the future or when the start of the chain is not exactly
        // known.
        return ZeroCoins
    } else if t >= cva.EndTime {
        return cva.OriginalVesting
    }

    x := t - cva.StartTime
    y := cva.EndTime - cva.StartTime

    return cva.OriginalVesting * (x / y)
}

func (cva ContinuousVestingAccount) GetVestingCoins(t Time) Coins {
    return cva.OriginalVesting - cva.GetVestedCoins(t)
}
```

### Periodic vesting account

Periodic vesting account yêu cầu tính lượng coin được mở khoá trong mỗi kỳ tại thời
điểm block `T`. Lưu ý có thể nhiều kỳ đã trôi qua khi gọi `GetVestedCoins`, nên ta
phải lặp qua từng kỳ cho tới khi kết thúc kỳ đó nằm sau `T`.

1. Đặt `CT := StartTime`
2. Đặt `V' := 0`

Với mỗi kỳ P:

  1. Tính `X := T - CT`
  2. NẾU `X >= P.Length`
      1. Tính `V' += P.Amount`
      2. Tính `CT += P.Length`
      3. NGƯỢC LẠI: dừng
  3. Tính `V := OV - V'`

```go
func (pva PeriodicVestingAccount) GetVestedCoins(t Time) Coins {
  if t < pva.StartTime {
    return ZeroCoins
  }
  ct := pva.StartTime // The start of the vesting schedule
  vested := 0
  periods = pva.GetPeriods()
  for _, period  := range periods {
    if t - ct < period.Length {
      break
    }
    vested += period.Amount
    ct += period.Length // increment ct to the start of the next vesting period
  }
  return vested
}

func (pva PeriodicVestingAccount) GetVestingCoins(t Time) Coins {
    return pva.OriginalVesting - cva.GetVestedCoins(t)
}
```

#### Delayed/Discrete vesting account

Delayed vesting account dễ suy luận hơn vì nó chỉ giữ toàn bộ lượng coin ở trạng
thái vesting cho tới một thời điểm nhất định, rồi toàn bộ coin trở thành vested
(mở khoá). Điều này không bao gồm các coin mở khoá sẵn mà tài khoản có thể đã có ban đầu.

```go
func (dva DelayedVestingAccount) GetVestedCoins(t Time) Coins {
    if t >= dva.EndTime {
        return dva.OriginalVesting
    }

    return ZeroCoins
}

func (dva DelayedVestingAccount) GetVestingCoins(t Time) Coins {
    return dva.OriginalVesting - dva.GetVestedCoins(t)
}
```

### Chuyển/Gửi

Tại bất kỳ thời điểm nào, một vesting account có thể chuyển: `min((BC + DV) - V, BC)`.

Nói cách khác, vesting account có thể chuyển giá trị nhỏ hơn giữa số dư base account
và số dư base account cộng số coin vesting hiện đang delegate trừ số coin đã vested
cho tới thời điểm đó.

Tuy nhiên, vì số dư tài khoản được theo dõi qua module `x/bank` và ta muốn tránh
phải tải toàn bộ số dư, ta có thể thay vào đó xác định số dư bị khoá (locked balance),
có thể định nghĩa là `max(V - DV, 0)`, và suy ra số dư có thể chi tiêu (spendable) từ đó.

```go
func (va VestingAccount) LockedCoins(t Time) Coins {
   return max(va.GetVestingCoins(t) - va.DelegatedVesting, 0)
}
```

`x/bank` `ViewKeeper` sau đó có thể cung cấp API để xác định coin bị khoá và coin
có thể chi tiêu cho bất kỳ tài khoản nào:

```go
func (k Keeper) LockedCoins(ctx Context, addr AccAddress) Coins {
    acc := k.GetAccount(ctx, addr)
    if acc != nil {
        if acc.IsVesting() {
            return acc.LockedCoins(ctx.BlockTime())
        }
    }

    // non-vesting accounts do not have any locked coins
    return NewCoins()
}
```

#### Keeper/Handler

Keeper của `x/bank` tương ứng nên xử lý việc gửi coin một cách phù hợp tuỳ theo
tài khoản có phải vesting account hay không.

```go
func (k Keeper) SendCoins(ctx Context, from Account, to Account, amount Coins) {
    bc := k.GetBalances(ctx, from)
    v := k.LockedCoins(ctx, from)

    spendable := bc - v
    newCoins := spendable - amount
    assert(newCoins >= 0)

    from.SetBalance(newCoins)
    to.AddBalance(amount)

    // save balances...
}
```

### Uỷ quyền stake (Delegating)

Với một vesting account muốn delegate `D` coin, thực hiện:

1. Xác minh `BC >= D > 0`
2. Tính `X := min(max(V - DV, 0), D)` (phần của `D` là coin vesting)
3. Tính `Y := D - X` (phần của `D` là coin free)
4. Đặt `DV += X`
5. Đặt `DF += Y`

```go
func (va VestingAccount) TrackDelegation(t Time, balance Coins, amount Coins) {
    assert(balance <= amount)
    x := min(max(va.GetVestingCoins(t) - va.DelegatedVesting, 0), amount)
    y := amount - x

    va.DelegatedVesting += x
    va.DelegatedFree += y
}
```

**Lưu ý** `TrackDelegation` chỉ sửa các field `DelegatedVesting` và `DelegatedFree`,
nên caller phía trên BẮT BUỘC phải sửa field `Coins` bằng cách trừ `amount`.

#### Keeper/Handler

```go
func DelegateCoins(t Time, from Account, amount Coins) {
    if isVesting(from) {
        from.TrackDelegation(t, amount)
    } else {
        from.SetBalance(sc - amount)
    }

    // save account...
}
```

### Huỷ uỷ quyền (Undelegating)

Với một vesting account muốn undelegate `D` coin, thực hiện:

> LƯU Ý: `DV < D` và `(DV + DF) < D` có thể xảy ra do đặc điểm “lạ” của việc làm tròn
> trong logic delegate/undelegate.

1. Xác minh `D > 0`
2. Tính `X := min(DF, D)` (phần của `D` nên trở thành free, ưu tiên free coin)
3. Tính `Y := min(DV, D - X)` (phần của `D` nên vẫn là vesting)
4. Đặt `DF -= X`
5. Đặt `DV -= Y`

```go
func (cva ContinuousVestingAccount) TrackUndelegation(amount Coins) {
    x := min(cva.DelegatedFree, amount)
    y := amount - x

    cva.DelegatedFree -= x
    cva.DelegatedVesting -= y
}
```

**Lưu ý** `TrackUnDelegation` chỉ sửa các field `DelegatedVesting` và `DelegatedFree`,
nên caller phía trên BẮT BUỘC phải sửa field `Coins` bằng cách cộng `amount`.

**Lưu ý**: Nếu một delegation bị slashed, continuous vesting account có thể kết
thúc với lượng `DV` dư thừa, ngay cả sau khi tất cả coin đã vested. Nguyên nhân là
do ưu tiên undelegate free coin.

**Lưu ý**: Lượng undelegation (bond refund) có thể lớn hơn lượng delegated vesting
(bond) do cách undelegation truncate bond refund, có thể làm tăng nhẹ exchange rate
(tokens/shares) của validator nếu token undelegate không nguyên.

#### Keeper/Handler

```go
func UndelegateCoins(to Account, amount Coins) {
    if isVesting(to) {
        if to.DelegatedFree + to.DelegatedVesting >= amount {
            to.TrackUndelegation(amount)
            // save account ...
        }
    } else {
        AddBalance(to, amount)
        // save account...
    }
}
```

## Keeper & Handler

Các hiện thực `VestingAccount` nằm trong `x/auth`. Tuy nhiên, bất kỳ keeper nào
trong module (ví dụ staking trong `x/staking`) muốn có thể sử dụng vesting coin
phải gọi các phương thức rõ ràng trên `x/bank` keeper (ví dụ `DelegateCoins`) thay
vì `SendCoins` và `SubtractCoins`.

Ngoài ra, vesting account cũng nên có thể chi tiêu bất kỳ coin nào nó nhận từ người
dùng khác. Vì vậy, handler `MsgSend` của module bank nên trả lỗi nếu vesting account
đang cố gửi một lượng vượt quá lượng coin đã mở khoá của họ.

Xem đặc tả bên trên để biết chi tiết triển khai đầy đủ.

## Khởi tạo Genesis

Để khởi tạo cả vesting lẫn non-vesting account, struct `GenesisAccount` có thêm
các field mới: `Vesting`, `StartTime`, và `EndTime`. Các tài khoản dự kiến là
`BaseAccount` hoặc bất kỳ loại non-vesting nào có `Vesting = false`. Logic khởi tạo
genesis (ví dụ `initFromGenesisState`) phải parse và trả về đúng loại tài khoản
tương ứng dựa trên các field này.

```go
type GenesisAccount struct {
    // ...

    // vesting account fields
    OriginalVesting  sdk.Coins `json:"original_vesting"`
    DelegatedFree    sdk.Coins `json:"delegated_free"`
    DelegatedVesting sdk.Coins `json:"delegated_vesting"`
    StartTime        int64     `json:"start_time"`
    EndTime          int64     `json:"end_time"`
}

func ToAccount(gacc GenesisAccount) Account {
    bacc := NewBaseAccount(gacc)

    if gacc.OriginalVesting > 0 {
        if ga.StartTime != 0 && ga.EndTime != 0 {
            // return a continuous vesting account
        } else if ga.EndTime != 0 {
            // return a delayed vesting account
        } else {
            // invalid genesis vesting account provided
            panic()
        }
    }

    return bacc
}
```

## Ví dụ

### Đơn giản

Cho một continuous vesting account với 10 coin vesting.

```text
OV = 10
DF = 0
DV = 0
BC = 10
V = 10
V' = 0
```

1. Ngay lập tức nhận thêm 1 coin

    ```text
    BC = 11
    ```

2. Thời gian trôi qua, 2 coin vest

    ```text
    V = 8
    V' = 2
    ```

3. Delegate 4 coin cho validator A

    ```text
    DV = 4
    BC = 7
    ```

4. Gửi 3 coin

    ```text
    BC = 4
    ```

5. Thời gian trôi qua tiếp, 2 coin nữa vest

    ```text
    V = 6
    V' = 4
    ```

6. Gửi 2 coin. Tại thời điểm này tài khoản không thể gửi thêm cho tới khi vest
thêm coin hoặc nhận thêm coin. Tuy nhiên nó vẫn có thể delegate.

    ```text
    BC = 2
    ```

### Slashing

Cùng điều kiện ban đầu như ví dụ đơn giản.

1. Thời gian trôi qua, 5 coin vest

    ```text
    V = 5
    V' = 5
    ```

2. Delegate 5 coin cho validator A

    ```text
    DV = 5
    BC = 5
    ```

3. Delegate 5 coin cho validator B

    ```text
    DF = 5
    BC = 0
    ```

4. Validator A bị slashed 50%, làm delegation tới A chỉ còn tương đương 2.5 coin
5. Undelegate khỏi validator A (2.5 coin)

    ```text
    DF = 5 - 2.5 = 2.5
    BC = 0 + 2.5 = 2.5
    ```

6. Undelegate khỏi validator B (5 coin). Tại điểm này tài khoản chỉ có thể gửi
2.5 coin trừ khi nhận thêm coin hoặc chờ vest thêm. Tuy nhiên nó vẫn có thể delegate.

    ```text
    DV = 5 - 2.5 = 2.5
    DF = 2.5 - 2.5 = 0
    BC = 2.5 + 5 = 7.5
    ```

    Lưu ý rằng ta có một lượng `DV` dư thừa.

### Periodic vesting

Một vesting account được tạo sao cho 100 token sẽ được mở khoá trong 1 năm, với
1/4 token vest mỗi quý. Lịch vesting sẽ như sau:

```yaml
Periods:
- amount: 25stake, length: 7884000
- amount: 25stake, length: 7884000
- amount: 25stake, length: 7884000
- amount: 25stake, length: 7884000
```

```text
OV = 100
DF = 0
DV = 0
BC = 100
V = 100
V' = 0
```

1. Ngay lập tức nhận thêm 1 coin

    ```text
    BC = 101
    ```

2. Kỳ vesting 1 kết thúc, 25 coin vest

    ```text
    V = 75
    V' = 25
    ```

3. Trong kỳ vesting 2, chuyển 5 coin và delegate 5 coin

    ```text
    DV = 5
    BC = 91
    ```

4. Kỳ vesting 2 kết thúc, 25 coin vest

    ```text
    V = 50
    V' = 50
    ```

## Thuật ngữ

* OriginalVesting: Lượng coin (theo mệnh giá) ban đầu thuộc về vesting account. Các coin này được thiết lập tại genesis.
* StartTime: Thời điểm BFT mà tại đó vesting account bắt đầu vest.
* EndTime: Thời điểm BFT mà tại đó vesting account vest hoàn toàn.
* DelegatedFree: Lượng coin (theo mệnh giá) được theo dõi đã delegate từ vesting account và đã vested hoàn toàn tại thời điểm delegate.
* DelegatedVesting: Lượng coin (theo mệnh giá) được theo dõi đã delegate từ vesting account và đang vesting tại thời điểm delegate.
* ContinuousVestingAccount: Hiện thực vesting account vest tuyến tính theo thời gian.
* DelayedVestingAccount: Hiện thực vesting account chỉ vest hoàn toàn toàn bộ coin tại một thời điểm nhất định.
* PeriodicVestingAccount: Hiện thực vesting account vest theo một lịch vesting tuỳ biến.
* PermanentLockedAccount: Không bao giờ mở khoá coin, khoá vô thời hạn. Coin trong tài khoản này vẫn có thể dùng để delegate và bỏ phiếu governance ngay cả khi bị khoá.

## CLI

Người dùng có thể truy vấn và tương tác với module `vesting` qua CLI.

### Transactions

Các lệnh `tx` cho phép người dùng tương tác với module `vesting`.

```bash
simd tx vesting --help
```

#### create-periodic-vesting-account

Lệnh `create-periodic-vesting-account` tạo một vesting account mới được cấp vốn
với một phân bổ token, trong đó có một chuỗi coin và độ dài kỳ (giây). Các kỳ là
tuần tự: thời lượng của một kỳ chỉ bắt đầu sau khi kỳ trước kết thúc. Thời lượng
kỳ đầu tiên bắt đầu khi tài khoản được tạo.

```bash
simd tx vesting create-periodic-vesting-account [to_address] [periods_json_file] [flags]
```

Ví dụ:

```bash
simd tx vesting create-periodic-vesting-account cosmos1.. periods.json
```

#### create-vesting-account

Lệnh `create-vesting-account` tạo một vesting account mới được cấp vốn với một
phân bổ token. Tài khoản có thể là delayed hoặc continuous vesting account, được
xác định bởi cờ `--delayed`. Tất cả vesting account được tạo sẽ có start time
đặt theo thời gian của block đã commit. `end_time` phải được cung cấp dưới dạng
UNIX epoch timestamp.

```bash
simd tx vesting create-vesting-account [to_address] [amount] [end_time] [flags]
```

Ví dụ:

```bash
simd tx vesting create-vesting-account cosmos1.. 100stake 2592000
```

