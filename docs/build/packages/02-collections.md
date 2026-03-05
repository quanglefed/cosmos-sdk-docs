# Collections (Bộ Sưu Tập)

Collections là một thư viện nhằm đơn giản hóa trải nghiệm liên quan đến việc xử lý trạng thái module.

Các Cosmos SDK module xử lý trạng thái của chúng bằng interface `KVStore`. Vấn đề khi làm việc với
`KVStore` là nó buộc bạn phải nghĩ về trạng thái như các cặp KV bytes trong khi thực tế phần lớn
trạng thái đến từ các đối tượng golang cụ thể phức tạp (string, int, struct, v.v.).

Collections cho phép bạn làm việc với trạng thái như thể chúng là các đối tượng golang bình thường và loại bỏ
nhu cầu nghĩ về trạng thái của bạn như các raw bytes trong code.

Nó cũng cho phép bạn migrate trạng thái hiện có mà không gây ra bất kỳ state breakage nào buộc bạn vào
các chain state migration phức tạp và tốn công.

## Cài Đặt

Để cài đặt collections trong dự án cosmos-sdk chain của bạn, chạy lệnh sau:

```shell
go get cosmossdk.io/collections
```

## Các Kiểu Core

Collections cung cấp 5 API khác nhau để làm việc với trạng thái, sẽ được khám phá trong các phần tiếp theo, các API này là:

* ``Map``: để làm việc với các cặp KV tùy ý có kiểu.
* ``KeySet``: để làm việc chỉ với các key có kiểu
* ``Item``: để làm việc chỉ với một giá trị có kiểu
* ``Sequence``: là một số tăng đơn điệu.
* ``IndexedMap``: kết hợp ``Map`` và `KeySet` để cung cấp `Map` với khả năng lập chỉ mục.

## Các Thành Phần Sơ Bộ

Trước khi khám phá các kiểu collections khác nhau và khả năng của chúng, cần giới thiệu
ba thành phần mà mọi collection đều dùng chung. Thực tế khi khởi tạo một kiểu collection bằng cách làm, ví dụ,
```collections.NewMap/collections.NewItem/...``` bạn sẽ phải truyền cho chúng một số đối số chung.

Ví dụ, trong code:

```go
package collections

import (
    "cosmossdk.io/collections"
    storetypes "cosmossdk.io/store/types"
    sdk "github.com/cosmos/cosmos-sdk/types"
)

var AllowListPrefix = collections.NewPrefix(0)

type Keeper struct {
	Schema    collections.Schema
	AllowList collections.KeySet[string]
}

func NewKeeper(storeKey *storetypes.KVStoreKey) Keeper {
	sb := collections.NewSchemaBuilder(sdk.OpenKVStore(storeKey))

	return Keeper{
		AllowList: collections.NewKeySet(sb, AllowListPrefix, "allow_list", collections.StringKey),
	}
}

```

Hãy phân tích các đối số dùng chung, chúng làm gì và tại sao chúng ta cần chúng.

### SchemaBuilder

Đối số đầu tiên được truyền là ``SchemaBuilder``

`SchemaBuilder` là một cấu trúc theo dõi toàn bộ trạng thái của một module, nó không được yêu cầu bởi collections
để xử lý trạng thái nhưng nó cung cấp một cách động và phản chiếu để clients khám phá trạng thái của module.

Chúng ta khởi tạo một ``SchemaBuilder`` bằng cách truyền cho nó một hàm mà với store key của module trả về store cụ thể của module.

Sau đó chúng ta cần truyền schema builder cho mọi kiểu collection mà chúng ta khởi tạo trong keeper của mình, trong trường hợp này là `AllowList`.

### Prefix

Đối số thứ hai được truyền cho ``KeySet`` của chúng ta là `collections.Prefix`, một prefix đại diện cho một phân vùng của `KVStore` của module
nơi tất cả trạng thái của một collection cụ thể sẽ được lưu.

Vì một module có thể có nhiều collection, điều sau được mong đợi:

* module params sẽ trở thành `collections.Item`
* `AllowList` là `collections.KeySet`

Chúng ta không muốn một collection ghi đè trạng thái của collection khác nên chúng ta truyền cho nó một prefix, định nghĩa phân vùng storage thuộc về collection.

Nếu bạn đã build modules, prefix tương ứng với các items bạn đang tạo trong file ``types/keys.go``, ví dụ: https://github.com/cosmos/cosmos-sdk/blob/v0.52.0-rc.1/x/feegrant/key.go#L16~L22

Cũ của bạn:

```go
var (
	// FeeAllowanceKeyPrefix là tập hợp kvstore cho fee allowance data
	// - 0x00<allowance_key_bytes>: allowance
	FeeAllowanceKeyPrefix = []byte{0x00}

	// FeeAllowanceQueueKeyPrefix là tập hợp kvstore cho fee allowance keys data
	// - 0x01<allowance_prefix_queue_key_bytes>: <empty value>
	FeeAllowanceQueueKeyPrefix = []byte{0x01}
)
```

trở thành:

```go
var (
	// FeeAllowanceKeyPrefix là tập hợp kvstore cho fee allowance data
	// - 0x00<allowance_key_bytes>: allowance
	FeeAllowanceKeyPrefix = collections.NewPrefix(0)

	// FeeAllowanceQueueKeyPrefix là tập hợp kvstore cho fee allowance keys data
	// - 0x01<allowance_prefix_queue_key_bytes>: <empty value>
	FeeAllowanceQueueKeyPrefix = collections.NewPrefix(1)
)
```

#### Quy Tắc

``collections.NewPrefix`` chấp nhận `uint8`, `string` hoặc `[]bytes`. Nên dùng `uint8` luôn tăng để tiết kiệm dung lượng đĩa.

Một collection **KHÔNG ĐƯỢC** chia sẻ cùng prefix với collection khác trong cùng module, và prefix của collection **KHÔNG BAO GIỜ** được bắt đầu bằng cùng prefix với collection khác, ví dụ:

```go
prefix1 := collections.NewPrefix("prefix")
prefix2 := collections.NewPrefix("prefix") // ĐÂY LÀ XẤU!
```

```go
prefix1 := collections.NewPrefix("a")
prefix2 := collections.NewPrefix("aa") // prefix2 bắt đầu giống prefix1: XẤU!!!
```

### Tên Dễ Đọc (Human-Readable Name)

