---
sidebar_position: 1
---

# Cấu Trúc Thư Mục Được Đề Xuất

:::note Tóm tắt
Tài liệu này trình bày cấu trúc được đề xuất của các Cosmos SDK module. Những ý tưởng này nhằm mục đích áp dụng như các gợi ý. Nhà phát triển ứng dụng được khuyến khích cải tiến và đóng góp vào cấu trúc module và thiết kế phát triển.
:::

## Cấu Trúc

Một Cosmos SDK module điển hình có thể được cấu trúc như sau:

```shell
proto
└── {project_name}
    └── {module_name}
        └── {proto_version}
            ├── {module_name}.proto
            ├── event.proto
            ├── genesis.proto
            ├── query.proto
            └── tx.proto
```

* `{module_name}.proto`: Các định nghĩa loại message chung của module.
* `event.proto`: Các định nghĩa loại message liên quan đến sự kiện của module.
* `genesis.proto`: Các định nghĩa loại message liên quan đến trạng thái genesis của module.
* `query.proto`: Query service của module và các định nghĩa loại message liên quan.
* `tx.proto`: Msg service của module và các định nghĩa loại message liên quan.

```shell
x/{module_name}
├── client
│   ├── cli
│   │   ├── query.go
│   │   └── tx.go
│   └── testutil
│       ├── cli_test.go
│       └── suite.go
├── exported
│   └── exported.go
├── keeper
│   ├── genesis.go
│   ├── grpc_query.go
│   ├── hooks.go
│   ├── invariants.go
│   ├── keeper.go
│   ├── keys.go
│   ├── msg_server.go
│   └── querier.go
├── module
│   └── module.go
│   └── abci.go
│   └── autocli.go
├── simulation
│   ├── decoder.go
│   ├── genesis.go
│   ├── operations.go
│   └── params.go
├── {module_name}.pb.go
├── codec.go
├── errors.go
├── events.go
├── events.pb.go
├── expected_keepers.go
├── genesis.go
├── genesis.pb.go
├── keys.go
├── msgs.go
├── params.go
├── query.pb.go
├── tx.pb.go
└── README.md
```

* `client/`: Triển khai chức năng CLI client của module và bộ kiểm thử CLI của module.
* `exported/`: Các kiểu được export của module - thường là các kiểu interface. Nếu một module phụ thuộc vào keeper từ module khác, nó dự kiến sẽ nhận keeper dưới dạng các hợp đồng interface thông qua file `expected_keepers.go` (xem bên dưới) để tránh phụ thuộc trực tiếp vào module triển khai keeper. Tuy nhiên, các hợp đồng interface này có thể định nghĩa các phương thức hoạt động trên và/hoặc trả về các kiểu đặc thù cho module đang triển khai keeper và đây là nơi `exported/` phát huy tác dụng. Các kiểu interface được định nghĩa trong `exported/` sử dụng các kiểu canonical, cho phép module nhận keeper dưới dạng hợp đồng interface thông qua file `expected_keepers.go`. Mẫu này cho phép code duy trì tính DRY và cũng giảm thiểu sự hỗn loạn chu kỳ import.
* `keeper/`: Triển khai `Keeper` và `MsgServer` của module.
* `module/`: Triển khai `AppModule` và `AppModuleBasic` của module.
    * `abci.go`: Các triển khai `BeginBlocker` và `EndBlocker` của module (file này chỉ cần thiết nếu `BeginBlocker` và/hoặc `EndBlocker` cần được định nghĩa).
    * `autocli.go`: Các tùy chọn [autocli](https://docs.cosmos.network/main/core/autocli) của module.
* `simulation/`: Gói [simulation](./14-simulator.md) của module định nghĩa các hàm được sử dụng bởi ứng dụng blockchain simulator (`simapp`).
* `README.md`: Các tài liệu đặc tả của module phác thảo các khái niệm quan trọng, cấu trúc lưu trữ trạng thái, và các định nghĩa loại message và sự kiện.
* Thư mục gốc bao gồm các định nghĩa kiểu cho các message, sự kiện và trạng thái genesis, bao gồm các định nghĩa kiểu được tạo bởi Protocol Buffers.
    * `codec.go`: Các phương thức đăng ký registry của module cho các kiểu interface.
    * `errors.go`: Các lỗi sentinel của module.
    * `events.go`: Các loại sự kiện và constructor của module.
    * `expected_keepers.go`: Các interface [expected keeper](./06-keeper.md#type-definition) của module.
    * `genesis.go`: Các phương thức trạng thái genesis và hàm helper của module.
    * `keys.go`: Các khóa store và các hàm helper liên quan của module.
    * `msgs.go`: Các định nghĩa loại message và các phương thức liên quan của module.
    * `params.go`: Các định nghĩa loại tham số và các phương thức liên quan của module.
    * `*.pb.go`: Các định nghĩa kiểu của module được tạo bởi Protocol Buffers (như được định nghĩa trong các file `*.proto` tương ứng ở trên).
