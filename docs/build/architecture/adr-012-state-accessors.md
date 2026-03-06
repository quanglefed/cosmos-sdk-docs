# ADR 012: Bộ Truy Cập Trạng Thái

## Changelog

* 04 tháng 9 năm 2019: Bản nháp đầu tiên

## Bối Cảnh

Các module Cosmos SDK hiện sử dụng interface `KVStore` và `Codec` để truy cập trạng thái tương ứng của chúng. Mặc dù điều này cung cấp mức độ tự do lớn cho nhà phát triển module, nhưng khó mô-đun hóa và UX còn kém.

Thứ nhất, mỗi lần một module cố gắng truy cập trạng thái, nó phải marshal giá trị và set hoặc get giá trị, rồi cuối cùng unmarshal. Thông thường điều này được thực hiện bằng cách khai báo các hàm `Keeper.GetXXX` và `Keeper.SetXXX`, vốn lặp đi lặp lại và khó bảo trì.

Thứ hai, điều này làm khó khăn hơn để căn chỉnh với định lý object capability: quyền truy cập trạng thái được định nghĩa là `StoreKey`, cung cấp toàn quyền truy cập vào toàn bộ cây Merkle, vì vậy một module không thể gửi quyền truy cập đến một cặp key-value cụ thể (hoặc tập hợp các cặp key-value) cho module khác một cách an toàn.

Cuối cùng, vì các hàm getter/setter được định nghĩa là phương thức của `Keeper` của module, những người review phải xem xét toàn bộ không gian cây Merkle khi họ review một hàm truy cập bất kỳ phần nào của trạng thái. Không có cách tĩnh nào để biết phần nào của trạng thái mà hàm đang truy cập (và phần nào không).

## Quyết Định

Chúng ta sẽ định nghĩa một kiểu tên là `Value`:

```go
type Value struct {
  m   Mapping
  key []byte
}
```

`Value` hoạt động như một tham chiếu cho một cặp key-value trong trạng thái, trong đó `Value.m` định nghĩa không gian key-value nó sẽ truy cập và `Value.key` định nghĩa khóa chính xác cho tham chiếu.

Chúng ta sẽ định nghĩa một kiểu tên là `Mapping`:

```go
type Mapping struct {
  storeKey sdk.StoreKey
  cdc      *codec.LegacyAmino
  prefix   []byte
}
```

`Mapping` hoạt động như một tham chiếu cho một không gian key-value trong trạng thái, trong đó `Mapping.storeKey` định nghĩa cây IAVL (con) và `Mapping.prefix` định nghĩa tiền tố subspace tùy chọn.

Chúng ta sẽ định nghĩa các phương thức core sau cho kiểu `Value`:

```go
// Get và unmarshal dữ liệu được lưu trữ, noop nếu không tồn tại, panic nếu không thể unmarshal
func (Value) Get(ctx Context, ptr interface{}) {}

// Get và unmarshal dữ liệu được lưu trữ, trả về lỗi nếu không tồn tại hoặc không thể unmarshal
func (Value) GetSafe(ctx Context, ptr interface{}) {}

// Get dữ liệu được lưu trữ dưới dạng slice byte thô
func (Value) GetRaw(ctx Context) []byte {}

// Marshal và set một giá trị thô
func (Value) Set(ctx Context, o interface{}) {}

// Kiểm tra xem giá trị thô có tồn tại không
func (Value) Exists(ctx Context) bool {}

// Xóa một giá trị thô
func (Value) Delete(ctx Context) {}
```

Chúng ta sẽ định nghĩa các phương thức core sau cho kiểu `Mapping`:

```go
// Xây dựng tham chiếu cặp key-value tương ứng với đối số key trong không gian Mapping
func (Mapping) Value(key []byte) Value {}

// Get và unmarshal dữ liệu được lưu trữ, noop nếu không tồn tại, panic nếu không thể unmarshal
func (Mapping) Get(ctx Context, key []byte, ptr interface{}) {}

// Get và unmarshal dữ liệu được lưu trữ, trả về lỗi nếu không tồn tại hoặc không thể unmarshal
func (Mapping) GetSafe(ctx Context, key []byte, ptr interface{})

// Get dữ liệu được lưu trữ dưới dạng slice byte thô
func (Mapping) GetRaw(ctx Context, key []byte) []byte {}

// Marshal và set một giá trị thô
func (Mapping) Set(ctx Context, key []byte, o interface{}) {}

// Kiểm tra xem giá trị thô có tồn tại không
func (Mapping) Has(ctx Context, key []byte) bool {}

// Xóa một giá trị thô
func (Mapping) Delete(ctx Context, key []byte) {}
```

Mỗi phương thức của kiểu `Mapping` được truyền các đối số `ctx`, `key` và `args...` sẽ proxy lời gọi đến `Mapping.Value(key)` với các đối số `ctx` và `args...`.

Ngoài ra, chúng ta sẽ định nghĩa và cung cấp một tập hợp chung các kiểu bắt nguồn từ kiểu `Value`:

```go
type Boolean struct { Value }
type Enum struct { Value }
type Integer struct { Value; enc IntEncoding }
type String struct { Value }
// ...
```

Trong đó các phương thức mã hóa có thể khác nhau, đối số `o` trong các phương thức core được gõ, và đối số `ptr` trong các phương thức core được thay thế bằng các kiểu trả về rõ ràng.

Cuối cùng, chúng ta sẽ định nghĩa một họ kiểu bắt nguồn từ kiểu `Mapping`:

```go
type Indexer struct {
  m   Mapping
  enc IntEncoding
}
```

Trong đó đối số `key` trong phương thức core được gõ.

Một số thuộc tính của các kiểu accessor là:

* Truy cập trạng thái chỉ xảy ra khi một hàm nhận `Context` làm đối số được gọi
* Các struct kiểu accessor cung cấp quyền truy cập trạng thái chỉ là những gì struct đang tham chiếu, không có gì khác
* Marshalling/Unmarshalling xảy ra ngầm định trong các phương thức core

## Trạng Thái

Đề Xuất

## Hậu Quả

### Tích Cực

* Tuần tự hóa sẽ được thực hiện tự động
* Kích thước code ngắn hơn, ít boilerplate hơn, UX tốt hơn
* Tham chiếu đến trạng thái có thể được chuyển giao an toàn
* Phạm vi truy cập rõ ràng

### Tiêu Cực

* Định dạng tuần tự hóa sẽ bị ẩn
* Kiến trúc khác với hiện tại, nhưng việc sử dụng các kiểu accessor có thể là tùy chọn
* Các kiểu cụ thể theo kiểu dữ liệu (ví dụ: `Boolean` và `Integer`) phải được định nghĩa thủ công

### Trung Lập

## Tài Liệu Tham Khảo

* [#4554](https://github.com/cosmos/cosmos-sdk/issues/4554)
