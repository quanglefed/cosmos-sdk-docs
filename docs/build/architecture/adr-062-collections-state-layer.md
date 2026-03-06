# ADR 062: Collections – Lớp Lưu Trữ Đơn Giản Hóa cho Cosmos SDK

## Changelog

* 30/11/2022: ĐỀ XUẤT

## Trạng Thái

ĐỀ XUẤT – Đã Triển Khai

## Tóm Tắt

Chúng tôi đề xuất một lớp lưu trữ module đơn giản hóa tận dụng golang generics để cho phép nhà phát triển module xử lý lưu trữ module theo cách đơn giản và trực tiếp, trong khi cung cấp tính an toàn, khả năng mở rộng và tiêu chuẩn hóa.

## Bối Cảnh

Nhà phát triển module bị buộc phải triển khai thủ công các chức năng lưu trữ trong module của họ, bao gồm:

* Định nghĩa định dạng key-to-bytes.
* Định nghĩa định dạng value-to-bytes.
* Định nghĩa secondary index.
* Định nghĩa phương thức query để phát lộ ra bên ngoài.
* Định nghĩa phương thức local để xử lý ghi lưu trữ.
* Xử lý genesis import và export.
* Viết test cho tất cả những điều trên.

Điều này gây ra nhiều vấn đề:

* Nó cản trở nhà phát triển tập trung vào phần quan trọng nhất: viết business logic.
* Các định dạng key-to-bytes phức tạp và dễ mắc lỗi.
* Thiếu tiêu chuẩn hóa khiến client gặp khó khăn, vấn đề càng trở nên tệ hơn khi cung cấp bằng chứng cho các đối tượng hiện diện trong state.

### Giải Pháp Hiện Tại: ORM

Giải pháp đề xuất hiện tại của SDK là [ORM](./adr-055-orm.md). Mặc dù ORM cung cấp nhiều chức năng tốt, nhưng có một số nhược điểm:

* Yêu cầu migration.
* Sử dụng API golang protobuf mới nhất, trong khi SDK vẫn chủ yếu sử dụng gogoproto.
* Có đường cong học tập cao, ngay cả đối với các lớp lưu trữ đơn giản.

### Giải Pháp CosmWasm: cw-storage-plus

Collections API lấy cảm hứng từ [cw-storage-plus](https://docs.cosmwasm.com/docs/1.0/smart-contracts/state/cw-plus/), đã chứng minh là công cụ mạnh mẽ để xử lý lưu trữ trong hợp đồng CosmWasm. Nó đơn giản, không yêu cầu công cụ bổ sung, dễ xử lý các cấu trúc lưu trữ phức tạp (index, snapshot, v.v.).

## Quyết Định

Chúng tôi đề xuất chuyển API `collections`, có triển khai trong [NibiruChain/collections](https://github.com/NibiruChain/collections) sang cosmos-sdk.

Collections triển khai bốn kiểu trình xử lý lưu trữ khác nhau:

* `Map`: xử lý ánh xạ `key=>object` đơn giản.
* `KeySet`: hoạt động như `Set` và chỉ giữ lại key (use case: allow-list).
* `Item`: luôn chỉ chứa một đối tượng (use case: Params).
* `Sequence`: triển khai một số luôn tăng đơn giản (use case: Nonces).
* `IndexedMap`: xây dựng trên `Map` và `KeySet`, cho phép tạo quan hệ với `Objects` và secondary key của `Objects`.

Collections hoàn toàn generic, có nghĩa là bất cứ thứ gì đều có thể được sử dụng làm `Key` và `Value`.

Các kiểu Collections ủy thác nhiệm vụ tuần tự hóa key và value cho thành phần API collections thứ cấp gọi là `ValueEncoders` và `KeyEncoders`.

* `ValueEncoders` chuyển đổi một value thành bytes. Collections đã cung cấp các `ValueEncoders` mặc định cho: đối tượng protobuf, các kiểu SDK đặc biệt (sdk.Int, sdk.Dec).
* `KeyEncoders` chuyển đổi key thành bytes. Collections đã cung cấp một số `KeyEncoders` mặc định cho một số kiểu golang nguyên thủy (uint64, string, time.Time, ...) và một số kiểu sdk được sử dụng rộng rãi.

Các triển khai mặc định này cũng cung cấp tính an toàn xung quanh sắp xếp lexicographic đúng và tránh xung đột namespace.

## Hậu Quả

### Tương Thích Ngược

Thiết kế `ValueEncoders` và `KeyEncoders` cho phép các module giữ nguyên cùng ánh xạ `byte(key)=>byte(value)`, làm cho việc nâng cấp lên lớp lưu trữ mới không phá vỡ state.

### Tích Cực

* ADR nhằm mục đích xóa code khỏi SDK hơn là thêm vào. Chỉ migrate `x/staking` sang collections sẽ mang lại sự giảm LOC net.
* Đơn giản hóa và tiêu chuẩn hóa các lớp lưu trữ trên các module trong SDK.
* Không yêu cầu xử lý protobuf.
* Code golang thuần túy.
* Generalization trên `KeyEncoders` và `ValueEncoders` cho phép không bị ràng buộc với framework tuần tự hóa dữ liệu.

### Tiêu Cực

* Golang generics chưa được battle-tested nhiều như các tính năng Golang khác, mặc dù đang được sử dụng trong production.
* Khởi tạo kiểu Collection cần được cải thiện.

## Thảo Luận Thêm

* Import/export genesis tự động (chưa được triển khai vì API breakage).
* Schema reflection.

## Tài Liệu Tham Khảo

* [NibiruChain/collections](https://github.com/NibiruChain/collections)
* [cw-storage-plus](https://docs.cosmwasm.com/docs/1.0/smart-contracts/state/cw-plus/)
