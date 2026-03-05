---
sidebar_position: 1
---

# Requests for Comments (Yêu Cầu Góp Ý)

Request for Comments (RFC) là bản ghi thảo luận về một chủ đề mở liên quan đến thiết kế và triển khai của Cosmos SDK, mà không cần quyết định ngay lập tức.

Mục đích của RFC là phục vụ như một bản ghi lịch sử của cuộc thảo luận cấp cao có thể chỉ được ghi lại theo cách tạm thời (ví dụ: qua gists hoặc Google docs) khó tìm kiếm cho người xem sau này. RFC _có thể_ dẫn đến các _quyết định_ kiến trúc cụ thể hơn cho Cosmos SDK, nhưng những quyết định đó phải được ghi lại riêng trong [Architecture Decision Records (ADR)](../architecture).

Theo nguyên tắc chung, nếu bạn có thể nêu rõ một câu hỏi cụ thể cần được trả lời, hãy viết một ADR. Nếu bạn cần khám phá chủ đề và nhận đầu vào từ người khác để biết câu hỏi nào cần được trả lời, RFC có thể phù hợp.

## Nội Dung RFC

Một RFC nên cung cấp:

* **Changelog** ghi lại khi nào và như thế nào RFC đã thay đổi.
* **Abstract** tóm tắt ngắn gọn chủ đề để người đọc có thể nhanh chóng xác định nó có liên quan đến mối quan tâm của họ hay không.
* Bất kỳ **background** nào mà người đọc sẽ cần để hiểu và tham gia vào nội dung của cuộc thảo luận (liên kết đến các tài liệu khác cũng được ở đây).
* **Discussion** — nội dung chính của tài liệu.

File [rfc-template.md](./rfc-template.md) bao gồm các placeholder cho các phần này.

## Mục Lục

* [RFC-001: Tx Validation](./rfc-001-tx-validation.md)
