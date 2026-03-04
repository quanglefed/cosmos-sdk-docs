---
sidebar_position: 1
---

# Bộ Mô Phỏng Blockchain Cosmos

Cosmos SDK cung cấp một framework mô phỏng đầy đủ để kiểm thử fuzz (fuzz test) mọi message được định nghĩa bởi một module.

Trong Cosmos SDK, chức năng này được cung cấp bởi [`SimApp`](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/simapp/app_di.go) — một ứng dụng `Baseapp` được dùng để chạy module [`simulation`](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/x/simulation). Module này định nghĩa toàn bộ logic mô phỏng cũng như các phép toán cho các tham số ngẫu nhiên như tài khoản, số dư, v.v.

## Mục tiêu

Bộ mô phỏng blockchain kiểm tra cách ứng dụng blockchain sẽ hoạt động trong các tình huống thực tế bằng cách tạo và gửi các message ngẫu nhiên. Mục tiêu là phát hiện và gỡ lỗi các sự cố có thể làm dừng chuỗi đang hoạt động, bằng cách cung cấp log và thống kê về các phép toán được chạy bởi simulator, cũng như xuất trạng thái ứng dụng mới nhất khi phát hiện lỗi.

Điểm khác biệt chính so với kiểm thử tích hợp là ứng dụng simulator cho phép bạn truyền các tham số để tùy chỉnh chuỗi đang được mô phỏng. Điều này rất tiện lợi khi cố gắng tái hiện các lỗi được tạo ra trong các phép toán được cung cấp (ngẫu nhiên hoặc không).

## Các lệnh Simulation

Ứng dụng simulation có các lệnh khác nhau, mỗi lệnh kiểm tra một loại lỗi khác nhau:

* `AppImportExport`: Simulator xuất trạng thái ứng dụng ban đầu rồi tạo một ứng dụng mới với `genesis.json` đã xuất làm đầu vào, kiểm tra sự không nhất quán giữa các store.
* `AppSimulationAfterImport`: Xếp hàng hai lần mô phỏng liên tiếp. Lần đầu cung cấp trạng thái ứng dụng (tức genesis) cho lần thứ hai. Hữu ích để kiểm tra nâng cấp phần mềm hoặc hard-fork từ một chuỗi đang hoạt động.
* `AppStateDeterminism`: Kiểm tra rằng tất cả các node trả về cùng giá trị, theo cùng thứ tự.
* `FullAppSimulation`: Chế độ mô phỏng tổng quát. Chạy chuỗi và các phép toán được chỉ định trong một số lượng block nhất định. Kiểm tra rằng không có `panics` nào xảy ra trong quá trình mô phỏng.

Mỗi lần mô phỏng phải nhận một tập hợp các đầu vào (tức flags) như số lượng block, seed, kích thước block, v.v. Xem danh sách đầy đủ các flags [tại đây](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/x/simulation/client/cli/flags.go#L43-L70).

## Các chế độ Simulator

Ngoài các đầu vào và lệnh khác nhau, simulator chạy ở ba chế độ:

1. Hoàn toàn ngẫu nhiên: trạng thái ban đầu, tham số module và tham số mô phỏng được **tạo ra theo kiểu giả ngẫu nhiên (pseudo-random)**.
2. Từ file `genesis.json`: trạng thái ban đầu và tham số module được định nghĩa sẵn. Chế độ này hữu ích khi chạy mô phỏng trên một trạng thái đã biết như xuất dữ liệu từ mạng đang hoạt động để kiểm tra phiên bản mới (thường có thay đổi phá vỡ tương thích).
3. Từ file `params.json`: trạng thái ban đầu được tạo ngẫu nhiên nhưng tham số module và mô phỏng có thể được cung cấp thủ công. Điều này cho phép thiết lập mô phỏng có kiểm soát và xác định hơn trong khi vẫn để không gian trạng thái được mô phỏng ngẫu nhiên. Danh sách các tham số có sẵn được liệt kê [tại đây](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/x/simulation/client/cli/flags.go#L72-L90).

:::tip
Các chế độ này không loại trừ lẫn nhau. Ví dụ, bạn có thể chạy genesis state được tạo ngẫu nhiên (chế độ 1) cùng với simulation params được tạo thủ công (chế độ 3).
:::

## Cách sử dụng

Đây là ví dụ tổng quát về cách chạy simulation. Để xem các ví dụ cụ thể hơn, hãy xem [Makefile](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/Makefile#L285-L320) của Cosmos SDK.

```bash
 $ go test -mod=readonly github.com/cosmos/cosmos-sdk/simapp \
  -run=TestApp<simulation_command> \
  ...<flags>
  -v -timeout 24h
```

## Mẹo gỡ lỗi

Một số gợi ý khi gặp lỗi trong quá trình mô phỏng:

* Xuất trạng thái ứng dụng tại chiều cao (height) phát hiện lỗi bằng cách truyền flag `-ExportStatePath` cho simulator.
* Dùng log `-Verbose`. Chúng có thể cung cấp gợi ý tốt hơn về tất cả các phép toán liên quan.
* Thử dùng `-Seed` khác. Nếu nó tái hiện cùng lỗi và thất bại sớm hơn, bạn sẽ tốn ít thời gian chạy simulation hơn.
* Giảm `-NumBlocks`. Trạng thái ứng dụng ở chiều cao trước khi xảy ra lỗi như thế nào?
* Thử thêm log cho các phép toán chưa được ghi log. Bạn cần định nghĩa một [Logger](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0-rc.2/x/staking/keeper/keeper.go#L77-L81) trên `Keeper` của mình.

## Sử dụng simulation trong ứng dụng dựa trên Cosmos SDK của bạn

Tìm hiểu cách tích hợp simulation vào ứng dụng dựa trên Cosmos SDK của bạn:

* Application Simulation Manager
* [Xây dựng module: Simulator](../../build/building-modules/14-simulator.md)
* Simulator tests
