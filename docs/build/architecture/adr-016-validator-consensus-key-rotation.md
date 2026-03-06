# ADR 016: Xoay Vòng Khóa Consensus Validator

## Changelog

* 23 tháng 10 năm 2019: Bản nháp đầu tiên
* 28 tháng 11 năm 2019: Thêm phí xoay vòng khóa

## Bối Cảnh

Tính năng xoay vòng khóa consensus validator đã được thảo luận và yêu cầu trong một thời gian dài, vì mục đích chính sách quản lý khóa validator an toàn hơn (ví dụ: https://github.com/tendermint/tendermint/issues/1136). Vì vậy, chúng tôi đề xuất một trong những hình thức triển khai xoay vòng khóa consensus validator đơn giản nhất chủ yếu vào Cosmos SDK.

Chúng ta không cần thực hiện bất kỳ cập nhật nào về logic đồng thuận trong Tendermint vì Tendermint không có bất kỳ thông tin ánh xạ nào về khóa consensus và khóa operator validator, có nghĩa là từ góc độ của Tendermint, xoay vòng khóa consensus của validator chỉ đơn giản là thay thế khóa consensus bằng khóa khác.

Ngoài ra, cần lưu ý rằng ADR này chỉ bao gồm hình thức xoay vòng khóa consensus đơn giản nhất mà không xem xét khái niệm nhiều khóa consensus. Khái niệm nhiều khóa consensus như vậy vẫn là mục tiêu dài hạn của Tendermint và Cosmos SDK.

## Quyết Định

### Quy Trình Giả Cho Xoay Vòng Khóa Consensus

* tạo cặp khóa consensus ngẫu nhiên mới.
* tạo và broadcast một giao dịch với `MsgRotateConsPubKey` nêu rõ khóa consensus mới hiện được ghép với operator validator với chữ ký từ khóa operator của validator.
* khóa consensus cũ trở nên không thể tham gia đồng thuận ngay sau khi cập nhật trạng thái ánh xạ khóa on-chain.
* bắt đầu xác thực với khóa consensus mới.
* các validator sử dụng HSM và KMS nên cập nhật khóa consensus trong HSM để sử dụng khóa đã xoay vòng mới sau chiều cao `h` khi `MsgRotateConsPubKey` được cam kết vào blockchain.

### Các Cân Nhắc

* chiến lược quản lý thông tin ánh xạ khóa consensus
    * lưu lịch sử của mỗi thay đổi ánh xạ khóa trong kvstore.
    * state machine có thể tìm kiếm khóa consensus tương ứng được ghép với operator validator nhất định cho bất kỳ chiều cao tùy ý nào trong unbonding period gần đây.
    * state machine không cần bất kỳ thông tin ánh xạ lịch sử nào cũ hơn unbonding period.
* chi phí xoay vòng khóa liên quan đến LCD và IBC
    * LCD và IBC sẽ có gánh nặng traffic/tính toán khi có thay đổi power thường xuyên
    * Trong thiết kế Tendermint hiện tại, xoay vòng khóa consensus được coi là thay đổi power từ góc độ LCD hoặc IBC
    * Do đó, để giảm thiểu hành vi xoay vòng khóa thường xuyên không cần thiết, chúng ta giới hạn số lần xoay vòng tối đa trong unbonding period gần đây và cũng áp dụng phí xoay vòng tăng theo cấp số nhân
* giới hạn
    * một validator không thể xoay vòng khóa consensus hơn `MaxConsPubKeyRotations` lần trong bất kỳ unbonding period nào, để ngăn spam.
    * tham số có thể được quyết định bởi quản trị và lưu trong genesis file.
* phí xoay vòng khóa
    * một validator nên trả `KeyRotationFee` để xoay vòng khóa consensus được tính như sau
    * `KeyRotationFee` = (max(`VotingPowerPercentage` *100, 1)* `InitialKeyRotationFee`) * 2^(số lần xoay vòng trong `ConsPubKeyRotationHistory` trong unbonding period gần đây)
* module evidence
    * module evidence có thể tìm kiếm khóa consensus tương ứng cho bất kỳ chiều cao nào từ slashing keeper để có thể quyết định khóa consensus nào được cho là được sử dụng cho chiều cao đã cho.
* abci.ValidatorUpdate
    * tendermint đã có khả năng thay đổi khóa consensus bằng giao tiếp ABCI (`ValidatorUpdate`).
    * cập nhật khóa consensus validator có thể được thực hiện bằng cách tạo mới + xóa cũ bằng cách thay đổi power về không.
    * do đó, chúng ta kỳ vọng thậm chí không cần thay đổi codebase Tendermint để triển khai tính năng này.
* tham số genesis mới trong module `staking`
    * `MaxConsPubKeyRotations`: số lần xoay vòng tối đa có thể được thực hiện bởi validator trong unbonding period gần đây. Giá trị mặc định 10 được đề xuất (lần xoay vòng khóa thứ 11 sẽ bị từ chối)
    * `InitialKeyRotationFee`: phí xoay vòng khóa ban đầu khi không có xoay vòng khóa nào xảy ra trong unbonding period gần đây. Giá trị mặc định 1atom được đề xuất (phí 1atom cho lần xoay vòng khóa đầu tiên trong unbonding period gần đây)

### Quy Trình Làm Việc

1. Validator tạo một cặp keypair consensus mới.
2. Validator tạo và ký một `MsgRotateConsPubKey` với khóa operator và ConsPubKey mới của họ

    ```go
    type MsgRotateConsPubKey struct {
        ValidatorAddress  sdk.ValAddress
        NewPubKey         crypto.PubKey
    }
    ```

3. `handleMsgRotateConsPubKey` nhận `MsgRotateConsPubKey`, gọi `RotateConsPubKey` với phát sự kiện
4. `RotateConsPubKey`
    * kiểm tra xem `NewPubKey` không bị trùng lặp trong `ValidatorsByConsAddr`
    * kiểm tra xem validator không vượt quá tham số `MaxConsPubKeyRotations` bằng cách iterate `ConsPubKeyRotationHistory`
    * kiểm tra xem tài khoản ký có đủ số dư để trả `KeyRotationFee` không
    * trả `KeyRotationFee` vào quỹ cộng đồng
    * ghi đè `NewPubKey` trong `validator.ConsPubKey`
    * xóa `ValidatorByConsAddr` cũ
    * `SetValidatorByConsAddr` cho `NewPubKey`
    * Thêm `ConsPubKeyRotationHistory` để theo dõi xoay vòng

    ```go
    type ConsPubKeyRotationHistory struct {
        OperatorAddress         sdk.ValAddress
        OldConsPubKey           crypto.PubKey
        NewConsPubKey           crypto.PubKey
        RotatedHeight           int64
    }
    ```

5. `ApplyAndReturnValidatorSetUpdates` kiểm tra xem có `ConsPubKeyRotationHistory` với `ConsPubKeyRotationHistory.RotatedHeight == ctx.BlockHeight()` không và nếu có, tạo 2 `ValidatorUpdate`, một để xóa validator và một để tạo validator mới

    ```go
    abci.ValidatorUpdate{
        PubKey: cmttypes.TM2PB.PubKey(OldConsPubKey),
        Power:  0,
    }

    abci.ValidatorUpdate{
        PubKey: cmttypes.TM2PB.PubKey(NewConsPubKey),
        Power:  v.ConsensusPower(),
    }
    ```

6. tại logic iterate `previousVotes` của `AllocateTokens`, `previousVote` sử dụng `OldConsPubKey` khớp với `ConsPubKeyRotationHistory`, và thay thế validator để phân bổ token
7. Di chuyển `ValidatorSigningInfo` và `ValidatorMissedBlockBitArray` từ `OldConsPubKey` sang `NewConsPubKey`

* Lưu ý: Tất cả các tính năng trên sẽ được triển khai trong module `staking`.

## Trạng Thái

Đề Xuất

## Hậu Quả

### Tích Cực

* Validator có thể ngay lập tức hoặc định kỳ xoay vòng khóa consensus của họ để có chính sách bảo mật tốt hơn
* cải thiện bảo mật chống lại các cuộc tấn công Long-Range (https://nearprotocol.com/blog/long-range-attacks-and-a-new-fork-choice-rule) khi validator bỏ đi (các) khóa consensus cũ

### Tiêu Cực

* Module Slash cần tính toán nhiều hơn vì nó cần tra cứu khóa consensus tương ứng của validator cho mỗi chiều cao
* xoay vòng khóa thường xuyên sẽ làm cho bisection client nhẹ kém hiệu quả hơn

### Trung Lập

## Tài Liệu Tham Khảo

* trên tendermint repo: https://github.com/tendermint/tendermint/issues/1136
* trên cosmos-sdk repo: https://github.com/cosmos/cosmos-sdk/issues/5231
* về nhiều khóa consensus: https://github.com/tendermint/tendermint/issues/1758#issuecomment-545291698
