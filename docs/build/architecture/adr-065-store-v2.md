# ADR-065: Store V2

## Changelog

* Feb 14, 2023: Bản nháp ban đầu (@alexanderbez)

## Trạng thái

BẢN NHÁP

## Tóm tắt

Các primitive về lưu trữ và state mà các ứng dụng dựa trên Cosmos SDK sử dụng nhìn
chung gần như không thay đổi kể từ khi Cosmos Hub đầu tiên ra mắt. Tuy nhiên, các
đòi hỏi và nhu cầu của các ứng dụng dựa trên Cosmos SDK, từ cả góc nhìn UX của
developer lẫn client, đã phát triển và mở rộng vượt khỏi hệ sinh thái kể từ khi
các primitive này được giới thiệu lần đầu.

Theo thời gian, khi các ứng dụng này được chấp nhận rộng rãi, nhiều thiếu sót và
khiếm khuyết nghiêm trọng đã bị bộc lộ trong các primitive về state và lưu trữ
của Cosmos SDK.

Để theo kịp nhu cầu đang tiến hoá của cả client lẫn developer, cần một cuộc đại
tu (overhaul) lớn đối với các primitive này.

## Bối cảnh

Cosmos SDK cung cấp cho developer ứng dụng nhiều primitive lưu trữ khác nhau để
làm việc với application state. Cụ thể, mỗi module có cấu trúc dữ liệu cam kết
Merkle riêng — một cây IAVL. Trong cấu trúc dữ liệu này, một module có thể lưu và
lấy các cặp khoá-giá trị (key-value) kèm các cam kết Merkle, tức các bằng chứng
(proofs), cho các cặp khoá-giá trị đó để chỉ ra rằng chúng có hoặc không tồn tại
trong application state toàn cục. Cấu trúc dữ liệu này là lớp nền tảng `KVStore`.

Ngoài ra, SDK cung cấp các abstraction trên đỉnh cấu trúc dữ liệu Merkle này. Cụ thể,
root multi-store (RMS) là một tập hợp các `KVStore` của mỗi module. Thông qua RMS,
ứng dụng có thể phục vụ truy vấn và cung cấp proof cho client, đồng thời cung cấp
cho module quyền truy cập `KVStore` riêng của nó thông qua `StoreKey`, một primitive OCAP.

Có thêm các lớp abstraction nằm giữa RMS và `KVStore` IAVL bên dưới. `GasKVStore`
chịu trách nhiệm theo dõi mức tiêu thụ gas cho IO (đọc/ghi) của state machine.
`CacheKVStore` chịu trách nhiệm cung cấp cơ chế cache cho đọc và buffer cho ghi
để các chuyển đổi state mang tính nguyên tử (atomic), ví dụ khi thực thi giao dịch
hoặc thực thi đề xuất governance.

Có một vài nhược điểm then chốt trong các lớp abstraction này và trong thiết kế
chung của storage trong Cosmos SDK:

