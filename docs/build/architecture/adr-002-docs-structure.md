# ADR 002: Cấu Trúc Tài Liệu SDK

## Bối Cảnh

Cần có một cấu trúc có khả năng mở rộng cho tài liệu Cosmos SDK. Tài liệu hiện tại bao gồm nhiều tài liệu không liên quan đến Cosmos SDK, khó bảo trì và khó theo dõi với người dùng.

Lý tưởng nhất, chúng ta sẽ có:

* Tất cả tài liệu liên quan đến framework hoặc công cụ dev nằm trong các repo github tương ứng (sdk repo chứa sdk docs, hub repo chứa hub docs, lotion repo chứa lotion docs, v.v.)
* Tất cả tài liệu khác (faqs, whitepaper, tài liệu high-level về Cosmos) sẽ nằm trên website.

## Quyết Định

Tái cấu trúc thư mục `/docs` của repo github Cosmos SDK như sau:

```text
docs/
├── README
├── intro/
├── concepts/
│   ├── baseapp
│   ├── types
│   ├── store
│   ├── server
│   ├── modules/
│   │   ├── keeper
│   │   ├── handler
│   │   ├── cli
│   ├── gas
│   └── commands
├── clients/
│   ├── lite/
│   ├── service-providers
├── modules/
├── spec/
├── translations/
└── architecture/
```

Các file trong mỗi thư mục con không quan trọng và có thể thay đổi. Điều quan trọng là phân mục:

* `README`: Trang đích của tài liệu.
* `intro`: Tài liệu giới thiệu. Mục tiêu là có một giải thích ngắn về Cosmos SDK rồi dẫn người dùng đến tài nguyên họ cần. [Hướng dẫn Cosmos SDK](https://github.com/cosmos/sdk-application-tutorial/) sẽ được nêu bật, cũng như `godocs`.
* `concepts`: Chứa các giải thích high-level về các trừu tượng của Cosmos SDK. Không chứa triển khai code cụ thể và không cần cập nhật thường xuyên. **Đây không phải là đặc tả API của các interface**. Đặc tả API là `godoc`.
* `clients`: Chứa các spec và thông tin về các Cosmos SDK client khác nhau.
* `spec`: Chứa spec của các module và các thứ khác.
* `modules`: Chứa các link đến `godocs` và spec của các module.
* `architecture`: Chứa các tài liệu liên quan đến kiến trúc như tài liệu hiện tại.
* `translations`: Chứa các bản dịch khác nhau của tài liệu.

Sidebar tài liệu website chỉ bao gồm các phần sau:

* `README`
* `intro`
* `concepts`
* `clients`

`architecture` không cần hiển thị trên website.

## Trạng Thái

Đã Chấp Nhận

## Hậu Quả

### Tích Cực

* Tổ chức tài liệu Cosmos SDK rõ ràng hơn nhiều.
* Thư mục `/docs` giờ chỉ chứa tài liệu liên quan đến Cosmos SDK và gaia. Sau này, nó chỉ chứa tài liệu liên quan đến Cosmos SDK.
* Nhà phát triển chỉ cần cập nhật thư mục `/docs` khi họ mở PR (không cần cập nhật `/examples` chẳng hạn).
* Dễ dàng hơn cho nhà phát triển tìm những gì cần cập nhật trong tài liệu nhờ kiến trúc được cải thiện.
* Build vuepress sạch hơn cho tài liệu website.
* Sẽ giúp xây dựng một tài liệu có thể thực thi (cf https://github.com/cosmos/cosmos-sdk/issues/2611)

### Trung Lập

* Chúng ta cần chuyển một số thứ đã lỗi thời vào thư mục `/_attic`.
* Chúng ta cần tích hợp nội dung trong `docs/sdk/docs/core` vào `concepts`.
* Chúng ta cần chuyển tất cả nội dung hiện đang nằm trong `docs` và không phù hợp với cấu trúc mới (như `lotion`, tài liệu giới thiệu, whitepaper) sang repository website.
* Cập nhật `DOCS_README.md`

## Tài Liệu Tham Khảo

* https://github.com/cosmos/cosmos-sdk/issues/1460
* https://github.com/cosmos/cosmos-sdk/pull/2695
* https://github.com/cosmos/cosmos-sdk/issues/2611
