---
sidebar_position: 1
---

# Architecture Decision Records (ADR)

Đây là nơi ghi chép tất cả các quyết định kiến trúc cấp cao trong Cosmos-SDK.

Architectural Decision (**AD**) là lựa chọn thiết kế phần mềm giải quyết yêu cầu chức năng hoặc phi chức năng có ý nghĩa về mặt kiến trúc.
Architecturally Significant Requirement (**ASR**) là yêu cầu có ảnh hưởng đo lường được đến kiến trúc và chất lượng của hệ thống phần mềm.
Architectural Decision Record (**ADR**) ghi lại một AD đơn lẻ, như thường được thực hiện khi viết ghi chú cá nhân hoặc biên bản cuộc họp; tập hợp các ADR được tạo và duy trì trong một dự án tạo thành nhật ký quyết định của nó. Tất cả những điều này nằm trong chủ đề Architectural Knowledge Management (AKM).

Bạn có thể đọc thêm về khái niệm ADR trong [bài viết blog](https://product.reverb.com/documenting-architecture-decisions-the-reverb-way-a3563bb24bd0#.78xhdix6t) này.

## Rationale

ADR được dự định là cơ chế chính để đề xuất thiết kế tính năng mới và quy trình mới, thu thập ý kiến cộng đồng về một vấn đề, và ghi chép các quyết định thiết kế.
Một ADR nên cung cấp:

* Bối cảnh về các mục tiêu liên quan và trạng thái hiện tại
* Các thay đổi đề xuất để đạt được mục tiêu
* Tóm tắt ưu và nhược điểm
* Tài liệu tham khảo
* Changelog

Lưu ý sự khác biệt giữa ADR và spec. ADR cung cấp bối cảnh, trực giác, lập luận và biện minh cho thay đổi trong kiến trúc, hoặc cho kiến trúc của điều gì đó mới. Spec là bản tóm tắt nén và tối ưu hóa hơn nhiều về mọi thứ như hiện tại.

Nếu các quyết định đã ghi chép hóa ra thiếu sót, hãy triệu tập thảo luận, ghi chép các quyết định mới ở đây, và sau đó sửa đổi mã để phù hợp.

## Tạo ADR mới

Đọc về [PROCESS](./PROCESS.md).

### Sử dụng từ khóa RFC 2119

Khi viết ADR, hãy tuân theo các best practice tương tự khi viết RFC. Khi viết RFC, các từ khóa được sử dụng để biểu thị các yêu cầu trong specification. Các từ này thường được viết hoa: "MUST," "MUST NOT," "REQUIRED," "SHALL," "SHALL NOT," "SHOULD," "SHOULD NOT," "RECOMMENDED," "MAY," và "OPTIONAL." Chúng được hiểu như mô tả trong [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Mục lục ADR

### Accepted

* [ADR 002: SDK Documentation Structure](./adr-002-docs-structure.md)
* [ADR 004: Split Denomination Keys](./adr-004-split-denomination-keys.md)
* [ADR 006: Secret Store Replacement](./adr-006-secret-store-replacement.md)
* [ADR 009: Evidence Module](./adr-009-evidence-module.md)
* [ADR 010: Modular AnteHandler](./adr-010-modular-antehandler.md)
* [ADR 019: Protocol Buffer State Encoding](./adr-019-protobuf-state-encoding.md)
* [ADR 020: Protocol Buffer Transaction Encoding](./adr-020-protobuf-transaction-encoding.md)
* [ADR 021: Protocol Buffer Query Encoding](./adr-021-protobuf-query-encoding.md)
* [ADR 023: Protocol Buffer Naming and Versioning](./adr-023-protobuf-naming.md)
* [ADR 029: Fee Grant Module](./adr-029-fee-grant-module.md)
* [ADR 030: Message Authorization Module](./adr-030-authz-module.md)
* [ADR 031: Protobuf Msg Services](./adr-031-msg-service.md)
* [ADR 055: ORM](./adr-055-orm.md)
* [ADR 058: Auto-Generated CLI](./adr-058-auto-generated-cli.md)
* [ADR 060: ABCI 1.0 (Phase I)](adr-060-abci-1.0.md)
* [ADR 061: Liquid Staking](./adr-061-liquid-staking.md)

### Proposed

* [ADR 003: Dynamic Capability Store](./adr-003-dynamic-capability-store.md)
* [ADR 011: Generalize Genesis Accounts](./adr-011-generalize-genesis-accounts.md)
* [ADR 012: State Accessors](./adr-012-state-accessors.md)
* [ADR 013: Metrics](./adr-013-metrics.md)
* [ADR 016: Validator Consensus Key Rotation](./adr-016-validator-consensus-key-rotation.md)
* [ADR 017: Historical Header Module](./adr-017-historical-header-module.md)
* [ADR 018: Extendable Voting Periods](./adr-018-extendable-voting-period.md)
* [ADR 022: Custom baseapp panic handling](./adr-022-custom-panic-handling.md)
* [ADR 024: Coin Metadata](./adr-024-coin-metadata.md)
* [ADR 027: Deterministic Protobuf Serialization](./adr-027-deterministic-protobuf-serialization.md)
* [ADR 028: Public Key Addresses](./adr-028-public-key-addresses.md)
* [ADR 032: Typed Events](./adr-032-typed-events.md)
* [ADR 033: Inter-module RPC](./adr-033-protobuf-inter-module-comm.md)
* [ADR 035: Rosetta API Support](./adr-035-rosetta-api-support.md)
* [ADR 037: Governance Split Votes](./adr-037-gov-split-vote.md)
* [ADR 038: State Listening](./adr-038-state-listening.md)
* [ADR 039: Epoched Staking](./adr-039-epoched-staking.md)
* [ADR 040: Storage and SMT State Commitments](./adr-040-storage-and-smt-state-commitments.md)
* [ADR 046: Module Params](./adr-046-module-params.md)
* [ADR 054: Semver Compatible SDK Modules](./adr-054-semver-compatible-modules.md)
* [ADR 057: App Wiring](./adr-057-app-wiring.md)
* [ADR 059: Test Scopes](./adr-059-test-scopes.md)
* [ADR 062: Collections State Layer](./adr-062-collections-state-layer.md)
* [ADR 063: Core Module API](./adr-063-core-module-api.md)
* [ADR 065: Store V2](./adr-065-store-v2.md)
* [ADR 076: Transaction Malleability Risk Review and Recommendations](./adr-076-tx-malleability.md)

### Draft

* [ADR 044: Guidelines for Updating Protobuf Definitions](./adr-044-protobuf-updates-guidelines.md)
* [ADR 047: Extend Upgrade Plan](./adr-047-extend-upgrade-plan.md)
* [ADR 053: Go Module Refactoring](./adr-053-go-module-refactoring.md)
* [ADR 068: Preblock](./adr-068-preblock.md)
