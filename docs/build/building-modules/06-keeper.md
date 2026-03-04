---
sidebar_position: 1
---

# Keepers

:::note Tóm tắt
`Keeper` đề cập đến một abstraction của Cosmos SDK có vai trò quản lý quyền truy cập vào tập con trạng thái được định nghĩa bởi các module khác nhau. `Keeper` đặc thù cho từng module, tức là tập con trạng thái được định nghĩa bởi một module chỉ có thể được truy cập bởi `keeper` được định nghĩa trong module đó. Nếu một module cần truy cập tập con trạng thái được định nghĩa bởi module khác, cần truyền tham chiếu đến `keeper` nội bộ của module thứ hai cho module thứ nhất. Điều này được thực hiện trong `app.go` trong quá trình khởi tạo module keepers.
:::

:::note Yêu Cầu Đọc Trước

* [Giới thiệu về Cosmos SDK Modules](./00-intro.md)

:::

## Động Lực

Cosmos SDK là một framework giúp các nhà phát triển dễ dàng xây dựng các ứng dụng phi tập trung phức tạp từ đầu, chủ yếu bằng cách kết hợp các module lại với nhau. Khi hệ sinh thái các module mã nguồn mở cho Cosmos SDK mở rộng, ngày càng có khả năng cao hơn là một số module này chứa các lỗ hổng, do sự cẩu thả hoặc ác ý của nhà phát triển.

