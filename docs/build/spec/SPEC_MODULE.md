# Đặc tả Module

File này nhằm phác thảo cấu trúc chung cho các đặc tả kỹ thuật trong thư mục này.

## Thì (Tense)

Để nhất quán, các đặc tả nên được viết ở thì hiện tại bị động (passive present tense).

## Mã giả (Pseudo-Code)

Nhìn chung, mã giả nên được giảm thiểu trong toàn bộ đặc tả. Thường thì các danh sách bullet đơn giản mô tả các thao tác của hàm là đủ và nên được ưu tiên. Trong một số trường hợp nhất định, do tính chất phức tạp của chức năng được mô tả, mã giả có thể là hình thức đặc tả phù hợp nhất. Trong các trường hợp này, việc sử dụng mã giả được phép, nhưng nên trình bày ngắn gọn, lý tưởng là chỉ giới hạn ở phần tử phức tạp như một phần của mô tả lớn hơn.

## Bố cục chung

Cấu trúc `README` tổng quát sau nên được sử dụng để phân tích đặc tả cho các module. Danh sách sau không bắt buộc và tất cả các phần đều là tùy chọn.

* `# {Tên Module}` - tổng quan về module
* `## Concepts` - mô tả các khái niệm và định nghĩa chuyên biệt được sử dụng trong đặc tả
* `## State` - chỉ định và mô tả các cấu trúc dự kiến được marshal vào store, và các key của chúng
* `## State Transitions` - các thao tác chuyển trạng thái tiêu chuẩn được kích hoạt bởi hooks, messages, v.v.
* `## Messages` - chỉ định cấu trúc message và hành vi state machine mong đợi
* `## Begin Block` - chỉ định các thao tác begin-block nếu có
* `## End Block` - chỉ định các thao tác end-block nếu có
* `## Hooks` - mô tả các hook có sẵn được gọi bởi/từ module này
* `## Events` - liệt kê và mô tả các event tag được sử dụng
* `## Client` - liệt kê và mô tả các lệnh CLI và endpoint gRPC và REST
* `## Params` - liệt kê tất cả tham số module, kiểu (trong JSON) và ví dụ
* `## Future Improvements` - mô tả các cải tiến tương lai của module này
* `## Tests` - các bài kiểm tra chấp nhận
* `## Appendix` - chi tiết bổ sung được tham chiếu ở nơi khác trong đặc tả

### Ký hiệu ánh xạ key-value

Trong `## State`, ký hiệu `->` sau nên được sử dụng để mô tả ánh xạ key sang value:

```text
key -> value
```

để biểu diễn nối byte, ký tự `|` có thể được sử dụng. Ngoài ra, kiểu encoding có thể được chỉ định, ví dụ:

```text
0x00 | addressBytes | address2Bytes -> amino(value_object)
```

Ngoài ra, ánh xạ index có thể được chỉ định bằng cách ánh xạ sang giá trị `nil`, ví dụ:

```text
0x01 | address2Bytes | addressBytes -> nil
```
