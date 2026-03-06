# ADR 034: Tái Khóa Tài Khoản (Account Rekeying)

## Nhật Ký Thay Đổi

* 30-09-2020: Bản nháp đầu tiên

## Trạng Thái

ĐỀ XUẤT

## Tóm Tắt

Tái khóa tài khoản là một quy trình cho phép tài khoản thay thế khóa công khai xác thực của nó bằng một khóa mới.

## Bối Cảnh

Hiện tại, trong Cosmos SDK, địa chỉ của `BaseAccount` auth dựa trên hàm băm của khóa công khai. Một khi tài khoản được tạo, khóa công khai cho tài khoản đó được cố định và không thể thay đổi. Điều này có thể gây ra vấn đề cho người dùng, vì xoay khóa là một thực hành bảo mật hữu ích, nhưng hiện tại không thể thực hiện được. Hơn nữa, vì multisig là một loại pubkey, một khi multisig cho một tài khoản được đặt, nó không thể được cập nhật. Điều này gây ra vấn đề, vì multisig thường được sử dụng bởi các tổ chức hoặc công ty, những người có thể cần thay đổi tập hợp người ký multisig của họ vì lý do nội bộ.

Việc chuyển tất cả tài sản của một tài khoản sang tài khoản mới với pubkey được cập nhật là không đủ, vì một số "cam kết" của tài khoản không dễ dàng chuyển được. Ví dụ, trong staking, để chuyển các Atom đã bond, một tài khoản sẽ phải unbond tất cả các delegation và chờ thời gian unbonding ba tuần. Thậm chí quan trọng hơn, đối với các operator validator, quyền sở hữu trên validator không thể chuyển nhượng được, có nghĩa là khóa operator cho validator không bao giờ có thể được cập nhật, dẫn đến bảo mật hoạt động kém cho các validator.

## Quyết Định

Chúng ta đề xuất thêm tính năng mới vào `x/auth` cho phép các tài khoản cập nhật khóa công khai liên kết với tài khoản của họ, trong khi giữ nguyên địa chỉ.

Điều này có thể thực hiện được vì Cosmos SDK `BaseAccount` lưu trữ khóa công khai cho tài khoản trong trạng thái, thay vì giả định rằng khóa công khai được bao gồm trong giao dịch (dù rõ ràng hay ngầm thông qua chữ ký) như trong các blockchain khác như Bitcoin và Ethereum. Vì khóa công khai được lưu trữ on-chain, không cần khóa công khai phải là hàm băm của địa chỉ tài khoản, vì địa chỉ không liên quan đến quá trình kiểm tra chữ ký.

Để xây dựng hệ thống này, chúng ta thiết kế một kiểu `Msg` mới như sau:

```protobuf
service Msg {
    rpc ChangePubKey(MsgChangePubKey) returns (MsgChangePubKeyResponse);
}

message MsgChangePubKey {
  string address = 1;
  google.protobuf.Any pub_key = 2;
}

message MsgChangePubKeyResponse {}
```

Giao dịch `MsgChangePubKey` cần được ký bởi pubkey hiện tại trong trạng thái.

Sau khi được phê duyệt, handler cho kiểu message này, nhận AccountKeeper, sẽ cập nhật pubkey trong trạng thái cho tài khoản và thay thế nó bằng pubkey từ Msg.

Một tài khoản đã được thay đổi pubkey của nó không thể tự động được dọn dẹp khỏi trạng thái. Điều này là vì nếu bị dọn dẹp, pubkey gốc của tài khoản sẽ cần thiết để tái tạo cùng một địa chỉ, nhưng chủ sở hữu địa chỉ có thể không còn pubkey gốc nữa. Hiện tại, chúng ta không tự động dọn dẹp bất kỳ tài khoản nào, nhưng chúng ta muốn giữ tùy chọn này mở trong tương lai (đây là mục đích của các số tài khoản). Để giải quyết điều này, chúng ta tính thêm phí gas cho thao tác này để bù đắp cho ngoại tác này (số gas ràng buộc này được cấu hình như tham số `PubKeyChangeCost`). Gas thưởng được tính bên trong handler, sử dụng hàm `ConsumeGas`. Hơn nữa, trong tương lai, chúng ta có thể cho phép các tài khoản đã tái khóa tự dọn dẹp thủ công bằng cách sử dụng kiểu `Msg` mới như `MsgDeleteAccount`. Việc dọn dẹp tài khoản thủ công có thể cung cấp hoàn trả gas như một khuyến khích để thực hiện hành động.

```go
	amount := ak.GetParams(ctx).PubKeyChangeCost
	ctx.GasMeter().ConsumeGas(amount, "pubkey change fee")
```

Mỗi khi khóa cho một địa chỉ được thay đổi, chúng ta sẽ lưu trữ nhật ký của thay đổi này trong trạng thái của chain, do đó tạo ra một ngăn xếp tất cả các khóa trước đó cho một địa chỉ và các khoảng thời gian chúng còn hoạt động. Điều này cho phép các dapp và client dễ dàng truy vấn các khóa cũ cho một tài khoản, có thể hữu ích cho các tính năng như xác minh các message đã ký off-chain có dấu thời gian.

## Hậu Quả

### Tích Cực

* Sẽ cho phép người dùng và operator validator sử dụng các thực hành bảo mật hoạt động tốt hơn với xoay khóa.
* Sẽ cho phép các tổ chức hoặc nhóm dễ dàng thay đổi và thêm/xóa người ký multisig.

### Tiêu Cực

Phá vỡ mối quan hệ giả định hiện tại giữa địa chỉ và pubkey là H(pubkey) = address. Điều này có một số hệ quả.

* Điều này làm cho các ví hỗ trợ tính năng này phức tạp hơn. Ví dụ, nếu một địa chỉ on-chain được cập nhật, khóa tương ứng trong ví CLI cũng cần được cập nhật.
* Không thể tự động dọn dẹp các tài khoản có số dư 0 đã thay đổi pubkey của chúng.

### Trung Lập

* Mặc dù mục đích của điều này là cho phép chủ sở hữu tài khoản cập nhật lên pubkey mới mà họ sở hữu, điều này về mặt kỹ thuật cũng có thể được sử dụng để chuyển quyền sở hữu tài khoản cho chủ sở hữu mới. Ví dụ, điều này có thể được sử dụng để bán một vị trí staked mà không cần unbond hoặc một tài khoản có các token vesting. Tuy nhiên, ma sát của điều này là rất cao vì điều này về cơ bản phải được thực hiện như một giao dịch OTC rất cụ thể. Hơn nữa, có thể thêm các ràng buộc bổ sung để ngăn các tài khoản có token Vesting sử dụng tính năng này.
* Sẽ yêu cầu rằng PubKeys cho một tài khoản được bao gồm trong các genesis export.

## Tham Khảo

* https://www.algorand.com/resources/blog/announcing-rekeying
