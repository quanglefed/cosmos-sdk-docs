# ADR 008: Nhóm Đội Phản Ứng Khẩn Cấp Máy Tính Phi Tập Trung (dCERT)

## Changelog

* 31 tháng 7 năm 2019: Bản nháp đầu tiên

## Bối Cảnh

Để giảm số lượng bên liên quan đến việc xử lý thông tin nhạy cảm trong tình huống khẩn cấp, chúng tôi đề xuất tạo một nhóm chuyên biệt hóa có tên Đội Phản Ứng Khẩn Cấp Máy Tính Phi Tập Trung (dCERT). Ban đầu, vai trò của nhóm này là đóng vai điều phối viên giữa các tác nhân khác nhau trong cộng đồng blockchain như validator, bug-hunter và nhà phát triển. Trong thời điểm khủng hoảng, nhóm dCERT sẽ tổng hợp và chuyển tiếp đầu vào từ nhiều stakeholder đến các nhà phát triển đang tích cực nghĩ ra bản vá cho phần mềm, theo cách này thông tin nhạy cảm không cần được công bố công khai trong khi vẫn có thể thu thập được một số đầu vào từ cộng đồng.

Ngoài ra, một đặc quyền đặc biệt được đề xuất cho nhóm dCERT: khả năng "circuit-break" (tức là tạm thời vô hiệu hóa) một đường dẫn message cụ thể. Lưu ý rằng đặc quyền này nên được bật/tắt toàn cục với một governance parameter để đặc quyền này có thể bắt đầu ở trạng thái bị vô hiệu hóa và sau đó được bật thông qua đề xuất thay đổi tham số, khi một nhóm dCERT đã được thành lập.

Trong tương lai, có thể dự kiến rằng cộng đồng có thể muốn mở rộng vai trò của dCERT với nhiều trách nhiệm hơn như khả năng "phê duyệt trước" một bản cập nhật bảo mật thay mặt cộng đồng trước khi có cuộc bỏ phiếu toàn cộng đồng, trong đó thông tin nhạy cảm sẽ được tiết lộ trước khi lỗ hổng được vá trên mạng trực tiếp.

## Quyết Định

Nhóm dCERT được đề xuất bao gồm triển khai một `SpecializationGroup` như được định nghĩa trong [ADR 007](./adr-007-specialization-groups.md). Điều này sẽ bao gồm triển khai:

* bỏ phiếu liên tục
* slashing do vi phạm hợp đồng mềm
* thu hồi thành viên do vi phạm hợp đồng mềm
* giải tán khẩn cấp toàn bộ nhóm dCERT (ví dụ: vì thông đồng độc hại)
* trợ cấp bồi thường từ pool cộng đồng hoặc các phương tiện khác do quản trị quyết định

Hệ thống này đòi hỏi các tham số mới sau:

* trợ cấp khối hàng ngày cho mỗi thành viên dCERT
* số lượng tối đa thành viên dCERT
* token có thể bị slashing bắt buộc stake cho mỗi thành viên dCERT
* quorum để đình chỉ một thành viên cụ thể
* wager đề xuất để giải tán nhóm dCERT
* giai đoạn ổn định cho chuyển đổi thành viên dCERT
* đặc quyền circuit break dCERT được bật

Các tham số này được kỳ vọng được triển khai thông qua param keeper để quản trị có thể thay đổi chúng vào bất kỳ thời điểm nào.

### Electionator Bỏ Phiếu Liên Tục

Một đối tượng `Electionator` sẽ được triển khai là bỏ phiếu liên tục và với các thông số kỹ thuật sau:

* Tất cả địa chỉ ủy quyền có thể gửi phiếu bầu vào bất kỳ thời điểm nào, cập nhật đại diện ưa thích của họ trên nhóm dCERT.
* Đại diện ưa thích có thể được chia tùy ý giữa các địa chỉ (ví dụ: 50% cho John, 25% cho Sally, 25% cho Carol)
* Để một thành viên mới được thêm vào nhóm dCERT, họ phải gửi một giao dịch chấp nhận việc gia nhập của mình, lúc đó tính hợp lệ của việc gia nhập sẽ được xác nhận.
    * Một số thứ tự được gán khi thành viên được thêm vào nhóm dCERT. Nếu thành viên rời khỏi nhóm dCERT và sau đó tái gia nhập, một số thứ tự mới được gán.
* Các địa chỉ kiểm soát lượng đại diện ưa thích lớn nhất đủ điều kiện tham gia nhóm dCERT (lên đến _số thành viên dCERT tối đa_). Nếu nhóm dCERT đã đầy và thành viên mới được tiếp nhận, thành viên dCERT hiện tại với số phiếu thấp nhất sẽ bị loại khỏi nhóm dCERT.
    * Trong tình huống ngang bằng khi nhóm dCERT đã đầy nhưng ứng viên mới có số phiếu bằng với thành viên dCERT hiện tại, thành viên hiện tại nên giữ vị trí của mình.
    * Trong tình huống ngang bằng khi ai đó phải bị loại nhưng hai địa chỉ có số phiếu nhỏ nhất bằng nhau, địa chỉ có số thứ tự nhỏ hơn giữ vị trí của mình.
