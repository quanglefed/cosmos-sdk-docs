---
sidebar_position: 1
---

# Depinject

> **TUYÊN BỐ TỪ CHỐI TRÁCH NHIỆM**: Đây là gói **beta**. Nhóm SDK đang tích cực làm việc trên tính năng này và chúng tôi đang tìm kiếm phản hồi từ cộng đồng. Vui lòng thử và cho chúng tôi biết suy nghĩ của bạn.

## Tổng Quan

`depinject` là một framework dependency injection (DI) cho Cosmos SDK, được thiết kế để đơn giản hóa quá trình xây dựng và cấu hình các ứng dụng blockchain. Nó hoạt động cùng với module `core/appconfig` để thay thế phần lớn boilerplate code trong `app.go` bằng file cấu hình theo định dạng Go, YAML hoặc JSON.

`depinject` đặc biệt hữu ích để phát triển các ứng dụng blockchain:

* Với nhiều thành phần, module hoặc service phụ thuộc lẫn nhau. Giúp quản lý các phụ thuộc của chúng một cách hiệu quả.
* Yêu cầu tách rời các thành phần này, giúp dễ dàng test, sửa đổi hoặc thay thế các phần riêng lẻ mà không ảnh hưởng đến toàn bộ hệ thống.
* Muốn đơn giản hóa việc thiết lập và khởi tạo các module và các phụ thuộc bằng cách giảm boilerplate code và tự động hóa quản lý phụ thuộc.

Bằng cách sử dụng `depinject`, nhà phát triển có thể đạt được:

* Code sạch hơn và có tổ chức hơn.
* Cải thiện tính module hóa và khả năng bảo trì.
* Cấu trúc có thể bảo trì và module hóa hơn cho các ứng dụng blockchain của họ, cuối cùng nâng cao tốc độ phát triển và chất lượng code.

