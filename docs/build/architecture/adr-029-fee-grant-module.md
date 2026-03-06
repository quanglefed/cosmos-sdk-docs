# ADR 029: Module Fee Grant (Cấp Phí)

## Nhật Ký Thay Đổi

* 18/08/2020: Bản nháp đầu tiên
* 05/05/2021: Xóa hỗ trợ hết hạn dựa theo chiều cao block và đơn giản hóa tên gọi.

## Trạng Thái

Đã Chấp Nhận

## Bối Cảnh

Để thực hiện các giao dịch blockchain, tài khoản ký phải sở hữu số dư đủ của đúng mệnh giá để trả phí. Có các loại giao dịch mà việc phải duy trì ví với phí đủ là rào cản để áp dụng.

Ví dụ, khi các quyền thích hợp được thiết lập, ai đó có thể tạm thời ủy quyền cho một tài khoản "burner" được lưu trữ trên điện thoại di động với bảo mật tối thiểu để bỏ phiếu về đề xuất.

Các trường hợp sử dụng khác bao gồm nhân viên theo dõi các mặt hàng trong chuỗi cung ứng hoặc nông dân gửi dữ liệu trường để phân tích hoặc tuân thủ.

Với tất cả các trường hợp sử dụng này, UX sẽ được cải thiện đáng kể bằng cách loại bỏ nhu cầu cho các tài khoản này phải luôn duy trì số dư phí thích hợp. Điều này đặc biệt đúng nếu chúng ta muốn đạt được áp dụng doanh nghiệp cho các thứ như theo dõi chuỗi cung ứng.

Mặc dù một giải pháp sẽ là có một dịch vụ tự động nạp các tài khoản này với phí thích hợp, một UX tốt hơn sẽ được cung cấp bằng cách cho phép các tài khoản này rút từ một tài khoản pool phí chung với giới hạn chi tiêu thích hợp. Một pool duy nhất sẽ giảm sự xáo trộn của việc thực hiện nhiều giao dịch "nạp tiền" nhỏ và cũng tận dụng hiệu quả hơn các nguồn lực của tổ chức thiết lập pool.

## Quyết Định

Như một giải pháp chúng tôi đề xuất một module, `x/feegrant` cho phép một tài khoản, "granter" cấp cho tài khoản khác, "grantee" một khoản trợ cấp để chi tiêu số dư tài khoản của granter cho phí trong các giới hạn được định nghĩa rõ ràng.

Trợ cấp phí được định nghĩa bởi interface `FeeAllowanceI` có thể mở rộng:

```go
type FeeAllowanceI {
  // Accept có thể sử dụng khoản phí được yêu cầu cũng như timestamp của block hiện tại
  // để xác định có xử lý hay không. Điều này được kiểm tra trong
  // Keeper.UseGrantedFees và các giá trị trả về nên khớp với cách nó được xử lý ở đó.
  //
  // Nếu trả về lỗi, khoản thanh toán phí bị từ chối, ngược lại nó được chấp nhận.
  // Triển khai FeeAllowance được kỳ vọng cập nhật trạng thái nội bộ của nó
  // và sẽ được lưu lại sau khi chấp nhận.
  //
  // Nếu remove là true (bất kể lỗi), FeeAllowance sẽ bị xóa khỏi storage
  // (vd: khi nó được sử dụng hết). (Xem lời gọi tới RevokeFeeAllowance trong Keeper.UseGrantedFees)
  Accept(ctx sdk.Context, fee sdk.Coins, msgs []sdk.Msg) (remove bool, err error)

  // ValidateBasic nên đánh giá FeeAllowance này về tính nhất quán nội bộ.
  // Không cho phép số âm, hoặc các khoảng thời gian âm chẳng hạn.
  ValidateBasic() error
}
```

Hai loại trợ cấp phí cơ bản, `BasicAllowance` và `PeriodicAllowance` được định nghĩa để hỗ trợ các trường hợp sử dụng đã biết:

