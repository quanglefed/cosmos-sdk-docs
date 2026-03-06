---
sidebar_position: 1
---

# Cosmovisor

`cosmovisor` là một process manager cho các application binary của Cosmos SDK giúp tự động hóa việc chuyển đổi binary ứng dụng khi nâng cấp chain.
Nó kiểm tra file `upgrade-info.json` được tạo bởi module x/upgrade tại upgrade height, và sau đó có thể tự động tải binary mới, dừng binary hiện tại, chuyển từ binary cũ sang binary mới, và cuối cùng khởi động lại node với binary mới.

* [Design (Thiết Kế)](#design)
* [Contributing (Đóng Góp)](#contributing)
* [Setup (Thiết Lập)](#setup)
    * [Installation (Cài Đặt)](#installation)
    * [Command Line Arguments And Environment Variables (Đối Số Dòng Lệnh Và Biến Môi Trường)](#command-line-arguments-and-environment-variables)
    * [Folder Layout (Cấu Trúc Thư Mục)](#folder-layout)
* [Usage (Sử Dụng)](#usage)
    * [Initialization (Khởi Tạo)](#initialization)
    * [Detecting Upgrades (Phát Hiện Nâng Cấp)](#detecting-upgrades)
    * [Adding Upgrade Binary (Thêm Binary Nâng Cấp)](#adding-upgrade-binary)
    * [Auto-Download (Tải Tự Động)](#auto-download)
    * [Preparing for an Upgrade (Chuẩn Bị Cho Việc Nâng Cấp)](#preparing-for-an-upgrade)
* [Ví dụ: Nâng cấp SimApp](#example-simapp-upgrade)
    * [Chain Setup (Thiết Lập Chain)](#chain-setup)
        * [Prepare Cosmovisor and Start the Chain (Chuẩn Bị Cosmovisor Và Khởi Động Chain)](#prepare-cosmovisor-and-start-the-chain)
    * [Update App (Cập Nhật Ứng Dụng)](#update-app)

## Design (Thiết Kế)

Cosmovisor được thiết kế để sử dụng như một wrapper cho ứng dụng `Cosmos SDK`:

* Nó sẽ truyền các đối số cho ứng dụng liên quan (được cấu hình bởi biến môi trường `DAEMON_NAME`).
  Chạy `cosmovisor run arg1 arg2 ....` sẽ chạy `app arg1 arg2 ...`;
* Nó sẽ quản lý một ứng dụng bằng cách khởi động lại và nâng cấp khi cần;
* Nó được cấu hình bằng biến môi trường, không phải đối số vị trí.

*Lưu ý: Nếu các phiên bản mới của ứng dụng không được thiết lập để chạy in-place store migration, các migration sẽ cần được chạy thủ công trước khi khởi động lại `cosmovisor` với binary mới. Vì lý do này, chúng tôi khuyến nghị các ứng dụng áp dụng in-place store migration.*

:::tip
Chỉ có phiên bản mới nhất của cosmovisor được tích cực phát triển/bảo trì.
:::

:::warning
Các phiên bản trước v1.0.0 có lỗ hổng có thể dẫn đến DOS. Vui lòng nâng cấp lên phiên bản mới nhất.
:::

## Contributing (Đóng Góp)

Cosmovisor là một phần của monorepo Cosmos SDK, nhưng đây là một module riêng biệt với lịch phát hành riêng của nó.

Các branch release có định dạng sau `release/cosmovisor/vA.B.x`, trong đó A và B là số (ví dụ: `release/cosmovisor/v1.3.x`). Các release được tag sử dụng định dạng sau: `cosmovisor/vA.B.C`.

## Setup (Thiết Lập)

### Installation (Cài Đặt)

Bạn có thể tải Cosmovisor từ [GitHub releases](https://github.com/cosmos/cosmos-sdk/releases/tag/cosmovisor%2Fv1.5.0).

Để cài đặt phiên bản mới nhất của `cosmovisor`, chạy lệnh sau:

```shell
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@latest
```

Để cài đặt một phiên bản cụ thể, bạn có thể chỉ định phiên bản:

```shell
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.5.0
```

Chạy `cosmovisor version` để kiểm tra phiên bản cosmovisor.

Ngoài ra, để build từ source, đơn giản chạy `make cosmovisor`. Binary sẽ nằm trong `tools/cosmovisor`.

:::warning
Cài đặt cosmovisor bằng `go install` sẽ hiển thị đúng phiên bản `cosmovisor`.
Build từ source (`make cosmovisor`) hoặc cài đặt `cosmovisor` bằng các phương tiện khác sẽ không hiển thị đúng phiên bản.
:::

### Command Line Arguments And Environment Variables (Đối Số Dòng Lệnh Và Biến Môi Trường)

Đối số đầu tiên được truyền cho `cosmovisor` là action để `cosmovisor` thực hiện. Các tùy chọn là:

* `help`, `--help`, hoặc `-h` - Xuất thông tin trợ giúp của `cosmovisor` và kiểm tra cấu hình `cosmovisor` của bạn.
* `run` - Chạy binary được cấu hình sử dụng các đối số còn lại được cung cấp.
* `version` - Xuất phiên bản `cosmovisor` và cũng chạy binary với đối số `version`.
* `config` - Hiển thị cấu hình `cosmovisor` hiện tại, tức là hiển thị giá trị biến môi trường mà `cosmovisor` đang sử dụng.
* `add-upgrade` - Thêm thủ công một upgrade vào `cosmovisor`. Lệnh này cho phép bạn dễ dàng thêm binary tương ứng với một upgrade trong cosmovisor.

Tất cả các đối số được truyền cho `cosmovisor run` sẽ được truyền cho binary ứng dụng (như một subprocess). `cosmovisor` sẽ trả về `/dev/stdout` và `/dev/stderr` của subprocess như của chính nó. Vì lý do này, `cosmovisor run` không thể chấp nhận bất kỳ đối số dòng lệnh nào ngoài những đối số có sẵn cho binary ứng dụng.

`cosmovisor` đọc cấu hình của nó từ các biến môi trường, hoặc file cấu hình của nó (sử dụng `--cosmovisor-config <path>`):

* `DAEMON_HOME` là vị trí nơi thư mục `cosmovisor/` được lưu giữ chứa genesis binary, upgrade binary, và bất kỳ file phụ trợ bổ sung nào liên quan đến từng binary (ví dụ: `$HOME/.gaiad`, `$HOME/.regend`, `$HOME/.simd`, v.v.).
* `DAEMON_NAME` là tên của binary (ví dụ: `gaiad`, `regend`, `simd`, v.v.).
* `DAEMON_ALLOW_DOWNLOAD_BINARIES` (*tùy chọn*), nếu đặt thành `true`, sẽ bật tính năng tự động tải binary mới (vì lý do bảo mật, tính năng này dành cho full nodes chứ không phải validators). Theo mặc định, `cosmovisor` sẽ không tự động tải binary mới.
* `DAEMON_DOWNLOAD_MUST_HAVE_CHECKSUM` (*tùy chọn*, mặc định = `false`), nếu `true` cosmovisor sẽ yêu cầu checksum được cung cấp trong upgrade plan để binary được tải. Nếu `false`, cosmovisor sẽ không yêu cầu checksum, nhưng vẫn kiểm tra checksum nếu có.
* `DAEMON_RESTART_AFTER_UPGRADE` (*tùy chọn*, mặc định = `true`), nếu `true`, khởi động lại subprocess với các đối số dòng lệnh và flag tương tự (nhưng với binary mới) sau khi upgrade thành công. Nếu không (`false`), `cosmovisor` dừng chạy sau khi upgrade và yêu cầu quản trị viên hệ thống khởi động lại thủ công. Lưu ý khởi động lại chỉ sau khi upgrade và không tự động khởi động lại subprocess sau khi xảy ra lỗi.
* `DAEMON_RESTART_DELAY` (*tùy chọn*, mặc định không có), cho phép người vận hành node định nghĩa độ trễ giữa việc node dừng (để upgrade) và backup theo thời gian được chỉ định. Giá trị phải là duration (ví dụ: `1s`).
* `DAEMON_SHUTDOWN_GRACE` (*tùy chọn*, mặc định không có), nếu được đặt, gửi interrupt đến binary và đợi thời gian được chỉ định để cho phép dọn dẹp/flush cache xuống đĩa trước khi gửi kill signal. Giá trị phải là duration (ví dụ: `1s`).
* `DAEMON_POLL_INTERVAL` (*tùy chọn*, mặc định 300 milliseconds), là độ dài khoảng thời gian để kiểm tra file upgrade plan. Giá trị phải là duration (ví dụ: `1s`).
* `DAEMON_DATA_BACKUP_DIR` tùy chọn để đặt thư mục backup tùy chỉnh. Nếu không được đặt, `DAEMON_HOME` được sử dụng.
* `UNSAFE_SKIP_BACKUP` (mặc định `false`), nếu đặt thành `true`, nâng cấp trực tiếp mà không thực hiện backup. Nếu không (`false`, mặc định) sao lưu dữ liệu trước khi thử upgrade. Giá trị mặc định false hữu ích và được khuyến nghị trong trường hợp thất bại và khi cần backup để rollback. Chúng tôi khuyến nghị sử dụng tùy chọn backup mặc định `UNSAFE_SKIP_BACKUP=false`.
* `DAEMON_PREUPGRADE_MAX_RETRIES` (mặc định `0`). Số lần tối đa gọi [`pre-upgrade`](https://docs.cosmos.network/main/build/building-apps/app-upgrade#pre-upgrade-handling) trong ứng dụng sau exit status `31`. Sau số lần thử tối đa, Cosmovisor sẽ thất bại trong việc upgrade.
* `COSMOVISOR_DISABLE_LOGS` (mặc định `false`). Nếu đặt thành true, điều này sẽ vô hiệu hóa hoàn toàn Cosmovisor logs (nhưng không phải process bên dưới). Điều này có thể hữu ích, ví dụ, khi một subcommand Cosmovisor bạn đang thực thi trả về JSON hợp lệ mà bạn sau đó phân tích cú pháp, vì các log được thêm bởi Cosmovisor làm cho output này không phải JSON hợp lệ.
* `COSMOVISOR_COLOR_LOGS` (mặc định `true`). Nếu đặt thành true, điều này sẽ tô màu Cosmovisor logs (nhưng không phải process bên dưới).
* `COSMOVISOR_TIMEFORMAT_LOGS` (mặc định `kitchen`). Nếu đặt thành một giá trị (`layout|ansic|unixdate|rubydate|rfc822|rfc822z|rfc850|rfc1123|rfc1123z|rfc3339|rfc3339nano|kitchen`), điều này sẽ thêm tiền tố timestamp vào Cosmovisor logs (nhưng không phải process bên dưới).
* `COSMOVISOR_CUSTOM_PREUPGRADE` (mặc định ``). Nếu được đặt, điều này sẽ chạy `$DAEMON_HOME/cosmovisor/$COSMOVISOR_CUSTOM_PREUPGRADE` trước khi upgrade với các đối số `[upgrade.Name, upgrade.Height]`. Thực thi script tùy chỉnh (tách biệt và trước lệnh pre-upgrade của chain daemon)
* `COSMOVISOR_DISABLE_RECASE` (mặc định `false`). Nếu đặt thành true, thư mục upgrade sẽ được mong đợi khớp với tên upgrade plan mà không có thay đổi case nào

### Folder Layout (Cấu Trúc Thư Mục)

`$DAEMON_HOME/cosmovisor` được mong đợi thuộc hoàn toàn về `cosmovisor` và các subprocess được điều khiển bởi nó. Nội dung thư mục được tổ chức như sau:

```text
.
├── current -> genesis or upgrades/<n>
├── genesis
│   └── bin
│       └── $DAEMON_NAME
└── upgrades
│   └── <n>
│       ├── bin
│       │   └── $DAEMON_NAME
│       └── upgrade-info.json
└── preupgrade.sh (optional)
```

Thư mục `cosmovisor/` bao gồm một thư mục con cho mỗi phiên bản của ứng dụng (tức là `genesis` hoặc `upgrades/<n>`). Trong mỗi thư mục con là binary ứng dụng (tức là `bin/$DAEMON_NAME`) và bất kỳ file phụ trợ bổ sung nào liên quan đến từng binary. `current` là một symbolic link đến thư mục đang hoạt động hiện tại (tức là `genesis` hoặc `upgrades/<n>`). Biến `name` trong `upgrades/<n>` là tên URI-encoded viết thường của upgrade như được chỉ định trong upgrade module plan. Lưu ý rằng đường dẫn tên upgrade được chuẩn hóa để viết thường: ví dụ, `MyUpgrade` được chuẩn hóa thành `myupgrade`, và đường dẫn của nó là `upgrades/myupgrade`.

Vui lòng lưu ý rằng `$DAEMON_HOME/cosmovisor` chỉ lưu trữ *các application binary*. Binary `cosmovisor` có thể được lưu trữ ở bất kỳ vị trí thông thường nào (ví dụ: `/usr/local/bin`). Ứng dụng sẽ tiếp tục lưu trữ dữ liệu trong thư mục dữ liệu mặc định (ví dụ: `$HOME/.simapp`) hoặc thư mục dữ liệu được chỉ định bằng flag `--home`. `$DAEMON_HOME` phụ thuộc vào thư mục dữ liệu và phải được đặt cùng thư mục với thư mục dữ liệu, bạn sẽ có cấu hình như sau:

```text
.simapp
├── config
├── data
└── cosmovisor
```

## Usage (Sử Dụng)

Quản trị viên hệ thống chịu trách nhiệm:

* Cài đặt binary `cosmovisor`
* Cấu hình hệ thống init của host (ví dụ: `systemd`, `launchd`, v.v.)
* Đặt phù hợp các biến môi trường
* Tạo thư mục `<DAEMON_HOME>/cosmovisor`
* Tạo thư mục `<DAEMON_HOME>/cosmovisor/genesis/bin`
* Tạo các thư mục `<DAEMON_HOME>/cosmovisor/upgrades/<n>/bin`
* Đặt các phiên bản khác nhau của file thực thi `<DAEMON_NAME>` trong các thư mục `bin` thích hợp.

`cosmovisor` sẽ đặt link `current` để trỏ đến `genesis` khi khởi động lần đầu (tức là khi không có link `current` nào tồn tại) và sau đó xử lý việc chuyển binary vào đúng thời điểm để quản trị viên hệ thống có thể chuẩn bị trước nhiều ngày và thư giãn vào thời điểm nâng cấp.

Để hỗ trợ các binary có thể tải, một tarball cho mỗi upgrade binary sẽ cần được đóng gói và cung cấp thông qua một URL canonical. Ngoài ra, một tarball bao gồm genesis binary và tất cả các upgrade binary có sẵn có thể được đóng gói và cung cấp để tất cả các binary cần thiết để sync một fullnode từ đầu có thể được tải xuống dễ dàng.

Các code và operations cụ thể của `DAEMON` (ví dụ: cấu hình cometBFT, application db, sync blocks, v.v.) đều hoạt động như mong đợi. Các chỉ thị của application binary như command-line flags và biến môi trường cũng hoạt động như mong đợi.

### Initialization (Khởi Tạo)

Lệnh `cosmovisor init <path to executable>` tạo cấu trúc thư mục cần thiết để sử dụng cosmovisor.

Nó thực hiện như sau:

* Tạo thư mục `<DAEMON_HOME>/cosmovisor` nếu chưa tồn tại
* Tạo thư mục `<DAEMON_HOME>/cosmovisor/genesis/bin` nếu chưa tồn tại
* Sao chép file thực thi được cung cấp vào `<DAEMON_HOME>/cosmovisor/genesis/bin/<DAEMON_NAME>`
* Tạo link `current`, trỏ đến thư mục `genesis`

Nó sử dụng các biến môi trường `DAEMON_HOME` và `DAEMON_NAME` cho vị trí thư mục và tên file thực thi.

Lệnh `cosmovisor init` dành riêng cho việc khởi tạo cosmovisor, và không nên nhầm lẫn với lệnh `init` của chain (ví dụ: `cosmovisor run init`).

### Detecting Upgrades (Phát Hiện Nâng Cấp)

`cosmovisor` đang kiểm tra file `$DAEMON_HOME/data/upgrade-info.json` để tìm các hướng dẫn upgrade mới. File được tạo bởi module x/upgrade trong `BeginBlocker` khi phát hiện upgrade và blockchain đạt đến upgrade height.
Heuristic sau được áp dụng để phát hiện upgrade:

* Khi khởi động, `cosmovisor` không biết nhiều về upgrade đang chạy hiện tại, ngoại trừ binary là `current/bin/`. Nó cố đọc file `current/update-info.json` để lấy thông tin về tên upgrade hiện tại.
* Nếu cả `cosmovisor/current/upgrade-info.json` lẫn `data/upgrade-info.json` không tồn tại, thì `cosmovisor` sẽ đợi file `data/upgrade-info.json` để kích hoạt upgrade.
* Nếu `cosmovisor/current/upgrade-info.json` không tồn tại nhưng `data/upgrade-info.json` tồn tại, thì `cosmovisor` giả định rằng bất cứ điều gì trong `data/upgrade-info.json` là yêu cầu upgrade hợp lệ. Trong trường hợp này `cosmovisor` ngay lập tức cố thực hiện upgrade theo thuộc tính `name` trong `data/upgrade-info.json`.
* Nếu không, `cosmovisor` đợi các thay đổi trong `upgrade-info.json`. Ngay khi một tên upgrade mới được ghi vào file, `cosmovisor` sẽ kích hoạt cơ chế upgrade.

Khi cơ chế upgrade được kích hoạt, `cosmovisor` sẽ:

1. Nếu `DAEMON_ALLOW_DOWNLOAD_BINARIES` được bật, bắt đầu bằng cách tự động tải binary mới vào `cosmovisor/<n>/bin` (trong đó `<n>` là thuộc tính `upgrade-info.json:name`);
2. Cập nhật symbolic link `current` để trỏ đến thư mục mới và lưu `data/upgrade-info.json` vào `cosmovisor/current/upgrade-info.json`.

### Adding Upgrade Binary (Thêm Binary Nâng Cấp)

`cosmovisor` có lệnh `add-upgrade` cho phép dễ dàng liên kết binary với một upgrade. Nó tạo một thư mục mới trong `cosmovisor/upgrades/<n>` và sao chép file thực thi được cung cấp vào `cosmovisor/upgrades/<n>/bin/<DAEMON_NAME>`.

Sử dụng flag `--upgrade-height` cho phép chỉ định ở height nào binary sẽ được chuyển đổi, mà không cần thông qua governance proposal.
Điều này hỗ trợ các upgrade khẩn cấp có điều phối nơi binary phải được chuyển đổi ở một height cụ thể, nhưng không có thời gian để thực hiện governance proposal.

:::warning
`--upgrade-height` tạo file `upgrade-info.json`. Điều này có nghĩa là nếu một chain upgrade qua governance proposal được thực thi trước height được chỉ định với `--upgrade-height`, governance proposal sẽ ghi đè lên plan `upgrade-info.json` được tạo bởi `add-upgrade --upgrade-height <height>`.
Hãy xem xét điều này khi sử dụng `--upgrade-height`.
:::

### Auto-Download (Tải Tự Động)

Nhìn chung, `cosmovisor` yêu cầu quản trị viên hệ thống đặt tất cả các binary liên quan lên đĩa trước khi upgrade xảy ra. Tuy nhiên, đối với những người không cần kiểm soát như vậy và muốn thiết lập tự động (có thể họ đang sync một non-validating fullnode và muốn ít bảo trì), có một tùy chọn khác.

**LƯU Ý: chúng tôi không khuyến nghị sử dụng auto-download** vì nó không xác minh trước nếu binary có sẵn. Nếu có bất kỳ vấn đề gì với việc tải binary, cosmovisor sẽ dừng lại và sẽ không khởi động lại ứng dụng (có thể dẫn đến chain halt).

Nếu `DAEMON_ALLOW_DOWNLOAD_BINARIES` được đặt thành `true`, và không tìm thấy binary local khi upgrade được kích hoạt, `cosmovisor` sẽ cố tải và cài đặt binary dựa trên các hướng dẫn trong thuộc tính `info` trong file `data/upgrade-info.json`. File được xây dựng bởi module x/upgrade và chứa dữ liệu từ đối tượng `Plan` của upgrade. `Plan` có trường info được mong đợi có một trong hai định dạng hợp lệ sau để chỉ định một lần tải:

1. Lưu trữ bản đồ os/architecture -> binary URI trong trường info của upgrade plan dưới dạng JSON dưới khóa `"binaries"`. Ví dụ:

    ```json
    {
      "binaries": {
        "linux/amd64":"https://example.com/gaia.zip?checksum=sha256:aec070645fe53ee3b3763059376134f058cc337247c978add178b6ccdfb0019f"
      }
    }
    ```

    Bạn có thể bao gồm nhiều binary cùng một lúc để đảm bảo nhiều hơn một môi trường sẽ nhận được binary đúng:

    ```json
    {
      "binaries": {
        "linux/amd64":"https://example.com/gaia.zip?checksum=sha256:aec070645fe53ee3b3763059376134f058cc337247c978add178b6ccdfb0019f",
        "linux/arm64":"https://example.com/gaia.zip?checksum=sha256:aec070645fe53ee3b3763059376134f058cc337247c978add178b6ccdfb0019f",
        "darwin/amd64":"https://example.com/gaia.zip?checksum=sha256:aec070645fe53ee3b3763059376134f058cc337247c978add178b6ccdfb0019f"
        }
    }
    ```

    Khi gửi cái này như một proposal, hãy đảm bảo không có khoảng trắng. Một ví dụ lệnh sử dụng `gaiad` có thể trông như sau:

    ```shell
    > gaiad tx upgrade software-upgrade Vega \
    --title Vega \
    --deposit 100uatom \
    --upgrade-height 7368420 \
    --upgrade-info '{"binaries":{"linux/amd64":"https://github.com/cosmos/gaia/releases/download/v6.0.0-rc1/gaiad-v6.0.0-rc1-linux-amd64","linux/arm64":"https://github.com/cosmos/gaia/releases/download/v6.0.0-rc1/gaiad-v6.0.0-rc1-linux-arm64","darwin/amd64":"https://github.com/cosmos/gaia/releases/download/v6.0.0-rc1/gaiad-v6.0.0-rc1-darwin-amd64"}}' \
    --summary "upgrade to Vega" \
    --gas 400000 \
    --from user \
    --chain-id test \
    --home test/val2 \
    --node tcp://localhost:36657 \
    --yes
    ```

2. Lưu trữ một link đến file chứa tất cả thông tin theo định dạng trên (ví dụ: nếu bạn muốn chỉ định nhiều binary, thông tin changelog, v.v. mà không làm đầy blockchain). Ví dụ:

    ```text
    https://example.com/testnet-1001-info.json?checksum=sha256:deaaa99fda9407c4dbe1d04bd49bab0cc3c1dd76fa392cd55a9425be074af01e
    ```

Khi `cosmovisor` được kích hoạt để tải binary mới, `cosmovisor` sẽ phân tích trường `"binaries"`, tải binary mới với [go-getter](https://github.com/hashicorp/go-getter), và giải nén binary mới trong thư mục `upgrades/<n>` để nó có thể chạy như thể được cài đặt thủ công.

Lưu ý rằng để cơ chế này cung cấp đảm bảo bảo mật mạnh, tất cả URL nên bao gồm checksum SHA 256/512. Điều này đảm bảo không có binary giả nào chạy, ngay cả khi ai đó hack máy chủ hoặc chiếm đoạt DNS. `go-getter` sẽ luôn đảm bảo file được tải khớp với checksum nếu được cung cấp. `go-getter` cũng sẽ xử lý giải nén các archive vào thư mục (trong trường hợp này link tải xuống nên trỏ đến file `zip` của tất cả dữ liệu trong thư mục `bin`).

Để tạo đúng sha256 checksum trên linux, bạn có thể sử dụng tiện ích `sha256sum`. Ví dụ:

```shell
sha256sum ./testdata/repo/zip_directory/autod.zip
```

Kết quả sẽ trông như sau: `29139e1381b8177aec909fab9a75d11381cab5adf7d3af0c05ff1c9c117743a7`.

Bạn cũng có thể sử dụng `sha512sum` nếu bạn muốn sử dụng hash dài hơn, hoặc `md5sum` nếu bạn muốn sử dụng hash bị phá vỡ. Bất kỳ cái nào bạn chọn, hãy đảm bảo đặt thuật toán hash đúng trong đối số checksum cho URL.

### Preparing for an Upgrade (Chuẩn Bị Cho Việc Nâng Cấp)

Để chuẩn bị cho một upgrade, sử dụng lệnh `prepare-upgrade`:

```shell
cosmovisor prepare-upgrade
```

Lệnh này thực hiện các hành động sau:

1. Lấy thông tin upgrade trực tiếp từ blockchain về upgrade được lên lịch tiếp theo.
2. Tải binary mới được chỉ định trong upgrade plan.
3. Xác minh checksum của binary (nếu được yêu cầu bởi cấu hình).
4. Đặt binary mới vào thư mục thích hợp để Cosmovisor sử dụng trong quá trình upgrade.

Lệnh `prepare-upgrade` cung cấp logging chi tiết trong suốt quá trình, bao gồm:

* Tên và height của upgrade sắp tới
* URL mà binary mới đang được tải
* Xác nhận tải xuống và xác minh thành công
* Đường dẫn nơi binary mới đã được đặt

Ví dụ output:

```bash
INFO Preparing for upgrade name=v1.0.0 height=1000000
INFO Downloading upgrade binary url=https://example.com/binary/v1.0.0?checksum=sha256:339911508de5e20b573ce902c500ee670589073485216bee8b045e853f24bce8
INFO Upgrade preparation complete name=v1.0.0 height=1000000
```

*Lưu ý: Cách hiện tại tải thủ công và đặt binary vào đúng vị trí vẫn hoạt động.*

## Ví dụ: Nâng cấp SimApp

Các hướng dẫn sau cung cấp minh chứng về `cosmovisor` sử dụng ứng dụng simulation (`simapp`) được cung cấp cùng với source code của Cosmos SDK. Các lệnh sau được chạy trong repository `cosmos-sdk`.

### Chain Setup (Thiết Lập Chain)

Hãy tạo một chain mới sử dụng phiên bản `v0.47.4` của simapp (ứng dụng demo Cosmos SDK):

```shell
git checkout v0.47.4
make build
```

Xóa sạch `~/.simapp` (đừng bao giờ làm điều này trong môi trường production):

```shell
./build/simd tendermint unsafe-reset-all
```

Thiết lập cấu hình app:

```shell
./build/simd config chain-id test
./build/simd config keyring-backend test
./build/simd config broadcast-mode sync
```

Khởi tạo node và ghi đè bất kỳ genesis file trước đó nào (đừng bao giờ làm điều này trong môi trường production):

```shell
./build/simd init test --chain-id test --overwrite
```

Để minh họa, sửa đổi `voting_period` trong `genesis.json` xuống thời gian rút ngắn là 20 giây (`20s`):

```shell
cat <<< $(jq '.app_state.gov.params.voting_period = "20s"' $HOME/.simapp/config/genesis.json) > $HOME/.simapp/config/genesis.json
```

Tạo validator và thiết lập genesis transaction:

```shell
./build/simd keys add validator
./build/simd genesis add-genesis-account validator 1000000000stake --keyring-backend test
./build/simd genesis gentx validator 1000000stake --chain-id test
./build/simd genesis collect-gentxs
```

#### Prepare Cosmovisor and Start the Chain (Chuẩn Bị Cosmovisor Và Khởi Động Chain)

Đặt các biến môi trường bắt buộc:

```shell
export DAEMON_NAME=simd
export DAEMON_HOME=$HOME/.simapp
```

Đặt biến môi trường tùy chọn để kích hoạt tự động khởi động lại ứng dụng:

```shell
export DAEMON_RESTART_AFTER_UPGRADE=true
```

Khởi tạo cosmovisor với binary hiện tại:

```shell
cosmovisor init ./build/simd
```

Bây giờ bạn có thể chạy cosmovisor với simapp v0.47.4:

```shell
cosmovisor run start
```

### Update App (Cập Nhật Ứng Dụng)

Cập nhật ứng dụng lên phiên bản mới nhất (ví dụ: v0.50.0).

:::note

Các migration plan được định nghĩa bằng module `x/upgrade` và được mô tả trong [In-Place Store Migrations](https://github.com/cosmos/cosmos-sdk/blob/main/docs/learn/advanced/15-upgrade.md). Các migration có thể thực hiện bất kỳ thay đổi state xác định nào.

Migration plan để nâng cấp simapp từ v0.47 lên v0.50 được định nghĩa trong `simapp/upgrade.go`.

:::

Build binary `simd` phiên bản mới:

```shell
make build
```

Thêm binary `simd` mới và tên upgrade:

:::warning

Tên migration phải khớp với tên được định nghĩa trong migration plan.

:::

```shell
cosmovisor add-upgrade v047-to-v050 ./build/simd
```

Mở cửa sổ terminal mới và gửi upgrade proposal cùng với deposit và vote (các lệnh này phải được chạy trong vòng 20 giây của nhau):

```shell
./build/simd tx upgrade software-upgrade v047-to-v050 --title upgrade --summary upgrade --upgrade-height 200 --upgrade-info "{}" --no-validate --from validator --yes
./build/simd tx gov deposit 1 10000000stake --from validator --yes
./build/simd tx gov vote 1 yes --from validator --yes
```

Upgrade sẽ xảy ra tự động ở height 200. Lưu ý: bạn có thể cần thay đổi upgrade height trong đoạn code trên nếu bài test của bạn mất nhiều thời gian hơn.