Tham số thứ ba chúng ta truyền cho một collection là một string, là tên dễ đọc.
Nó cần thiết để làm cho vai trò của một collection có thể hiểu được bởi các client không biết gì về
những gì một module đang lưu trữ trong state.

#### Quy Tắc

Mỗi collection trong một module **PHẢI** có một tên dễ đọc duy nhất.

## Key và Value Codecs

Một collection là generic trên kiểu bạn có thể sử dụng làm key hoặc value.
Điều này làm cho collections "ngu ngốc" nhưng cũng có nghĩa là về mặt lý thuyết chúng ta có thể lưu trữ mọi thứ
có thể là một kiểu go vào một collection. Chúng ta không bị giới hạn với bất kỳ loại encoding nào (dù là proto, json hay bất kỳ thứ gì khác)

Vì vậy, một collection cần được cung cấp một cách để hiểu cách chuyển đổi keys và values của bạn sang bytes.
Điều này được thực hiện thông qua ``KeyCodec`` và `ValueCodec`, là các đối số bạn truyền cho
collections khi khởi tạo chúng bằng các hàm khởi tạo ```collections.NewMap/collections.NewItem/...```.

LƯU Ý: Nhìn chung bạn sẽ không bao giờ được yêu cầu triển khai ``Key/ValueCodec`` của riêng mình vì
SDK và thư viện collections đã đi kèm với triển khai mặc định, an toàn và nhanh của chúng.
Bạn có thể cần triển khai chúng chỉ khi bạn đang migrate sang collections và có sự không tương thích layout trạng thái.

Hãy khám phá một ví dụ:

```go
package collections

import (
	"cosmossdk.io/collections"
	storetypes "cosmossdk.io/store/types"
	sdk "github.com/cosmos/cosmos-sdk/types"
)

var IDsPrefix = collections.NewPrefix(0)

type Keeper struct {
	Schema    collections.Schema
	IDs   collections.Map[string, uint64]
}

func NewKeeper(storeKey *storetypes.KVStoreKey) Keeper {
	sb := collections.NewSchemaBuilder(sdk.OpenKVStore(storeKey))

	return Keeper{
		IDs: collections.NewMap(sb, IDsPrefix, "ids", collections.StringKey, collections.Uint64Value),
	}
}
```

Chúng ta đang khởi tạo một map trong đó key là string và value là `uint64`.
Chúng ta đã biết ba đối số đầu tiên của hàm ``NewMap``.

Tham số thứ tư là `KeyCodec` của chúng ta, chúng ta biết rằng ``Map`` có `string` làm key nên chúng ta truyền cho nó một `KeyCodec` xử lý strings làm keys.

Tham số thứ năm là `ValueCodec` của chúng ta, chúng ta biết rằng `Map` có `uint64` làm value nên chúng ta truyền cho nó một `ValueCodec` xử lý uint64.

Collections đã đi kèm với tất cả các triển khai cần thiết cho các kiểu primitive của golang.

Hãy xem một ví dụ khác, cái này gần hơn với những gì chúng ta build bằng cosmos SDK, giả sử chúng ta muốn
tạo `collections.Map` ánh xạ các địa chỉ tài khoản với base account của chúng. Vì vậy chúng ta muốn ánh xạ `sdk.AccAddress` sang `auth.BaseAccount` (là proto):

```go
package collections

import (
	"cosmossdk.io/collections"
	storetypes "cosmossdk.io/store/types"
	"github.com/cosmos/cosmos-sdk/codec"
	sdk "github.com/cosmos/cosmos-sdk/types"
	authtypes "github.com/cosmos/cosmos-sdk/x/auth/types"
)

var AccountsPrefix = collections.NewPrefix(0)

type Keeper struct {
	Schema    collections.Schema
	Accounts   collections.Map[sdk.AccAddress, authtypes.BaseAccount]
}

func NewKeeper(storeKey *storetypes.KVStoreKey, cdc codec.BinaryCodec) Keeper {
	sb := collections.NewSchemaBuilder(sdk.OpenKVStore(storeKey))
	return Keeper{
		Accounts: collections.NewMap(sb, AccountsPrefix, "accounts",
			sdk.AccAddressKey, codec.CollValue[authtypes.BaseAccount](cdc)),
	}
}
```

Như chúng ta có thể thấy ở đây vì `collections.Map` của chúng ta ánh xạ `sdk.AccAddress` sang `authtypes.BaseAccount`,
chúng ta sử dụng `sdk.AccAddressKey` là triển khai `KeyCodec` cho `AccAddress` và chúng ta sử dụng `codec.CollValue` để
encode kiểu proto `BaseAccount` của chúng ta.

Nhìn chung bạn sẽ luôn tìm thấy key và value codec tương ứng cho các kiểu trong đường dẫn `go.mod` bạn đang sử dụng
để import kiểu đó. Nếu bạn muốn encode proto values, tham khảo hàm `codec.CollValue` của codec, cho phép bạn
encode bất kỳ kiểu nào triển khai interface `proto.Message`.

## Map

Chúng ta phân tích kiểu collection đầu tiên và quan trọng nhất, ``collections.Map``.
Đây là kiểu mà mọi thứ khác xây dựng trên đó.

### Trường Hợp Sử Dụng

`collections.Map` được sử dụng để ánh xạ các key tùy ý với các value tùy ý.

### Ví Dụ

Dễ giải thích khả năng của `collections.Map` thông qua một ví dụ:

```go
package collections

import (
	"cosmossdk.io/collections"
	storetypes "cosmossdk.io/store/types"
	"fmt"
	"github.com/cosmos/cosmos-sdk/codec"
	sdk "github.com/cosmos/cosmos-sdk/types"
	authtypes "github.com/cosmos/cosmos-sdk/x/auth/types"
)

var AccountsPrefix = collections.NewPrefix(0)

type Keeper struct {
	Schema    collections.Schema
	Accounts   collections.Map[sdk.AccAddress, authtypes.BaseAccount]
}

func NewKeeper(storeKey *storetypes.KVStoreKey, cdc codec.BinaryCodec) Keeper {
	sb := collections.NewSchemaBuilder(sdk.OpenKVStore(storeKey))
	return Keeper{
		Accounts: collections.NewMap(sb, AccountsPrefix, "accounts",
			sdk.AccAddressKey, codec.CollValue[authtypes.BaseAccount](cdc)),
	}
}

func (k Keeper) CreateAccount(ctx sdk.Context, addr sdk.AccAddress, account authtypes.BaseAccount) error {
	has, err := k.Accounts.Has(ctx, addr)
	if err != nil {
		return err
	}
	if has {
		return fmt.Errorf("account already exists: %s", addr)
	}
	
	err = k.Accounts.Set(ctx, addr, account)
	if err != nil {
		return err
	}
	return nil
}

func (k Keeper) GetAccount(ctx sdk.Context, addr sdk.AccAddress) (authtypes.BaseAccount, error) {
	acc, err := k.Accounts.Get(ctx, addr)
	if err != nil {
		return authtypes.BaseAccount{}, err
	}
	
	return acc,	nil
}

func (k Keeper) RemoveAccount(ctx sdk.Context, addr sdk.AccAddress) error {
	err := k.Accounts.Remove(ctx, addr)
	if err != nil {
		return err
	}
	return nil
}
```