```protobuf
// BasicAllowance triển khai FeeAllowanceI với một khoản cấp phát token một lần
// tùy chọn hết hạn. Người được ủy quyền có thể sử dụng tối đa SpendLimit để chi phí.
message BasicAllowance {
  // spend_limit chỉ định số lượng token tối đa có thể được chi tiêu
  // bởi khoản trợ cấp này và sẽ được cập nhật khi token được chi tiêu. Nếu rỗng,
  // không có giới hạn chi tiêu và có thể chi tiêu bất kỳ lượng coin nào.
  repeated cosmos_sdk.v1.Coin spend_limit = 1;

  // expiration chỉ định thời gian tùy chọn khi khoản trợ cấp này hết hạn
  google.protobuf.Timestamp expiration = 2;
}

// PeriodicAllowance mở rộng FeeAllowanceI để cho phép cả giới hạn tối đa,
// cũng như giới hạn mỗi khoảng thời gian.
message PeriodicAllowance {
  BasicAllowance basic = 1;

  // period chỉ định khoảng thời gian mà period_spend_limit coin có thể
  // được chi tiêu trước khi trợ cấp đó được đặt lại
  google.protobuf.Duration period = 2;

  // period_spend_limit chỉ định số coin tối đa có thể được chi tiêu
  // trong khoảng thời gian
  repeated cosmos_sdk.v1.Coin period_spend_limit = 3;

  // period_can_spend là số coin còn lại có thể chi tiêu trước thời gian period_reset
  repeated cosmos_sdk.v1.Coin period_can_spend = 4;

  // period_reset là thời gian mà khoảng thời gian này đặt lại và bắt đầu khoảng mới,
  // nó được tính từ thời gian bắt đầu của giao dịch đầu tiên sau khi
  // khoảng thời gian cuối kết thúc
  google.protobuf.Timestamp period_reset = 5;
}
```

Các khoản trợ cấp có thể được cấp và thu hồi sử dụng `MsgGrantAllowance` và `MsgRevokeAllowance`:

```protobuf
// MsgGrantAllowance thêm quyền cho Grantee chi tiêu tối đa Allowance
// phí từ tài khoản của Granter.
message MsgGrantAllowance {
     string granter = 1;
     string grantee = 2;
     google.protobuf.Any allowance = 3;
 }

 // MsgRevokeAllowance xóa bất kỳ FeeAllowance hiện tại nào từ Granter tới Grantee.
 message MsgRevokeAllowance {
     string granter = 1;
     string grantee = 2;
 }
```

Để sử dụng các khoản trợ cấp trong các giao dịch, chúng ta thêm trường mới `granter` vào kiểu `Fee` của giao dịch:

```protobuf
package cosmos.tx.v1beta1;

message Fee {
  repeated cosmos.base.v1beta1.Coin amount = 1;
  uint64 gas_limit = 2;
  string payer = 3;
  string granter = 4;
}
```

`granter` phải được để trống hoặc phải tương ứng với một tài khoản đã cấp trợ cấp phí cho người trả phí (người ký đầu tiên hoặc giá trị của trường `payer`).

Một `AnteDecorator` mới có tên `DeductGrantedFeeDecorator` sẽ được tạo ra để xử lý các giao dịch có `fee_payer` được đặt và khấu trừ đúng phí dựa trên trợ cấp phí.

## Hậu Quả

### Tích Cực

* Cải thiện UX cho các trường hợp sử dụng mà việc duy trì số dư tài khoản chỉ để trả phí là cồng kềnh

### Tiêu Cực

### Trung Lập

* Một trường mới phải được thêm vào message `Fee` của giao dịch và một `AnteDecorator` mới phải được tạo ra để sử dụng nó

## Tham Khảo

* Bài viết blog mô tả công việc ban đầu: https://medium.com/regen-network/hacking-the-cosmos-cosmwasm-and-key-management-a08b9f561d1b
* Đặc tả công khai ban đầu: https://gist.github.com/aaronc/b60628017352df5983791cad30babe56
* Đề xuất subkeys gốc từ B-harvest đã ảnh hưởng đến thiết kế này: https://github.com/cosmos/cosmos-sdk/issues/4480
