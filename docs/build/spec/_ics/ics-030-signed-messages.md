# ICS 030: Cosmos Signed Messages

>TODO: Replace with valid ICS number and possibly move to new location.

* [Changelog](#changelog)
* [Tóm tắt](#abstract)
* [Sơ bộ](#preliminary)
* [Đặc tả](#specification)
* [Các thích ứng tương lai](#future-adaptations)
* [API](#api)
* [Tài liệu tham khảo](#references)

## Trạng thái

Đề xuất.

## Changelog

## Tóm tắt {#abstract}

Khả năng ký thông điệp off-chain đã chứng minh là một khía cạnh cơ bản của hầu hết mọi blockchain. Việc ký thông điệp off-chain mang lại nhiều lợi ích như tiết kiệm chi phí tính toán và giảm thông lượng giao dịch cũng như chi phí phụ. Trong bối cảnh Cosmos, một số ứng dụng chính của việc ký dữ liệu như vậy bao gồm, nhưng không giới hạn ở, cung cấp phương tiện mật mã an toàn và có thể xác minh để chứng minh danh tính validator và có thể liên kết nó với một framework hoặc tổ chức khác. Ngoài ra, khả năng ký thông điệp Cosmos bằng Ledger hoặc thiết bị HSM tương tự.

Một giao thức chuẩn hóa cho việc băm, ký và xác minh thông điệp mà có thể được triển khai bởi Cosmos SDK và các tổ chức bên thứ ba khác là cần thiết. Giao thức chuẩn hóa như vậy tuân theo các điều sau:

* Chứa đặc tả của dữ liệu có cấu trúc có kiểu có thể đọc được bởi con người và có thể xác minh bởi máy
* Chứa framework cho việc mã hóa dữ liệu có cấu trúc mang tính xác định và đơn ánh
* Sử dụng các thuật toán băm và ký mật mã an toàn
* Framework hỗ trợ mở rộng và tách biệt miền
* Không dễ bị tấn công bởi chosen ciphertext attacks
* Có bảo vệ chống lại việc ký các giao dịch mà người dùng không có ý định

Đặc tả này chỉ quan tâm đến lý do và triển khai chuẩn hóa của Cosmos signed messages. Nó **không** quan tâm đến khái niệm replay attacks vì điều đó sẽ được để cho triển khai ứng dụng cấp cao hơn xử lý. Nếu bạn xem signed messages như phương tiện để ủy quyền một hành động hoặc dữ liệu nào đó, thì ứng dụng đó sẽ phải coi đây là idempotent hoặc có cơ chế để từ chối các signed messages đã biết.

## Sơ bộ {#preliminary}

Giao thức ký thông điệp Cosmos sẽ được tham số hóa với thuật toán băm mật mã an toàn `SHA-256` và thuật toán ký `S` chứa các thao tác `sign` và `verify` cung cấp chữ ký số trên một tập byte và xác minh chữ ký tương ứng.

Lưu ý, mục tiêu của chúng ta ở đây không phải cung cấp ngữ cảnh và lý do về việc tại sao các thuật toán này được chọn ngoài thực tế chúng là các thuật toán de facto được sử dụng trong CometBFT và Cosmos SDK và chúng đáp ứng nhu cầu của chúng ta về các thuật toán mật mã như khả năng chống collision và second pre-image attacks, cũng như [deterministic](https://en.wikipedia.org/wiki/Hash_function#Determinism) và [uniform](https://en.wikipedia.org/wiki/Hash_function#Uniformity).

## Đặc tả {#specification}

CometBFT có giao thức được thiết lập tốt cho việc ký thông điệp sử dụng biểu diễn JSON chuẩn như được định nghĩa [tại đây](https://github.com/cometbft/cometbft/blob/master/types/canonical.go).

Một ví dụ về cấu trúc JSON chuẩn như vậy là cấu trúc vote của CometBFT:

```go
type CanonicalJSONVote struct {
    ChainID   string               `json:"@chain_id"`
    Type      string               `json:"@type"`
    BlockID   CanonicalJSONBlockID `json:"block_id"`
    Height    int64                `json:"height"`
    Round     int                  `json:"round"`
    Timestamp string               `json:"timestamp"`
    VoteType  byte                 `json:"type"`
}
```

Với các cấu trúc JSON chuẩn như vậy, đặc tả yêu cầu chúng bao gồm các trường meta: `@chain_id` và `@type`. Các trường meta này được dành riêng và phải được bao gồm. Cả hai đều có kiểu `string`. Ngoài ra, các trường phải được sắp xếp theo thứ tự từ điển tăng dần.

Để ký thông điệp Cosmos, trường `@chain_id` phải tương ứng với định danh chuỗi Cosmos. User-agent **phải từ chối** ký nếu trường `@chain_id` không khớp với chuỗi đang hoạt động! Trường `@type` phải bằng hằng số `"message"`. Trường `@type` tương ứng với loại cấu trúc mà người dùng sẽ ký trong một ứng dụng. Hiện tại, người dùng chỉ được phép ký các byte văn bản ASCII hợp lệ ([xem tại đây](https://github.com/cometbft/cometbft/blob/v0.37.0/libs/strings/string.go#L35-L64)). Tuy nhiên, điều này sẽ thay đổi và phát triển để hỗ trợ các cấu trúc cụ thể ứng dụng bổ sung có thể đọc được bởi con người và có thể xác minh bởi máy ([xem Các thích ứng tương lai](#future-adaptations)).

Do đó, chúng ta có thể có cấu trúc JSON chuẩn cho việc ký thông điệp Cosmos sử dụng đặc tả [JSON schema](http://json-schema.org/) như sau:

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "$id": "cosmos/signing/typeData/schema",
  "title": "The Cosmos signed message typed data schema.",
  "type": "object",
  "properties": {
    "@chain_id": {
      "type": "string",
      "description": "The corresponding Cosmos chain identifier.",
      "minLength": 1
    },
    "@type": {
      "type": "string",
      "description": "The message type. It must be 'message'.",
      "enum": [
        "message"
      ]
    },
    "text": {
      "type": "string",
      "description": "The valid ASCII text to sign.",
      "pattern": "^[\\x20-\\x7E]+$",
      "minLength": 1
    }
  },
  "required": [
    "@chain_id",
    "@type",
    "text"
  ]
}
```

ví dụ:

```json
{
  "@chain_id": "1",
  "@type": "message",
  "text": "Hello, you can identify me as XYZ on keybase."
}
```

## Các thích ứng tương lai {#future-adaptations}

Vì các ứng dụng có thể rất khác nhau về miền, việc hỗ trợ cả tách biệt miền và các cấu trúc có thể đọc được bởi con người và có thể xác minh bởi máy sẽ rất quan trọng.

Tách biệt miền sẽ cho phép các nhà phát triển ứng dụng ngăn chặn va chạm của các cấu trúc giống hệt nhau. Nó nên được thiết kế để duy nhất cho mỗi ứng dụng và nên được sử dụng trực tiếp trong chính việc mã hóa chữ ký.

Các cấu trúc có thể đọc được bởi con người và có thể xác minh bởi máy sẽ cho phép người dùng cuối ký các cấu trúc phức tạp hơn, ngoài chỉ thông điệp chuỗi, và vẫn có thể biết chính xác họ đang ký gì (thay vì ký một đống byte tùy ý).

Do đó, trong tương lai, đặc tả Cosmos signing message dự kiến sẽ mở rộng cấu trúc JSON chuẩn của nó để bao gồm chức năng như vậy.

## API {#api}

Các nhà phát triển và thiết kế ứng dụng nên chính thức hóa một tập API chuẩn tuân theo đặc tả sau:

-----

### **cosmosSignBytes**

Tham số:

* `data`: cấu trúc JSON chuẩn Cosmos signed message
* `address`: địa chỉ tài khoản Cosmos Bech32 để ký dữ liệu

Trả về:

* `signature`: chữ ký Cosmos được tạo ra bằng thuật toán ký `S`

-----

### Ví dụ

Sử dụng `secp256k1` làm DSA, `S`:

```javascript
data = {
  "@chain_id": "1",
  "@type": "message",
  "text": "I hereby claim I am ABC on Keybase!"
}

cosmosSignBytes(data, "cosmos1pvsch6cddahhrn5e8ekw0us50dpnugwnlfngt3")
> "0x7fc4a495473045022100dec81a9820df0102381cdbf7e8b0f1e2cb64c58e0ecda1324543742e0388e41a02200df37905a6505c1b56a404e23b7473d2c0bc5bcda96771d2dda59df6ed2b98f8"
```

## Tài liệu tham khảo {#references}