Cosmos SDK áp dụng [phương pháp dựa trên object-capabilities](../../docs/learn/advanced/10-ocap.md) để giúp nhà phát triển bảo vệ ứng dụng của họ khỏi các tương tác giữa module không mong muốn, và `keeper` là trọng tâm của phương pháp này. Một `keeper` có thể được coi theo nghĩa đen là người gác cổng của các store trong module. Mỗi store (thường là [`IAVL` Store](../../learn/advanced/04-store.md#iavl-store)) được định nghĩa trong một module đi kèm với một `storeKey`, cấp quyền truy cập không giới hạn vào nó. `Keeper` của module giữ `storeKey` này (vốn phải được giữ không tiết lộ), và định nghĩa [các phương thức](#implementing-methods) để đọc và ghi vào store.

Ý tưởng cốt lõi đằng sau phương pháp object-capabilities là chỉ tiết lộ những gì cần thiết để hoàn thành công việc. Trong thực tế, điều này có nghĩa là thay vì xử lý quyền của các module thông qua danh sách kiểm soát truy cập, các `keeper` của module được truyền tham chiếu đến phiên bản cụ thể của `keeper` của module khác mà chúng cần truy cập (điều này được thực hiện trong [hàm constructor của ứng dụng](../../learn/beginner/00-app-anatomy.md#constructor-function)). Kết quả là, một module chỉ có thể tương tác với tập con trạng thái được định nghĩa trong module khác thông qua các phương thức được hiển thị bởi phiên bản `keeper` của module khác đó. Đây là cách tuyệt vời để nhà phát triển kiểm soát các tương tác mà module của họ có thể có với các module do nhà phát triển bên ngoài phát triển.

## Định Nghĩa Kiểu

`Keeper` thường được triển khai trong file `/keeper/keeper.go` nằm trong thư mục của module. Theo quy ước, kiểu `keeper` của một module được đặt tên đơn giản là `Keeper` và thường có cấu trúc sau:

```go
type Keeper struct {
    // External keepers, nếu có

    // Store key(s)

    // codec

    // authority 
}
```

Ví dụ, đây là định nghĩa kiểu của `keeper` từ module `staking`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/staking/keeper/keeper.go#L23-L31
```

Hãy xem qua các tham số khác nhau:

* Một `keeper` mong đợi (expected keeper) là một `keeper` bên ngoài module được yêu cầu bởi `keeper` nội bộ của module đó. Các `keeper` bên ngoài được liệt kê trong định nghĩa kiểu của `keeper` nội bộ dưới dạng các interface. Các interface này được định nghĩa trong file `expected_keepers.go` ở thư mục gốc của module. Trong ngữ cảnh này, các interface được sử dụng để giảm số lượng phụ thuộc, cũng như tạo điều kiện bảo trì module.
* `storeKey` cấp quyền truy cập vào các store của [multistore](../../learn/advanced/04-store.md) được quản lý bởi module. Chúng phải luôn được giữ không tiết lộ cho các module bên ngoài.
* `cdc` là [codec](../../learn/advanced/05-encoding.md) được sử dụng để marshal và unmarshal các struct sang/từ `[]byte`. `cdc` có thể là bất kỳ `codec.BinaryCodec`, `codec.JSONCodec` hoặc `codec.Codec` nào dựa trên yêu cầu của bạn. Nó có thể là codec proto hoặc amino miễn là chúng triển khai các interface này.
* Authority được liệt kê là một tài khoản module hoặc tài khoản người dùng có quyền thay đổi các tham số ở cấp module. Trước đây điều này được xử lý bởi module param, vốn đã bị loại bỏ.

Tất nhiên, có thể định nghĩa các loại `keeper` nội bộ khác nhau cho cùng một module (ví dụ: một `keeper` chỉ đọc). Mỗi loại `keeper` có hàm constructor riêng, được gọi từ [hàm constructor của ứng dụng](../../learn/beginner/00-app-anatomy.md). Đây là nơi các `keeper` được khởi tạo, và nơi nhà phát triển đảm bảo truyền đúng phiên bản các `keeper` của modules cho các module khác yêu cầu chúng.

## Triển Khai Các Phương Thức

`Keeper` chủ yếu hiển thị các phương thức getter và setter cho các store được quản lý bởi module của chúng. Các phương thức này phải đơn giản nhất có thể và chỉ giới hạn trong việc lấy hoặc đặt giá trị được yêu cầu, vì các kiểm tra hợp lệ đã được thực hiện bởi [`Msg` server](./03-msg-services.md) khi các phương thức của `keeper` được gọi.

Thông thường, một phương thức *getter* sẽ có chữ ký sau:

```go
func (k Keeper) Get(ctx context.Context, key string) returnType
```

và phương thức sẽ thực hiện các bước sau:

1. Lấy store thích hợp từ `ctx` bằng cách sử dụng `storeKey`. Điều này được thực hiện thông qua phương thức `KVStore(storeKey sdk.StoreKey)` của `ctx`. Sau đó ưu tiên sử dụng `prefix.Store` để chỉ truy cập tập con giới hạn mong muốn của store cho tiện lợi và an toàn.
2. Nếu tồn tại, lấy giá trị `[]byte` được lưu trữ tại vị trí `[]byte(key)` bằng cách sử dụng phương thức `Get(key []byte)` của store.
3. Unmarshal giá trị đã lấy từ `[]byte` sang `returnType` bằng cách sử dụng codec `cdc`. Trả về giá trị.

Tương tự, một phương thức *setter* sẽ có chữ ký sau:

```go
func (k Keeper) Set(ctx context.Context, key string, value valueType)
```

và phương thức sẽ thực hiện các bước sau:

1. Lấy store thích hợp từ `ctx` bằng cách sử dụng `storeKey`. Ưu tiên sử dụng `prefix.Store` để chỉ truy cập tập con giới hạn mong muốn của store cho tiện lợi và an toàn.
2. Marshal `value` sang `[]byte` bằng cách sử dụng codec `cdc`.
3. Đặt giá trị được mã hóa vào store tại vị trí `key` bằng cách sử dụng phương thức `Set(key []byte, value []byte)` của store.

Để biết thêm, xem ví dụ về [triển khai các phương thức của `keeper` từ module `staking`](https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/staking/keeper/keeper.go).

[`KVStore` của module](../../learn/advanced/04-store.md#kvstore-and-commitkvstore-interfaces) cũng cung cấp phương thức `Iterator()` trả về một đối tượng `Iterator` để duyệt qua một miền khóa.

Đây là ví dụ từ module `auth` để duyệt qua các tài khoản:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/auth/keeper/account.go
```
