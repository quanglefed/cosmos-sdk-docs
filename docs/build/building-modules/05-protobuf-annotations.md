---
sidebar_position: 1
---

# Các Annotation ProtocolBuffer

Tài liệu này giải thích các scalar protobuf khác nhau đã được thêm vào để giúp việc làm việc với protobuf trở nên dễ dàng hơn cho các nhà phát triển ứng dụng Cosmos SDK.

## Signer

Signer chỉ định trường nào nên được sử dụng để xác định signer của một message trong Cosmos SDK. Trường này cũng có thể được sử dụng cho các client để suy ra trường nào nên được dùng để xác định signer của một message.

Đọc thêm về trường signer [tại đây](./02-messages-and-queries.md).

```protobuf reference 
https://github.com/cosmos/cosmos-sdk/blob/e6848d99b55a65d014375b295bdd7f9641aac95e/proto/cosmos/bank/v1beta1/tx.proto#L40
```

```proto
option (cosmos.msg.v1.signer) = "from_address";
```

## Scalar

Kiểu scalar định nghĩa cách cho các client hiểu cách xây dựng các protobuf message theo những gì module và SDK mong đợi.

```proto
(cosmos_proto.scalar) = "cosmos.AddressString"
```

Ví dụ về scalar địa chỉ tài khoản dạng string:

```proto reference 
https://github.com/cosmos/cosmos-sdk/blob/e6848d99b55a65d014375b295bdd7f9641aac95e/proto/cosmos/bank/v1beta1/tx.proto#L46
```

Ví dụ về scalar địa chỉ validator dạng string:

```proto reference 
https://github.com/cosmos/cosmos-sdk/blob/e8f28bf5db18b8d6b7e0d94b542ce4cf48fed9d6/proto/cosmos/distribution/v1beta1/query.proto#L87
```

Ví dụ về scalar Decimals:

```proto reference
https://github.com/cosmos/cosmos-sdk/blob/e8f28bf5db18b8d6b7e0d94b542ce4cf48fed9d6/proto/cosmos/distribution/v1beta1/distribution.proto#L26
```

Ví dụ về scalar Int:

```proto reference
https://github.com/cosmos/cosmos-sdk/blob/e8f28bf5db18b8d6b7e0d94b542ce4cf48fed9d6/proto/cosmos/gov/v1/gov.proto#L137
```

Có một vài tùy chọn có thể cung cấp làm scalar: `cosmos.AddressString`, `cosmos.ValidatorAddressString`, `cosmos.ConsensusAddressString`, `cosmos.Int`, `cosmos.Dec`.

## Implements_Interface

`Implements_Interface` được sử dụng để cung cấp thông tin cho công cụ client như [telescope](https://github.com/cosmology-tech/telescope) về cách mã hóa và giải mã các protobuf message.

```proto
option (cosmos_proto.implements_interface) = "cosmos.auth.v1beta1.AccountI";
```

## Method, Field, Message Added In (Được Thêm Vào Trong Phiên Bản)

`method_added_in`, `field_added_in` và `message_added_in` là các annotation để thông báo cho client rằng một trường đã được hỗ trợ trong phiên bản mới hơn. Điều này hữu ích khi các phương thức hoặc trường mới được thêm vào trong các phiên bản sau và client cần biết những gì có thể gọi.

Annotation nên được viết như sau:

```proto
option (cosmos_proto.method_added_in) = "cosmos-sdk v0.50.1";
option (cosmos_proto.method_added_in) = "x/epochs v1.0.0";
option (cosmos_proto.method_added_in) = "simapp v24.0.0";
```

## Amino

Amino codec đã bị xóa trong `v0.50+`, điều này có nghĩa là không cần đăng ký `legacyAminoCodec` nữa. Để thay thế amino codec, các annotation protobuf Amino được sử dụng để cung cấp thông tin cho amino codec về cách mã hóa và giải mã các protobuf message.

Các annotation Amino chỉ được sử dụng cho khả năng tương thích ngược với amino. Các module mới không bắt buộc phải sử dụng annotation amino.

Các annotation dưới đây được sử dụng để cung cấp thông tin cho amino codec về cách mã hóa và giải mã các protobuf message theo cách tương thích ngược.

### Name

Name chỉ định tên amino sẽ hiển thị cho người dùng để họ thấy message nào họ đang ký.

```proto
option (amino.name) = "cosmos-sdk/BaseAccount";
```

```proto reference
https://github.com/cosmos/cosmos-sdk/blob/e8f28bf5db18b8d6b7e0d94b542ce4cf48fed9d6/proto/cosmos/bank/v1beta1/tx.proto#L41
```

### Field_Name

Field name chỉ định tên amino sẽ hiển thị cho người dùng để họ thấy trường nào họ đang ký.

```proto
uint64 height = 1 [(amino.field_name) = "public_key"];
```

```proto reference
https://github.com/cosmos/cosmos-sdk/blob/e8f28bf5db18b8d6b7e0d94b542ce4cf48fed9d6/proto/cosmos/distribution/v1beta1/distribution.proto#L166
```

### Dont_OmitEmpty

Dont omitempty chỉ định rằng trường không nên bị bỏ qua khi mã hóa sang amino.

```proto
repeated cosmos.base.v1beta1.Coin amount = 3 [(amino.dont_omitempty)   = true];
```

```proto reference
https://github.com/cosmos/cosmos-sdk/blob/e8f28bf5db18b8d6b7e0d94b542ce4cf48fed9d6/proto/cosmos/bank/v1beta1/bank.proto#L56
```

### Encoding

Encoding hướng dẫn amino json marshaler cách mã hóa một số trường nhất định có thể khác với hành vi mã hóa tiêu chuẩn. Ví dụ phổ biến nhất là cách `repeated cosmos.base.v1beta1.Coin` được mã hóa khi sử dụng định dạng mã hóa amino json. Tùy chọn `legacy_coins` cho json marshaler biết [cách mã hóa một slice null](https://github.com/cosmos/cosmos-sdk/blob/e8f28bf5db18b8d6b7e0d94b542ce4cf48fed9d6/x/tx/signing/aminojson/json_marshal.go#L65) của `cosmos.base.v1beta1.Coin`.

```proto
(amino.encoding)         = "legacy_coins",
```

```proto reference
https://github.com/cosmos/cosmos-sdk/blob/e8f28bf5db18b8d6b7e0d94b542ce4cf48fed9d6/proto/cosmos/bank/v1beta1/genesis.proto#L23
```
