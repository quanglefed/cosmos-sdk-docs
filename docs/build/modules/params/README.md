---
sidebar_position: 1
---

# `x/params`

LƯU Ý: `x/params` đã bị deprecate từ Cosmos SDK v0.53 và sẽ bị loại bỏ trong bản phát hành tiếp theo.

## Tóm tắt

Gói params cung cấp một kho tham số (parameter store) dùng chung toàn cục.

Có hai kiểu chính: Keeper và Subspace. Subspace là một namespace tách biệt cho
paramstore, trong đó các key được prefix bởi spacename cấu hình sẵn. Keeper có
quyền truy cập tất cả các space hiện có.

Subspace có thể được dùng bởi các keeper riêng lẻ (của từng module), khi cần một
kho tham số riêng tư mà các keeper khác không thể chỉnh sửa. Params Keeper có thể
được dùng để thêm route vào router của `x/gov` nhằm sửa bất kỳ tham số nào nếu một
proposal được thông qua.

Phần nội dung sau giải thích cách dùng module params cho master module và user module.

## Nội dung

* [Keeper](#keeper)
* [Subspace](#subspace)
  * [Key](#key)
  * [KeyTable](#keytable)
  * [ParamSet](#paramset)

## Keeper

Trong giai đoạn khởi tạo ứng dụng, các [subspace](#subspace) có thể được cấp phát
cho keeper của các module khác bằng `Keeper.Subspace` và được lưu trong `Keeper.spaces`.
Sau đó, các module đó có thể tham chiếu tới kho tham số cụ thể của mình thông qua
`Keeper.GetSubspace`.

Ví dụ:

```go
type ExampleKeeper struct {
	paramSpace paramtypes.Subspace
}

func (k ExampleKeeper) SetParams(ctx sdk.Context, params types.Params) {
	k.paramSpace.SetParamSet(ctx, &params)
}
```

## Subspace

`Subspace` là một subspace có prefix của parameter store. Mỗi module dùng parameter
store sẽ nhận một `Subspace` để cô lập quyền truy cập.

### Key

Key tham số là các chuỗi chữ-số dễ đọc (alphanumeric). Một tham số có key
`"ExampleParameter"` được lưu dưới dạng `[]byte("SubspaceName" + "/" + "ExampleParameter")`,
trong đó `"SubspaceName"` là tên subspace.

Subkey là key tham số phụ, được dùng cùng với key tham số chính (primary key).
Subkey có thể dùng để nhóm hoặc tạo key tham số động trong lúc chạy (runtime).

### KeyTable

Tất cả key tham số sẽ được dùng nên được đăng ký ở compile-time. `KeyTable` về
bản chất là một `map[string]attribute`, trong đó `string` là key tham số.

Hiện tại, `attribute` gồm một `reflect.Type` (để kiểm tra kiểu tham số tương
thích với key và value được cung cấp và đã đăng ký), cùng một hàm `ValueValidatorFn`
để validate giá trị.

Chỉ primary key cần được đăng ký trong `KeyTable`. Subkey thừa hưởng attribute
của primary key.

### ParamSet

Các module thường định nghĩa tham số như một proto message. Struct được generate
có thể hiện thực interface `ParamSet` để dùng với các phương thức sau:

* `KeyTable.RegisterParamSet()`: đăng ký tất cả tham số trong struct
* `Subspace.{Get, Set}ParamSet()`: lấy/đặt tham số từ struct

Implementer nên là con trỏ (pointer) để có thể dùng `GetParamSet()`.

