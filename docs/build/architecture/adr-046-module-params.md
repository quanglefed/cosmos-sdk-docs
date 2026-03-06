# ADR 046: Tham Số Module

## Changelog

* 22 tháng 9 năm 2021: Bản nháp đầu tiên

## Trạng Thái

Đề Xuất

## Tóm Tắt

ADR này mô tả một cách tiếp cận thay thế về cách các module Cosmos SDK sử dụng, tương tác và lưu trữ các tham số tương ứng của chúng.

## Bối Cảnh

Hiện tại, trong Cosmos SDK, các module yêu cầu sử dụng các tham số dùng module `x/params`. Module `x/params` hoạt động bằng cách yêu cầu các module định nghĩa các tham số, thường thông qua một struct `Params` đơn giản, và đăng ký struct đó trong module `x/params` thông qua một `Subspace` duy nhất thuộc về module đăng ký tương ứng. Module đăng ký sau đó có quyền truy cập duy nhất vào `Subspace` tương ứng của nó. Thông qua `Subspace` này, module có thể lấy và đặt struct `Params` của nó.

Ngoài ra, module `x/gov` của Cosmos SDK có hỗ trợ trực tiếp để thay đổi tham số on-chain thông qua loại đề xuất governance `ParamChangeProposal`, nơi các stakeholder có thể bỏ phiếu về các thay đổi tham số được đề xuất.

Có nhiều đánh đổi khi sử dụng module `x/params` để quản lý các tham số module riêng lẻ. Cụ thể, việc quản lý tham số về cơ bản đến "miễn phí" ở chỗ nhà phát triển chỉ cần định nghĩa struct `Params`, `Subspace` và các hàm phụ trợ khác nhau, ví dụ `ParamSetPairs`, trên kiểu `Params`. Tuy nhiên, có một số hạn chế đáng chú ý. Những hạn chế này bao gồm thực tế là các tham số được tuần tự hóa trong state thông qua JSON, rất chậm. Ngoài ra, các thay đổi tham số thông qua đề xuất governance `ParamChangeProposal` không có cách đọc từ hoặc ghi vào state. Nói cách khác, hiện tại không thể có bất kỳ chuyển đổi trạng thái nào trong ứng dụng trong khi cố gắng thay đổi tham số.

## Quyết Định

Chúng ta sẽ xây dựng dựa trên sự liên kết của công việc `x/gov` và `x/authz` theo [#9810](https://github.com/cosmos/cosmos-sdk/pull/9810). Cụ thể, nhà phát triển module sẽ tạo một hoặc nhiều cấu trúc dữ liệu tham số duy nhất phải được tuần tự hóa vào state. Cấu trúc dữ liệu Param phải triển khai interface `sdk.Msg` với phương thức dịch vụ Protobuf Msg tương ứng sẽ xác thực và cập nhật các tham số với tất cả các thay đổi cần thiết. Module `x/gov` thông qua công việc được thực hiện trong [#9810](https://github.com/cosmos/cosmos-sdk/pull/9810), sẽ dispatch các message Param, sẽ được xử lý bởi các dịch vụ Protobuf Msg.

Ví dụ các tham số được định nghĩa trong `x/auth`:

```go
type Params struct {
    MaxMemoCharacters      uint64
    TxSigLimit             uint64
    TxSizeCostPerByte      uint64
    SigVerifyCostED25519   uint64
    SigVerifyCostSecp256k1 uint64
}

type MsgUpdateParams struct {
    MaxMemoCharacters      uint64
    TxSigLimit             uint64
    TxSizeCostPerByte      uint64
    SigVerifyCostED25519   uint64
    SigVerifyCostSecp256k1 uint64
}

func (ms msgServer) UpdateParams(goCtx context.Context, msg *types.MsgUpdateParams) (*types.MsgUpdateParamsResponse, error) {
  ctx := sdk.UnwrapSDKContext(goCtx)
  params := ParamsFromMsg(msg)
  ms.SaveParams(ctx, params)
  return &types.MsgUpdateParamsResponse{}, nil
}
```

Cũng nên cung cấp một gRPC `Service` query:

```protobuf
service Query {
  rpc Params(QueryParamsRequest) returns (QueryParamsResponse) {
    option (google.api.http).get = "/cosmos/<module>/v1beta1/params";
  }
}
```

## Hậu Quả

Kết quả của việc triển khai phương pháp tham số module này, chúng ta có được khả năng thay đổi tham số module có trạng thái và có thể mở rộng để phù hợp với gần như mọi use case của ứng dụng. Chúng ta sẽ có thể phát sự kiện, gọi các phương thức dịch vụ Msg khác hoặc thực hiện migration. Ngoài ra, sẽ có những cải thiện đáng kể về hiệu suất khi đọc và ghi tham số từ và vào state.

Tuy nhiên, phương pháp này sẽ yêu cầu nhà phát triển triển khai nhiều kiểu và phương thức dịch vụ Msg hơn. Ngoài ra, nhà phát triển phải triển khai logic lưu trữ của các tham số module.

### Tương Thích Ngược

Phương thức mới để làm việc với tham số module không tương thích ngược với module `x/params` hiện tại. Tuy nhiên, `x/params` sẽ vẫn còn trong Cosmos SDK và sẽ được đánh dấu là deprecated.

### Tích Cực

* Tham số module được tuần tự hóa hiệu quả hơn.
* Các module có thể phản ứng với các thay đổi tham số và thực hiện các hành động bổ sung.
* Có thể phát các sự kiện đặc biệt, cho phép các hook được kích hoạt.

### Tiêu Cực

* Tham số module trở nên phức tạp hơn một chút cho nhà phát triển module.

## Tài Liệu Tham Khảo

* https://github.com/cosmos/cosmos-sdk/pull/9810
* https://github.com/cosmos/cosmos-sdk/issues/9438
