---
sidebar_position: 1
---

# Nâng Cấp Ứng Dụng

:::note
Tài liệu này mô tả cách nâng cấp ứng dụng của bạn. Nếu bạn đang tìm kiếm cụ thể các thay đổi cần thực hiện giữa các phiên bản SDK, hãy xem [tài liệu migrations SDK](https://docs.cosmos.network/main/migrations/intro).
:::

:::warning
Phần này hiện chưa hoàn chỉnh. Theo dõi tiến trình của tài liệu này [tại đây](https://github.com/cosmos/cosmos-sdk/issues/11504).
:::

:::note Đọc Trước

* [Tài liệu `x/upgrade`](https://docs.cosmos.network/main/modules/upgrade)

:::

## Quy Trình Chung

Giả sử chúng ta đang chạy v0.38.0 phần mềm trên testnet và muốn nâng cấp lên v0.40.0. Điều này sẽ diễn ra như thế nào trong thực tế? Đầu tiên, chúng ta muốn hoàn thiện release candidate v0.40.0, sau đó cài đặt một upgrade handler được đặt tên đặc biệt (ví dụ: "testnet-v2" hoặc thậm chí "v0.40.0"). Một upgrade handler nên được định nghĩa trong phiên bản mới của phần mềm để định nghĩa các migration nào cần chạy để di chuyển từ phiên bản cũ. Tất nhiên, đây là đặc thù của ứng dụng chứ không phải module, và phải được định nghĩa trong `app.go`, ngay cả khi nó import logic từ nhiều module khác nhau để thực hiện các hành động. Bạn có thể đăng ký chúng với `upgradeKeeper.SetUpgradeHandler` trong quá trình khởi tạo ứng dụng (trước khi khởi động abci server), và chúng không chỉ phục vụ để thực hiện migration mà còn để xác định liệu đây là phiên bản cũ hay mới (ví dụ: sự hiện diện của handler được đăng ký cho upgrade được đặt tên).

Khi release candidate cùng với upgrade handler phù hợp đã được đóng băng, chúng ta có thể có một governance vote để phê duyệt upgrade này ở một chiều cao block trong tương lai (ví dụ: 200000). Đây được gọi là upgrade.Plan. Code v0.38.0 sẽ không biết về handler này, nhưng sẽ tiếp tục chạy cho đến block 200000, khi plan được kích hoạt tại `BeginBlock`. Nó sẽ kiểm tra sự tồn tại của handler, và khi thấy nó bị thiếu, biết rằng nó đang chạy phần mềm lỗi thời và thoát ra một cách graceful.

Thông thường binary ứng dụng sẽ khởi động lại khi thoát, nhưng sau đó sẽ thực thi BeginBlocker này một lần nữa và thoát, gây ra vòng lặp khởi động lại. Operator có thể cài đặt phần mềm mới theo cách thủ công, hoặc bạn có thể sử dụng một daemon giám sát bên ngoài để có thể tải xuống và chuyển đổi binary, đồng thời có thể tạo bản sao lưu. Công cụ SDK cho mục đích này được gọi là [Cosmovisor](https://docs.cosmos.network/main/tooling/cosmovisor).

Khi binary khởi động lại với phiên bản nâng cấp (ở đây là v0.40.0), nó sẽ phát hiện rằng chúng ta đã đăng ký upgrade handler "testnet-v2" trong code, và nhận ra đây là phiên bản mới. Sau đó nó sẽ chạy upgrade handler và *migrate database tại chỗ*. Khi hoàn thành, nó đánh dấu upgrade là đã hoàn thành, và tiếp tục xử lý phần còn lại của block bình thường. Khi 2/3 voting power đã nâng cấp, blockchain sẽ ngay lập tức tiếp tục cơ chế đồng thuận. Nếu phần lớn operator thêm script `do-upgrade` tùy chỉnh, điều này chỉ mất vài phút và thậm chí không cần họ phải thức dậy vào thời điểm đó.

## Tích Hợp Với Ứng Dụng

:::tip
Phần sau đây không bắt buộc với người dùng sử dụng `depinject`, vì điều này được trừu tượng hóa cho họ.
:::

Ngoài việc wiring module cơ bản, hãy thiết lập upgrade Keeper cho ứng dụng và sau đó định nghĩa một `PreBlocker` gọi phương thức PreBlocker của upgrade keeper:

```go
func (app *myApp) PreBlocker(ctx sdk.Context, req req.RequestFinalizeBlock) (*sdk.ResponsePreBlock, error) {
      // Để minh họa, app PreBlocker chỉ trả về upgrade module pre-blocker.
      // Trong ứng dụng thực, module manager nên gọi tất cả pre-blocker
      // return app.ModuleManager.PreBlock(ctx, req)
      return app.upgradeKeeper.PreBlocker(ctx, req)
}
```

Sau đó ứng dụng phải tích hợp upgrade keeper với module governance của nó một cách phù hợp. Module governance nên gọi ScheduleUpgrade để lên lịch nâng cấp và ClearUpgradePlan để hủy nâng cấp đang chờ xử lý.

## Thực Hiện Nâng Cấp

Các nâng cấp có thể được lên lịch ở một chiều cao block được định sẵn. Khi chiều cao block này đạt được, phần mềm hiện tại sẽ ngừng xử lý các message ABCI và phiên bản mới với code xử lý nâng cấp phải được triển khai. Tất cả nâng cấp được phối hợp bởi một tên nâng cấp duy nhất không thể tái sử dụng trên cùng blockchain. Để module upgrade biết rằng nâng cấp đã được áp dụng an toàn, một handler với tên của nâng cấp phải được cài đặt. Đây là ví dụ handler cho nâng cấp tên "my-fancy-upgrade":

```go
app.upgradeKeeper.SetUpgradeHandler("my-fancy-upgrade", func(ctx context.Context, plan upgrade.Plan) {
 // Thực hiện bất kỳ migration store state cần thiết cho nâng cấp này
})
```

Upgrade handler này thực hiện chức năng kép là thông báo cho module upgrade rằng nâng cấp được đặt tên đã được áp dụng, cũng như cung cấp cơ hội cho phần mềm nâng cấp thực hiện bất kỳ migration trạng thái cần thiết nào. Cả việc dừng (với binary cũ) và áp dụng migration (với binary mới) đều được thực thi trong state machine. Thực sự việc chuyển đổi binary là công việc vận hành và không được xử lý bên trong sdk / abci app.

Đây là code mẫu để đặt store migrations với nâng cấp:

```go
// cấu hình no-op upgrade handler cho nâng cấp "my-fancy-upgrade"
app.UpgradeKeeper.SetUpgradeHandler("my-fancy-upgrade", func(ctx context.Context, plan upgrade.Plan) {
 // các thay đổi nâng cấp ở đây
})
upgradeInfo, err := app.UpgradeKeeper.ReadUpgradeInfoFromDisk()
if err != nil {
 // xử lý lỗi
}
if upgradeInfo.Name == "my-fancy-upgrade" && !app.UpgradeKeeper.IsSkipHeight(upgradeInfo.Height) {
 storeUpgrades := store.StoreUpgrades{
  Renamed: []store.StoreRename{{
   OldKey: "foo",
   NewKey: "bar",
  }},
  Deleted: []string{},
 }
 // cấu hình store loader kiểm tra version == upgradeHeight và áp dụng store upgrades
 app.SetStoreLoader(upgrade.UpgradeStoreLoader(upgradeInfo.Height, &storeUpgrades))
}
```

## Hành Vi Dừng (Halt)

Trước khi dừng ABCI state machine trong phương thức BeginBlocker, module upgrade sẽ ghi log một lỗi trông như sau:

```text
 UPGRADE "<Name>" NEEDED at height <NNNN>: <Info>
```

trong đó `Name` và `Info` là giá trị của các trường tương ứng trong upgrade Plan.

Để thực sự dừng blockchain, upgrade keeper chỉ đơn giản là panic, ngăn ABCI state machine tiến hành nhưng không thực sự thoát tiến trình. Việc thoát tiến trình có thể gây ra vấn đề cho các node khác bắt đầu mất kết nối với các node đang thoát, do đó module này ưa thích chỉ dừng chứ không thoát.

## Tự Động Hóa

Đọc thêm về [Cosmovisor](https://docs.cosmos.network/main/tooling/cosmovisor), công cụ để tự động hóa nâng cấp.

## Hủy Nâng Cấp

Có hai cách để hủy nâng cấp đã lên kế hoạch — với on-chain governance hoặc off-chain social consensus. Với cách đầu tiên, có một governance proposal `CancelSoftwareUpgrade`, có thể được bỏ phiếu và sẽ xóa kế hoạch nâng cấp đã lên lịch. Tất nhiên điều này yêu cầu nâng cấp được biết là ý tưởng tồi tệ từ lâu trước khi chính nâng cấp xảy ra, để có đủ thời gian bỏ phiếu. Nếu bạn muốn cho phép khả năng như vậy, bạn nên đặt chiều cao nâng cấp là `2 * (votingperiod + depositperiod) + (safety delta)` tính từ đầu đề xuất nâng cấp đầu tiên. Safety delta là thời gian có sẵn từ khi đề xuất nâng cấp thành công cho đến khi nhận ra đó là ý tưởng tệ (do kiểm thử bên ngoài).

Tuy nhiên, giả sử chúng ta không nhận ra nâng cấp có lỗi cho đến khi nó sắp xảy ra (hoặc khi chúng ta thử — gặp panic trong migration). Dường như blockchain bị kẹt, nhưng chúng ta cần cho phép thoát khỏi để social consensus ghi đè kế hoạch nâng cấp. Để làm vậy, có flag `--unsafe-skip-upgrades` cho lệnh start, sẽ khiến node đánh dấu nâng cấp là hoàn thành khi đạt chiều cao nâng cấp đã lên kế hoạch, mà không dừng và không thực sự thực hiện migration. Nếu hơn hai phần ba chạy node của họ với flag này trên binary cũ, nó sẽ cho phép chain tiếp tục qua nâng cấp với manual override. (Điều này phải được ghi chép rõ ràng cho bất kỳ ai đồng bộ từ genesis sau này).

Ví dụ:

```shell
<appd> start --unsafe-skip-upgrades <height1> <optional_height_2> ... <optional_height_N>
```

## Xử Lý Trước Khi Nâng Cấp (Pre-Upgrade)

Cosmovisor hỗ trợ xử lý tùy chỉnh trước khi nâng cấp. Sử dụng xử lý pre-upgrade khi bạn cần triển khai các thay đổi cấu hình ứng dụng cần thiết trong phiên bản mới hơn trước khi thực hiện nâng cấp.

Việc sử dụng xử lý pre-upgrade của Cosmovisor là tùy chọn. Nếu xử lý pre-upgrade không được triển khai, nâng cấp vẫn tiếp tục.

Ví dụ, thực hiện các thay đổi cần thiết cho phiên bản mới vào cài đặt `app.toml` trong quá trình xử lý pre-upgrade. Quá trình xử lý pre-upgrade có nghĩa là file không cần phải cập nhật thủ công sau khi nâng cấp.

Trước khi binary ứng dụng được nâng cấp, Cosmovisor gọi lệnh `pre-upgrade` có thể được triển khai bởi ứng dụng.

Lệnh `pre-upgrade` không nhận bất kỳ đối số dòng lệnh nào và được kỳ vọng sẽ kết thúc với các exit code sau:

| Exit status code | Cách xử lý trong Cosmovisor                                                                                         |
|------------------|---------------------------------------------------------------------------------------------------------------------|
| `0`              | Giả định lệnh `pre-upgrade` đã thực thi thành công và tiếp tục nâng cấp.                                           |
| `1`              | Exit code mặc định khi lệnh `pre-upgrade` chưa được triển khai.                                                    |
| `30`             | Lệnh `pre-upgrade` đã được thực thi nhưng thất bại. Điều này làm thất bại toàn bộ nâng cấp.                        |
| `31`             | Lệnh `pre-upgrade` đã được thực thi nhưng thất bại. Nhưng lệnh được thử lại cho đến khi trả về exit code `1` hoặc `30`. |

## Mẫu

Đây là cấu trúc mẫu của lệnh `pre-upgrade`:

```go
func preUpgradeCommand() *cobra.Command {
 cmd := &cobra.Command{
  Use:   "pre-upgrade",
  Short: "Lệnh Pre-upgrade",
        Long: "Lệnh pre-upgrade để triển khai xử lý pre-upgrade tùy chỉnh",
  Run: func(cmd *cobra.Command, args []string) {

   err := HandlePreUpgrade()

   if err != nil {
    os.Exit(30)
   }

   os.Exit(0)

  },
 }

 return cmd
}
```

Đảm bảo rằng lệnh pre-upgrade đã được đăng ký trong ứng dụng:

```go
rootCmd.AddCommand(
  // ..
  preUpgradeCommand(),
  // ..
 )
```

Khi không sử dụng Cosmovisor, hãy đảm bảo chạy `<appd> pre-upgrade` trước khi khởi động binary ứng dụng.