* Có thể bao gồm giai đoạn ổn định tùy chọn để giảm "flip-flopping" của các thành viên dCERT ở đuôi. Nếu giai đoạn ổn định được cung cấp lớn hơn 0, khi các thành viên bị loại vì hỗ trợ không đủ, một mục hàng đợi được tạo ghi lại thành viên nào sẽ thay thế thành viên nào khác. Trong khi mục này trong hàng đợi, không có mục mới nào để loại cùng thành viên dCERT đó có thể được tạo. Khi mục đáo hạn vào thời gian của giai đoạn ổn định, thành viên mới được khởi tạo và thành viên cũ bị loại.

### Staking/Slashing

Tất cả thành viên nhóm dCERT phải stake token _cụ thể_ để duy trì đủ điều kiện làm thành viên dCERT. Những token này có thể được stake trực tiếp bởi thành viên dCERT hoặc từ thiện của bên thứ 3 (người sẽ không nhận được bất kỳ lợi ích on-chain nào cho việc đó). Cơ chế staking này nên sử dụng thời gian unbonding toàn cục hiện tại của token được stake cho bảo mật validator mạng. Thành viên dCERT _chỉ có thể là_ thành viên nếu họ có token bắt buộc được stake dưới cơ chế này. Nếu những token đó được unbond thì thành viên dCERT phải bị tự động loại khỏi nhóm.

Slashing một thành viên dCERT cụ thể do vi phạm hợp đồng mềm nên được thực hiện bởi quản trị trên cơ sở từng thành viên dựa trên mức độ vi phạm. Quy trình dự kiến là thành viên dCERT bị đình chỉ bởi nhóm dCERT trước khi bị slash bởi quản trị.

Đình chỉ thành viên bởi nhóm dCERT diễn ra thông qua quy trình bỏ phiếu của các thành viên nhóm dCERT. Sau khi đình chỉ này diễn ra, phải gửi một governance proposal để slash thành viên dCERT, nếu đề xuất không được phê duyệt trong thời gian thành viên bị thu hồi đã hoàn thành unbonding token của họ, thì các token không còn được stake và không thể bị slash.

Ngoài ra, trong trường hợp khẩn cấp của một nhóm dCERT thông đồng và độc hại, cộng đồng cần khả năng giải tán toàn bộ nhóm dCERT và có khả năng slash toàn bộ họ. Điều này có thể đạt được thông qua một loại đề xuất mới đặc biệt (được triển khai như governance proposal chung) sẽ tạm dừng chức năng của nhóm dCERT cho đến khi đề xuất được kết luận. Loại đề xuất đặc biệt này có thể sẽ cần có wager khá lớn có thể bị slash nếu người tạo đề xuất có ý đồ xấu. Lý do wager lớn cần được yêu cầu là vì ngay khi đề xuất được tạo, khả năng của nhóm dCERT để dừng các route message tạm thời bị đình chỉ, có nghĩa là một tác nhân độc hại tạo ra đề xuất như vậy sau đó có thể khai thác một lỗ hổng trong thời gian này mà không có nhóm dCERT có thể tắt các route message có thể bị khai thác.

### Giao Dịch Thành Viên dCERT

Thành viên dCERT đang hoạt động

* thay đổi mô tả của nhóm dCERT
* circuit break một route message
* bỏ phiếu để đình chỉ thành viên dCERT.

Ở đây circuit-breaking đề cập đến khả năng vô hiệu hóa một nhóm message. Ví dụ có thể là: "vô hiệu hóa tất cả message staking-delegation", hoặc "vô hiệu hóa tất cả message distribution". Điều này có thể được thực hiện bằng cách xác minh rằng route message chưa bị "circuit-broken" tại thời điểm CheckTx (trong `baseapp/baseapp.go`).

"Unbreaking" một circuit được dự kiến chỉ xảy ra trong quá trình nâng cấp hard fork, có nghĩa là không cần khả năng unbreak một route message trên chain đang hoạt động.

Lưu ý rằng nếu có vấn đề với bỏ phiếu quản trị (ví dụ như khả năng bỏ phiếu nhiều lần) thì quản trị sẽ bị hỏng và nên bị dừng bằng cơ chế này, sau đó sẽ do validator set phối hợp và nâng cấp hard-fork sang phiên bản đã vá của phần mềm trong đó quản trị được bật lại (và sửa). Nếu nhóm dCERT lạm dụng đặc quyền này, tất cả họ nên bị slash nặng nề.

## Trạng Thái

Đề Xuất

## Hậu Quả

### Tích Cực

* Tiềm năng giảm số lượng bên cần phối hợp trong trường hợp khẩn cấp
* Giảm khả năng tiết lộ thông tin nhạy cảm cho các bên độc hại

### Tiêu Cực

* Rủi ro tập trung hóa

### Trung Lập

## Tài Liệu Tham Khảo

  [ADR Nhóm Chuyên Biệt Hóa](./adr-007-specialization-groups.md)
