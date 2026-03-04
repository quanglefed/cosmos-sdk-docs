---
sidebar_position: 1
---

# Errors (Lỗi)

:::note Tóm tắt
Tài liệu này trình bày cách sử dụng được khuyến nghị và các API để xử lý lỗi trong các Cosmos SDK module.
:::

Các module được khuyến khích định nghĩa và đăng ký các lỗi của riêng chúng để cung cấp ngữ cảnh tốt hơn về việc thực thi message hoặc handler thất bại. Thông thường, những lỗi này nên là các lỗi phổ biến hoặc tổng quát có thể được bọc thêm để cung cấp ngữ cảnh thực thi cụ thể hơn.

## Đăng Ký (Registration)

Các module nên định nghĩa và đăng ký các lỗi tùy chỉnh của họ trong `x/{module}/errors.go`. Việc đăng ký lỗi được xử lý thông qua [`gói errors`](https://github.com/cosmos/cosmos-sdk/blob/main/errors/errors.go).

Ví dụ:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/distribution/types/errors.go
```

Mỗi lỗi module tùy chỉnh phải cung cấp codespace, thường là tên module (ví dụ: "distribution") và là duy nhất cho mỗi module, và một mã uint32. Cùng nhau, codespace và mã cung cấp một lỗi Cosmos SDK duy nhất trên toàn cầu. Thông thường, mã tăng đơn điệu nhưng không nhất thiết phải như vậy. Các hạn chế duy nhất đối với mã lỗi là:

* Phải lớn hơn một, vì giá trị mã bằng một được dành riêng cho các lỗi nội bộ.
* Phải là duy nhất trong module.

Lưu ý, Cosmos SDK cung cấp một bộ lỗi *phổ biến* cốt lõi. Các lỗi này được định nghĩa trong [`types/errors/errors.go`](https://github.com/cosmos/cosmos-sdk/blob/main/types/errors/errors.go).

## Bọc (Wrapping)

Các lỗi module tùy chỉnh có thể được trả về như kiểu cụ thể của chúng vì chúng đã thỏa mãn interface `error`. Tuy nhiên, các lỗi module có thể được bọc để cung cấp thêm ngữ cảnh và ý nghĩa cho việc thực thi thất bại.

Ví dụ:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/bank/keeper/keeper.go#L141-L182
```

Bất kể lỗi có được bọc hay không, gói `errors` của Cosmos SDK cung cấp hàm để xác định xem một lỗi có thuộc một loại cụ thể hay không thông qua `Is`.

## ABCI

Nếu một lỗi module được đăng ký, gói `errors` của Cosmos SDK cho phép thông tin ABCI được trích xuất thông qua hàm `ABCIInfo`. Gói này cũng cung cấp `ResponseCheckTx` và `ResponseDeliverTx` như các hàm phụ trợ để tự động lấy phản hồi `CheckTx` và `DeliverTx` từ một lỗi.
