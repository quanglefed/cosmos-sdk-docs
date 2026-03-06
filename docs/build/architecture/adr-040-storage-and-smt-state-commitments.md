# ADR 040: Cam Kết Trạng Thái Storage và SMT

## Nhật Ký Thay Đổi

* 2020-01-15: Bản nháp đầu tiên

## Trạng Thái

Đề Xuất

## Tóm Tắt

Thiết kế phân tách `Storage` (bộ nhớ trạng thái bền vững) và `Commitment` (cam kết mật mã của trạng thái). ADR này đề xuất sử dụng SMT (Sparse Merkle Tree) cho cam kết trạng thái, đồng thời sử dụng cơ sở dữ liệu riêng biệt cho bộ nhớ hiệu quả.

## Bối Cảnh

Hiện tại Cosmos SDK sử dụng IAVL (Immutable AVL Tree) cho cả lưu trữ trạng thái và cam kết trạng thái. Thiết kế này có một số hạn chế:

1. **Hiệu suất**: IAVL có hiệu suất đọc/ghi chậm hơn so với các DB thông thường do cấu trúc cây phức tạp.
2. **Không thể chứng minh**: Không thể tạo bằng chứng Merkle cho việc không tồn tại một key hiệu quả với IAVL.
3. **Tính mô đun**: Không thể dễ dàng thay đổi cấu trúc dữ liệu cơ bản mà không ảnh hưởng đến cả lưu trữ lẫn cam kết.

## Quyết Định

### Tách Biệt Bộ Nhớ và Cam Kết

Chúng ta đề xuất tách biệt lưu trữ trạng thái thành hai lớp:

1. **Storage (SC - State Commitment)**: Sử dụng Sparse Merkle Tree để tính toán root hash và tạo bằng chứng Merkle. Đây là lớp mật mã học.

2. **Storage (SS - State Storage)**: Sử dụng cơ sở dữ liệu hiệu quả (như IAVL hoặc B-tree đơn giản) cho lưu trữ và truy vấn thực tế. Đây là lớp bộ nhớ.

```
     ┌──────────────────────────────────────────┐
     │                                          │
     │       CommitMultiStore (CMS)             │
     │                                          │
     └──────────────┬───────────────────────────┘
                    │
         ┌──────────┴──────────┐
         │                     │
         ▼                     ▼
  ┌─────────────┐     ┌──────────────────┐
  │  State      │     │  State           │
  │  Storage    │     │  Commitment      │
  │  (SS)       │     │  (SC)            │
  │             │     │  SMT             │
  └─────────────┘     └──────────────────┘
```

### Sparse Merkle Tree (SMT)

SMT là một cấu trúc dữ liệu Merkle tree thưa có các thuộc tính sau:

* Tất cả các key trong không gian key (2^256 với key 256-bit) đều hợp lệ
* Chứng minh không tồn tại có kích thước cố định
* Bằng chứng Merkle là O(log n) trong không gian key, không phải số phần tử

```go
// SMT định nghĩa interface cho Sparse Merkle Tree
type SMT interface {
    // Get trả về giá trị tại key, hoặc nil nếu không tồn tại
    Get(key []byte) ([]byte, error)
    
    // Set đặt giá trị tại key
    Set(key, value []byte) error
    
    // Delete xóa giá trị tại key
    Delete(key []byte) error
    
    // Root trả về root hash hiện tại của tree
    Root() []byte
    
    // Prove tạo bằng chứng Merkle cho key
    Prove(key []byte) (Proof, error)
}
```

### CommitMultiStore Mới

`CommitMultiStore` sẽ được cập nhật để sử dụng cả SS và SC:

```go
type CommitStore interface {
    // Commit cam kết tất cả các thay đổi đang chờ và trả về CommitID
    Commit() CommitID
    
    // GetLatestVersion trả về phiên bản trạng thái mới nhất
    GetLatestVersion() int64
    
    // SS trả về State Storage
    SS() StateStorage
    
    // SC trả về State Commitment (SMT)
    SC() StateCommitment
}
```

### Định Dạng Bằng Chứng

Bằng chứng Merkle sẽ có định dạng mới tương thích với SMT:

```protobuf
message SMTProof {
    // LeafOp xác định cách hash một node lá
    LeafOp leaf = 1;
    
    // InnerOps là chuỗi thao tác để đạt tới root
    repeated InnerOp path = 2;
}

message LeafOp {
    HashOp hash = 1;
    HashOp prehash_key = 2;
    HashOp prehash_value = 3;
    LengthOp length = 4;
    bytes prefix = 5;
}
```

### Tích Hợp ICS-23

ICS-23 định nghĩa các đặc tả bằng chứng chuẩn cho các cấu trúc dữ liệu Merkle. Chúng ta sẽ triển khai một đặc tả ICS-23 cho SMT của chúng ta:

```go
var SmtSpec = &ics23.ProofSpec{
    LeafSpec: &ics23.LeafOp{
        Hash:         ics23.HashOp_SHA256,
        PrehashKey:   ics23.HashOp_NO_HASH,
        PrehashValue: ics23.HashOp_SHA256,
        Length:       ics23.LengthOp_REQUIRE_32_BYTES,
        Prefix:       []byte{0x00},
    },
    InnerSpec: &ics23.InnerSpec{
        ChildOrder:      []int32{0, 1},
        ChildSize:       32,
        MinPrefixLength: 1,
        MaxPrefixLength: 1,
        EmptyChild:      make([]byte, 32),
        Hash:            ics23.HashOp_SHA256,
    },
    MaxDepth: 64,
}
```

### Lộ Trình Di Chuyển

Để di chuyển từ IAVL sang thiết kế mới:

1. **Giai đoạn 1**: Giới thiệu các interface mới, vẫn hỗ trợ IAVL bên dưới
2. **Giai đoạn 2**: Triển khai SMT như một tùy chọn cấu hình
3. **Giai đoạn 3**: Di chuyển SS sang cơ sở dữ liệu hiệu quả hơn (như LevelDB/RocksDB trực tiếp)
4. **Giai đoạn 4**: Không dùng nữa IAVL cho các deployment mới

## Hậu Quả

### Tích Cực

* Cải thiện hiệu suất đọc/ghi bằng cách tối ưu riêng cho SS và SC
* Bằng chứng không tồn tại nhỏ gọn với SMT
* Tính mô đun cao hơn - có thể thay đổi backend lưu trữ mà không thay đổi cam kết
* Tương thích ICS-23 tốt hơn cho light clients

### Tiêu Cực

* Độ phức tạp triển khai cao hơn
* Cần công cụ di chuyển cho các chain hiện có
* Sự phụ thuộc vào việc triển khai SMT mới

### Trung Lập

* Các node validator sẽ cần thêm dung lượng bộ nhớ tạm thời trong quá trình di chuyển

## Tham Khảo

* [Thiết kế ban đầu của Celestia](https://github.com/lazyledger/smt)
* [ICS-23: Vector Commitment](https://github.com/cosmos/ics/tree/master/spec/core/ics-023-vector-commitments)
* [IAVL: Immutable AVL Tree](https://github.com/cosmos/iavl)
