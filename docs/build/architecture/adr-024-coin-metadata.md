# ADR 024: Metadata Coin

## Nhật Ký Thay Đổi

* 19/05/2020: Bản nháp đầu tiên

## Trạng Thái

Đề Xuất

## Bối Cảnh

Các tài sản trong Cosmos SDK được biểu diễn thông qua kiểu `Coins` bao gồm `amount` và `denom`, trong đó `amount` có thể là bất kỳ giá trị lớn hoặc nhỏ tùy ý nào. Ngoài ra, Cosmos SDK sử dụng mô hình dựa trên tài khoản trong đó có hai loại tài khoản chính -- tài khoản cơ bản và tài khoản module. Tất cả các loại tài khoản có một tập hợp số dư được cấu thành từ `Coins`. Module `x/bank` theo dõi tất cả số dư cho tất cả tài khoản và cũng theo dõi tổng nguồn cung số dư trong một ứng dụng.

Đối với số lượng số dư `amount`, Cosmos SDK giả định một đơn vị mệnh giá tĩnh và cố định, bất kể chính mệnh giá đó là gì. Nói cách khác, các client và ứng dụng được xây dựng trên một chain dựa trên Cosmos SDK có thể chọn định nghĩa và sử dụng các đơn vị mệnh giá tùy ý để cung cấp UX phong phú hơn, tuy nhiên, khi một tx hoặc thao tác đến máy trạng thái Cosmos SDK, `amount` được coi là một đơn vị duy nhất. Ví dụ, đối với Cosmos Hub (Gaia), các client giả định 1 ATOM = 10^6 uatom, và vì vậy tất cả các tx và thao tác trong Cosmos SDK hoạt động theo đơn vị 10^6.

Điều này rõ ràng cung cấp UX kém và hạn chế đặc biệt khi khả năng tương tác của các mạng tăng lên và do đó tổng số loại tài sản tăng lên. Chúng tôi đề xuất việc `x/bank` cũng theo dõi thêm metadata cho mỗi `denom` để giúp các client, nhà cung cấp ví và trình khám phá cải thiện UX của họ và loại bỏ yêu cầu đưa ra bất kỳ giả định nào về đơn vị mệnh giá.

## Quyết Định

Module `x/bank` sẽ được cập nhật để lưu trữ và lập chỉ mục metadata theo `denom`, cụ thể là đơn vị "cơ sở" hoặc nhỏ nhất -- đơn vị mà máy trạng thái Cosmos SDK làm việc.

Metadata cũng có thể bao gồm một danh sách không rỗng các mệnh giá. Mỗi mục chứa tên của mệnh giá `denom`, số mũ đối với cơ sở và danh sách bí danh. Một mục được hiểu là `1 denom = 10^exponent base_denom` (vd: `1 ETH = 10^18 wei` và `1 uatom = 10^0 uatom`).

Có hai mệnh giá có tầm quan trọng cao với các client: `base`, là đơn vị nhỏ nhất có thể và `display`, là đơn vị thường được đề cập trong giao tiếp của con người và trên các sàn giao dịch. Các giá trị trong các trường đó liên kết tới một mục trong danh sách mệnh giá.

Danh sách trong `denom_units` và mục `display` có thể được thay đổi thông qua quản trị.

Kết quả là, chúng ta có thể định nghĩa kiểu như sau:

```protobuf
message DenomUnit {
  string denom    = 1;
  uint32 exponent = 2;  
  repeated string aliases = 3;
}

message Metadata {
  string description = 1;
  repeated DenomUnit denom_units = 2;
  string base = 3;
  string display = 4;
}
```

Ví dụ, metadata của ATOM có thể được định nghĩa như sau:

```json
{
  "name": "atom",
  "description": "Token staking gốc của Cosmos Hub.",
  "denom_units": [
    {
      "denom": "uatom",
      "exponent": 0,
      "aliases": [
        "microatom"
      ]
    },
    {
      "denom": "matom",
      "exponent": 3,
      "aliases": [
        "milliatom"
      ]
    },
    {
      "denom": "atom",
      "exponent": 6
    }
  ],
  "base": "uatom",
  "display": "atom"
}
```

Với metadata ở trên, một client có thể suy ra những điều sau:

* 4.3atom = 4.3 * (10^6) = 4.300.000uatom
* Chuỗi "atom" có thể được sử dụng làm tên hiển thị trong danh sách token.
* Số dư 4300000 có thể được hiển thị là 4.300.000uatom hoặc 4.300matom hoặc 4.3atom. Mệnh giá `display` 4.3atom là mặc định tốt nếu các tác giả của client không đưa ra quyết định rõ ràng để chọn một biểu diễn khác.

Một client nên có khả năng truy vấn metadata theo denom thông qua cả giao diện CLI và REST. Ngoài ra, chúng ta sẽ thêm các handler vào các giao diện này để chuyển đổi từ bất kỳ đơn vị nào sang đơn vị khác, vì khung cơ bản cho điều này đã tồn tại trong Cosmos SDK.

Cuối cùng, chúng ta cần đảm bảo metadata tồn tại trong `GenesisState` của module `x/bank`, cũng được lập chỉ mục theo `denom` cơ sở.

```go
type GenesisState struct {
  SendEnabled   bool        `json:"send_enabled" yaml:"send_enabled"`
  Balances      []Balance   `json:"balances" yaml:"balances"`
  Supply        sdk.Coins   `json:"supply" yaml:"supply"`
  DenomMetadata []Metadata  `json:"denom_metadata" yaml:"denom_metadata"`
}
```

## Công Việc Tương Lai

Để tránh cho các client phải chuyển đổi tài sản sang mệnh giá cơ sở -- dù thủ công hay qua endpoint, chúng ta có thể xem xét hỗ trợ tự động chuyển đổi đầu vào đơn vị đã cho.

## Hậu Quả

### Tích Cực

* Cung cấp cho các client, nhà cung cấp ví và trình khám phá block thêm dữ liệu về mệnh giá tài sản để cải thiện UX và loại bỏ bất kỳ nhu cầu đưa ra giả định về đơn vị mệnh giá.

### Tiêu Cực

* Cần thêm một lượng nhỏ bộ nhớ trong module `x/bank`. Lượng bộ nhớ bổ sung nên là tối thiểu vì số lượng tổng tài sản không nên quá lớn.

### Trung Lập

## Tham Khảo
