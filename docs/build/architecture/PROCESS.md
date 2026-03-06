# Quy trình tạo ADR

1. Sao chép file `adr-template.md`. Sử dụng pattern tên file sau: `adr-next_number-title.md`
2. Tạo Pull Request nháp nếu bạn muốn nhận phản hồi sớm.
3. Đảm bảo bối cảnh và giải pháp rõ ràng và được ghi chép đầy đủ.
4. Thêm mục vào danh sách trong file [README](./README.md).
5. Tạo Pull Request để đề xuất ADR mới.

## ADR là gì?

ADR là tài liệu ghi chép triển khai và thiết kế có thể đã hoặc chưa được thảo luận trong RFC. Trong khi RFC nhằm thay thế giao tiếp đồng bộ trong môi trường phân tán, ADR nhằm ghi chép quyết định đã được đưa ra. ADR không đi kèm nhiều chi phí giao tiếp vì thảo luận đã được ghi chép trong RFC hoặc thảo luận đồng bộ. Nếu consensus đến từ thảo luận đồng bộ, thì nên thêm đoạn trích ngắn vào ADR để giải thích các mục tiêu.

## Vòng đời ADR

Việc tạo ADR là quy trình **lặp đi lặp lại**. Thay vì có chi phí giao tiếp cao, ADR được sử dụng khi đã có quyết định và cần thêm chi tiết triển khai. ADR nên ghi chép consensus tập thể cho vấn đề cụ thể là gì và cách giải quyết.

1. Mọi ADR nên bắt đầu bằng RFC hoặc thảo luận nơi đã đạt được consensus.

2. Khi đạt consensus, tạo GitHub Pull Request (PR) với tài liệu mới dựa trên `adr-template.md`.

3. Nếu ADR ở trạng thái _proposed_ được merge, thì nó nên ghi chép rõ ràng các vấn đề còn tồn đọng trong ghi chú tài liệu ADR hoặc trong GitHub Issue.

4. PR LUÔN NÊN được merge. Trong trường hợp ADR có lỗi, chúng ta vẫn ưu tiên merge nó với trạng thái _rejected_. Chỉ khi tác giả từ bỏ thì ADR MỚI KHÔNG NÊN được merge.

5. Các ADR đã merge KHÔNG NÊN bị xóa.

### Trạng thái ADR

Trạng thái có hai thành phần:

```text
{CONSENSUS STATUS} {IMPLEMENTATION STATUS}
```

IMPLEMENTATION STATUS là `Implemented` hoặc `Not Implemented`.

#### Consensus Status

```text
DRAFT -> PROPOSED -> LAST CALL yyyy-mm-dd -> ACCEPTED | REJECTED -> SUPERSEDED by ADR-xxx
                  \        |
                   \       |
                    v      v
                     ABANDONED
```

* `DRAFT`: [tùy chọn] ADR đang trong quá trình phát triển, chưa sẵn sàng cho đánh giá chung. Dùng để trình bày công việc sớm và nhận phản hồi sớm dưới dạng Draft Pull Request.
* `PROPOSED`: ADR bao gồm kiến trúc giải pháp đầy đủ và vẫn đang trong giai đoạn đánh giá - các bên liên quan dự án chưa đạt thỏa thuận.
* `LAST CALL <ngày cho last call>`: [tùy chọn] Thông báo rằng chúng ta sắp chấp nhận cập nhật. Thay đổi trạng thái thành `LAST CALL` có nghĩa là consensus xã hội (của các maintainer Cosmos SDK) đã đạt được, và chúng ta vẫn muốn dành thời gian để cộng đồng phản ứng hoặc phân tích.
* `ACCEPTED`: ADR đại diện cho thiết kế kiến trúc hiện đã triển khai hoặc sẽ được triển khai.
* `REJECTED`: ADR có thể chuyển từ PROPOSED hoặc ACCEPTED sang rejected nếu consensus giữa các bên liên quan dự án quyết định như vậy.
* `SUPERSEDED by ADR-xxx`: ADR đã được thay thế bởi ADR mới.
* `ABANDONED`: ADR không còn được tác giả ban đầu theo đuổi.

## Ngôn ngữ sử dụng trong ADR

* Phần bối cảnh/background nên được viết ở thì hiện tại.
* Tránh sử dụng ngôi thứ nhất.
