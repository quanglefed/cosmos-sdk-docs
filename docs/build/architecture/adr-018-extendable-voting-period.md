# ADR 18: Giai Đoạn Bỏ Phiếu Có Thể Mở Rộng

## Changelog

* 1 tháng 1 năm 2020: Bắt đầu phiên bản đầu tiên

## Bối Cảnh

Hiện tại, giai đoạn bỏ phiếu cho tất cả các governance proposal là như nhau. Tuy nhiên, điều này không tối ưu vì không phải tất cả governance proposal đều cần cùng khoảng thời gian. Đối với các đề xuất ít gây tranh cãi hơn, chúng có thể được xử lý hiệu quả hơn với giai đoạn nhanh hơn, trong khi các đề xuất gây tranh cãi hoặc phức tạp hơn có thể cần giai đoạn dài hơn để thảo luận/xem xét mở rộng.

## Quyết Định

Chúng ta muốn thiết kế một cơ chế để làm cho giai đoạn bỏ phiếu của governance proposal có thể thay đổi dựa trên nhu cầu của người bỏ phiếu. Chúng ta muốn nó dựa trên quan điểm của những người tham gia quản trị, chứ không chỉ người đề xuất governance proposal (do đó, cho phép người đề xuất chọn độ dài giai đoạn bỏ phiếu là không đủ).

Tuy nhiên, chúng ta muốn tránh tạo ra toàn bộ quy trình bỏ phiếu thứ hai để xác định độ dài giai đoạn bỏ phiếu, vì nó chỉ đẩy vấn đề sang việc xác định độ dài giai đoạn bỏ phiếu đầu tiên đó.

Do đó, chúng tôi đề xuất cơ chế sau:

### Tham Số

* Tham số gov hiện tại `VotingPeriod` được thay thế bằng tham số `MinVotingPeriod`. Đây là giai đoạn bỏ phiếu mặc định mà tất cả giai đoạn bỏ phiếu governance proposal bắt đầu.
* Có một tham số gov mới gọi là `MaxVotingPeriodExtension`.

### Cơ Chế

Có một kiểu `Msg` mới gọi là `MsgExtendVotingPeriod`, có thể được gửi bởi bất kỳ tài khoản có stake nào trong giai đoạn bỏ phiếu của một đề xuất. Nó cho phép người gửi đơn phương mở rộng độ dài giai đoạn bỏ phiếu thêm `MaxVotingPeriodExtension * phần chia sẻ quyền bỏ phiếu của người gửi`. Mỗi địa chỉ chỉ có thể gọi `MsgExtendVotingPeriod` một lần mỗi đề xuất.

Ví dụ, nếu `MaxVotingPeriodExtension` được đặt thành 100 ngày, thì bất kỳ ai có 1% quyền bỏ phiếu có thể mở rộng quyền bỏ phiếu thêm 1 ngày. Nếu 33% quyền bỏ phiếu đã gửi message, giai đoạn bỏ phiếu sẽ được mở rộng thêm 33 ngày. Do đó, nếu hoàn toàn mọi người chọn mở rộng giai đoạn bỏ phiếu, giai đoạn bỏ phiếu tối đa tuyệt đối sẽ là `MinVotingPeriod + MaxVotingPeriodExtension`.

Hệ thống này hoạt động như một loại phối hợp phân tán, trong đó các stakeholder cá nhân chọn mở rộng hay không cho phép hệ thống đánh giá mức độ gây tranh cãi/phức tạp của đề xuất. Rất không thể nhiều stakeholder sẽ chọn mở rộng vào đúng cùng một thời điểm, nó cho phép stakeholder xem người khác đã mở rộng bao lâu cho đến nay, để quyết định có nên mở rộng thêm hay không.

### Xử Lý Unbonding/Redelegation

Có một điều cần được giải quyết. Cách xử lý redelegation/unbonding trong giai đoạn bỏ phiếu. Nếu một stakeholder 5% gọi `MsgExtendVotingPeriod` và sau đó unbond, giai đoạn bỏ phiếu có giảm lại 5 ngày không? Điều này không tốt vì nó có thể khiến mọi người có cảm giác sai về thời gian họ có để đưa ra quyết định. Vì lý do này, chúng ta muốn thiết kế sao cho độ dài giai đoạn bỏ phiếu chỉ có thể được mở rộng, không thể rút ngắn. Để làm điều này, lượng mở rộng hiện tại dựa trên tỷ lệ phần trăm cao nhất đã bỏ phiếu mở rộng vào bất kỳ thời điểm nào. Điều này được giải thích rõ nhất bằng ví dụ:

1. Giả sử 2 stakeholder có quyền bỏ phiếu 4% và 3% tương ứng bỏ phiếu mở rộng. Giai đoạn bỏ phiếu sẽ được mở rộng thêm 7 ngày.
2. Bây giờ stakeholder 3% quyết định unbond trước khi kết thúc giai đoạn bỏ phiếu. Mở rộng giai đoạn bỏ phiếu vẫn là 7 ngày.
3. Bây giờ, giả sử một stakeholder khác có 2% quyền bỏ phiếu quyết định mở rộng giai đoạn bỏ phiếu. Bây giờ có 6% quyền bỏ phiếu đang hoạt động chọn mở rộng. Quyền bỏ phiếu vẫn là 7 ngày.
4. Nếu một stakeholder thứ tư có 10% chọn mở rộng bây giờ, có tổng cộng 16% quyền bỏ phiếu đang hoạt động muốn mở rộng. Giai đoạn bỏ phiếu sẽ được mở rộng lên 16 ngày.

### Người Ủy Quyền

Giống như các phiếu bầu trong giai đoạn bỏ phiếu thực tế, người ủy quyền tự động kế thừa phần mở rộng của validator của họ. Nếu validator của họ chọn mở rộng, quyền bỏ phiếu của họ sẽ được sử dụng trong phần mở rộng của validator. Tuy nhiên, người ủy quyền không thể override validator và "unextend" vì điều đó sẽ mâu thuẫn với nguyên tắc "độ dài quyền bỏ phiếu chỉ có thể được ratchet lên" được mô tả trong phần trước. Tuy nhiên, người ủy quyền có thể chọn mở rộng bằng quyền bỏ phiếu cá nhân của họ, nếu validator của họ chưa làm vậy.

## Trạng Thái

Đề Xuất

## Hậu Quả

### Tích Cực

* Các governance proposal phức tạp/gây tranh cãi hơn sẽ có nhiều thời gian hơn để xem xét và thảo luận đúng đắn

### Tiêu Cực

* Quy trình quản trị trở nên phức tạp hơn và đòi hỏi hiểu biết sâu hơn để tương tác hiệu quả
* Không còn có thể dự đoán khi nào governance proposal sẽ kết thúc. Không thể giả định thứ tự kết thúc của các governance proposal.

### Trung Lập

* Giai đoạn bỏ phiếu tối thiểu có thể được rút ngắn

## Tài Liệu Tham Khảo

* [Bài viết trên Cosmos Forum nơi ý tưởng xuất hiện lần đầu](https://forum.cosmos.network/t/proposal-draft-reduce-governance-voting-period-to-7-days/3032/9)
