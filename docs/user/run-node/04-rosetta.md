# Rosetta

Dự án `rosetta` triển khai [Rosetta API](https://www.rosetta-api.org) của Coinbase. Tài liệu này cung cấp hướng dẫn về cách sử dụng tích hợp Rosetta API. Để biết thông tin về động lực và các lựa chọn thiết kế, hãy tham khảo [ADR 035](https://docs.cosmos.network/main/architecture/adr-035-rosetta-api-support).

## Cài Đặt Rosetta

Máy chủ Rosetta API là một dịch vụ độc lập kết nối đến một node của chuỗi được phát triển với Cosmos SDK.

Rosetta có thể được thêm vào bất kỳ node chuỗi Cosmos nào. Có thể chạy độc lập hoặc tích hợp gốc.

### Độc Lập (Standalone)

Rosetta có thể được thực thi như một dịch vụ độc lập, nó kết nối đến các endpoint của node và hiển thị các endpoint cần thiết.

Cài đặt máy chủ Rosetta độc lập bằng lệnh sau:

```bash
go install github.com/cosmos/rosetta
```

Hoặc, để xây dựng từ mã nguồn, đơn giản chạy `make build`. Binary sẽ nằm trong thư mục gốc.

### Tích Hợp Gốc - Là Một Lệnh Node

Để bật hỗ trợ Rosetta API tích hợp gốc, cần thêm `RosettaCommand` vào file lệnh gốc của ứng dụng (ví dụ: `simd/cmd/root.go`).

Import gói `rosettaCmd`:

```go
import "github.com/cosmos/rosetta/cmd"
```

Tìm dòng sau:

```go
initRootCmd(rootCmd, encodingConfig)
```

Sau dòng đó, thêm đoạn sau:

```go
rootCmd.AddCommand(
  rosettaCmd.RosettaCommand(encodingConfig.InterfaceRegistry, encodingConfig.Codec)
)
```

Hàm `RosettaCommand` xây dựng lệnh gốc `rosetta` và được định nghĩa trong gói `rosettaCmd` (`github.com/cosmos/rosetta/cmd`).

Vì chúng ta đã cập nhật Cosmos SDK để hoạt động với Rosetta API, việc cập nhật file lệnh gốc của ứng dụng là tất cả những gì bạn cần làm.

Có thể tìm thấy ví dụ triển khai trong gói `simapp`.

## Sử Dụng Lệnh Rosetta

Để chạy Rosetta trong CLI ứng dụng của bạn, sử dụng lệnh sau:

> **Lưu ý:** nếu sử dụng phương pháp tích hợp gốc, hãy thêm tên node của bạn trước bất kỳ lệnh rosetta nào.

```shell
rosetta --help
```

Để kiểm thử và chạy các endpoint Rosetta API cho các ứng dụng đang chạy và được hiển thị, sử dụng lệnh sau:

```shell
rosetta
     --blockchain "tên ứng dụng của bạn (vd: gaia)"
     --network "mã định danh chuỗi của bạn (vd: testnet-1)"
     --tendermint "endpoint tendermint (vd: localhost:26657)"
     --grpc "endpoint gRPC (vd: localhost:9090)"
     --addr "địa chỉ binding của rosetta (vd: :8080)"
     --grpc-types-server (tùy chọn) "endpoint gRPC cho các kiểu mô tả thông điệp"
```

## Plugin - Kết Nối Đa Chuỗi

Rosetta sẽ cố gắng phản chiếu (reflect) các kiểu của node thông qua reflection qua các endpoint gRPC của node, có thể có những trường hợp cách tiếp cận này không đủ. Có thể dễ dàng mở rộng hoặc triển khai các kiểu cần thiết thông qua plugin.

Để sử dụng Rosetta trên bất kỳ chuỗi nào, cần thiết lập các tiền tố (prefix) và đăng ký các interface đặc thù của zone thông qua plugin.

Mỗi plugin là một triển khai tối giản của `InitZone` và `RegisterInterfaces`, cho phép Rosetta phân tích dữ liệu đặc thù của chuỗi. Có một ví dụ cho chuỗi cosmos-hub trong thư mục `plugins/cosmos-hub/`:
- **InitZone**: Một phương thức rỗng được thực thi đầu tiên và định nghĩa các tiền tố, tham số và các cài đặt khác.
- **RegisterInterfaces**: Phương thức này nhận một registry interface, nơi các kiểu và interface đặc thù của zone sẽ được tải.

Để thêm plugin mới:
1. Tạo một thư mục trong thư mục `plugins` với tên của zone mong muốn
2. Thêm file `main.go` với các phương thức đã đề cập ở trên.
3. Build binary mã bằng `go build -buildmode=plugin -o main.so main.go`

Thư mục plugin được chọn thông qua cờ `--plugin` của CLI và được tải vào máy chủ Rosetta.

## Mở Rộng

Có hai cách để bạn tùy chỉnh và mở rộng triển khai với các cài đặt tùy chỉnh của mình.

### Mở Rộng Thông Điệp

Để làm cho một `sdk.Msg` được Rosetta hiểu, điều duy nhất cần thiết là thêm các phương thức vào các thông điệp của bạn để thỏa mãn interface `rosetta.Msg`. Có thể tìm thấy ví dụ về cách thực hiện trong các kiểu staking như `MsgDelegate`, hoặc trong các kiểu bank như `MsgSend`.

### Ghi Đè Interface Client

Trong trường hợp cần tùy chỉnh nhiều hơn, có thể nhúng (embed) kiểu Client và ghi đè các phương thức cần tùy chỉnh.

Ví dụ:

```go
package custom_client
import (

"context"
"github.com/coinbase/rosetta-sdk-go/types"
"github.com/cosmos/rosetta/lib"
)

// CustomClient nhúng cosmos client tiêu chuẩn
// có nghĩa là nó triển khai interface Client của cosmos-rosetta-gateway
// đồng thời cho phép tùy chỉnh một số phương thức nhất định
type CustomClient struct {
    *rosetta.Client
}

func (c *CustomClient) ConstructionPayload(_ context.Context, request *types.ConstructionPayloadsRequest) (resp *types.ConstructionPayloadsResponse, err error) {
    // cung cấp các byte chữ ký tùy chỉnh
    panic("implement me")
}
```

LƯU Ý: khi sử dụng client tùy chỉnh, không thể sử dụng lệnh vì các constructor cần thiết **có thể** khác nhau, vì vậy cần tạo một lệnh mới. Chúng tôi có ý định cung cấp cách khởi tạo client tùy chỉnh mà không cần viết thêm code trong tương lai.

### Mở Rộng Lỗi

Vì rosetta yêu cầu cung cấp các lỗi 'đã trả về' cho các tùy chọn mạng, để khai báo một lỗi rosetta mới, chúng ta sử dụng gói `errors` trong cosmos-rosetta-gateway.

Ví dụ:

```go
package custom_errors
import crgerrs "github.com/cosmos/rosetta/lib/errors"

var customErrRetriable = true
var CustomError = crgerrs.RegisterError(100, "custom message", customErrRetriable, "description")
```

Lưu ý: các lỗi phải được đăng ký trước khi phương thức `Start` của `Server` trong cosmos-rosetta-gateway được gọi. Nếu không, việc đăng ký sẽ bị bỏ qua. Các lỗi có cùng mã cũng sẽ bị bỏ qua.
