# Kiểm Thử Module Oracle

Chúng ta sẽ hướng dẫn bạn qua quá trình kiểm thử module Oracle trong ứng dụng của bạn. Module Oracle sử dụng vote extension để cung cấp dữ liệu giá hiện tại. Nếu bạn muốn xem module oracle hoàn chỉnh đang hoạt động, vui lòng xem [tại đây](https://github.com/cosmos/sdk-tutorials/blob/master/tutorials/oracle/base/x/oracle).

## Bước 1: Biên Dịch và Cài Đặt Ứng Dụng

Đầu tiên, chúng ta cần biên dịch và cài đặt ứng dụng. Hãy đảm bảo bạn đang ở trong thư mục `tutorials/oracle/base`. Chạy lệnh sau trong terminal:

```shell
make install
```

Lệnh này biên dịch ứng dụng và chuyển binary kết quả đến một vị trí trong PATH của hệ thống.

## Bước 2: Khởi Tạo Ứng Dụng

Tiếp theo, chúng ta cần khởi tạo ứng dụng. Chạy lệnh sau trong terminal:

```shell
make init
```

Lệnh này chạy script `tutorials/oracle/base/scripts/init.sh`, thiết lập cấu hình cần thiết để ứng dụng chạy. Bao gồm tạo file cấu hình `app.toml` và khởi tạo blockchain với genesis block.

## Bước 3: Khởi Động Ứng Dụng

Bây giờ, chúng ta có thể khởi động ứng dụng. Chạy lệnh sau trong terminal:

```shell
exampled start
```

Lệnh này khởi động ứng dụng của bạn, bắt đầu blockchain node và bắt đầu xử lý giao dịch.

## Bước 4: Truy Vấn Giá Oracle

Cuối cùng, chúng ta có thể truy vấn các giá hiện tại từ module Oracle. Chạy lệnh sau trong terminal:

```shell
exampled q oracle prices
```

Lệnh này truy vấn các giá hiện tại từ module Oracle. Đầu ra dự kiến cho thấy rằng vote extension đã được đưa vào block thành công và module Oracle đã có thể lấy dữ liệu giá.

## Hiểu Vote Extension Trong Oracle

Trong module Oracle, hàm `ExtendVoteHandler` chịu trách nhiệm tạo vote extension. Hàm này lấy giá hiện tại từ provider, tạo struct `OracleVoteExtension` với các giá này, sau đó marshal struct này thành bytes. Các bytes này sau đó được đặt làm vote extension.

Trong bối cảnh kiểm thử, module Oracle sử dụng một mock provider để mô phỏng hành vi của một price provider thực. Mock provider này được định nghĩa trong package `mockprovider` và được dùng để trả về các giá được định nghĩa trước cho các cặp tiền tệ cụ thể.

## Kết Luận

Trong tutorial này, chúng ta đã đi sâu vào khái niệm Oracle trong công nghệ blockchain, tập trung vào vai trò của chúng trong việc cung cấp dữ liệu bên ngoài cho mạng lưới blockchain. Chúng ta đã khám phá vote extension — một tính năng mạnh mẽ của ABCI++ — và tích hợp chúng vào một ứng dụng Cosmos SDK để tạo ra một module price oracle.

Qua các bài tập thực hành, bạn đã triển khai vote extension và kiểm tra hiệu quả của chúng trong việc cung cấp thông tin giá tài sản kịp thời và chính xác. Bạn đã có được những hiểu biết thực tế bằng cách thiết lập mock provider để kiểm thử và phân tích quá trình mở rộng phiếu bầu, xác minh vote extension, cũng như chuẩn bị và xử lý proposal.

Hãy tiếp tục thử nghiệm với các khái niệm này, tham gia cộng đồng và cập nhật các tiến bộ mới. Kiến thức bạn đã có được ở đây rất quan trọng để phát triển các ứng dụng blockchain mạnh mẽ và đáng tin cậy có thể tương tác với dữ liệu thực tế.
