---
sidebar_position: 1
---

# Thiết Lập Keyring

:::note Tóm tắt
Tài liệu này mô tả cách cấu hình và sử dụng keyring cùng các backend khác nhau của nó cho một [**ứng dụng**](../../learn/beginner/00-app-anatomy.md).
:::

Keyring lưu giữ các cặp khóa private/public được dùng để tương tác với một node. Ví dụ, khóa validator cần được thiết lập trước khi chạy blockchain node, để các block có thể được ký đúng cách. Khóa private có thể được lưu trữ ở các vị trí khác nhau, gọi là "backend", chẳng hạn như một file hoặc bộ lưu trữ khóa riêng của hệ điều hành.

## Các Backend Có Sẵn Cho Keyring

Bắt đầu từ phiên bản v0.38.0, Cosmos SDK đi kèm với một triển khai keyring mới cung cấp một tập hợp lệnh để quản lý các khóa mật mã theo cách bảo mật. Keyring mới hỗ trợ nhiều backend lưu trữ, một số trong đó có thể không khả dụng trên tất cả hệ điều hành.

### Backend `os`

Backend `os` dựa vào các giá trị mặc định dành riêng cho hệ điều hành để xử lý lưu trữ khóa một cách bảo mật. Thông thường, hệ thống con thông tin xác thực của hệ điều hành xử lý các yêu cầu mật khẩu, lưu trữ khóa private và phiên người dùng theo chính sách mật khẩu của người dùng. Dưới đây là danh sách các hệ điều hành phổ biến nhất và trình quản lý mật khẩu tương ứng của chúng:

* macOS: [Keychain](https://support.apple.com/en-gb/guide/keychain-access/welcome/mac)
* Windows: [Credentials Management API](https://docs.microsoft.com/en-us/windows/win32/secauthn/credentials-management)
* GNU/Linux:
    * [libsecret](https://gitlab.gnome.org/GNOME/libsecret)
    * [kwallet](https://api.kde.org/frameworks/kwallet/html/index.html)
    * [keyctl](https://www.kernel.org/doc/html/latest/security/keys/core.html)

Các bản phân phối GNU/Linux sử dụng GNOME làm môi trường desktop mặc định thường đi kèm với [Seahorse](https://wiki.gnome.org/Apps/Seahorse). Người dùng các bản phân phối dựa trên KDE thường được cung cấp [KDE Wallet Manager](https://userbase.kde.org/KDE_Wallet_Manager). Trong khi cái trước thực chất là một frontend tiện lợi cho `libsecret`, thì cái sau là một client `kwallet`. `keyctl` là một backend bảo mật tận dụng hệ thống quản lý khóa bảo mật của Linux kernel để lưu trữ các khóa mật mã an toàn trong bộ nhớ.

`os` là tùy chọn mặc định vì các trình quản lý thông tin xác thực mặc định của hệ điều hành được thiết kế để đáp ứng nhu cầu phổ biến nhất của người dùng và mang lại trải nghiệm thoải mái mà không ảnh hưởng đến bảo mật.

Các backend được khuyến nghị cho môi trường headless là `file` và `pass`.

### Backend `file`

Backend `file` gần giống hơn với triển khai keybase được dùng trước v0.38.1. Nó lưu trữ keyring được mã hóa trong thư mục cấu hình của ứng dụng. Keyring này sẽ yêu cầu mật khẩu mỗi lần truy cập, điều này có thể xảy ra nhiều lần trong một lệnh dẫn đến các yêu cầu mật khẩu lặp lại. Nếu sử dụng bash script để thực thi lệnh với tùy chọn `file`, bạn có thể muốn sử dụng định dạng sau cho nhiều yêu cầu:

```shell
# giả sử KEYPASSWD được đặt trong môi trường
$ gaiacli config keyring-backend file                             # sử dụng file backend
$ (echo $KEYPASSWD; echo $KEYPASSWD) | gaiacli keys add me        # nhiều yêu cầu
$ echo $KEYPASSWD | gaiacli keys show me                          # một yêu cầu
```

:::tip
Lần đầu tiên bạn thêm khóa vào keyring rỗng, bạn sẽ được yêu cầu nhập mật khẩu hai lần.
:::

### Backend `pass`

Backend `pass` sử dụng tiện ích [pass](https://www.passwordstore.org/) để quản lý mã hóa trên đĩa cho dữ liệu nhạy cảm và metadata của các khóa. Các khóa được lưu trữ bên trong các file được mã hóa bằng `gpg` trong các thư mục dành riêng cho ứng dụng. `pass` có sẵn cho các hệ điều hành UNIX phổ biến nhất cũng như các bản phân phối GNU/Linux. Vui lòng tham khảo trang hướng dẫn của nó để biết thông tin về cách tải xuống và cài đặt.

:::tip
**pass** sử dụng [GnuPG](https://gnupg.org/) để mã hóa. `gpg` tự động khởi chạy daemon `gpg-agent` khi thực thi, daemon này xử lý việc cache thông tin xác thực GnuPG. Vui lòng tham khảo trang hướng dẫn của `gpg-agent` để biết thêm thông tin về cách cấu hình các tham số cache như TTL thông tin xác thực và hết hạn passphrase.
:::

Password store phải được thiết lập trước khi sử dụng lần đầu:

```shell
pass init <GPG_KEY_ID>
```

Thay `<GPG_KEY_ID>` bằng ID khóa GPG của bạn. Bạn có thể sử dụng khóa GPG cá nhân hoặc một khóa thay thế mà bạn muốn dùng riêng để mã hóa password store.

### Backend `kwallet`

Backend `kwallet` sử dụng `KDE Wallet Manager`, được cài đặt mặc định trên các bản phân phối GNU/Linux đi kèm KDE làm môi trường desktop mặc định. Vui lòng tham khảo [tài liệu KWallet API](https://api.kde.org/frameworks/kwallet/html/index.html) để biết thêm thông tin.

### Backend `keyctl`

*Kernel Key Retention Service* là một cơ sở bảo mật đã được thêm vào Linux kernel tương đối gần đây. Nó cho phép lưu trữ dữ liệu mật mã nhạy cảm như mật khẩu, khóa private, token xác thực, v.v. một cách bảo mật trong bộ nhớ.

Backend `keyctl` chỉ khả dụng trên nền tảng Linux.

### Backend `test`

Backend `test` là biến thể không có mật khẩu của backend `file`. Các khóa được lưu trữ không mã hóa trên đĩa.

**Chỉ cung cấp cho mục đích kiểm thử. Backend `test` không được khuyến nghị sử dụng trong môi trường production**.

### Backend `memory`

Backend `memory` lưu trữ các khóa trong bộ nhớ. Các khóa bị xóa ngay lập tức sau khi chương trình thoát.

**Chỉ cung cấp cho mục đích kiểm thử. Backend `memory` không được khuyến nghị sử dụng trong môi trường production**.

### Đặt Backend Sử Dụng Biến Môi Trường

Bạn có thể đặt keyring-backend bằng biến môi trường: `BINNAME_KEYRING_BACKEND`. Ví dụ, nếu tên binary của bạn là `gaia-v5` thì đặt: `export GAIA_V5_KEYRING_BACKEND=pass`

## Thêm Khóa Vào Keyring

:::warning
Hãy đảm bảo bạn có thể build binary của riêng mình, và thay `simd` bằng tên binary của bạn trong các đoạn code.
:::

Các ứng dụng được phát triển bằng Cosmos SDK đi kèm với lệnh con `keys`. Để phục vụ mục đích của tutorial này, chúng ta đang chạy CLI `simd`, là một ứng dụng được xây dựng bằng Cosmos SDK để kiểm thử và học tập. Để biết thêm thông tin, xem [`simapp`](https://github.com/cosmos/cosmos-sdk/tree/main/simapp).

Bạn có thể dùng `simd keys` để được trợ giúp về lệnh keys và `simd keys [command] --help` để biết thêm thông tin về một lệnh con cụ thể.

Để tạo khóa mới trong keyring, chạy lệnh con `add` với đối số `<key_name>`. Để phục vụ mục đích của tutorial này, chúng ta sẽ chỉ sử dụng backend `test` và đặt tên khóa mới là `my_validator`. Khóa này sẽ được sử dụng trong phần tiếp theo.

```bash
$ simd keys add my_validator --keyring-backend test

# Đặt địa chỉ được tạo ra vào một biến để sử dụng sau.
MY_VALIDATOR_ADDRESS=$(simd keys show my_validator -a --keyring-backend test)
```

Lệnh này tạo ra một cụm từ gợi nhớ 24 từ mới, lưu nó vào backend liên quan và xuất thông tin về cặp khóa. Nếu cặp khóa này sẽ được dùng để lưu giữ token có giá trị, hãy nhớ ghi lại cụm từ gợi nhớ ở một nơi an toàn!

Theo mặc định, keyring tạo ra một cặp khóa `secp256k1`. Keyring cũng hỗ trợ khóa `ed25519`, có thể được tạo bằng cách truyền flag `--algo ed25519`. Tất nhiên một keyring có thể lưu giữ cả hai loại khóa đồng thời, và module `x/auth` của Cosmos SDK hỗ trợ tự nhiên hai thuật toán khóa công khai này.