* Vì mỗi module có `KVStore` IAVL riêng, các cam kết không mang tính [nguyên tử](https://github.com/cosmos/cosmos-sdk/issues/14625)
  * Lưu ý, ta vẫn có thể cho phép module có `KVStore` IAVL riêng, nhưng thư viện
    IAVL sẽ cần hỗ trợ khả năng truyền một instance DB như một tham số cho nhiều API IAVL.
* Vì IAVL chịu trách nhiệm cả lưu trữ state lẫn cam kết, việc chạy một archive node
  ngày càng trở nên đắt đỏ khi dung lượng đĩa tăng theo cấp số nhân.
* Khi kích thước mạng tăng, nhiều nút thắt hiệu năng bắt đầu xuất hiện ở nhiều khu vực
  như hiệu năng truy vấn, nâng cấp mạng, migration state, và hiệu năng tổng thể của ứng dụng.
* UX cho developer kém vì không cho phép developer thử nghiệm với các cách tiếp cận
  khác nhau cho lưu trữ và cam kết, đồng thời bị phức tạp hoá bởi nhiều tầng abstraction
  như đã nêu.

Xem [Storage Discussion](https://github.com/cosmos/cosmos-sdk/discussions/13545) để biết thêm thông tin.

## Phương án thay thế

Đã có một nỗ lực trước đó để refactor tầng storage được mô tả trong [ADR-040](./adr-040-storage-and-smt-state-commitments.md).
Tuy nhiên, cách tiếp cận này chủ yếu xuất phát từ các thiếu sót của IAVL và các
vấn đề hiệu năng xung quanh nó. Dù đã có một triển khai (một phần) của
[ADR-040](./adr-040-storage-and-smt-state-commitments.md), nó chưa bao giờ được
áp dụng vì nhiều lý do, chẳng hạn phụ thuộc vào SMT (vốn còn ở pha nghiên cứu),
và một số lựa chọn thiết kế không thể đạt đồng thuận, ví dụ cơ chế snapshotting
dẫn tới bùng nổ state (state bloat) rất lớn.

## Quyết định

Chúng ta đề xuất xây dựng dựa trên một số ý tưởng tốt đã được giới thiệu trong
[ADR-040](./adr-040-storage-and-smt-state-commitments.md), nhưng linh hoạt hơn
với các triển khai nền tảng và ít “xâm lấn” hơn về tổng thể. Cụ thể, chúng ta đề xuất:

* Tách biệt mối quan tâm giữa cam kết state (**SC**) — cần cho đồng thuận, và
  lưu trữ state (**SS**) — cần cho state machine và client.
* Giảm số lớp abstraction cần thiết giữa RMS và các store nền tảng.
* Cung cấp cam kết store theo module mang tính nguyên tử bằng cách cung cấp một
  đối tượng DB batch cho các API IAVL cốt lõi.
* Giảm độ phức tạp trong triển khai `CacheKVStore` đồng thời cải thiện hiệu năng<sup>[3]</sup>.

Hơn nữa, trong thời gian hiện tại chúng ta vẫn giữ IAVL làm store [commitment](https://cryptography.fandom.com/wiki/Commitment_scheme)
nền tảng. Dù có thể về dài hạn ta chưa hoàn toàn “chốt” việc dùng IAVL, hiện chưa
có bằng chứng thực nghiệm mạnh mẽ để cho thấy một lựa chọn tốt hơn. Vì SDK cung
cấp các interface cho store, điều này đủ để thay đổi backing commitment store
trong tương lai nếu có bằng chứng biện minh cho một lựa chọn tốt hơn. Tuy nhiên,
đang có những công việc hứa hẹn được thực hiện cho IAVL có thể mang lại cải thiện
hiệu năng đáng kể <sup>[1,2]</sup>.

### Tách SS và SC

Bằng cách tách SS và SC, chúng ta có thể tối ưu theo các use-case và access pattern
chính để truy cập state. Cụ thể, lớp SS sẽ chịu trách nhiệm truy cập trực tiếp
dữ liệu dưới dạng cặp (key, value), trong khi lớp SC (IAVL) sẽ chịu trách nhiệm
cam kết dữ liệu và cung cấp Merkle proof.

Lưu ý, cơ sở dữ liệu lưu trữ vật lý (physical storage DB) nền tảng sẽ là cùng
một DB cho cả hai lớp SS và SC. Vì vậy để tránh xung đột giữa các cặp (key, value),
cả hai lớp sẽ được phân không gian tên (namespaced).

#### Cam kết state (SC)

Vì giải pháp hiện tại vừa đóng vai trò SS vừa đóng vai trò SC, chúng ta có thể
tái sử dụng nó để chỉ đóng vai trò lớp SC mà không cần thay đổi đáng kể về access
pattern hay hành vi. Nói cách khác, toàn bộ tập hợp `KVStore` theo module dựa
trên IAVL hiện có sẽ đóng vai trò lớp SC.

Tuy nhiên, để lớp SC vẫn “nhẹ” và không nhân đôi phần lớn dữ liệu nằm trong lớp SS,
chúng ta khuyến nghị operator chạy node duy trì chiến lược pruning chặt.

#### Lưu trữ state (SS)

Trong RMS, chúng ta sẽ expose một `KVStore` *duy nhất* được backing bởi cùng physical
DB mà lớp SC dùng. `KVStore` này sẽ được namespaced rõ ràng để tránh xung đột và sẽ
đóng vai trò lưu trữ chính cho các cặp (key, value).

Trong khi có khả năng chúng ta vẫn tiếp tục dùng `cosmos-db` hoặc một interface
local nào đó để linh hoạt và có thể lặp (iterate) về các backend lưu trữ vật lý
ưa thích khi nghiên cứu và benchmark tiếp tục. Tuy nhiên, chúng ta đề xuất hardcode
việc dùng RocksDB làm backend lưu trữ vật lý chính.

Vì lớp SS sẽ được triển khai như một `KVStore`, nó sẽ hỗ trợ chức năng sau:

* Range query
* Thao tác CRUD
* Truy vấn lịch sử và versioning
* Pruning

RMS sẽ theo dõi tất cả buffered writes bằng một `MemoryListener` nội bộ chuyên
dụng cho mỗi `StoreKey`. Với mỗi block height, khi `Commit`, lớp SS sẽ ghi tất
cả các cặp (key, value) đã buffer vào một column family theo
[RocksDB user-defined timestamp](https://github.com/facebook/rocksdb/wiki/User-defined-Timestamp-%28Experimental%29),
dùng block height làm timestamp, một số nguyên không dấu. Điều này cho phép client
fetch các cặp (key, value) ở height lịch sử và hiện tại, đồng thời giúp iteration
và range query tương đối hiệu năng vì timestamp là hậu tố khoá (key suffix).

Lưu ý, chúng ta chọn không dùng cách tiếp cận tổng quát hơn cho phép bất kỳ
embedded key/value DB nào, như LevelDB hay PebbleDB, bằng cách dùng khoá prefixed
theo height để “version hoá” state, bởi vì phần lớn các DB này dùng khoá có độ
dài biến thiên, làm cho các thao tác như iteration và range query kém hiệu năng hơn.

Vì operator có thể muốn chiến lược pruning ở SS khác với SC, ví dụ SC pruning
rất chặt trong khi SS pruning “lỏng” hơn, chúng ta đề xuất thêm một cấu hình pruning
bổ sung với các tham số giống hệt các tham số đang có trong SDK hiện tại, và cho
phép operator điều khiển chiến lược pruning của lớp SS độc lập với lớp SC.

Lưu ý, chiến lược pruning SC phải tương thích (congruent) với cấu hình state sync
của operator, để snapshot state sync có thể chạy thành công; nếu không, snapshot
có thể bị kích hoạt tại một height không còn tồn tại trong SC.

#### State Sync

Quy trình state sync nhìn chung sẽ không bị ảnh hưởng nhiều bởi việc tách lớp SC
và SS. Tuy nhiên, nếu một node đồng bộ qua state sync, lớp SS của node sẽ không có
height state đã sync sẵn, vì quy trình import IAVL không được thiết kế theo cách
cho phép chèn key/value trực tiếp một cách dễ dàng. Cần sửa đổi quy trình import
IAVL để có thể hỗ trợ việc có sẵn height state sync trong SS.

Lưu ý, điều này không gây vấn đề cho chính state machine bởi vì khi một truy vấn
được tạo, RMS sẽ tự động điều hướng truy vấn đúng (xem [Queries](#queries)).

#### Truy vấn (Queries)

Để hợp nhất việc điều hướng truy vấn giữa lớp SC và SS, chúng ta đề xuất có khái
niệm “query router” được xây dựng trong RMS. Query router này sẽ được cung cấp
cho mỗi triển khai `KVStore`. Query router sẽ điều hướng truy vấn tới lớp SC hoặc
lớp SS dựa trên một vài tham số. Nếu `prove: true`, truy vấn phải được điều hướng
tới lớp SC. Nếu không, nếu height truy vấn có sẵn trong lớp SS, truy vấn sẽ được
phục vụ từ lớp SS. Nếu không nữa, ta fallback về lớp SC.

Nếu không cung cấp height, lớp SS sẽ giả định height mới nhất. Lớp SS sẽ lưu một
reverse index để tra cứu `LatestVersion -> timestamp(version)`, được thiết lập khi `Commit`.

#### Proof

Vì lớp SS tự nhiên chỉ là lớp lưu trữ, không có cam kết cho các cặp (key, value),
nó không thể cung cấp Merkle proof cho client khi truy vấn.

Vì chiến lược pruning đối với lớp SC do operator cấu hình, RMS có thể điều hướng
truy vấn tới lớp SC nếu version tồn tại và `prove: true`. Nếu không, truy vấn sẽ
fallback về lớp SS mà không có proof.

Chúng ta có thể khám phá ý tưởng dùng snapshot state để dựng lại một cây IAVL
trong bộ nhớ theo thời gian thực dựa trên version gần nhất với version được cung
cấp trong truy vấn. Tuy nhiên, chưa rõ tác động hiệu năng của cách tiếp cận này.

### Cam kết nguyên tử (Atomic Commitment)

Chúng ta đề xuất sửa các API IAVL hiện có để chấp nhận một đối tượng DB batch
thay vì dựa vào một batch nội bộ trong `nodeDB`. Vì mỗi `KVStore` IAVL nền tảng
chia sẻ cùng một DB trong lớp SC, điều này sẽ cho phép các commit mang tính nguyên tử.

Cụ thể, chúng ta đề xuất:

* Loại bỏ field `dbm.Batch` khỏi `nodeDB`
* Cập nhật phương thức `SaveVersion` của kiểu IAVL `MutableTree` để chấp nhận một batch object
* Cập nhật phương thức `Commit` của interface `CommitKVStore` để chấp nhận một batch object
* Tạo một batch object trong RMS trong `Commit` và truyền batch này cho mỗi `KVStore`
* Ghi database batch sau khi tất cả store commit thành công

Lưu ý, điều này sẽ yêu cầu IAVL được cập nhật để không phụ thuộc hay giả định có
batch hiện diện trong `SaveVersion`.

## Hệ quả

Do có một gói store V2 mới, chúng ta kỳ vọng cải thiện hiệu năng cho truy vấn và
giao dịch nhờ tách biệt mối quan tâm. Chúng ta cũng kỳ vọng cải thiện UX cho
developer trong việc thử nghiệm các lược đồ cam kết (commitment scheme) và backend
lưu trữ để tăng hiệu năng hơn nữa, đồng thời giảm bớt mức độ abstraction quanh
KVStore để các thao tác như caching và state branching trực quan hơn.

Tuy nhiên, do thiết kế đề xuất, sẽ có nhược điểm liên quan tới việc cung cấp
proof state cho các truy vấn lịch sử.

### Tương thích ngược

ADR này đề xuất các thay đổi đối với triển khai storage trong Cosmos SDK thông
qua một package mới hoàn toàn. Các interface có thể được mượn và mở rộng từ các
kiểu hiện có trong `store`, nhưng sẽ không có triển khai hoặc interface hiện có
nào bị phá vỡ hoặc bị sửa đổi.

### Tích cực

* Cải thiện hiệu năng của hai lớp SS và SC hoạt động độc lập
* Giảm lớp abstraction giúp primitive lưu trữ dễ hiểu hơn
* Cam kết nguyên tử cho SC
* Thiết kế lại kiểu và interface lưu trữ cho phép thử nghiệm nhiều hơn, như các
  backend lưu trữ vật lý khác nhau và các commitment scheme khác nhau cho các module khác nhau

### Tiêu cực

* Việc cung cấp proof cho state lịch sử là thách thức

### Trung tính

* Giữ IAVL là cấu trúc dữ liệu cam kết chính, dù đang có những cải tiến hiệu năng mạnh

## Thảo luận thêm

### Kiểm soát lưu trữ theo module

Nhiều module lưu các chỉ mục phụ (secondary index) thường chỉ dùng để hỗ trợ truy
vấn client, nhưng thực tế không cần cho chuyển đổi state của state machine. Điều
này nghĩa là các chỉ mục này về mặt kỹ thuật không có lý do tồn tại trong lớp SC,
vì chúng chiếm không gian không cần thiết. Cần khám phá một API cho phép module
chỉ định các cặp (key, value) nào họ muốn được lưu trong lớp SC (gián tiếp chỉ
ra lớp SS), thay vì chỉ lưu cặp (key, value) trong lớp SS.

### Proof state lịch sử

Chưa rõ mức độ quan trọng hay nhu cầu trong cộng đồng đối với việc cung cấp proof
cam kết cho state lịch sử. Dù có thể nghĩ ra giải pháp như dựng lại cây theo thời
gian thực dựa trên snapshot state, nhưng chưa rõ tác động hiệu năng.

### Backend DB vật lý

ADR này đề xuất dùng RocksDB để tận dụng user-defined timestamp làm cơ chế versioning.
Tuy nhiên, có các backend DB vật lý khác có thể cung cấp cách khác để triển khai
versioning và cũng có thể cải thiện hiệu năng so với RocksDB. Ví dụ: PebbleDB cũng
hỗ trợ MVCC timestamp, nhưng cần khám phá cách PebbleDB xử lý compaction và tăng trưởng
state theo thời gian.

## Tham khảo

* [1] https://github.com/cosmos/iavl/pull/676
* [2] https://github.com/cosmos/iavl/pull/664
* [3] https://github.com/cosmos/cosmos-sdk/issues/14990

