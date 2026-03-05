---
sidebar_position: 1
---

# SDK Migrations (Di Chuyển SDK)

Để giúp việc cập nhật lên phiên bản ổn định mới nhất diễn ra thuận lợi, SDK bao gồm một lệnh CLI cho các hard-fork migration (trong lệnh con `<appd> genesis migrate`).
Ngoài ra, SDK còn bao gồm các in-place migration cho các module core của nó. Các in-place migration này hữu ích để di chuyển giữa các major release.

* Hard-fork migration được hỗ trợ từ major release cuối cùng lên phiên bản hiện tại.
* [In-place module migration](https://docs.cosmos.network/main/core/upgrade#overwriting-genesis-functions) được hỗ trợ từ hai major release cuối cùng lên phiên bản hiện tại.

Migration từ phiên bản cũ hơn hai major release cuối không được hỗ trợ.

Khi migrate từ phiên bản trước, hãy tham khảo [`UPGRADING.md`](../../../../UPGRADING.md) và `CHANGELOG.md` của phiên bản bạn đang migrate đến.
