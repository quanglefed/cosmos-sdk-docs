---
sidebar_position: 1
---

# `x/nft`

⚠️ **DEPRECATED**: Gói này đã bị deprecate và sẽ bị loại bỏ trong bản phát hành major tiếp theo. Module `x/nft` sẽ được chuyển sang repo riêng `github.com/cosmos/cosmos-sdk-legacy`.

## Tóm tắt

`x/nft` là một triển khai module Cosmos SDK theo [ADR 43](https://github.com/cosmos/cosmos-sdk/blob/main/docs/architecture/adr-043-nft-module.md), cho phép bạn tạo phân loại NFT (nft classification), tạo NFT, chuyển NFT, cập nhật NFT, và hỗ trợ nhiều truy vấn khác nhau bằng cách tích hợp module. Module tương thích hoàn toàn với đặc tả ERC721.

## Nội dung

* [Khái niệm](#khái-niệm)
  * [Class](#class)
  * [NFT](#nft)
* [State](#state)
  * [Class](#class-1)
  * [NFT](#nft-1)
  * [NFTOfClassByOwner](#nftofclassbyowner)
  * [Owner](#owner)
  * [TotalSupply](#totalsupply)
* [Messages](#messages)
  * [MsgSend](#msgsend)
* [Events](#events)

## Khái niệm

### Class

Module `x/nft` định nghĩa struct `Class` để mô tả các đặc tính chung của một lớp NFT.
Trong một class, bạn có thể tạo nhiều NFT khác nhau, tương đương với một hợp đồng
ERC721 trên Ethereum. Thiết kế được định nghĩa trong [ADR 043](https://github.com/cosmos/cosmos-sdk/blob/main/docs/architecture/adr-043-nft-module.md).

### NFT

Tên đầy đủ của NFT là Non-Fungible Tokens. Do tính không thể thay thế của NFT,
nó có thể được dùng để đại diện cho các vật/thực thể độc nhất. NFT được triển khai
bởi module này tương thích hoàn toàn với chuẩn ERC721 của Ethereum.

## State

### Class

Class chủ yếu gồm `id`, `name`, `symbol`, `description`, `uri`, `uri_hash`, `data`,
trong đó `id` là định danh duy nhất của class, tương tự địa chỉ hợp đồng ERC721
trên Ethereum; các trường còn lại là tuỳ chọn.

* Class: `0x01 | classID | -> ProtocolBuffer(Class)`

### NFT

NFT chủ yếu gồm `class_id`, `id`, `uri`, `uri_hash` và `data`. Trong đó `class_id`
và `id` là cặp (two-tuple) xác định tính duy nhất của NFT; `uri` và `uri_hash` là
tuỳ chọn, dùng để chỉ ra vị trí lưu trữ off-chain của NFT; và `data` là kiểu Any.
Các chain dùng `x/nft` có thể tuỳ biến bằng cách mở rộng trường này.

* NFT: `0x02 | classID | 0x00 | nftID |-> ProtocolBuffer(NFT)`

### NFTOfClassByOwner

NFTOfClassByOwner chủ yếu nhằm triển khai truy vấn tất cả NFT theo classID và owner,
không có chức năng dư thừa khác.

* NFTOfClassByOwner: `0x03 | owner | 0x00 | classID | 0x00 | nftID |-> 0x01`

### Owner

Vì trong NFT không có field bổ sung để chỉ ra owner, một cặp key-value bổ sung
được dùng để lưu quyền sở hữu NFT. Khi NFT được chuyển, cặp key-value này được
cập nhật đồng bộ.

* OwnerKey: `0x04 | classID | 0x00  | nftID |-> owner`

### TotalSupply

TotalSupply chịu trách nhiệm theo dõi số lượng tất cả NFT trong một class cụ thể.
Khi mint, supply tăng 1; khi burn, supply giảm 1.

* OwnerKey: `0x05 | classID |-> totalSupply`

## Messages

Phần này mô tả xử lý message cho module NFT.

:::warning
Việc validate `ClassID` và `NftID` để lại cho app developer.
SDK không cung cấp bất kỳ validate nào cho các field này.
:::

### MsgSend

Bạn có thể dùng message `MsgSend` để chuyển quyền sở hữu NFT. Đây là chức năng do
module `x/nft` cung cấp. Tất nhiên, bạn có thể dùng phương thức `Transfer` để tự
triển khai logic chuyển nhượng của riêng mình, nhưng bạn cần chú ý thêm về quyền
chuyển nhượng (transfer permissions).

Xử lý message sẽ thất bại nếu:

* `ClassID` được cung cấp không tồn tại.
* `Id` được cung cấp không tồn tại.
* `Sender` được cung cấp không phải chủ sở hữu NFT.

## Events

Module nft phát ra các proto event được định nghĩa trong [tham chiếu Protobuf](https://buf.build/cosmos/cosmos-sdk/docs/main:cosmos.nft.v1beta1).