* [Go Doc](https://pkg.go.dev/cosmossdk.io/depinject)

## Sử Dụng

Framework `depinject`, dựa trên các khái niệm dependency injection, đơn giản hóa việc quản lý các phụ thuộc trong ứng dụng blockchain của bạn bằng cách sử dụng Configuration API. API này cung cấp một bộ hàm và phương thức để tạo các cấu hình dễ sử dụng, giúp đơn giản hóa việc định nghĩa, sửa đổi và truy cập các phụ thuộc cũng như các mối quan hệ của chúng.

Một thành phần cốt lõi của [Configuration API](https://pkg.go.dev/github.com/cosmos/cosmos-sdk/depinject#Config) là hàm `Provide`, cho phép bạn đăng ký các hàm provider cung cấp các phụ thuộc. Lấy cảm hứng từ constructor injection, các hàm provider này tạo thành nền tảng của dependency tree, cho phép quản lý và giải quyết các phụ thuộc theo cách có cấu trúc và dễ bảo trì. Ngoài ra, `depinject` hỗ trợ các kiểu interface làm đầu vào cho các hàm provider, cung cấp sự linh hoạt và tách rời giữa các thành phần, tương tự như các khái niệm interface injection.

Bằng cách tận dụng `depinject` và Configuration API của nó, bạn có thể xử lý hiệu quả các phụ thuộc trong ứng dụng blockchain của mình, đảm bảo một codebase sạch, module hóa và có tổ chức tốt.

Ví dụ:

```go
package main

import (
 "fmt"

 "cosmossdk.io/depinject"
)

type AnotherInt int

func GetInt() int               { return 1 }
func GetAnotherInt() AnotherInt { return 2 }

func main() {
 var (
  x int
  y AnotherInt
 )

 fmt.Printf("Before (%v, %v)\n", x, y)
 depinject.Inject(
  depinject.Provide(
   GetInt,
   GetAnotherInt,
  ),
  &x,
  &y,
 )
 fmt.Printf("After (%v, %v)\n", x, y)
}
```

Trong ví dụ này, `depinject.Provide` đăng ký hai hàm provider trả về các giá trị `int` và `AnotherInt`. Hàm `depinject.Inject` sau đó được sử dụng để inject các giá trị này vào các biến `x` và `y`.

Các hàm provider đóng vai trò là nền tảng cho dependency tree. Chúng được phân tích để xác định các đầu vào của chúng là các phụ thuộc và các đầu ra là các dependent. Các dependent này có thể được sử dụng bởi một hàm provider khác hoặc được lưu trữ bên ngoài container DI (ví dụ: `&x` và `&y` trong ví dụ trên). Các hàm provider phải được export.

### Giải Quyết Kiểu Interface

`depinject` hỗ trợ sử dụng kiểu interface làm đầu vào cho các hàm provider, giúp tách rời các phụ thuộc giữa các module. Cách tiếp cận này đặc biệt hữu ích để quản lý các hệ thống phức tạp với nhiều module, như Cosmos SDK, nơi các phụ thuộc cần linh hoạt và dễ bảo trì.

Ví dụ, `x/bank` mong đợi một interface [AccountKeeper](https://pkg.go.dev/github.com/cosmos/cosmos-sdk/x/bank/types#AccountKeeper) làm [đầu vào cho ProvideModule](https://github.com/cosmos/cosmos-sdk/tree/release/v0.50.x/x/bank/module.go#L208-L260). `SimApp` sử dụng triển khai trong `x/auth`, nhưng thiết kế module hóa cho phép dễ dàng thay đổi triển khai nếu cần.

Xét ví dụ sau:

```go
package duck

type Duck interface {
 quack()
}

type AlsoDuck interface {
 quack()
}

type Mallard struct{}
type Canvasback struct{}

func (duck Mallard) quack()    {}
func (duck Canvasback) quack() {}

type Pond struct {
 Duck AlsoDuck
}
```

Và các hàm provider sau:

```go
func GetMallard() duck.Mallard {
 return Mallard{}
}

func GetPond(duck Duck) Pond {
 return Pond{Duck: duck}
}

func GetCanvasback() Canvasback {
 return Canvasback{}
}
```

Trong ví dụ này, có một struct `Pond` có trường `Duck` kiểu `AlsoDuck`. Framework `depinject` có thể tự động giải quyết triển khai phù hợp khi chỉ có một triển khai có sẵn, như được hiển thị bên dưới:

```go
var pond Pond

depinject.Inject(
  depinject.Provide(
   GetMallard,
   GetPond,
  ),
   &pond)
```

Đoạn code này dẫn đến trường `Duck` của `Pond` được liên kết ngầm với triển khai `Mallard` vì đây là triển khai duy nhất của interface `Duck` trong container.

Tuy nhiên, nếu có nhiều triển khai của interface `Duck`, như trong ví dụ sau, bạn sẽ gặp lỗi:

```go
var pond Pond

depinject.Inject(
 depinject.Provide(
  GetMallard,
  GetCanvasback,
  GetPond,
 ),
 &pond)
```

Cần có tùy chọn binding cụ thể cho `Duck`.

#### API `BindInterface`

Trong tình huống trên, đăng ký một binding cho một interface binding nhất định có thể trông như sau:

```go
depinject.Inject(
 depinject.Configs(
  depinject.BindInterface(
   "duck/duck.Duck",
   "duck/duck.Mallard",
  ),
  depinject.Provide(
   GetMallard,
   GetCanvasback,
   GetPond,
  ),
 ),
 &pond)
```

Bây giờ `depinject` có đủ thông tin để cung cấp `Mallard` như đầu vào cho `APond`.

### Ví Dụ Đầy Đủ Trong Ứng Dụng Thực Tế

:::warning
Khi sử dụng `depinject.Inject`, các kiểu được inject phải là con trỏ.
:::

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/simapp/app_di.go#L165-L188
```

## Debugging (Gỡ Lỗi)

Các vấn đề trong việc giải quyết các phụ thuộc trong container có thể được xử lý bằng log và các rendering [Graphviz](https://graphviz.org) của container tree.
Theo mặc định, bất cứ khi nào có lỗi, log sẽ được in ra stderr và một rendering của dependency graph theo định dạng Graphviz DOT sẽ được lưu vào `debug_container.dot`.

Đây là ví dụ về rendering Graphviz của một dependency graph được build thành công:
![Graphviz Example](https://raw.githubusercontent.com/cosmos/cosmos-sdk/ff39d243d421442b400befcd959ec3ccd2525154/depinject/testdata/example.svg)

Hình chữ nhật đại diện cho các hàm, hình oval đại diện cho các kiểu, hình chữ nhật bo góc đại diện cho các module và hình lục giác đơn
đại diện cho hàm đã gọi `Build`. Các hình màu đen đánh dấu các hàm và kiểu đã được gọi/giải quyết
mà không có lỗi. Các node màu xám đánh dấu các hàm và kiểu có thể đã được gọi/giải quyết trong container nhưng
không được sử dụng.

Đây là ví dụ về rendering Graphviz của một dependency graph build thất bại:
![Graphviz Error Example](https://raw.githubusercontent.com/cosmos/cosmos-sdk/ff39d243d421442b400befcd959ec3ccd2525154/depinject/testdata/example_error.svg)

Các file Graphviz DOT có thể được chuyển đổi thành SVG để xem trong trình duyệt web bằng công cụ dòng lệnh `dot`, ví dụ:

```txt
dot -Tsvg debug_container.dot > debug_container.svg
```

Nhiều công cụ khác bao gồm một số IDE hỗ trợ làm việc với các file DOT.
