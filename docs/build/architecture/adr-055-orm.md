# ADR 055: ORM

## Changelog

* 2022-04-27: Bản nháp đầu tiên

## Trạng Thái

ACCEPTED Đã Triển Khai

## Tóm Tắt

Để giúp nhà phát triển dễ dàng hơn trong việc xây dựng các module Cosmos SDK và để client truy vấn, lập index và xác minh bằng chứng đối với dữ liệu state, chúng tôi đã triển khai một lớp ORM (object-relational mapping) cho Cosmos SDK.

## Bối Cảnh

Về mặt lịch sử, các module trong Cosmos SDK luôn sử dụng trực tiếp key-value store và tạo ra các hàm viết tay khác nhau để quản lý định dạng key cũng như xây dựng các secondary index. Điều này tốn nhiều thời gian khi xây dựng một module và dễ mắc lỗi. Vì các định dạng key không chuẩn, đôi khi được tài liệu hóa kém và có thể thay đổi, rất khó để client lập index chung, truy vấn và xác minh bằng chứng merkle đối với dữ liệu state.

Trường hợp đầu tiên được biết đến của "ORM" trong hệ sinh thái Cosmos là trong [weave](https://github.com/iov-one/weave/tree/master/orm). Phiên bản sau được xây dựng cho [regen-ledger](https://github.com/regen-network/regen-ledger/tree/157181f955823149e1825263a317ad8e16096da4/orm) để sử dụng trong group module và sau đó được [chuyển sang SDK](https://github.com/cosmos/cosmos-sdk/tree/35d3312c3be306591fcba39892223f1244c8d108/x/group/internal/orm) chỉ cho mục đích đó.

## Quyết Định

Các nỗ lực trước đây đã dẫn đến việc tạo ra go module `orm` của Cosmos SDK sử dụng các annotation protobuf để chỉ định định nghĩa bảng ORM. ORM này dựa trên API `google.golang.org/protobuf/reflect/protoreflect` mới và hỗ trợ:

* Sorted index cho tất cả các kiểu protobuf đơn giản (ngoại trừ `bytes`, `enum`, `float`, `double`) cũng như `Timestamp` và `Duration`
* Unsorted `bytes` và `enum` index
* Composite primary và secondary key
* Unique index
* Auto-incrementing `uint64` primary key
* Complex prefix và range query
* Paginated query
* Giải mã logic đầy đủ của dữ liệu KV-store

Gần như tất cả thông tin cần thiết để giải mã state trực tiếp được chỉ định trong các file .proto. Mỗi định nghĩa bảng chỉ định một ID duy nhất theo file .proto và mỗi index trong một bảng là duy nhất trong bảng đó.

ORM tối ưu hóa xung quanh dung lượng lưu trữ bằng cách không lặp lại các giá trị trong primary key trong key value khi lưu trữ các bản ghi primary key. Ví dụ, nếu đối tượng `{"a":0,"b":1}` có primary key `a`, nó sẽ được lưu trữ trong key value store là `Key: '0', Value: {"b":1}`.

Một code generator được bao gồm với ORM tạo ra các wrapper type-safe xung quanh triển khai `Table` động của ORM và là cách được khuyến nghị cho các module sử dụng ORM.

Các test ORM cung cấp một demo module bank đơn giản hóa minh họa:

* [Tùy chọn proto ORM](https://github.com/cosmos/cosmos-sdk/blob/0d846ae2f0424b2eb640f6679a703b52d407813d/orm/internal/testpb/bank.proto)
* [Code Được Tạo](https://github.com/cosmos/cosmos-sdk/blob/0d846ae2f0424b2eb640f6679a703b52d407813d/orm/internal/testpb/bank.cosmos_orm.go)
* [Ví Dụ Sử Dụng trong Module Keeper](https://github.com/cosmos/cosmos-sdk/blob/0d846ae2f0424b2eb640f6679a703b52d407813d/orm/model/ormdb/module_test.go)

## Hậu Quả

### Tương Thích Ngược

Code state machine áp dụng ORM sẽ cần migration vì bố cục state nói chung không tương thích ngược. Các state machine này cũng sẽ cần migrate sang https://github.com/cosmos/cosmos-proto ít nhất cho dữ liệu state.

### Tích Cực

* Dễ dàng hơn để xây dựng module.
* Dễ dàng hơn để thêm secondary index vào state.
* Có thể viết một indexer chung cho ORM state.
* Dễ dàng hơn để viết client thực hiện state proof.
* Có thể tự động viết các query layer thay vì cần triển khai gRPC query thủ công.

### Tiêu Cực

* Hiệu suất kém hơn so với các key viết tay (hiện tại).

## Thảo Luận Thêm

Các công việc đang được lên kế hoạch và tiếp tục bao gồm:

* Tự động tạo query layer phía client.
* Thư viện query phía client tự động xác minh bằng chứng light client.
* Index dữ liệu ORM vào cơ sở dữ liệu SQL.
* Cải thiện hiệu suất.

## Tài Liệu Tham Khảo

* https://github.com/iov-one/weave/tree/master/orm
* https://github.com/cosmos/cosmos-sdk/discussions/9156
* https://github.com/cosmos/cosmos-sdk/pull/10454
