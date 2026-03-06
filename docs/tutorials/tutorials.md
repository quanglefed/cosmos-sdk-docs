---
sidebar_position: 0
---
# Tutorials

## Advanced Tutorials

Phần này cung cấp tổng quan ngắn gọn về các hướng dẫn tập trung vào việc triển khai vote extensions trong Cosmos SDK. Vote extensions là tính năng mạnh mẽ để nâng cao bảo mật và công bằng của các ứng dụng blockchain, đặc biệt trong các kịch bản như triển khai oracle và giảm thiểu front-running đấu giá.

*   **Implementing Oracle with Vote Extensions** - Hướng dẫn này mô tả chi tiết cách sử dụng vote extensions để triển khai oracle an toàn và đáng tin cậy trong ứng dụng blockchain. Nó minh họa việc sử dụng vote extensions để đưa dữ liệu gửi từ oracle vào các block một cách an toàn, đảm bảo tính toàn vẹn và độ tin cậy của dữ liệu cho blockchain.

*   **Mitigating Auction Front-Running with Vote Extensions** - Khám phá cách ngăn chặn front-running đấu giá bằng vote extensions. Hướng dẫn này phác thảo việc tạo một module nhằm giảm thiểu front-running trong các phiên đấu giá nameservice, nhấn mạnh các hàm `ExtendVote`, `PrepareProposal`, và `ProcessProposal` để tạo điều kiện cho quy trình đấu giá công bằng.
