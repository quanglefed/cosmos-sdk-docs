---
sidebar_position: 1
---

# Mô Hình Object-Capability

## Giới thiệu

Khi xem xét về bảo mật, điều tốt nhất là bắt đầu từ một mô hình mối đe dọa cụ thể. Mô hình mối đe dọa của chúng tôi như sau:

> Chúng tôi giả định rằng một hệ sinh thái phong phú gồm các module Cosmos SDK dễ dàng kết hợp thành một ứng dụng blockchain sẽ chứa các module có lỗi hoặc độc hại.

Cosmos SDK được thiết kế để giải quyết mối đe dọa này bằng cách là nền tảng của một hệ thống object capability.

> Các thuộc tính cấu trúc của hệ thống object capability ủng hộ tính mô-đun trong thiết kế mã và đảm bảo tính đóng gói đáng tin cậy trong việc triển khai mã.
>
> Những thuộc tính cấu trúc này tạo điều kiện cho việc phân tích một số thuộc tính bảo mật của chương trình hoặc hệ điều hành có object-capability. Một số thuộc tính trong đó — đặc biệt là các thuộc tính luồng thông tin — có thể được phân tích ở mức độ tham chiếu đối tượng và khả năng kết nối, độc lập với bất kỳ kiến thức hay phân tích nào về mã xác định hành vi của các đối tượng.
>
> Kết quả là, các thuộc tính bảo mật này có thể được thiết lập và duy trì ngay cả khi có các đối tượng mới chứa mã chưa biết và có thể độc hại.
>
> Những thuộc tính cấu trúc này xuất phát từ hai quy tắc quản lý quyền truy cập vào các đối tượng hiện có:
>
> 1. Đối tượng A chỉ có thể gửi tin nhắn đến B nếu đối tượng A giữ một tham chiếu đến B.
> 2. Đối tượng A chỉ có thể lấy tham chiếu đến C nếu đối tượng A nhận được tin nhắn chứa tham chiếu đến C. Kết quả của hai quy tắc này là, một đối tượng chỉ có thể lấy tham chiếu đến đối tượng khác thông qua một chuỗi tham chiếu đã có từ trước. Nói ngắn gọn: "Chỉ kết nối mới sinh ra kết nối" (Only connectivity begets connectivity).

Để tìm hiểu thêm về object-capability, xem [bài viết Wikipedia](https://en.wikipedia.org/wiki/Object-capability_model) này.

## Ocaps trong thực tế

Ý tưởng là chỉ tiết lộ những gì cần thiết để hoàn thành công việc.

Ví dụ, đoạn code sau đây vi phạm nguyên tắc object capabilities:

```go
type AppAccount struct {...}
account := &AppAccount{
    Address: pub.Address(),
    Coins: sdk.Coins{sdk.NewInt64Coin("ATM", 100)},
}
sumValue := externalModule.ComputeSumValue(account)
```

Phương thức `ComputeSumValue` gợi ý một hàm thuần túy, nhưng khả năng ngầm định của việc chấp nhận giá trị con trỏ là khả năng sửa đổi giá trị đó. Chữ ký phương thức ưa thích nên nhận một bản sao thay thế:

```go
sumValue := externalModule.ComputeSumValue(*account)
```

Trong Cosmos SDK, bạn có thể thấy việc áp dụng nguyên tắc này trong simapp.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/simapp/app.go
```

Sơ đồ sau đây thể hiện các phụ thuộc hiện tại giữa các keeper.

![Phụ thuộc giữa các Keeper](https://raw.githubusercontent.com/cosmos/cosmos-sdk/release/v0.46.x/docs/uml/svg/keeper_dependencies.svg)
