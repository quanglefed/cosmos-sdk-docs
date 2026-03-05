# Quy Trình Tạo RFC

1. Sao chép file `rfc-template.md`. Sử dụng mẫu tên file sau: `rfc-next_number-title.md`
2. Tạo Draft Pull Request nếu bạn muốn nhận phản hồi sớm.
3. Đảm bảo ngữ cảnh và giải pháp rõ ràng và được ghi chép đầy đủ.
4. Thêm một entry vào danh sách trong file [README](./README.md).
5. Tạo Pull Request để đề xuất một ADR mới.

## RFC là gì?

RFC là một loại phiên làm việc bảng trắng không đồng bộ. Nó nhằm thay thế nhu cầu của một nhóm phân tán phải tập hợp lại để đưa ra quyết định. Hiện tại, nhóm Cosmos SDK và các người đóng góp phân tán trên khắp thế giới. Nhóm tổ chức các working group để có thảo luận đồng bộ và RFC có thể được sử dụng để ghi lại thảo luận cho đối tượng rộng hơn hiểu rõ hơn về những thay đổi đang đến với phần mềm.

Sự khác biệt chính mà Cosmos SDK đang định nghĩa để phân biệt giữa RFC và ADR là một cái dùng để đạt được đồng thuận và lưu hành thông tin về thay đổi hoặc tính năng tiềm năng. ADR được sử dụng nếu đã có đồng thuận về một tính năng hoặc thay đổi và không cần phải trình bày rõ ràng về thay đổi sắp đến với phần mềm. ADR sẽ trình bày các thay đổi và có lượng giao tiếp ít hơn.

## Vòng Đời RFC

Việc tạo RFC là một quá trình **lặp đi lặp lại**. RFC được coi là một phiên hợp tác phân tán, nó có thể có nhiều bình luận và thường là sản phẩm phụ của việc không có working group hay giao tiếp đồng bộ.

1. Các đề xuất có thể bắt đầu từ một GitHub Issue mới, là kết quả của các Issue hiện có hoặc một cuộc thảo luận.

2. RFC không nhất thiết phải đến `main` với trạng thái _accepted_ trong một PR đơn lẻ. Nếu động cơ rõ ràng và giải pháp hợp lý, chúng ta NÊN có thể merge nó và giữ trạng thái _proposed_. Tốt hơn là có cách tiếp cận lặp đi lặp lại hơn là Pull Requests dài, chưa được merge.

3. Nếu một RFC _proposed_ được merge, thì nó nên ghi chép rõ ràng các vấn đề còn tồn tại trong ghi chú tài liệu RFC hoặc trong một GitHub Issue.

4. PR NÊN luôn được merge. Trong trường hợp RFC có lỗi, chúng tôi vẫn muốn merge nó với trạng thái _rejected_. Trường hợp duy nhất RFC KHÔNG NÊN được merge là nếu tác giả từ bỏ nó.

5. Các RFC đã merge KHÔNG NÊN bị xóa.

6. Nếu có đồng thuận và đủ phản hồi thì RFC có thể được chấp nhận.

> Lưu ý: RFC được viết khi không có working group hay session của nhóm về vấn đề. RFC được coi là phiên làm việc bảng trắng phân tán. Nếu có working group về đề xuất thì không cần có RFC vì đang có làm việc bảng trắng đồng bộ diễn ra.

### Trạng Thái RFC

Trạng thái có hai thành phần:

```text
{CONSENSUS STATUS}
```

#### Consensus Status (Trạng Thái Đồng Thuận)

```text
DRAFT -> PROPOSED -> LAST CALL yyyy-mm-dd -> ACCEPTED | REJECTED -> SUPERSEDED by ADR-xxx
                  \        |
                   \       |
                    v      v
                     ABANDONED
```

* `DRAFT`: [tùy chọn] một ADR đang trong quá trình phát triển, chưa sẵn sàng cho việc review chung. Đây là để trình bày công việc sớm và nhận phản hồi sớm trong dạng Draft Pull Request.
* `PROPOSED`: một ADR bao gồm kiến trúc giải pháp đầy đủ và vẫn đang trong giai đoạn review - các bên liên quan của dự án chưa đạt được thỏa thuận.
* `LAST CALL <date for the last call>`: [tùy chọn] thông báo rõ ràng rằng chúng ta đang tiến gần đến việc chấp nhận cập nhật. Thay đổi trạng thái thành `LAST CALL` có nghĩa là đã đạt được đồng thuận xã hội (của các maintainer Cosmos SDK) và chúng ta vẫn muốn dành thời gian để cộng đồng phản ứng hoặc phân tích.
* `ACCEPTED`: ADR đại diện cho thiết kế kiến trúc hiện đang được hoặc sẽ được triển khai.
* `REJECTED`: ADR có thể chuyển từ PROPOSED hoặc ACCEPTED sang rejected nếu sự đồng thuận giữa các bên liên quan của dự án quyết định như vậy.
* `SUPERSEEDED by ADR-xxx`: ADR đã được thay thế bởi một ADR mới.
* `ABANDONED`: ADR không còn được các tác giả ban đầu theo đuổi.

## Ngôn Ngữ Sử Dụng Trong RFC

* Background/mục tiêu nên được viết ở thì hiện tại.
* Tránh sử dụng dạng ngôi thứ nhất, cá nhân.
