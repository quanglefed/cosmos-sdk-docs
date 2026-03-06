# ADR 43: Module NFT

## Changelog

* 2021-05-01: Bản nháp đầu tiên
* 2021-07-02: Cập nhật từ review
* 2022-06-15: Thêm thao tác batch
* 2022-11-11: Bỏ xác thực nghiêm ngặt classID và tokenID

## Trạng Thái

ĐỀ XUẤT

## Tóm Tắt

ADR này định nghĩa module `x/nft`, là một triển khai NFT chung, tương thích với ERC721. **Các ứng dụng sử dụng module `x/nft` phải triển khai các hàm sau**:

* `MsgNewClass` - Nhận yêu cầu của người dùng để tạo một class, và gọi `NewClass` của module `x/nft`.
* `MsgUpdateClass` - Nhận yêu cầu của người dùng để cập nhật một class, và gọi `UpdateClass` của module `x/nft`.
* `MsgMintNFT` - Nhận yêu cầu của người dùng để mint một nft, và gọi `MintNFT` của module `x/nft`.
* `BurnNFT` - Nhận yêu cầu của người dùng để burn một nft, và gọi `BurnNFT` của module `x/nft`.
* `UpdateNFT` - Nhận yêu cầu của người dùng để cập nhật một nft, và gọi `UpdateNFT` của module `x/nft`.

## Bối Cảnh

NFT không chỉ là crypto art, điều này rất hữu ích để tích lũy giá trị cho hệ sinh thái Cosmos. Kết quả là, Cosmos Hub nên triển khai các chức năng NFT và kích hoạt một cơ chế thống nhất để lưu trữ và gửi đại diện quyền sở hữu của NFT như đã thảo luận tại https://github.com/cosmos/cosmos-sdk/discussions/9065.

