# ADR 036: Đặc Tả Chữ Ký Message Tùy Ý

## Nhật Ký Thay Đổi

* 28/10/2020 - Bản nháp đầu tiên

## Tác Giả

* Antoine Herzog (@antoineherzog)
* Zaki Manian (@zmanian)
* Aleksandr Bezobchuk (alexanderbez) [1]
* Frojdi Dymylja (@fdymylja)

## Trạng Thái

Bản Nháp

## Tóm Tắt

Hiện tại, trong Cosmos SDK, không có quy ước để ký các message tùy ý như trong Ethereum. Chúng tôi đề xuất với đặc tả này, cho hệ sinh thái Cosmos SDK, một cách để ký và xác nhận các message tùy ý off-chain.

Đặc tả này phục vụ mục đích bao gồm mọi trường hợp sử dụng; điều này có nghĩa là các nhà phát triển ứng dụng Cosmos SDK quyết định cách tuần tự hóa và biểu diễn `Data` cho người dùng.

## Bối Cảnh

Khả năng ký các message off-chain đã được chứng minh là một khía cạnh cơ bản của gần như bất kỳ blockchain nào. Khái niệm ký các message off-chain có nhiều lợi ích được thêm vào như tiết kiệm chi phí tính toán và giảm thông lượng giao dịch và overhead. Trong bối cảnh Cosmos, một số ứng dụng chính của việc ký dữ liệu như vậy bao gồm, nhưng không giới hạn, cung cấp phương tiện mật mã học an toàn và có thể xác minh để chứng minh danh tính validator và có thể liên kết nó với một số framework hoặc tổ chức khác. Ngoài ra, có khả năng ký các message Cosmos bằng thiết bị Ledger hoặc HSM tương tự.

## Quyết Định

Mục tiêu là có thể ký các message tùy ý, ngay cả khi sử dụng thiết bị Ledger hoặc HSM tương tự.

Kết quả là, các message đã ký nên trông giống các message Cosmos SDK nhưng **không được** là một giao dịch hợp lệ on-chain. `chain-id`, `account_number` và `sequence` đều có thể được gán các giá trị không hợp lệ.

Cosmos SDK 0.40 cũng giới thiệu khái niệm "auth_info" có thể chỉ định SIGN_MODES.

Một spec nên bao gồm `auth_info` hỗ trợ SIGN_MODE_DIRECT và SIGN_MODE_LEGACY_AMINO.

Để tạo các định nghĩa proto `offchain`, chúng ta mở rộng module auth với gói `offchain` để cung cấp chức năng xác minh và ký các message ngoại tuyến.

Một giao dịch offchain tuân theo các quy tắc sau:

* memo phải rỗng
* nonce, sequence number phải bằng 0
* chain-id phải bằng ""
* fee gas phải bằng 0
* fee amount phải là mảng rỗng

Xác minh giao dịch offchain tuân theo cùng quy tắc như giao dịch onchain, ngoại trừ các khác biệt về spec được nêu bật ở trên.

Message đầu tiên được thêm vào gói `offchain` là `MsgSignData`.

`MsgSignData` cho phép các nhà phát triển ký các byte tùy ý chỉ có thể xác nhận off-chain. `Signer` là địa chỉ tài khoản của người ký. `Data` là các byte tùy ý có thể đại diện cho `text`, `file`, `object`. Quyết định của nhà phát triển ứng dụng là `Data` nên được giải tuần tự hóa, tuần tự hóa như thế nào và object mà nó có thể đại diện trong bối cảnh của họ.

Quyết định của nhà phát triển ứng dụng là `Data` nên được xử lý như thế nào, theo ý nghĩa xử lý là quá trình tuần tự hóa và giải tuần tự hóa và Object mà `Data` nên đại diện.

Định nghĩa Proto:

```protobuf
// MsgSignData định nghĩa một message off-chain tùy ý, mục đích chung
message MsgSignData {
    // Signer là sdk.AccAddress của người ký message
    bytes Signer = 1 [(gogoproto.jsontag) = "signer", (gogoproto.casttype) = "github.com/cosmos/cosmos-sdk/types.AccAddress"];
    // Data đại diện cho các byte thô của nội dung được ký (văn bản, json, v.v.)
    bytes Data = 2 [(gogoproto.jsontag) = "data"];
}
```

Ví dụ json MsgSignData đã ký:

```json
{
  "type": "cosmos-sdk/StdTx",
  "value": {
    "msg": [
      {
        "type": "sign/MsgSignData",
        "value": {
          "signer": "cosmos1hftz5ugqmpg9243xeegsqqav62f8hnywsjr4xr",
          "data": "cmFuZG9t"
        }
      }
    ],
    "fee": {
      "amount": [],
      "gas": "0"
    },
    "signatures": [
      {
        "pub_key": {
          "type": "tendermint/PubKeySecp256k1",
          "value": "AqnDSiRoFmTPfq97xxEb2VkQ/Hm28cPsqsZm9jEVsYK9"
        },
        "signature": "8y8i34qJakkjse9pOD2De+dnlc4KvFgh0wQpes4eydN66D9kv7cmCEouRrkka9tlW9cAkIL52ErB+6ye7X5aEg=="
      }
    ],
    "memo": ""
  }
}
```

## Hậu Quả

Có một đặc tả về cách các message không dự định được phát sóng tới một chain trực tiếp nên được hình thành.

### Tương Thích Ngược

Tương thích ngược được duy trì vì đây là định nghĩa đặc tả message mới.

### Tích Cực

* Một định dạng chung có thể được sử dụng bởi nhiều ứng dụng để ký và xác minh các message off-chain.
* Đặc tả mang tính nguyên thủy có nghĩa là nó có thể bao gồm mọi trường hợp sử dụng mà không hạn chế những gì có thể chứa trong nó.
* Nó tạo không gian cho các đặc tả message off-chain khác nhắm đến các trường hợp sử dụng cụ thể và phổ biến hơn như các lớp authN/authZ dựa trên off-chain [2].

### Tiêu Cực

* Đề xuất hiện tại yêu cầu mối quan hệ cố định giữa địa chỉ tài khoản và khóa công khai.
* Không hoạt động với các tài khoản multisig.

## Thảo Luận Thêm

* Về bảo mật trong `MsgSignData`, nhà phát triển sử dụng `MsgSignData` chịu trách nhiệm làm cho nội dung trong `Data` không thể phát lại khi và nếu cần thiết.
* Gói offchain sẽ được mở rộng thêm với các message bổ sung nhắm đến các trường hợp sử dụng cụ thể như, nhưng không giới hạn, xác thực trong các ứng dụng, kênh thanh toán, giải pháp L2 nói chung.

## Tham Khảo

1. https://github.com/cosmos/ics/pull/33
2. https://github.com/cosmos/cosmos-sdk/pull/7727#discussion_r515668204
3. https://github.com/cosmos/cosmos-sdk/pull/7727#issuecomment-722478477
4. https://github.com/cosmos/cosmos-sdk/pull/7727#issuecomment-721062923
