# ADR 048: Hệ Thống Giá Gas Đa Tầng

## Changelog

* 1 tháng 12 năm 2021: Bản nháp đầu tiên

## Trạng Thái

Bị Từ Chối

## Tóm Tắt

ADR này mô tả một cơ chế linh hoạt để duy trì giá gas ở cấp độ đồng thuận, trong đó người ta có thể chọn hệ thống giá gas đa tầng hoặc hệ thống tương tự EIP-1559 thông qua cấu hình.

## Bối Cảnh

Hiện tại, mỗi validator cấu hình `minimal-gas-prices` của riêng họ trong `app.yaml`. Nhưng việc thiết lập giá gas tối thiểu phù hợp rất quan trọng để bảo vệ mạng khỏi tấn công DoS, và rất khó để tất cả validator chọn một giá trị hợp lý, vì vậy chúng tôi đề xuất duy trì giá gas ở cấp độ đồng thuận.

## Hệ Thống Giá Đa Tầng

Chúng tôi đề xuất hệ thống giá đa tầng trên đồng thuận để cung cấp tính linh hoạt tối đa:

* Tầng 1: giá gas không đổi, chỉ có thể sửa đổi thỉnh thoảng thông qua đề xuất governance.
* Tầng 2: giá gas động được điều chỉnh theo tải block trước.
* Tầng 3: giá gas động được điều chỉnh theo tải block trước ở tốc độ cao hơn.

Giá gas của tầng cao hơn phải lớn hơn tầng thấp hơn.

Schema tham số:

```protobuf
message TierParams {
  uint32 priority = 1
  Coin initial_gas_price = 2
  uint32 parent_gas_target = 3
  uint32 change_denominator = 4
  Coin min_gas_price = 5
  Coin max_gas_price = 6
}

message Params {
  repeated TierParams tiers = 1;
}
```

### Tùy Chọn Mở Rộng

Để cho phép người dùng chỉ định tầng dịch vụ cho giao dịch, chúng ta thêm một tùy chọn mở rộng trong `AuthInfo`:

```protobuf
message ExtensionOptionsTieredTx {
  uint32 fee_tier = 1
}
```

### Ưu Tiên Tx

Giao dịch được ưu tiên dựa trên tầng — tầng càng cao, ưu tiên càng cao.

### Điều Chỉnh Theo Tải Block

Đối với giao dịch tầng 2 và tầng 3, giá gas được điều chỉnh theo tải block trước, logic tương tự EIP-1559:

```python
def adjust_gas_price(gas_price, parent_gas_used, tier):
  if parent_gas_used == tier.parent_gas_target:
    return gas_price
  elif parent_gas_used > tier.parent_gas_target:
    gas_used_delta = parent_gas_used - tier.parent_gas_target
    gas_price_delta = max(gas_price * gas_used_delta // tier.parent_gas_target // tier.change_speed, 1)
    return gas_price + gas_price_delta
  else:
    gas_used_delta = parent_gas_target - parent_gas_used
    gas_price_delta = gas_price * gas_used_delta // parent_gas_target // tier.change_speed
    return gas_price - gas_price_delta
```

## Hậu Quả

### Tương Thích Ngược

* Tham số giao thức mới.
* State đồng thuận mới.
* Trường mới/thay đổi trong thân giao dịch.

### Tích Cực

* Tầng mặc định giữ trải nghiệm giá gas dự đoán được.
* Giá gas tầng cao hơn có thể thích nghi với tải block.
* Không có xung đột ưu tiên với ưu tiên tùy chỉnh dựa trên loại giao dịch.

### Tiêu Cực

* Ví & công cụ cần cập nhật để hỗ trợ tham số `tier` mới.

## Tài Liệu Tham Khảo

* https://eips.ethereum.org/EIPS/eip-1559