Như đã thảo luận trong [#9065](https://github.com/cosmos/cosmos-sdk/discussions/9065), có thể xem xét một số giải pháp tiềm năng:

* irismod/nft và modules/incubator/nft
* CW721
* DID NFTs
* interNFT

Vì các chức năng/use case của NFT gắn chặt với logic của chúng, gần như không thể hỗ trợ tất cả các use case NFT trong một module Cosmos SDK bằng cách định nghĩa và triển khai các loại giao dịch khác nhau.

Xem xét tính sử dụng chung và khả năng tương thích của các giao thức interchain bao gồm IBC và Gravity Bridge, nên ưu tiên thiết kế module NFT chung xử lý logic NFT chung. Ý tưởng thiết kế này có thể kích hoạt khả năng kết hợp, nơi các chức năng dành riêng cho ứng dụng nên được quản lý bởi các module khác trên Cosmos Hub hoặc trên các Zone khác bằng cách nhập module NFT.

Thiết kế hiện tại dựa trên công việc được thực hiện bởi [nhóm IRISnet](https://github.com/irisnet/irismod/tree/master/modules/nft) và triển khai cũ hơn trong [kho lưu trữ Cosmos](https://github.com/cosmos/modules/tree/master/incubator/nft).

## Quyết Định

Chúng ta tạo một module `x/nft`, chứa các chức năng sau:

* Lưu trữ NFT và theo dõi quyền sở hữu của chúng.
* Phát lộ interface `Keeper` cho các module kết hợp để chuyển, mint và burn NFT.
* Phát lộ interface `Message` bên ngoài cho người dùng để chuyển quyền sở hữu NFT của họ.
* Truy vấn NFT và thông tin về tổng cung.

Module được đề xuất là module cơ sở cho logic NFT của ứng dụng. Mục tiêu của nó là cung cấp một lớp chung cho lưu trữ, chức năng chuyển giao cơ bản và IBC. Module không nên được sử dụng như một standalone. Thay vào đó, ứng dụng nên tạo một module chuyên biệt để xử lý logic dành riêng cho ứng dụng (ví dụ: xây dựng NFT ID, royalty), mint và burn ở cấp độ người dùng. Hơn nữa, một module chuyên biệt của ứng dụng nên xử lý dữ liệu phụ trợ để hỗ trợ logic ứng dụng (ví dụ: index, ORM, dữ liệu business).

Tất cả dữ liệu mang qua IBC phải là một phần của kiểu `NFT` hoặc `Class` được mô tả dưới đây. Dữ liệu NFT dành riêng cho ứng dụng nên được mã hóa trong `NFT.data` để đảm bảo tính toàn vẹn cross-chain. Các đối tượng khác liên quan đến NFT, không quan trọng cho tính toàn vẹn, có thể là một phần của module dành riêng cho ứng dụng.

### Các Kiểu Dữ Liệu

Chúng tôi đề xuất hai kiểu chính:

* `Class` -- mô tả NFT class. Chúng ta có thể coi nó như một địa chỉ smart contract.
* `NFT` -- đối tượng đại diện cho tài sản độc đáo, không thể thay thế. Mỗi NFT được liên kết với một Class.

#### Class

**Class** NFT tương đương với một smart contract ERC-721 (cung cấp mô tả về một smart contract), trong đó có thể tạo và quản lý một bộ sưu tập NFT.

```protobuf
message Class {
  string id          = 1;
  string name        = 2;
  string symbol      = 3;
  string description = 4;
  string uri         = 5;
  string uri_hash    = 6;
  google.protobuf.Any data = 7;
}
```

* `id` được dùng làm chỉ mục chính để lưu trữ class; _bắt buộc_
* `name` là tên mô tả của NFT class; _tùy chọn_
* `symbol` là ký hiệu thường hiển thị trên các sàn cho NFT class; _tùy chọn_
* `description` là mô tả chi tiết của NFT class; _tùy chọn_
* `uri` là URI cho metadata của class được lưu trữ off chain. Nên là file JSON chứa metadata về NFT class và schema dữ liệu NFT ([ví dụ OpenSea](https://docs.opensea.io/docs/contract-level-metadata)); _tùy chọn_
* `uri_hash` là hash của tài liệu được trỏ bởi uri; _tùy chọn_
* `data` là metadata dành riêng cho ứng dụng của class; _tùy chọn_

#### NFT

Chúng tôi định nghĩa một mô hình chung cho `NFT` như sau.

```protobuf
message NFT {
  string class_id           = 1;
  string id                 = 2;
  string uri                = 3;
  string uri_hash           = 4;
  google.protobuf.Any data  = 10;
}
```

* `class_id` là định danh của NFT class mà NFT thuộc về; _bắt buộc_
* `id` là định danh của NFT, duy nhất trong phạm vi class của nó. Nó được người tạo NFT chỉ định và có thể được mở rộng để sử dụng DID trong tương lai. `class_id` kết hợp với `id` định danh duy nhất một NFT và được dùng làm chỉ mục chính để lưu trữ NFT; _bắt buộc_

  ```text
  {class_id}/{id} --> NFT (bytes)
  ```

* `uri` là URI cho metadata NFT được lưu trữ off chain. Nên trỏ đến một file JSON chứa metadata về NFT này (Tham chiếu: [tiêu chuẩn ERC721 và phần mở rộng OpenSea](https://docs.opensea.io/docs/metadata-standards)); _bắt buộc_
* `uri_hash` là hash của tài liệu được trỏ bởi uri; _tùy chọn_
* `data` là dữ liệu dành riêng cho ứng dụng của NFT. CÓ THỂ được sử dụng bởi các module kết hợp để chỉ định các thuộc tính bổ sung của NFT; _tùy chọn_

ADR này không chỉ định các giá trị mà `data` có thể có; tuy nhiên, thực hành tốt nhất khuyến nghị các module NFT cấp trên chỉ định rõ ràng nội dung của chúng. Mặc dù giá trị của trường này không cung cấp ngữ cảnh bổ sung cần thiết để quản lý các bản ghi NFT, có nghĩa là trường này về mặt kỹ thuật có thể được xóa khỏi đặc tả, sự tồn tại của trường cho phép các chức năng thông tin/UI cơ bản.

### Interface `Keeper`

```go
type Keeper interface {
  NewClass(ctx sdk.Context,class Class)
  UpdateClass(ctx sdk.Context,class Class)

  Mint(ctx sdk.Context,nft NFT，receiver sdk.AccAddress)   // cập nhật totalSupply
  BatchMint(ctx sdk.Context, tokens []NFT,receiver sdk.AccAddress) error

  Burn(ctx sdk.Context, classId string, nftId string)    // cập nhật totalSupply
  BatchBurn(ctx sdk.Context, classID string, nftIDs []string) error

  Update(ctx sdk.Context, nft NFT)
  BatchUpdate(ctx sdk.Context, tokens []NFT) error

  Transfer(ctx sdk.Context, classId string, nftId string, receiver sdk.AccAddress)
  BatchTransfer(ctx sdk.Context, classID string, nftIDs []string, receiver sdk.AccAddress) error

  GetClass(ctx sdk.Context, classId string) Class
  GetClasses(ctx sdk.Context) []Class

  GetNFT(ctx sdk.Context, classId string, nftId string) NFT
  GetNFTsOfClassByOwner(ctx sdk.Context, classId string, owner sdk.AccAddress) []NFT
  GetNFTsOfClass(ctx sdk.Context, classId string) []NFT

  GetOwner(ctx sdk.Context, classId string, nftId string) sdk.AccAddress
  GetBalance(ctx sdk.Context, classId string, owner sdk.AccAddress) uint64
  GetTotalSupply(ctx sdk.Context, classId string) uint64
}
```

Các triển khai logic business khác nên được định nghĩa trong các module kết hợp nhập `x/nft` và sử dụng `Keeper` của nó.

### Dịch Vụ `Msg`

```protobuf
service Msg {
  rpc Send(MsgSend)         returns (MsgSendResponse);
}

message MsgSend {
  string class_id = 1;
  string id       = 2;
  string sender   = 3;
  string receiver = 4;
}
message MsgSendResponse {}
```

`MsgSend` có thể được sử dụng để chuyển quyền sở hữu của một NFT sang địa chỉ khác.

Phác thảo triển khai của server như sau:

```go
type msgServer struct{
  k Keeper
}

func (m msgServer) Send(ctx context.Context, msg *types.MsgSend) (*types.MsgSendResponse, error) {
  // kiểm tra quyền sở hữu hiện tại
  assertEqual(msg.Sender, m.k.GetOwner(msg.ClassId, msg.Id))

  // chuyển quyền sở hữu
  m.k.Transfer(msg.ClassId, msg.Id, msg.Receiver)

  return &types.MsgSendResponse{}, nil
}
```

Các phương thức dịch vụ query cho module `x/nft` là:

```protobuf
service Query {
  // Balance truy vấn số lượng NFT của một class nhất định thuộc sở hữu của owner, tương tự balanceOf trong ERC721
  rpc Balance(QueryBalanceRequest) returns (QueryBalanceResponse) {
    option (google.api.http).get = "/cosmos/nft/v1beta1/balance/{owner}/{class_id}";
  }

  // Owner truy vấn chủ sở hữu của NFT dựa trên class và id của nó, tương tự ownerOf trong ERC721
  rpc Owner(QueryOwnerRequest) returns (QueryOwnerResponse) {
    option (google.api.http).get = "/cosmos/nft/v1beta1/owner/{class_id}/{id}";
  }

  // Supply truy vấn số lượng NFT từ class nhất định, tương tự totalSupply của ERC721.
  rpc Supply(QuerySupplyRequest) returns (QuerySupplyResponse) {
    option (google.api.http).get = "/cosmos/nft/v1beta1/supply/{class_id}";
  }

  // NFTs truy vấn tất cả NFT của một class hoặc owner nhất định, chọn ít nhất một trong hai, tương tự tokenByIndex trong ERC721Enumerable
  rpc NFTs(QueryNFTsRequest) returns (QueryNFTsResponse) {
    option (google.api.http).get = "/cosmos/nft/v1beta1/nfts";
  }

  // NFT truy vấn một NFT dựa trên class và id của nó.
  rpc NFT(QueryNFTRequest) returns (QueryNFTResponse) {
    option (google.api.http).get = "/cosmos/nft/v1beta1/nfts/{class_id}/{id}";
  }

  // Class truy vấn một NFT class dựa trên id của nó
  rpc Class(QueryClassRequest) returns (QueryClassResponse) {
    option (google.api.http).get = "/cosmos/nft/v1beta1/classes/{class_id}";
  }

  // Classes truy vấn tất cả NFT class
  rpc Classes(QueryClassesRequest) returns (QueryClassesResponse) {
    option (google.api.http).get = "/cosmos/nft/v1beta1/classes";
  }
}
```

### Khả Năng Tương Tác

Khả năng tương tác là về việc tái sử dụng tài sản giữa các module và chain. Cái trước đạt được bởi ADR-33: Giao tiếp client-server Protobuf. Tại thời điểm viết ADR-33 chưa được hoàn thiện. Cái sau đạt được bởi IBC. Ở đây chúng ta sẽ tập trung vào phía IBC.

IBC được triển khai theo module. Ở đây, chúng ta đồng ý rằng NFT sẽ được ghi lại và quản lý trong x/nft. Điều này đòi hỏi việc tạo một tiêu chuẩn IBC mới và triển khai nó.

Để có khả năng tương tác IBC, các module NFT tùy chỉnh PHẢI sử dụng kiểu đối tượng NFT được hiểu bởi IBC client. Vì vậy, để có khả năng tương tác x/nft, các triển khai NFT tùy chỉnh (ví dụ: x/cryptokitty) nên sử dụng module x/nft canonical và proxy tất cả chức năng quản lý số dư NFT sang x/nft hoặc triển khai lại tất cả chức năng sử dụng kiểu đối tượng NFT được hiểu bởi IBC client. Nói cách khác: x/nft trở thành registry NFT tiêu chuẩn cho tất cả Cosmos NFT (ví dụ: x/cryptokitty sẽ đăng ký một kitty NFT trong x/nft và sử dụng x/nft để book keeping).

## Hậu Quả

### Tương Thích Ngược

Không có sự không tương thích ngược.

### Tương Thích Tiến

Đặc tả này tuân thủ đặc tả smart contract ERC-721 cho định danh NFT.

### Tích Cực

* Định danh NFT có sẵn trên Cosmos Hub.
* Khả năng xây dựng các module NFT khác nhau cho Cosmos Hub, ví dụ ERC-721.
* Module NFT hỗ trợ khả năng tương tác với IBC và các cơ sở hạ tầng cross-chain khác như Gravity Bridge.

### Tiêu Cực

* Cần ứng dụng IBC mới cho x/nft.
* Cần adapter CW721.

### Trung Lập

* Các chức năng khác cần nhiều module hơn. Ví dụ, cần module custody cho chức năng giao dịch NFT, cần module collectible để định nghĩa các thuộc tính NFT.

## Thảo Luận Thêm

Đối với các loại ứng dụng khác trên Hub, có thể phát triển thêm nhiều module dành riêng cho ứng dụng trong tương lai:

* `x/nft/custody`: lưu ký NFT để hỗ trợ chức năng giao dịch.
* `x/nft/marketplace`: bán và mua NFT sử dụng sdk.Coins.
* `x/fractional`: module để chia quyền sở hữu tài sản (NFT hoặc tài sản khác) cho nhiều stakeholder. `x/group` nên phù hợp cho hầu hết các trường hợp.

## Tài Liệu Tham Khảo

* Thảo luận ban đầu: https://github.com/cosmos/cosmos-sdk/discussions/9065
* x/nft: khởi tạo module: https://github.com/cosmos/cosmos-sdk/pull/9174
* [ADR 033](https://github.com/cosmos/cosmos-sdk/blob/main/docs/architecture/adr-033-protobuf-inter-module-comm.md)
