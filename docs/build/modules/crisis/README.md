---
sidebar_position: 1
---

# `x/crisis`

LƯU Ý: `x/crisis` đã bị deprecate từ Cosmos SDK v0.53 và sẽ bị loại bỏ trong bản phát hành tiếp theo.

## Tổng quan

Module crisis sẽ dừng (halt) blockchain trong trường hợp một invariant của
blockchain bị phá vỡ. Các invariant có thể được đăng ký với ứng dụng trong quá
trình khởi tạo ứng dụng.

## Nội dung

* [State](#state)
* [Messages](#messages)
* [Events](#events)
* [Parameters](#parameters)
* [Client](#client)
  * [CLI](#cli)

## State

### ConstantFee

Do dự đoán việc xác minh một invariant tốn gas rất lớn (và có khả năng vượt quá
giới hạn gas tối đa của block), một mức phí cố định (constant fee) được dùng thay
vì cơ chế tiêu thụ gas tiêu chuẩn. Constant fee được thiết kế để lớn hơn mức gas
chi phí dự kiến khi chạy invariant theo cơ chế tiêu thụ gas tiêu chuẩn.

Tham số ConstantFee được lưu trong state params của module với prefix `0x01`,
có thể được cập nhật bằng governance hoặc địa chỉ authority.

* Params: `mint/params -> legacy_amino(sdk.Coin)`

## Messages

Phần này mô tả xử lý các message của crisis và các cập nhật state tương ứng.

### MsgVerifyInvariant

Các invariant của blockchain có thể được kiểm tra bằng message `MsgVerifyInvariant`.

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.47.0-rc1/proto/cosmos/crisis/v1beta1/tx.proto#L26-L42
```

Message này dự kiến sẽ thất bại nếu:

* người gửi không có đủ coin để trả constant fee
* invariant route chưa được đăng ký

Message này kiểm tra invariant được cung cấp, và nếu invariant bị phá vỡ nó sẽ
panic, làm blockchain bị halt. Nếu invariant bị phá vỡ, constant fee sẽ không
bị trừ vì giao dịch không bao giờ được commit vào block (tương đương được hoàn
lại). Tuy nhiên, nếu invariant không bị phá vỡ, constant fee sẽ không được hoàn lại.

## Events

Module crisis phát ra các event sau:

### Handlers

#### MsgVerifyInvariant

| Type      | Attribute Key | Attribute Value  |
|-----------|---------------|------------------|
| invariant | route         | {invariantRoute} |
| message   | module        | crisis           |
| message   | action        | verify_invariant |
| message   | sender        | {senderAddress}  |

## Parameters

Module crisis có các tham số sau:

| Key         | Type          | Ví dụ                             |
|-------------|---------------|-----------------------------------|
| ConstantFee | object (coin) | {"denom":"uatom","amount":"1000"} |

## Client

### CLI

Người dùng có thể truy vấn và tương tác với module `crisis` bằng CLI.

#### Transactions

Các lệnh `tx` cho phép người dùng tương tác với module `crisis`.

```bash
simd tx crisis --help
```

##### invariant-broken

Lệnh `invariant-broken` gửi bằng chứng rằng một invariant đã bị phá vỡ để halt chain.

```bash
simd tx crisis invariant-broken [module-name] [invariant-route] [flags]
```

Ví dụ:

```bash
simd tx crisis invariant-broken bank total-supply --from=[keyname or address]
```

