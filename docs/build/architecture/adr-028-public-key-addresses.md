# ADR 028: Địa Chỉ Khóa Công Khai

## Nhật Ký Thay Đổi

* 28/10/2020: Bản nháp đầu tiên

## Trạng Thái

Đề Xuất

## Tóm Tắt

ADR này định nghĩa một hàm tạo địa chỉ mới cho các tài khoản dựa trên khóa công khai mới và được định nghĩa trong module auth, cung cấp khả năng tương thích ngược cao.

## Bối Cảnh

Vấn đề: Địa chỉ Cosmos hiện tại sử dụng hàm băm `sha256(pubkey)[:20]`, mà cách này không đủ phân biệt với các loại khóa khác nhau. Hơn nữa, đây là giới hạn địa chỉ 20 byte, có thể không đủ cho bảo mật lâu dài. Kiểu tài khoản Cosmos mới sẽ được sử dụng trong tương lai, và chúng cũng sẽ cần địa chỉ.

Thay đổi giao thức liên quan đến địa chỉ là nhạy cảm, vì nó cần thay đổi cách địa chỉ được tạo ra và có thể yêu cầu các bản cập nhật khó khăn cho người dùng, ứng dụng và trình khám phá hiện có.

## Quyết Định

Chúng ta định nghĩa hàm tạo địa chỉ như sau:

```
address = H(type_prefix + H(pub_key) + account_id)[:20]
```

Trong đó:
- `H` là SHA256
- `type_prefix` là chuỗi byte phân biệt loại địa chỉ
- `pub_key` là byte khóa công khai
- `account_id` là số 0-indexed xác định phiên bản của tài khoản

Đối với khóa thông thường (các khóa đơn):

```
address = H("cosmos_account" + sha256(pub_key))[:20]
```

Đối với khóa module:

```
address = H("cosmos_module" + module_name)[:20]
```

Đối với địa chỉ nhóm:

```
address = H("cosmos_group" + members_addresses)[:20]
```

### Định Nghĩa Hàm Tạo Địa Chỉ

Hàm `AddressHash` được định nghĩa như sau:

```go
func AddressHash(typ string, key []byte) []byte {
    hasher := sha256.New()
    _, err := hasher.Write([]byte(typ))
    // ...
    th := hasher.Sum(nil)

    hasher.Reset()
    _, err = hasher.Write(th)
    // ...
    _, err = hasher.Write(key)
    // ...
    return hasher.Sum(nil)[:20]
}
```

### Ví Dụ Địa Chỉ

Khóa đơn secp256k1 (tương thích ngược):
```
address = Hash(pubkey) = sha256(pubkey)[:20]
```

Khóa đơn ed25519:
```
address = AddressHash("cosmos_addr_ed25519_pubkey", pubkey)
```

Địa chỉ multisig:
```
address = AddressHash("cosmos_addr_multisig_legacy", amino(pubkeys, threshold))
```

### Địa Chỉ Module

Một module có địa chỉ được tạo từ tên module của nó. Điều này để đảm bảo sự tách biệt giữa các tài khoản người dùng và tài khoản module.

```go
// Module account address = Hash("cosmos_module" | module_name)
func ModuleAddress(moduleName string) sdk.AccAddress {
    return AddressHash("cosmos_module", []byte(moduleName))
}
```

### Địa Chỉ Submodule

Các tài khoản submodule cung cấp cấu trúc phân cấp:

```go
func SubmoduleAddress(moduleName, key string) sdk.AccAddress {
    return AddressHash(moduleName, []byte(key))
}
```

### Kế Hoạch Chuyển Đổi

Có thể xem xét hai chiến lược chuyển đổi:

1. **Chuyển đổi toàn bộ**: Tất cả địa chỉ được tái tạo với hàm tạo mới. Điều này yêu cầu di chuyển trạng thái và là gián đoạn.

2. **Tương thích ngược**: Địa chỉ hiện tại được giữ nguyên (secp256k1 20 byte tiếp tục sử dụng sha256[:20]). Chỉ các loại khóa mới sử dụng sơ đồ tạo địa chỉ mới.

Chiến lược tương thích ngược được ưa thích vì nó ít gián đoạn hơn.

## Hậu Quả

### Tích Cực

* Cung cấp không gian địa chỉ phân biệt cho các loại khóa khác nhau
* Tương thích ngược với địa chỉ hiện tại cho khóa secp256k1
* Nền tảng cho địa chỉ tài khoản module được tiêu chuẩn hóa

### Tiêu Cực

* Tăng độ phức tạp của logic tạo địa chỉ
* Người dùng mới với các loại khóa khác nhau sẽ có định dạng địa chỉ khác

### Trung Lập

* Cần cập nhật tài liệu và công cụ

## Tham Khảo

* Đề xuất ban đầu: [#7875](https://github.com/cosmos/cosmos-sdk/issues/7875)
