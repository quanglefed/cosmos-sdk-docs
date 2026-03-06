# ADR 050: SIGN_MODE_TEXTUAL: Phụ Lục 1 – Value Renderer

## Changelog

* 06 tháng 12 năm 2021: Bản nháp đầu tiên
* 07 tháng 2 năm 2022: Nhóm Ledger đọc và concept-ACK.
* 01 tháng 12 năm 2022: Xóa tiền tố `Object: ` trên màn hình header của Any.
* 13 tháng 12 năm 2022: Ký qua hash bytes khi độ dài bytes > 32.
* 27 tháng 3 năm 2023: Cập nhật value renderer `Any` để bỏ qua màn hình header message.

## Trạng Thái

Accepted. Triển khai đã bắt đầu. Một số chi tiết nhỏ về value renderer vẫn cần được tinh chỉnh.

## Tóm Tắt

Phụ lục này mô tả value renderer, được dùng để hiển thị các giá trị Protobuf theo cách thân thiện với người dùng sử dụng mảng chuỗi.

## Value Renderer

Value Renderer mô tả cách các giá trị của các kiểu Protobuf khác nhau nên được mã hóa thành mảng chuỗi. Value renderer có thể được hình thức hóa như một tập hợp các hàm song ánh `func renderT(value T) []string`.

### Protobuf `number`

* Áp dụng cho các kiểu số nguyên (`int{32,64}`, `uint{32,64}`, v.v.) và chuỗi/bytes có `customtype` là `sdk.Int` hoặc `sdk.Dec`.
* Các số không ở cuối phần thập phân luôn bị xóa.
* Định dạng với `'` cho mỗi ba chữ số nguyên.
* Sử dụng `.` để biểu thị dấu phân cách thập phân.

#### Ví Dụ

* `1000` (uint64) → `1'000`
* `"1000000.00"` (string biểu diễn Dec) → `1'000'000`
* `"1000000.10"` (string biểu diễn Dec) → `1'000'000.1`

### `coin`

* Áp dụng cho `cosmos.base.v1beta1.Coin`.
* Denom được chuyển đổi sang denom `display` sử dụng `Metadata` (nếu có). **Điều này yêu cầu truy vấn state.**
* Một khoảng cách giữa denom và số lượng (ví dụ `10 atom`).

#### Ví Dụ

* `1000000000uatom` → `["1'000 atom"]` vì atom là denom display của metadata.

### `coins`

* Mảng `coin` được hiển thị là sự nối tiếp của mỗi `coin` được mã hóa theo đặc tả trên, nối với nhau bằng dấu phân cách `", "`.
* Danh sách coins được sắp xếp theo điểm code unicode của denom display.
* Nếu danh sách coins có 0 mục thì hiển thị là `zero`.

#### Ví Dụ

* `["3cosm", "2000000uatom"]` → `2 atom, 3 COSM`
* `[]` → `zero`

### `repeated`

* Áp dụng cho tất cả các trường `repeated`, ngoại trừ `cosmos.tx.v1beta1.TxBody#Messages`.
* Một kiểu repeated có template:

```
<field_name>: <int> <field_kind>
<field_name> (<index>/<int>): <value rendered 1st line>
<optional value rendered in the next lines>
End of <field_name>.
```

### `message`

* Áp dụng cho tất cả các message Protobuf không có mã hóa tùy chỉnh.
* Tên trường theo [sentence case](https://en.wiktionary.org/wiki/sentence_case)
* Tên trường được sắp xếp theo số trường Protobuf.
* Lồng nhau được biểu thị bằng tiền tố `>`.

### Enum

* Hiển thị tên biến thể enum dưới dạng chuỗi.

### `google.protobuf.Any`

* Được render là:

```
<type_url>
> <value rendered underlying message>
```

Ngoại lệ: khi message bên dưới là message Protobuf không có mã hóa tùy chỉnh thì màn hình header message bị bỏ qua và một mức thụt lề được xóa.

### `google.protobuf.Timestamp`

Được render sử dụng [RFC 3339](https://www.rfc-editor.org/rfc/rfc3339). Rendering luôn sử dụng "Z" (UTC) làm múi giờ. Chỉ sử dụng các chữ số thập phân cần thiết của giây.

#### Ví Dụ

* Timestamp với 1136214245 giây và 700000000 nanoseconds → `2006-01-02T15:04:05.7Z`
* Timestamp với 1136214245 giây và không nanoseconds → `2006-01-02T15:04:05Z`

### `google.protobuf.Duration`

Duration được render như các đơn vị thời gian dài hơn là ngày, giờ, phút, cộng với giây còn lại theo thứ tự đó.

#### Ví Dụ

* `1 day`
* `30 days`
* `-1 day, 12 hours`
* `3 hours, 0 minutes, 53.025 seconds`

### bytes

* Bytes có độ dài ngắn hơn hoặc bằng 35 được render ở dạng hex, tất cả chữ hoa, không có tiền tố `0x`.
* Bytes có độ dài lớn hơn 35 được băm sử dụng SHA256. Văn bản được render là `SHA-256=`, theo sau là hash 32-byte ở dạng hex.
* Chuỗi hex được phân chia thành các nhóm 4 chữ số, với khoảng cách `' '` làm dấu phân cách.

#### Ví Dụ

* `[0]`: `00`
* `[0,1,2]`: `0001 02`
* `[0,1,2,..,35]`: `SHA-256=5D7E 2D9B 1DCB C85E 7C89 0036 A2CF 2F9F E7B6 6554 F2DF 08CE C6AA 9C0A 25C9 9C21`

### strings

Chuỗi được render nguyên trạng.

### Giá Trị Mặc Định

* Các giá trị Protobuf mặc định cho mỗi trường bị bỏ qua.

### bool

Giá trị boolean được render là `True` hoặc `False`.
