# ADR-061: Liquid Staking

## Changelog

* 2022-09-10: Bản nháp đầu tiên (@zmanian)

## Trạng Thái

ACCEPTED

## Tóm Tắt

Thêm một primitive liquid staking bán-fungible vào module staking mặc định của Cosmos SDK. Điều này nâng cấp proof of stake để cho phép các thiết kế an toàn hơn với mức phát hành tiền tệ tổng thể thấp hơn và tích hợp với nhiều giao thức liquid staking như Stride, Persistence, Quicksilver, Lido v.v.

## Bối Cảnh

Phát hành ban đầu của Cosmos Hub có triển khai proof of stake tiên phong với delegation, slashing, phân phối phần thưởng trong giao thức và phát hành adaptive. Thiết kế này là state-of-the-art năm 2016.

Vì cả Proof of Stake và các use case blockchain đã trưởng thành, thiết kế này đã lỗi thời. Khiếm khuyết quan trọng nhất của thiết kế staking legacy là nó kết hợp kém với các giao thức on-chain cho giao dịch, cho vay, phái sinh được gọi chung là DeFi.

Điều quan trọng cần lưu ý là Interchain Accounts đã có sẵn trong triển khai IBC mặc định và có thể được sử dụng để [rehypothecate](https://www.investopedia.com/terms/h/hypothecation.asp#toc-what-is-rehypothecation) delegation. Do đó liquid staking đã có thể và những thay đổi này chỉ cải thiện UX của liquid staking. ADR này đưa ra quan điểm rằng việc áp dụng liquid staking trong giao thức là kết quả được ưu tiên.

Những thay đổi đối với module staking này đã được phát triển hơn một năm và được ngành công nghiệp áp dụng đáng kể.

## Quyết Định

Chúng ta triển khai hệ thống liquid staking bán-fungible và hệ thống exemption factor trong cosmos sdk. Mặc dù được đăng ký là tài sản fungible, các tokenized share này có tính fungibility cực kỳ hạn chế, chỉ trong số bản ghi delegation cụ thể được tạo khi share được tokenize.

Một tham số governance mới được giới thiệu định nghĩa tỷ lệ giữa exempt và issued tokenized share. Đây là **exemption factor**. Exemption factor lớn hơn cho phép nhiều tokenized share được phát hành hơn với số lượng exempt delegation nhỏ hơn.

Min self delegation bị xóa khỏi hệ thống staking với kỳ vọng rằng nó sẽ được thay thế bởi hệ thống exempt delegation. Hệ thống exempt delegation cho phép nhiều tài khoản thể hiện sự liên kết kinh tế với operator validator mà không trộn lẫn quỹ.

Khi share được tokenize, các share cơ bản được chuyển đến một module account và phần thưởng đi đến module account cho TokenizedShareRecord.

Không còn cơ chế nào để override phiếu bầu của validator cho TokenizedShares.

### `MsgTokenizeShares`

Message `MsgTokenizeShares` được sử dụng để tokenize delegation. Sau khi thực thi, lượng delegation cụ thể biến mất khỏi tài khoản và share token được cung cấp. Share token được mệnh giá theo validator và record id của delegation cơ bản, ví dụ `cosmosvaloper1xxxx/5`.

`MsgTokenizeShares` thất bại nếu tài khoản là VestingAccount.

Tổng số tokenized share nổi bật cho validator được kiểm tra với tổng của exempt delegation nhân với exemption factor. Nếu tokenized share vượt quá giới hạn này, thực thi thất bại.

### `MsgRedeemTokensforShares`

Message `MsgRedeemTokensforShares` được sử dụng để đổi delegation từ share token. Sau khi thực thi, delegation sẽ xuất hiện với người dùng.

### `MsgTransferTokenizeShareRecord`

Message `MsgTransferTokenizeShareRecord` được sử dụng để chuyển quyền sở hữu phần thưởng được tạo ra từ số lượng delegation được tokenize. TokenizeShareRecord được tạo khi người dùng tokenize delegation của họ và bị xóa khi toàn bộ share token được đổi lại.

Điều này được thiết kế để hoạt động với các thiết kế liquid staking không đổi lại tokenized share.

### `MsgExemptDelegation`

Message `MsgExemptDelegation` được sử dụng để exempt một delegation đối với validator. Nếu exemption factor lớn hơn 0, điều này sẽ cho phép phát hành thêm delegation share từ validator.

## Hậu Quả

### Tương Thích Ngược

Bằng cách đặt exemption factor thành không, module này hoạt động như staking legacy. Thay đổi đáng kể duy nhất là việc xóa min-self-bond.

### Tích Cực

Cách tiếp cận này nên cho phép tích hợp với các nhà cung cấp liquid staking và cải thiện trải nghiệm người dùng. Nó cung cấp con đường đến bảo mật dưới các chính sách phát hành không theo cấp số nhân.