#### Phương Thức Set

Set ánh xạ với `AccAddress` được cung cấp (key) sang `auth.BaseAccount` (value).

Bên dưới, `collections.Map` sẽ chuyển đổi key và value sang bytes bằng cách sử dụng [key và value codec](README.md#key-and-value-codecs).
Nó sẽ thêm tiền tố [prefix](README.md#prefix) vào bytes key của chúng ta và lưu trữ nó trong KVStore của module.

#### Phương Thức Has

Phương thức has báo cáo xem key được cung cấp có tồn tại trong store hay không.

#### Phương Thức Get

Phương thức get chấp nhận `AccAddress` và trả về `auth.BaseAccount` liên quan nếu tồn tại, nếu không nó báo lỗi.

#### Phương Thức Remove

Phương thức remove chấp nhận `AccAddress` và xóa nó khỏi store. Nó sẽ không báo lỗi
nếu không tồn tại, để kiểm tra sự tồn tại trước khi xóa hãy sử dụng phương thức ``Has``.

#### Iteration (Duyệt Qua)

Iteration có phần riêng.

## KeySet

Kiểu collection thứ hai là `collections.KeySet`, như từ gợi ý nó duy trì
chỉ một tập hợp các key mà không có values.

#### Sự Tò Mò Về Triển Khai

`collections.KeySet` chỉ là `collections.Map` với một `key` nhưng không có value.
Value bên trong luôn giống nhau và được biểu diễn là byte slice rỗng ```[]byte{}```.

### Ví Dụ

Như thường lệ chúng ta khám phá kiểu collection thông qua một ví dụ:

```go
package collections

import (
	"cosmossdk.io/collections"
	storetypes "cosmossdk.io/store/types"
	"fmt"
	sdk "github.com/cosmos/cosmos-sdk/types"
)

var ValidatorsSetPrefix = collections.NewPrefix(0)

type Keeper struct {
	Schema        collections.Schema
	ValidatorsSet collections.KeySet[sdk.ValAddress]
}

func NewKeeper(storeKey *storetypes.KVStoreKey) Keeper {
	sb := collections.NewSchemaBuilder(sdk.OpenKVStore(storeKey))
	return Keeper{
		ValidatorsSet: collections.NewKeySet(sb, ValidatorsSetPrefix, "validators_set", sdk.ValAddressKey),
	}
}

func (k Keeper) AddValidator(ctx sdk.Context, validator sdk.ValAddress) error {
	has, err := k.ValidatorsSet.Has(ctx, validator)
	if err != nil {
		return err
	}
	if has {
		return fmt.Errorf("validator already in set: %s", validator)
	}
	
	err = k.ValidatorsSet.Set(ctx, validator)
	if err != nil {
		return err
	}
	
	return nil
}

func (k Keeper) RemoveValidator(ctx sdk.Context, validator sdk.ValAddress) error {
	err := k.ValidatorsSet.Remove(ctx, validator)
	if err != nil {
		return err
	}
	return nil
}
```

Sự khác biệt đầu tiên chúng ta nhận thấy là `KeySet` cần chúng ta chỉ định một tham số kiểu: key (`sdk.ValAddress` trong trường hợp này).
Sự khác biệt thứ hai chúng ta nhận thấy là `KeySet` trong hàm `NewKeySet` của nó không yêu cầu
chúng ta chỉ định `ValueCodec` mà chỉ cần `KeyCodec`. Điều này là vì `KeySet` chỉ lưu keys chứ không lưu values.

Hãy khám phá các phương thức.

#### Phương Thức Has

Has cho phép chúng ta hiểu xem một key có hiện diện trong `collections.KeySet` hay không, hoạt động giống như `collections.Map.Has`

#### Phương Thức Set

Set chèn key được cung cấp vào `KeySet`.

#### Phương Thức Remove

Remove xóa key được cung cấp khỏi `KeySet`, nó không báo lỗi nếu key không tồn tại,
nếu cần kiểm tra sự tồn tại trước khi xóa nó cần được kết hợp với phương thức `Has`.

## Item

Kiểu collection thứ ba là `collections.Item`.
Nó chỉ lưu một item đơn lẻ, hữu ích ví dụ cho parameters, luôn chỉ có một instance
của parameters trong state.

### Sự Tò Mò Về Triển Khai

`collections.Item` chỉ là `collections.Map` không có key mà chỉ có value.
Key là prefix của collection!

### Ví Dụ

```go
package collections

import (
	"cosmossdk.io/collections"
	storetypes "cosmossdk.io/store/types"
	"github.com/cosmos/cosmos-sdk/codec"
	sdk "github.com/cosmos/cosmos-sdk/types"
	stakingtypes "cosmossdk.io/x/staking/types"
)

var ParamsPrefix = collections.NewPrefix(0)

type Keeper struct {
	Schema        collections.Schema
	Params collections.Item[stakingtypes.Params]
}

func NewKeeper(storeKey *storetypes.KVStoreKey, cdc codec.BinaryCodec) Keeper {
	sb := collections.NewSchemaBuilder(sdk.OpenKVStore(storeKey))
	return Keeper{
		Params: collections.NewItem(sb, ParamsPrefix, "params", codec.CollValue[stakingtypes.Params](cdc)),
	}
}

func (k Keeper) UpdateParams(ctx sdk.Context, params stakingtypes.Params) error {
	err := k.Params.Set(ctx, params)
	if err != nil {
		return err
	}
	return nil
}

func (k Keeper) GetParams(ctx sdk.Context) (stakingtypes.Params, error) {
	return k.Params.Get(ctx)
}
```

Sự khác biệt quan trọng đầu tiên chúng ta nhận thấy là chúng ta chỉ định một tham số kiểu, là value chúng ta đang lưu trữ.
Sự khác biệt quan trọng thứ hai là chúng ta không chỉ định `KeyCodec`, vì chúng ta chỉ lưu một item chúng ta đã biết key
và thực tế là nó là hằng số.

## Iteration (Duyệt Qua)

Một trong những tính năng chính của ``KVStore`` là duyệt qua các key.

Các collection xử lý keys (vậy là `Map`, `KeySet` và `IndexedMap`) cho phép bạn duyệt qua
các key theo cách an toàn và có kiểu. Tất cả đều dùng chung API, sự khác biệt duy nhất là
``KeySet`` trả về kiểu `Iterator` khác vì `KeySet` chỉ xử lý keys.

:::note

Mọi collection đều dùng chung ngữ nghĩa `Iterator` giống nhau.

:::

Hãy xem phương thức `Map.Iterate`:

```go
func (m Map[K, V]) Iterate(ctx context.Context, ranger Ranger[K]) (Iterator[K, V], error) 
```

Nó chấp nhận `collections.Ranger[K]`, là một API hướng dẫn map cách duyệt qua các key.
Như thường lệ chúng ta không cần triển khai bất cứ thứ gì ở đây vì `collections` đã cung cấp một số triển khai `Ranger` generic
hiển thị tất cả những gì bạn cần để làm việc với các range.

### Ví Dụ

Chúng ta có một `collections.Map` ánh xạ các tài khoản bằng ID `uint64`.

```go
package collections

import (
	"cosmossdk.io/collections"
	storetypes "cosmossdk.io/store/types"
	"github.com/cosmos/cosmos-sdk/codec"
	sdk "github.com/cosmos/cosmos-sdk/types"
	authtypes "github.com/cosmos/cosmos-sdk/x/auth/types"
)

var AccountsPrefix = collections.NewPrefix(0)

type Keeper struct {
	Schema   collections.Schema
	Accounts collections.Map[uint64, authtypes.BaseAccount]
}

func NewKeeper(storeKey *storetypes.KVStoreKey, cdc codec.BinaryCodec) Keeper {
	sb := collections.NewSchemaBuilder(sdk.OpenKVStore(storeKey))
	return Keeper{
		Accounts: collections.NewMap(sb, AccountsPrefix, "accounts", collections.Uint64Key, codec.CollValue[authtypes.BaseAccount](cdc)),
	}
}

func (k Keeper) GetAllAccounts(ctx sdk.Context) ([]authtypes.BaseAccount, error) {
	// truyền nil Ranger có nghĩa là: duyệt qua mọi key có thể
	iter, err := k.Accounts.Iterate(ctx, nil)
	if err != nil {
		return nil, err
	}
	accounts, err := iter.Values()
	if err != nil {
		return nil, err
	}

	return accounts, err
}

func (k Keeper) IterateAccountsBetween(ctx sdk.Context, start, end uint64) ([]authtypes.BaseAccount, error) {
	// API collections.Range cung cấp nhiều khả năng
	// như định nghĩa nơi iteration bắt đầu hoặc kết thúc.
	rng := new(collections.Range[uint64]).
		StartInclusive(start).
		EndExclusive(end).
		Descending()

	iter, err := k.Accounts.Iterate(ctx, rng)
	if err != nil {
		return nil, err
	}
	accounts, err := iter.Values()
	if err != nil {
		return nil, err
	}

	return accounts, nil
}

func (k Keeper) IterateAccounts(ctx sdk.Context, do func(id uint64, acc authtypes.BaseAccount) (stop bool)) error {
	iter, err := k.Accounts.Iterate(ctx, nil)
	if err != nil {
		return err
	}
	defer iter.Close()

	for ; iter.Valid(); iter.Next() {
		kv, err := iter.KeyValue()
		if err != nil {
			return err
		}

		if do(kv.Key, kv.Value) {
			break
		}
	}
	return nil
}
```

Hãy phân tích từng phương thức trong ví dụ và cách nó sử dụng API `Iterate` và `Iterator` được trả về.

#### GetAllAccounts

Trong `GetAllAccounts` chúng ta truyền cho `Iterate` một `Ranger` nil. Điều này có nghĩa là `Iterator` được trả về sẽ bao gồm
tất cả các key hiện có trong collection.

Sau đó chúng ta sử dụng phương thức `Values` từ API `Iterator` được trả về để thu thập tất cả các value vào một slice.

`Iterator` cung cấp các phương thức khác như `Keys()` để chỉ thu thập các key chứ không phải các value và `KeyValues` để thu thập
tất cả các key và value.


#### IterateAccountsBetween

Ở đây chúng ta sử dụng helper `collections.Range` để chuyên biệt hóa range của mình.
Chúng ta làm cho nó bắt đầu ở một điểm thông qua `StartInclusive` và kết thúc ở điểm khác với `EndExclusive`, sau đó
chúng ta hướng dẫn nó báo cáo kết quả theo thứ tự ngược qua `Descending`

Sau đó chúng ta truyền hướng dẫn range cho `Iterate` và nhận một `Iterator`, sẽ chỉ chứa các kết quả
chúng ta đã chỉ định trong range.

Sau đó chúng ta lại sử dụng phương thức `Values` của `Iterator` để thu thập tất cả kết quả.

`collections.Range` cũng cung cấp API `Prefix` không áp dụng cho tất cả các kiểu key,
ví dụ uint64 không thể được prefix vì nó có kích thước cố định, nhưng một key `string`
có thể được prefix.

#### IterateAccounts

Ở đây chúng ta trình bày cách thu thập value từ Iterator một cách lười biếng.

:::note

`Keys/Values/KeyValues` tiêu thụ và đóng `Iterator` hoàn toàn, ở đây chúng ta cần thực hiện lời gọi `defer iterator.Close()` tường minh.

:::

`Iterator` cũng hiển thị phương thức `Value` và `Key` để chỉ thu thập value hoặc key hiện tại, nếu không cần thu thập cả hai.

:::note

Đối với pattern `callback` này, collections hiển thị API `Walk`.

:::

## Composite Keys (Khóa Tổng Hợp)

Cho đến nay chúng ta chỉ làm việc với các key đơn giản, như `uint64`, địa chỉ tài khoản, v.v.
Có một số trường hợp phức tạp hơn, trong đó chúng ta cần xử lý composite keys.

Một key là composite khi nó được tạo thành từ nhiều key, ví dụ bank balances được lưu trữ như composite key
`(AccAddress, string)` trong đó phần đầu tiên là địa chỉ giữ coins và phần thứ hai là denom.

Ví dụ, giả sử địa chỉ `BOB` giữ `10atom,15osmo`, đây là cách nó được lưu trữ trong state:

```
(bob, atom) => 10
(bob, osmos) => 15
```

Bây giờ điều này cho phép lấy hiệu quả số dư denom cụ thể của một địa chỉ, đơn giản bằng cách `get` `(address, denom)`, hoặc lấy tất cả số dư
của một địa chỉ bằng cách prefix trên `(address)`.

Hãy xem bây giờ chúng ta có thể làm việc với composite keys bằng cách sử dụng collections như thế nào.

### Ví Dụ

Trong ví dụ của chúng ta, chúng ta sẽ trình bày cách chúng ta có thể sử dụng collections khi xử lý balances, tương tự bank,
một balance là một mapping giữa `(address, denom) => math.Int`, composite key trong trường hợp của chúng ta là `(address, denom)`.

## Khởi Tạo Collection Composite Key

```go
package collections

import (
	"cosmossdk.io/collections"
	"cosmossdk.io/math"
	storetypes "cosmossdk.io/store/types"
	sdk "github.com/cosmos/cosmos-sdk/types"
)


var BalancesPrefix = collections.NewPrefix(1)

type Keeper struct {
	Schema   collections.Schema
	Balances collections.Map[collections.Pair[sdk.AccAddress, string], math.Int]
}

func NewKeeper(storeKey *storetypes.KVStoreKey) Keeper {
	sb := collections.NewSchemaBuilder(sdk.OpenKVStore(storeKey))
	return Keeper{
		Balances: collections.NewMap(
			sb, BalancesPrefix, "balances",
			collections.PairKeyCodec(sdk.AccAddressKey, collections.StringKey),
			sdk.IntValue,
		),
	}
}
```

### Định Nghĩa Map Key

Đầu tiên chúng ta có thể thấy rằng để định nghĩa một composite key gồm hai phần tử chúng ta sử dụng kiểu `collections.Pair`:

```go
collections.Map[collections.Pair[sdk.AccAddress, string], math.Int]
```

`collections.Pair` định nghĩa một key được tạo thành từ hai key khác, trong trường hợp của chúng ta phần đầu tiên là `sdk.AccAddress`, phần thứ hai
là `string`.

#### Khởi Tạo Key Codec

Các đối số để khởi tạo luôn giống nhau, điều duy nhất thay đổi là cách chúng ta khởi tạo
``KeyCodec``, vì key này được tạo thành từ hai key chúng ta sử dụng `collections.PairKeyCodec`, tạo ra
một `KeyCodec` được tạo thành từ hai key codec. Cái đầu tiên sẽ encode phần đầu tiên của key, cái thứ hai sẽ
encode phần thứ hai của key.


### Làm Việc Với Collection Composite Key

Hãy mở rộng ví dụ chúng ta đã sử dụng trước đó:

```go
var BalancesPrefix = collections.NewPrefix(1)

type Keeper struct {
	Schema   collections.Schema
	Balances collections.Map[collections.Pair[sdk.AccAddress, string], math.Int]
}

func NewKeeper(storeKey *storetypes.KVStoreKey) Keeper {
	sb := collections.NewSchemaBuilder(sdk.OpenKVStore(storeKey))
	return Keeper{
		Balances: collections.NewMap(
			sb, BalancesPrefix, "balances",
			collections.PairKeyCodec(sdk.AccAddressKey, collections.StringKey),
			sdk.IntValue,
		),
	}
}

func (k Keeper) SetBalance(ctx sdk.Context, address sdk.AccAddress, denom string, amount math.Int) error {
	key := collections.Join(address, denom)
	return k.Balances.Set(ctx, key, amount)
}

func (k Keeper) GetBalance(ctx sdk.Context, address sdk.AccAddress, denom string) (math.Int, error) {
	return k.Balances.Get(ctx, collections.Join(address, denom))
}

func (k Keeper) GetAllAddressBalances(ctx sdk.Context, address sdk.AccAddress) (sdk.Coins, error) {
	balances := sdk.NewCoins()

	rng := collections.NewPrefixedPairRange[sdk.AccAddress, string](address)

	iter, err := k.Balances.Iterate(ctx, rng)
	if err != nil {
		return nil, err
	}

	kvs, err := iter.KeyValues()
	if err != nil {
		return nil, err
	}

	for _, kv := range kvs {
		balances = balances.Add(sdk.NewCoin(kv.Key.K2(), kv.Value))
	}
	return balances, nil
}

func (k Keeper) GetAllAddressBalancesBetween(ctx sdk.Context, address sdk.AccAddress, startDenom, endDenom string) (sdk.Coins, error) {
    rng := collections.NewPrefixedPairRange[sdk.AccAddress, string](address).
        StartInclusive(startDenom).
        EndInclusive(endDenom)

    iter, err := k.Balances.Iterate(ctx, rng)
    if err != nil {
        return nil, err
	}
    ...
}
```

#### SetBalance

Như chúng ta có thể thấy ở đây chúng ta đang đặt số dư của một địa chỉ cho một denom cụ thể.
Chúng ta sử dụng hàm `collections.Join` để tạo composite key.
`collections.Join` trả về một `collections.Pair` (là key của `collections.Map` của chúng ta)

`collections.Pair` chứa hai key chúng ta đã nối, nó cũng hiển thị hai phương thức: `K1` để lấy phần đầu tiên của key và `K2` để lấy phần thứ hai.

Như thường lệ, chúng ta sử dụng phương thức `collections.Map.Set` để ánh xạ composite key sang value của chúng ta (`math.Int` trong trường hợp này)

#### GetBalance

Để lấy một value trong collection composite key, chúng ta đơn giản sử dụng `collections.Join` để tạo key.

#### GetAllAddressBalances

Chúng ta sử dụng `collections.PrefixedPairRange` để duyệt qua tất cả các key bắt đầu bằng địa chỉ được cung cấp.
Cụ thể iteration sẽ báo cáo tất cả các số dư thuộc về địa chỉ được cung cấp.

Phần đầu tiên là chúng ta khởi tạo `PrefixedPairRange`, là một triển khai `Ranger` nhằm giúp
duyệt qua các `Pair` key.

```go
	rng := collections.NewPrefixedPairRange[sdk.AccAddress, string](address)
```

Như chúng ta có thể thấy ở đây chúng ta đang truyền các tham số kiểu của `collections.Pair` vì suy luận kiểu golang
đối với generics không linh hoạt như các ngôn ngữ khác, vì vậy chúng ta cần chỉ định rõ ràng các kiểu của pair key là gì.

#### GetAllAddressesBalancesBetween

Điều này trình bày cách chúng ta có thể chuyên biệt hóa thêm range để giới hạn kết quả hơn nữa, bằng cách chỉ định
range giữa phần thứ hai của key (trong trường hợp của chúng ta là các denom, là strings).

## IndexedMap

`collections.IndexedMap` là một collection sử dụng bên dưới `collections.Map`, và có một struct chứa các index mà chúng ta cần định nghĩa.

### Ví Dụ

Giả sử chúng ta có một struct `auth.BaseAccount` trông như sau:

```go
type BaseAccount struct {
	AccountNumber uint64     `protobuf:"varint,3,opt,name=account_number,json=accountNumber,proto3" json:"account_number,omitempty"`
	Sequence      uint64     `protobuf:"varint,4,opt,name=sequence,proto3" json:"sequence,omitempty"`
}
```

Đầu tiên, khi chúng ta lưu các tài khoản trong state chúng ta ánh xạ chúng bằng primary key `sdk.AccAddress`.
Nếu là `collections.Map`, nó sẽ là `collections.Map[sdk.AccAddress, authtypes.BaseAccount]`.

Sau đó chúng ta cũng muốn có thể lấy một tài khoản không chỉ bằng `sdk.AccAddress`, mà còn bằng `AccountNumber`.

Vì vậy chúng ta có thể nói chúng ta muốn tạo một `Index` ánh xạ `BaseAccount` sang `AccountNumber` của nó.

Chúng ta cũng biết rằng `Index` này là unique. Unique có nghĩa là chỉ có thể có một `BaseAccount` ánh xạ đến một
`AccountNumber` cụ thể.

Đầu tiên, chúng ta bắt đầu bằng cách định nghĩa đối tượng chứa index của chúng ta:

```go
var AccountsNumberIndexPrefix = collections.NewPrefix(1)

type AccountsIndexes struct {
	Number *indexes.Unique[uint64, sdk.AccAddress, authtypes.BaseAccount]
}

func NewAccountIndexes(sb *collections.SchemaBuilder) AccountsIndexes {
	return AccountsIndexes{
		Number: indexes.NewUnique(
			sb, AccountsNumberIndexPrefix, "accounts_by_number",
			collections.Uint64Key, sdk.AccAddressKey,
			func(_ sdk.AccAddress, v authtypes.BaseAccount) (uint64, error) {
				return v.AccountNumber, nil
			},
		),
	}
}
```

Chúng ta tạo một struct `AccountIndexes` chứa một trường: `Number`. Trường này đại diện cho index `AccountNumber` của chúng ta.
`AccountNumber` là một trường của `authtypes.BaseAccount` và nó là `uint64`.

Sau đó chúng ta có thể thấy trong struct `AccountIndexes` của chúng ta trường `Number` được định nghĩa là:

```go
*indexes.Unique[uint64, sdk.AccAddress, authtypes.BaseAccount]
```

Trong đó tham số kiểu đầu tiên là `uint64`, là kiểu trường của index của chúng ta.
Tham số kiểu thứ hai là primary key `sdk.AccAddress`.
Và tham số kiểu thứ ba là đối tượng thực tế chúng ta đang lưu trữ `authtypes.BaseAccount`.

Sau đó chúng ta tạo hàm `NewAccountIndexes` khởi tạo và trả về struct `AccountsIndexes`.

Hàm nhận một `SchemaBuilder`. Sau đó chúng ta khởi tạo `indexes.Unique`, hãy phân tích các đối số chúng ta truyền cho
`indexes.NewUnique`.

#### LƯU Ý: danh sách index

Struct `AccountsIndexes` chứa các index, hàm `NewIndexedMap` sẽ suy ra các index từ struct đó
bằng cách sử dụng reflection, điều này chỉ xảy ra khi init và không tốn kém về mặt tính toán. Trong trường hợp bạn muốn khai báo rõ ràng
các index: triển khai interface `Indexes` trong struct `AccountsIndexes`:

```go
func (a AccountsIndexes) IndexesList() []collections.Index[sdk.AccAddress, authtypes.BaseAccount] {
    return []collections.Index[sdk.AccAddress, authtypes.BaseAccount]{a.Number}
}
```

#### Khởi Tạo `indexes.Unique`

Ba đối số đầu tiên chúng ta đã biết, chúng là: `SchemaBuilder`, `Prefix` là prefix index của chúng ta (phân vùng
nơi mối quan hệ index keys cho `Number` index sẽ được duy trì), và tên dễ đọc cho `Number` index.

Đối số thứ hai là `collections.Uint64Key` là key codec để xử lý `uint64` key, chúng ta truyền điều đó vì
key chúng ta đang cố index là `uint64` key (account number), và sau đó chúng ta truyền đối số thứ năm là primary key codec,
trong trường hợp của chúng ta là `sdk.AccAddress` (hãy nhớ: chúng ta ánh xạ `sdk.AccAddress` => `BaseAccount`).

Sau đó là tham số cuối cùng chúng ta truyền một hàm: cho `BaseAccount` trả về `AccountNumber` của nó.

Sau đó chúng ta có thể tiến hành khởi tạo `IndexedMap`.

```go
var AccountsPrefix = collections.NewPrefix(0)

type Keeper struct {
	Schema   collections.Schema
	Accounts *collections.IndexedMap[sdk.AccAddress, authtypes.BaseAccount, AccountsIndexes]
}

func NewKeeper(storeKey *storetypes.KVStoreKey, cdc codec.BinaryCodec) Keeper {
	sb := collections.NewSchemaBuilder(sdk.OpenKVStore(storeKey))
	return Keeper{
		Accounts: collections.NewIndexedMap(
			sb, AccountsPrefix, "accounts",
			sdk.AccAddressKey, codec.CollValue[authtypes.BaseAccount](cdc),
			NewAccountIndexes(sb),
		),
	}
}
```

Như chúng ta có thể thấy ở đây những gì chúng ta làm, hiện tại, giống như những gì chúng ta đã làm cho `collections.Map`.
Chúng ta truyền cho nó `SchemaBuilder`, `Prefix` nơi chúng ta dự định lưu trữ mapping giữa `sdk.AccAddress` và `authtypes.BaseAccount`,
tên dễ đọc và key codec `sdk.AccAddress` và value codec `authtypes.BaseAccount` tương ứng.

Sau đó chúng ta truyền khởi tạo của `AccountIndexes` qua `NewAccountIndexes`.

Ví dụ đầy đủ:

```go
package docs

import (
	"cosmossdk.io/collections"
	"cosmossdk.io/collections/indexes"
	storetypes "cosmossdk.io/store/types"
	"github.com/cosmos/cosmos-sdk/codec"
	sdk "github.com/cosmos/cosmos-sdk/types"
	authtypes "github.com/cosmos/cosmos-sdk/x/auth/types"
)

var AccountsNumberIndexPrefix = collections.NewPrefix(1)

type AccountsIndexes struct {
	Number *indexes.Unique[uint64, sdk.AccAddress, authtypes.BaseAccount]
}

func (a AccountsIndexes) IndexesList() []collections.Index[sdk.AccAddress, authtypes.BaseAccount] {
	return []collections.Index[sdk.AccAddress, authtypes.BaseAccount]{a.Number}
}

func NewAccountIndexes(sb *collections.SchemaBuilder) AccountsIndexes {
	return AccountsIndexes{
		Number: indexes.NewUnique(
			sb, AccountsNumberIndexPrefix, "accounts_by_number",
			collections.Uint64Key, sdk.AccAddressKey,
			func(_ sdk.AccAddress, v authtypes.BaseAccount) (uint64, error) {
				return v.AccountNumber, nil
			},
		),
	}
}

var AccountsPrefix = collections.NewPrefix(0)

type Keeper struct {
	Schema   collections.Schema
	Accounts *collections.IndexedMap[sdk.AccAddress, authtypes.BaseAccount, AccountsIndexes]
}

func NewKeeper(storeKey *storetypes.KVStoreKey, cdc codec.BinaryCodec) Keeper {
	sb := collections.NewSchemaBuilder(sdk.OpenKVStore(storeKey))
	return Keeper{
		Accounts: collections.NewIndexedMap(
			sb, AccountsPrefix, "accounts",
			sdk.AccAddressKey, codec.CollValue[authtypes.BaseAccount](cdc),
			NewAccountIndexes(sb),
		),
	}
}
```

### Làm Việc Với IndexedMaps

Trong khi khởi tạo `collections.IndexedMap` khá tẻ nhạt, làm việc với chúng lại cực kỳ mượt mà.

Hãy lấy ví dụ đầy đủ và mở rộng nó với một số trường hợp sử dụng.

```go
package docs

import (
	"cosmossdk.io/collections"
	"cosmossdk.io/collections/indexes"
	storetypes "cosmossdk.io/store/types"
	"github.com/cosmos/cosmos-sdk/codec"
	sdk "github.com/cosmos/cosmos-sdk/types"
	authtypes "github.com/cosmos/cosmos-sdk/x/auth/types"
)

var AccountsNumberIndexPrefix = collections.NewPrefix(1)

type AccountsIndexes struct {
	Number *indexes.Unique[uint64, sdk.AccAddress, authtypes.BaseAccount]
}

func (a AccountsIndexes) IndexesList() []collections.Index[sdk.AccAddress, authtypes.BaseAccount] {
	return []collections.Index[sdk.AccAddress, authtypes.BaseAccount]{a.Number}
}

func NewAccountIndexes(sb *collections.SchemaBuilder) AccountsIndexes {
	return AccountsIndexes{
		Number: indexes.NewUnique(
			sb, AccountsNumberIndexPrefix, "accounts_by_number",
			collections.Uint64Key, sdk.AccAddressKey,
			func(_ sdk.AccAddress, v authtypes.BaseAccount) (uint64, error) {
				return v.AccountNumber, nil
			},
		),
	}
}

var AccountsPrefix = collections.NewPrefix(0)

type Keeper struct {
	Schema   collections.Schema
	Accounts *collections.IndexedMap[sdk.AccAddress, authtypes.BaseAccount, AccountsIndexes]
}

func NewKeeper(storeKey *storetypes.KVStoreKey, cdc codec.BinaryCodec) Keeper {
	sb := collections.NewSchemaBuilder(sdk.OpenKVStore(storeKey))
	return Keeper{
		Accounts: collections.NewIndexedMap(
			sb, AccountsPrefix, "accounts",
			sdk.AccAddressKey, codec.CollValue[authtypes.BaseAccount](cdc),
			NewAccountIndexes(sb),
		),
	}
}

func (k Keeper) CreateAccount(ctx sdk.Context, addr sdk.AccAddress) error {
	nextAccountNumber := k.getNextAccountNumber()
	
	newAcc := authtypes.BaseAccount{
		AccountNumber: nextAccountNumber,
		Sequence:      0,
	}
	
	return k.Accounts.Set(ctx, addr, newAcc)
}

func (k Keeper) RemoveAccount(ctx sdk.Context, addr sdk.AccAddress) error {
	return k.Accounts.Remove(ctx, addr)
} 

func (k Keeper) GetAccountByNumber(ctx sdk.Context, accNumber uint64) (sdk.AccAddress, authtypes.BaseAccount, error) {
	accAddress, err := k.Accounts.Indexes.Number.MatchExact(ctx, accNumber)
	if err != nil {
		return nil, authtypes.BaseAccount{}, err
	}
	
	acc, err := k.Accounts.Get(ctx, accAddress)
	return accAddress, acc, nil
}

func (k Keeper) GetAccountsByNumber(ctx sdk.Context, startAccNum, endAccNum uint64) ([]authtypes.BaseAccount, error) {
	rng := new(collections.Range[uint64]).
		StartInclusive(startAccNum).
		EndInclusive(endAccNum)
	
	iter, err := k.Accounts.Indexes.Number.Iterate(ctx, rng)
	if err != nil {
		return nil, err
	}
	
	return indexes.CollectValues(ctx, k.Accounts, iter)
}


func (k Keeper) getNextAccountNumber() uint64 {
	return 0
}
```

## Collections Với Interface Làm Values

Mặc dù cosmos-sdk đang chuyển hướng khỏi việc sử dụng interface registry, vẫn còn một số nơi nó được sử dụng.
Để hỗ trợ code cũ, chúng ta phải hỗ trợ collections với interface values.

`codec.CollValue` generic không thể xử lý interface values, vì vậy chúng ta cần sử dụng kiểu đặc biệt `codec.CollValueInterface`.
`codec.CollValueInterface` nhận `codec.BinaryCodec` làm đối số và sử dụng nó để marshal và unmarshal các value như interface.
`codec.CollValueInterface` nằm trong package `codec`, đường dẫn import là `github.com/cosmos/cosmos-sdk/codec`.

### Khởi Tạo Collections Với Interface Values

Để khởi tạo collection với interface values, chúng ta cần sử dụng `codec.CollValueInterface` thay vì `codec.CollValue`.

```go
package example

import (
    "cosmossdk.io/collections"
    storetypes "cosmossdk.io/store/types"
    "github.com/cosmos/cosmos-sdk/codec"
    sdk "github.com/cosmos/cosmos-sdk/types"
	authtypes "github.com/cosmos/cosmos-sdk/x/auth/types"
)

var AccountsPrefix = collections.NewPrefix(0)

type Keeper struct {
    Schema   collections.Schema
    Accounts *collections.Map[sdk.AccAddress, sdk.AccountI]
}

func NewKeeper(cdc codec.BinaryCodec, storeKey *storetypes.KVStoreKey) Keeper {
    sb := collections.NewSchemaBuilder(sdk.OpenKVStore(storeKey))
    return Keeper{
        Accounts: collections.NewMap(
            sb, AccountsPrefix, "accounts",
            sdk.AccAddressKey, codec.CollInterfaceValue[sdk.AccountI](cdc),
        ),
    }
}

func (k Keeper) SaveBaseAccount(ctx sdk.Context, account authtypes.BaseAccount) error {
    return k.Accounts.Set(ctx, account.GetAddress(), account)
}

func (k Keeper) SaveModuleAccount(ctx sdk.Context, account authtypes.ModuleAccount) error {
    return k.Accounts.Set(ctx, account.GetAddress(), account)
}

func (k Keeper) GetAccount(ctx sdk.context, addr sdk.AccAddress) (sdk.AccountI, error) {
    return k.Accounts.Get(ctx, addr)
}
```

## Triple Key (Khóa Ba)

`collections.Triple` là một kiểu key đặc biệt được tạo thành từ ba key, giống hệt `collections.Pair`.

Hãy xem một ví dụ.

```go
package example

import (
 "context"

 "cosmossdk.io/collections"
 storetypes "cosmossdk.io/store/types"
 "github.com/cosmos/cosmos-sdk/codec"
)

type AccAddress = string
type ValAddress = string

type Keeper struct {
 // giả sử chúng ta có các redelegation được lưu trữ như một triple key bao gồm
 // người ủy thác, validator nguồn và validator đích.
 Redelegations collections.KeySet[collections.Triple[AccAddress, ValAddress, ValAddress]]
}

func NewKeeper(storeKey *storetypes.KVStoreKey) Keeper {
 sb := collections.NewSchemaBuilder(sdk.OpenKVStore(storeKey))
 return Keeper{
  Redelegations: collections.NewKeySet(sb, collections.NewPrefix(0), "redelegations", collections.TripleKeyCodec(collections.StringKey, collections.StringKey, collections.StringKey)
 }
}

// RedelegationsByDelegator duyệt qua tất cả các redelegation của một người ủy thác và gọi onResult cung cấp
// mỗi redelegation từ source validator đến destination validator.
func (k Keeper) RedelegationsByDelegator(ctx context.Context, delegator AccAddress, onResult func(src, dst ValAddress) (stop bool, err error)) error {
 rng := collections.NewPrefixedTripleRange[AccAddress, ValAddress, ValAddress](delegator)
 return k.Redelegations.Walk(ctx, rng, func(key collections.Triple[AccAddress, ValAddress, ValAddress]) (stop bool, err error) {
  return onResult(key.K2(), key.K3())
 })
}

// RedelegationsByDelegatorAndValidator duyệt qua tất cả các redelegation của một người ủy thác và source validator của nó và gọi onResult cho mỗi
// destination validator.
func (k Keeper) RedelegationsByDelegatorAndValidator(ctx context.Context, delegator AccAddress, validator ValAddress, onResult func(dst ValAddress) (stop bool, err error)) error {
 rng := collections.NewSuperPrefixedTripleRange[AccAddress, ValAddress, ValAddress](delegator, validator)
 return k.Redelegations.Walk(ctx, rng, func(key collections.Triple[AccAddress, ValAddress, ValAddress]) (stop bool, err error) {
  return onResult(key.K3())
 })
}
```

## Sử Dụng Nâng Cao

### Alternative Value Codec (Codec Value Thay Thế)

`codec.AltValueCodec` cho phép một collection decode các value bằng cách sử dụng codec khác với codec được sử dụng để encode chúng.
Về cơ bản nó cho phép decode hai biểu diễn bytes khác nhau của cùng một value cụ thể.
Nó có thể được sử dụng để migrate lazily các value từ một biểu diễn bytes sang biểu diễn khác, miễn là biểu diễn mới
không thể decode biểu diễn cũ.

Một ví dụ cụ thể có thể tìm thấy trong `x/bank` nơi số dư ban đầu được lưu trữ là `Coin` và sau đó được migrate sang `Int`.

```go

var BankBalanceValueCodec = codec.NewAltValueCodec(sdk.IntValue, func(b []byte) (sdk.Int, error) {
    coin := sdk.Coin{}
    err := coin.Unmarshal(b)
    if err != nil {
        return sdk.Int{}, err
    }
    return coin.Amount, nil
})
```

Ví dụ trên cho thấy cách tạo `AltValueCodec` có thể decode cả giá trị `sdk.Int` và `sdk.Coin`. Hàm decoder được cung cấp
sẽ được sử dụng làm fallback trong trường hợp decoder mặc định thất bại. Khi value được encode trở lại vào state
nó sẽ sử dụng encoder mặc định. Điều này cho phép migrate lazily các value sang biểu diễn bytes mới.
