---
sidebar_position: 0
---

# Packages (Gói)

Cosmos SDK là một tập hợp các module Go. Phần này cung cấp tài liệu về các gói khác nhau có thể được sử dụng khi phát triển một Cosmos SDK chain.
Nó liệt kê tất cả các module Go độc lập là một phần của Cosmos SDK.

:::tip
Để biết thêm thông tin về các SDK module, xem phần [SDK Modules](https://docs.cosmos.network/main/modules).
Để biết thêm thông tin về SDK tooling, xem phần [Tooling](https://docs.cosmos.network/main/tooling).
:::

## Core (Lõi)

* [Core](https://pkg.go.dev/cosmossdk.io/core) - Thư viện Core định nghĩa các interface SDK ([ADR-063](https://docs.cosmos.network/main/architecture/adr-063-core-module-api))
* [API](https://pkg.go.dev/cosmossdk.io/api) - Thư viện API chứa Pulsar API SDK được tạo ra
* [Store](https://pkg.go.dev/cosmossdk.io/store) - Triển khai Cosmos SDK store

## State Management (Quản Lý Trạng Thái)

* [Collections](./02-collections.md) - Thư viện quản lý trạng thái

## Automation (Tự Động Hóa)

* [Depinject](./01-depinject.md) - Framework dependency injection
* [Client/v2](https://pkg.go.dev/cosmossdk.io/client/v2) - Thư viện hỗ trợ [AutoCLI](https://docs.cosmos.network/main/core/autocli)

## Utilities (Tiện Ích)

* [Log](https://pkg.go.dev/cosmossdk.io/log) - Thư viện logging
* [Errors](https://pkg.go.dev/cosmossdk.io/errors) - Thư viện xử lý lỗi
* [Math](https://pkg.go.dev/cosmossdk.io/math) - Thư viện toán học cho các phép tính số học SDK

## Example (Ví Dụ)

* [SimApp](https://pkg.go.dev/cosmossdk.io/simapp) - SimApp là **ví dụ** Cosmos SDK chain mẫu. Package này không nên được import trong ứng dụng của bạn.
